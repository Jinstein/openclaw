# KakaoTalk Channel Integration - Design Document

## Overview

This document outlines the design for integrating KakaoTalk messaging platform with OpenClaw. KakaoTalk is South Korea's dominant messaging platform with over 50 million active users, making it an essential channel for services targeting Korean markets.

## Goals

1. **Primary Goal**: Enable bidirectional communication between OpenClaw agents and KakaoTalk users
2. **Secondary Goals**:
   - Support AlimTalk for informational/notification messages
   - Support FriendTalk for marketing and promotional messages
   - Enable 1:1 channel conversations
   - Provide seamless OAuth 2.0 authentication flow
   - Support message templates (default and custom)

## Architecture

### Component Overview

```
┌─────────────────────────────────────────────────────────┐
│                    OpenClaw Gateway                      │
│  ┌──────────────────────────────────────────────────┐   │
│  │         KakaoTalk Channel Plugin                 │   │
│  │  ┌────────────┐  ┌──────────────┐  ┌─────────┐  │   │
│  │  │  Channel   │  │    OAuth     │  │ Message │  │   │
│  │  │  Provider  │  │   Handler    │  │ Handler │  │   │
│  │  └────────────┘  └──────────────┘  └─────────┘  │   │
│  │  ┌────────────┐  ┌──────────────┐  ┌─────────┐  │   │
│  │  │  Template  │  │  API Client  │  │  Media  │  │   │
│  │  │  Manager   │  │              │  │ Handler │  │   │
│  │  └────────────┘  └──────────────┘  └─────────┘  │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                           │
                           │ HTTPS REST API
                           │ OAuth 2.0
                           ▼
┌─────────────────────────────────────────────────────────┐
│              Kakao Platform Services                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  KakaoTalk   │  │    Kakao     │  │  Message     │  │
│  │   Channel    │  │    OAuth     │  │  Template    │  │
│  │     API      │  │              │  │   Service    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### Core Components

#### 1. Channel Provider (`src/provider.ts`)

Implements the OpenClaw channel provider interface:
- Registers with the gateway
- Manages connection lifecycle
- Routes messages between gateway and Kakao API
- Handles session management

**Key responsibilities:**
- Initialize connection on gateway startup
- Handle reconnection/retry logic
- Normalize incoming messages to OpenClaw format
- Format outgoing messages to Kakao API format
- Manage channel configuration

#### 2. OAuth Handler (`src/oauth.ts`)

Manages OAuth 2.0 authentication flow:
- Authorization URL generation
- Token exchange (authorization code → access token)
- Token refresh (access token + refresh token management)
- Token storage and retrieval
- Scope management

**OAuth Flow:**
```
User → Authorization Request → Kakao Login
                                    ↓
                            User Approves
                                    ↓
Authorization Code ← Redirect ← Kakao
         ↓
Token Exchange → Access Token + Refresh Token
         ↓
   Store Tokens
```

**Token management:**
- Access tokens expire in ~6 hours
- Refresh tokens expire in ~2 months
- Automatic refresh before expiration
- Secure storage in `~/.openclaw/credentials/kakao/`

#### 3. Message Handler (`src/messages.ts`)

Processes incoming and outgoing messages:
- Parse Kakao webhook events
- Convert Kakao message format to OpenClaw message envelope
- Convert OpenClaw responses to Kakao API calls
- Handle message types (text, image, template, button)
- Track message delivery status

**Inbound message flow:**
```
Kakao Webhook → Parse Event → Validate → Normalize
                                              ↓
                                    Create Message Envelope
                                              ↓
                                    Enqueue to Gateway
```

**Outbound message flow:**
```
Gateway Response → Route to Channel → Format Message
                                              ↓
                                     Apply Template
                                              ↓
                                      Kakao API Call
                                              ↓
                                    Track Delivery
```

#### 4. Template Manager (`src/templates.ts`)

Manages message templates:
- Load template definitions
- Validate template parameters
- Render templates with context variables
- Cache approved templates
- Handle template approval status

**Template types:**
- Default templates (text, feed, list, location, commerce)
- Custom templates (user-defined in Kakao console)
- AlimTalk templates (pre-approved informational messages)

#### 5. API Client (`src/api-client.ts`)

Low-level HTTP client for Kakao APIs:
- REST API calls with authentication
- Request/response serialization
- Error handling and retry logic
- Rate limiting compliance
- API endpoint management

**Key endpoints:**
- `/v2/api/talk/memo/default/send` - Send to self
- `/v1/api/talk/friends/message/default/send` - Send to friends
- `/v2/user/me` - Get user info
- `/v1/api/talk/friends` - Get friend list

#### 6. Media Handler (`src/media.ts`)

Handles media uploads and downloads:
- Image upload to Kakao
- Image download from Kakao
- Media format validation
- Size limit enforcement (2MB default)
- Content-type detection

## Message Types Support

### 1. AlimTalk (알림톡)

**Use case:** Transactional and informational messages

**Characteristics:**
- Requires pre-approved templates
- Template approval takes 1-2 business days
- High deliverability (shown even if not a friend)
- Lower cost compared to SMS
- Strict content guidelines

**Implementation:**
- Template registration via Kakao Business Channel
- Template ID storage in config
- Variable substitution at send time
- Fallback to SMS/LMS if template fails

**Example template:**
```json
{
  "template_id": "template_12345",
  "variables": {
    "orderNumber": "ORD-2026-001",
    "shippingDate": "2026-02-10",
    "trackingUrl": "https://tracking.example.com/ORD-2026-001"
  }
}
```

### 2. FriendTalk (친구톡)

**Use case:** Marketing and promotional messages to channel friends

**Characteristics:**
- No template pre-approval required
- Only sent to users who added channel as friend
- More flexible content
- Supports rich media (images, buttons)
- Better for conversational tone

**Implementation:**
- Check friendship status before sending
- Support wide message and carousel message formats
- Handle button actions
- Track click-through rates

### 3. Channel 1:1 Chat

**Use case:** Real-time customer service and support

**Characteristics:**
- Bidirectional conversation
- Real-time messaging
- User-initiated or bot-initiated
- Session-based context

**Implementation:**
- Webhook for incoming messages
- Session management per user
- Context preservation across messages
- Typing indicators

## Authentication & Security

### OAuth 2.0 Flow

**Required scopes:**
- `talk_message` - Send messages
- `friends` - Access friend list (optional)
- `profile` - Read user profile (optional)

**Token storage:**
```
~/.openclaw/credentials/kakao/
├── tokens.json          # Current tokens
├── refresh_tokens.json  # Refresh tokens
└── oauth_config.json    # OAuth configuration
```

**Security measures:**
- Tokens encrypted at rest
- HTTPS-only communication
- Token rotation on security events
- Rate limiting to prevent abuse
- Webhook signature verification

### API Authentication

**Headers required:**
```
Authorization: Bearer {access_token}
Content-Type: application/json; charset=utf-8
```

**Error handling:**
- `401 Unauthorized` → Refresh token and retry
- `403 Forbidden` → Check permissions/scopes
- `429 Too Many Requests` → Exponential backoff
- `500 Server Error` → Retry with backoff

## Configuration Schema

### Gateway Configuration

```typescript
interface KakaoChannelConfig {
  // Basic settings
  enabled: boolean;
  appKey: string;           // JavaScript key from developer console
  restApiKey: string;       // REST API key
  channelId: string;        // Channel ID (format: _abc123)

  // Access control
  dmPolicy: "pairing" | "allowlist" | "open" | "disabled";
  allowFrom: string[];      // User IDs allowed to message

  // Message types
  messageTypes: {
    alimtalk: {
      enabled: boolean;
      senderKey: string;    // Business sender key
      templates: Record<string, AlimTalkTemplate>;
    };
    friendtalk: {
      enabled: boolean;
    };
  };

  // OAuth settings
  oauth: {
    redirectUri: string;    // OAuth callback URL
    scopes: string[];       // Required scopes
  };

  // API settings
  apiBaseUrl?: string;      // Default: https://kapi.kakao.com
  mediaMaxMb?: number;      // Default: 2
  retry?: RetryPolicy;

  // Multi-account support
  accounts?: Record<string, KakaoAccountConfig>;
}
```

### Template Definition

```typescript
interface AlimTalkTemplate {
  templateId: string;       // Template ID from Kakao
  templateCode: string;     // Business template code
  variables: string[];      // Required variable names
  buttons?: TemplateButton[];
  fallbackConfig?: {
    type: "SMS" | "LMS";
    message: string;
  };
}
```

## API Integration

### REST API Endpoints

**Base URL:** `https://kapi.kakao.com`

**Send message to self (memo):**
```
POST /v2/api/talk/memo/default/send
Authorization: Bearer {access_token}

{
  "template_object": {
    "object_type": "text",
    "text": "Hello from OpenClaw!",
    "link": {
      "web_url": "https://openclaw.ai"
    }
  }
}
```

**Send message to friends:**
```
POST /v1/api/talk/friends/message/default/send
Authorization: Bearer {access_token}

{
  "receiver_uuids": ["uuid1", "uuid2"],
  "template_object": {
    "object_type": "feed",
    "content": {
      "title": "New Message",
      "description": "You have a new message",
      "image_url": "https://example.com/image.jpg",
      "link": {
        "web_url": "https://example.com"
      }
    }
  }
}
```

**Get user info:**
```
GET /v2/user/me
Authorization: Bearer {access_token}
```

**Get friend list:**
```
GET /v1/api/talk/friends
Authorization: Bearer {access_token}
```

### Webhook Events

**Webhook endpoint:** `POST /webhook/kakao` (configured in gateway)

**Event types:**
- `message` - New message received
- `friend_added` - User added channel as friend
- `friend_deleted` - User removed channel
- `chat_opened` - User opened 1:1 chat
- `chat_closed` - User closed 1:1 chat

**Message event structure:**
```json
{
  "type": "message",
  "timestamp": 1707210000000,
  "user": {
    "id": "user123",
    "nickname": "John Doe"
  },
  "message": {
    "id": "msg123",
    "type": "text",
    "text": "Hello!",
    "extras": {}
  }
}
```

## Implementation Phases

### Phase 1: Core Integration (MVP)
- [ ] Basic REST API client
- [ ] OAuth 2.0 authentication flow
- [ ] Send text messages (memo API)
- [ ] Receive webhook events
- [ ] Basic error handling

### Phase 2: Advanced Messaging
- [ ] AlimTalk template support
- [ ] FriendTalk messaging
- [ ] Media uploads (images)
- [ ] Button and action support
- [ ] Message templates (default types)

### Phase 3: Channel Management
- [ ] Friend list management
- [ ] Channel profile updates
- [ ] Custom template management
- [ ] Analytics and tracking
- [ ] Bulk messaging support

### Phase 4: Enterprise Features
- [ ] Multi-account support
- [ ] Advanced rate limiting
- [ ] Message scheduling
- [ ] A/B testing support
- [ ] Comprehensive logging and monitoring

## Error Handling

### Common Errors

**Authentication errors:**
- `401`: Token expired → Refresh and retry
- `403`: Insufficient permissions → Check scopes
- Invalid credentials → Re-authenticate

**API errors:**
- `429`: Rate limit exceeded → Exponential backoff
- `500/502/503`: Server errors → Retry with backoff
- `400`: Bad request → Validate and log

**Template errors:**
- Template not found → Check template ID
- Template not approved → Wait for approval
- Invalid variables → Validate before send

### Retry Strategy

```typescript
interface RetryPolicy {
  maxAttempts: number;      // Default: 3
  initialDelayMs: number;   // Default: 1000
  maxDelayMs: number;       // Default: 30000
  backoffMultiplier: number; // Default: 2
  retryableErrors: number[]; // [429, 500, 502, 503]
}
```

## Testing Strategy

### Unit Tests
- API client methods
- OAuth flow components
- Message parsing/formatting
- Template rendering
- Error handling

### Integration Tests
- End-to-end message flow
- OAuth token refresh
- Webhook event processing
- Multi-account scenarios

### Manual Testing
- Real Kakao channel setup
- Message sending/receiving
- Template approval workflow
- Friend management

## Performance Considerations

### Rate Limiting

**Kakao API limits:**
- 100 requests/second per app
- 1000 messages/hour per sender key (AlimTalk)
- May vary based on business tier

**Implementation:**
- Token bucket algorithm
- Per-endpoint rate tracking
- Exponential backoff on 429 errors
- Request queuing for burst handling

### Caching

**What to cache:**
- Friend lists (30 min TTL)
- Approved templates (24 hr TTL)
- User profiles (1 hr TTL)
- OAuth tokens (until expiry)

**Cache invalidation:**
- Time-based expiration
- Event-based invalidation (friend_added/deleted)
- Manual cache clear command

## Monitoring & Logging

### Metrics to track

- Messages sent/received per minute
- API call latency (p50, p95, p99)
- Error rate by error type
- Token refresh frequency
- Template approval status
- Friend count changes

### Log levels

- `INFO`: Normal operations (message sent, received)
- `WARN`: Recoverable errors (rate limit, retry)
- `ERROR`: Failures (auth failed, API error)
- `DEBUG`: Detailed traces (OAuth flow, API calls)

## Security Considerations

1. **Token Security**
   - Store tokens encrypted at rest
   - Use secure channels (HTTPS only)
   - Rotate tokens on breach detection

2. **Webhook Validation**
   - Verify webhook signatures
   - Check timestamp to prevent replay attacks
   - Whitelist Kakao IP ranges

3. **Data Privacy**
   - Minimal data collection
   - User consent for friend list access
   - GDPR/CCPA compliance for user data

4. **Rate Limiting**
   - Prevent abuse of API
   - Implement per-user limits
   - Monitor for suspicious patterns

## Migration & Backwards Compatibility

### For existing users
- No breaking changes to OpenClaw core
- Plugin-based architecture allows opt-in
- Clear migration guide for configuration

### Version compatibility
- Support Node.js 22+
- Compatible with OpenClaw core API
- Semantic versioning for plugin releases

## Documentation Requirements

1. **User Documentation**
   - Quick start guide
   - Configuration reference
   - Troubleshooting guide
   - API examples

2. **Developer Documentation**
   - Architecture overview
   - API client usage
   - Extension points
   - Testing guide

3. **Operations Documentation**
   - Deployment guide
   - Monitoring setup
   - Incident response
   - Performance tuning

## Future Enhancements

### Short-term (3-6 months)
- Group chat support (when available from Kakao)
- Rich card templates
- Payment integration
- Location sharing

### Long-term (6-12 months)
- KakaoTalk Shopping integration
- Mini-app support
- AI-powered response suggestions
- Advanced analytics dashboard

## References

### Official Documentation
- [Kakao Developers](https://developers.kakao.com)
- [REST API Reference](https://developers.kakao.com/docs/latest/en/kakaotalk-message/rest-api)
- [Message Templates](https://developers.kakao.com/docs/latest/ko/message/common)
- [OAuth 2.0 Guide](https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api)

### Third-party Resources
- [Sinch Conversation API](https://developers.sinch.com/docs/conversation/channel-support/kakaotalk)
- [Sendbird Business Messaging](https://sendbird.com/products/business-messaging/kakaotalk)
- [Infobip KakaoTalk API](https://www.infobip.com/docs/kakaotalk/kakaotalk-over-api)

### Related OpenClaw Docs
- [Channel Architecture](/concepts/channels)
- [Plugin Development](/plugin)
- [OAuth Integration](/concepts/oauth)
- [Message Routing](/concepts/routing)

## Appendix: API Examples

### Example 1: Send Simple Text Message

```typescript
import { kakaoApiClient } from './api-client';

async function sendTextMessage(accessToken: string, text: string) {
  const response = await kakaoApiClient.post('/v2/api/talk/memo/default/send', {
    headers: {
      'Authorization': `Bearer ${accessToken}`,
    },
    body: {
      template_object: {
        object_type: 'text',
        text: text,
        link: {
          web_url: 'https://openclaw.ai',
        },
      },
    },
  });

  return response;
}
```

### Example 2: Send AlimTalk with Template

```typescript
async function sendAlimTalk(
  senderKey: string,
  phoneNumber: string,
  templateCode: string,
  variables: Record<string, string>
) {
  const response = await kakaoApiClient.post('/alimtalk/v1/message', {
    headers: {
      'Authorization': `Bearer ${accessToken}`,
    },
    body: {
      senderKey: senderKey,
      phoneNumber: phoneNumber,
      templateCode: templateCode,
      templateParameter: variables,
    },
  });

  return response;
}
```

### Example 3: OAuth Token Refresh

```typescript
async function refreshAccessToken(refreshToken: string): Promise<TokenResponse> {
  const response = await fetch('https://kauth.kakao.com/oauth/token', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded',
    },
    body: new URLSearchParams({
      grant_type: 'refresh_token',
      client_id: config.appKey,
      refresh_token: refreshToken,
    }),
  });

  const data = await response.json();

  return {
    accessToken: data.access_token,
    refreshToken: data.refresh_token || refreshToken, // Kakao may not return new refresh token
    expiresIn: data.expires_in,
    refreshTokenExpiresIn: data.refresh_token_expires_in,
  };
}
```

## Conclusion

This design provides a comprehensive roadmap for integrating KakaoTalk with OpenClaw. The plugin-based architecture ensures clean separation from the core, while the phased implementation approach allows for incremental delivery of features.

Key success factors:
- Robust OAuth 2.0 implementation
- Reliable message delivery
- Comprehensive error handling
- Clear documentation
- Strong security practices

Next steps:
1. Review and approve design
2. Set up development environment
3. Implement Phase 1 (MVP)
4. Conduct integration testing
5. Deploy beta version for testing
