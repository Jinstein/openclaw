---
summary: "KakaoTalk channel support status, capabilities, and configuration"
read_when:
  - Working on KakaoTalk features or webhooks
title: "KakaoTalk"
---

# KakaoTalk (Channel API)

Status: experimental. Direct messages and channel interactions; group support in development.

## Plugin required

KakaoTalk ships as a plugin and is not bundled with the core install.

- Install via CLI: `openclaw plugins install @openclaw/kakao`
- Or select **KakaoTalk** during onboarding and confirm the install prompt
- Details: [Plugins](/plugin)

## Quick setup (beginner)

1. **Create a KakaoTalk Channel**
   - Visit [KakaoTalk Channel Admin](https://center-pf.kakao.com)
   - Create a new channel (free for anyone)
   - Note your Channel ID

2. **Register a Kakao Developer App**
   - Go to [Kakao Developers](https://developers.kakao.com)
   - Create a new application
   - Enable KakaoTalk Channel API
   - Copy your App Key and REST API Key

3. **Install the KakaoTalk plugin**
   - From npm: `openclaw plugins install @openclaw/kakao`
   - From source: `openclaw plugins install ./extensions/kakao`
   - Or select **KakaoTalk** during onboarding

4. **Configure credentials**
   - Env: `KAKAO_APP_KEY=...` and `KAKAO_REST_API_KEY=...`
   - Or config:
     ```json5
     {
       channels: {
         kakao: {
           enabled: true,
           appKey: "your-app-key",
           restApiKey: "your-rest-api-key",
           channelId: "your-channel-id",
           dmPolicy: "pairing",
         },
       },
     }
     ```

5. **Restart the gateway**
6. DM access defaults to pairing; approve the pairing code on first contact

## What it is

- A KakaoTalk Channel API integration owned by the Gateway
- Deterministic routing: replies go back to KakaoTalk; the model never chooses channels
- DMs share the agent's main session
- Supports AlimTalk (informational messages) and FriendTalk (friend messages)

## Message Types

### AlimTalk (알림톡)

AlimTalk is designed for sending informational messages to users, such as:
- Order confirmations
- Shipping notifications
- Appointment reminders
- Payment receipts

**Requirements:**
- Messages must follow pre-approved templates
- Templates must be registered and approved in the Kakao Business Channel
- Template approval typically takes 1-2 business days

**Template structure:**
```json5
{
  template_object: {
    object_type: "text",
    text: "Your order #{{orderNumber}} has been shipped!",
    link: {
      web_url: "https://example.com/orders/{{orderNumber}}",
      mobile_web_url: "https://example.com/orders/{{orderNumber}}",
    },
  },
}
```

### FriendTalk (친구톡)

FriendTalk allows sending messages to users who have added your channel as a friend:
- More flexible content than AlimTalk
- Supports text, images, and rich media
- No template pre-approval required
- Users must be channel friends to receive messages

### Channel Messages

Direct 1:1 conversations through your KakaoTalk channel:
- Real-time chat support
- Customer service interactions
- Automated responses

## Setup (fast path)

### 1) Create a KakaoTalk Channel

1. Visit [KakaoTalk Channel Admin Center](https://center-pf.kakao.com)
2. Click "Create Channel"
3. Fill in channel information (name, profile image, description)
4. Submit for approval (instant for most channels)
5. Note your Channel ID from the channel settings

### 2) Register Kakao Developer Application

1. Go to [Kakao Developers Console](https://developers.kakao.com/console)
2. Click "Create Application"
3. Enter application name and details
4. Navigate to "App Keys" and copy:
   - App Key (JavaScript Key)
   - REST API Key
5. Go to "Platform" and add your domain
6. Go to "KakaoTalk Channel" settings and link your channel

### 3) Enable required APIs

Enable these APIs in the developer console:
- KakaoTalk Message API
- KakaoTalk Channel API
- (Optional) KakaoTalk Social API for friend list access

### 4) Configure OAuth 2.0

1. Set redirect URI in developer console (e.g., `https://your-gateway.com/oauth/kakao`)
2. Configure consent items:
   - "Send message in KakaoTalk"
   - "View KakaoTalk channel relationship"
   - "Manage KakaoTalk channel relationship"

### 5) Configure the plugin

Example config:

```json5
{
  channels: {
    kakao: {
      enabled: true,
      appKey: "abc123...",
      restApiKey: "xyz789...",
      channelId: "_abc123",
      dmPolicy: "pairing",
      messageTypes: {
        alimtalk: {
          enabled: true,
          senderKey: "your-sender-key",
        },
        friendtalk: {
          enabled: true,
        },
      },
    },
  },
}
```

Environment variables (for default account):
- `KAKAO_APP_KEY`
- `KAKAO_REST_API_KEY`
- `KAKAO_CHANNEL_ID`

Multi-account support: use `channels.kakao.accounts` with per-account credentials.

## How it works (behavior)

- Inbound messages are normalized into the shared channel envelope
- Replies route back to the same KakaoTalk conversation
- Message templates are rendered with context variables
- OAuth tokens are refreshed automatically
- Friend relationships are cached locally

## Authentication Flow

KakaoTalk uses OAuth 2.0 for user authentication:

1. User initiates login via Kakao
2. Gateway redirects to Kakao authorization URL
3. User approves permissions
4. Kakao redirects back with authorization code
5. Gateway exchanges code for access token and refresh token
6. Tokens are stored securely for API calls

Token storage: `~/.openclaw/credentials/kakao/`

## API Endpoints

### REST API Base URL
`https://kapi.kakao.com`

### Key endpoints used:
- `/v2/api/talk/memo/default/send` - Send message to self
- `/v1/api/talk/friends/message/default/send` - Send to friends
- `/v2/user/me` - Get user info
- `/v1/user/service/terms` - Check service agreements

## Limits

- AlimTalk: Rate limited by Kakao (typically 100 msg/sec per sender key)
- FriendTalk: Must respect user preferences and friendship status
- Text messages: 1000 characters max
- Image uploads: 2MB max per image
- Template approval: Required for AlimTalk (1-2 business days)

## Access control (DMs)

### DM access

- Default: `channels.kakao.dmPolicy = "pairing"`
- Unknown senders receive a pairing code; messages ignored until approved
- Approve via:
  - `openclaw pairing list kakao`
  - `openclaw pairing approve kakao <CODE>`
- `channels.kakao.allowFrom` accepts Kakao user IDs

### Finding your Kakao user ID

1. Start the gateway and send a message to your channel
2. Run `openclaw logs --follow` and look for `userId` field
3. Or call the `/v2/user/me` API endpoint

## Message Templates

### Default Templates

Kakao provides default templates for common message types:
- Text
- Feed
- List
- Location
- Commerce

Example text template:
```json5
{
  object_type: "text",
  text: "Hello from OpenClaw!",
  link: {
    web_url: "https://openclaw.ai",
    mobile_web_url: "https://openclaw.ai",
  },
}
```

### Custom Templates

Create custom templates in the [Message Template Tool](https://developers.kakao.com/console/app/message):
1. Design your template with the visual editor
2. Test the template
3. Submit for approval
4. Reference approved templates by template ID

## Capabilities

| Feature          | Status                    |
| ---------------- | ------------------------- |
| Direct messages  | ✅ Supported              |
| Groups           | 🚧 In development         |
| Media (images)   | ✅ Supported              |
| Buttons          | ✅ Supported              |
| Templates        | ✅ Supported              |
| AlimTalk         | ✅ Supported              |
| FriendTalk       | ✅ Supported              |
| Reactions        | ❌ Not supported by Kakao |
| Threads          | ❌ Not supported by Kakao |
| Streaming        | ⚠️ Limited                |

## Delivery targets (CLI/cron)

- Use a user ID as the target
- Example: `openclaw message send --channel kakao --target 123456789 --message "hi"`

## Troubleshooting

**Bot doesn't respond:**
- Check OAuth tokens: `openclaw channels status --probe`
- Verify channel is active and not blocked
- Check gateway logs: `openclaw logs --follow`
- Ensure user has added channel as friend (for FriendTalk)

**Template messages fail:**
- Verify template is approved in developer console
- Check template ID matches registered template
- Ensure all required template variables are provided

**Authentication fails:**
- Check app keys and REST API key are correct
- Verify redirect URI matches configuration
- Check that required consent items are configured

**API rate limiting:**
- Kakao enforces rate limits per sender key
- Implement exponential backoff for retries
- Consider upgrading to business API tier for higher limits

## Configuration reference (KakaoTalk)

Full configuration: [Configuration](/gateway/configuration)

Provider options:

- `channels.kakao.enabled`: enable/disable channel startup
- `channels.kakao.appKey`: JavaScript app key from developer console
- `channels.kakao.restApiKey`: REST API key from developer console
- `channels.kakao.channelId`: KakaoTalk channel ID (format: `_abc123`)
- `channels.kakao.dmPolicy`: `pairing | allowlist | open | disabled` (default: pairing)
- `channels.kakao.allowFrom`: DM allowlist (user IDs)
- `channels.kakao.messageTypes.alimtalk.enabled`: enable AlimTalk messages
- `channels.kakao.messageTypes.alimtalk.senderKey`: sender key for AlimTalk
- `channels.kakao.messageTypes.friendtalk.enabled`: enable FriendTalk messages
- `channels.kakao.mediaMaxMb`: inbound/outbound media cap (MB, default 2)
- `channels.kakao.retry`: retry policy for API calls

Multi-account options:

- `channels.kakao.accounts.<id>.appKey`: per-account app key
- `channels.kakao.accounts.<id>.restApiKey`: per-account REST API key
- `channels.kakao.accounts.<id>.channelId`: per-account channel ID
- `channels.kakao.accounts.<id>.name`: display name
- `channels.kakao.accounts.<id>.enabled`: enable/disable account
- `channels.kakao.accounts.<id>.dmPolicy`: per-account DM policy
- `channels.kakao.accounts.<id>.allowFrom`: per-account allowlist

## Resources

- [Kakao Developers](https://developers.kakao.com)
- [REST API Documentation](https://developers.kakao.com/docs/latest/en/kakaotalk-message/rest-api)
- [Message Templates](https://developers.kakao.com/docs/latest/ko/message/common)
- [KakaoTalk Channel](https://center-pf.kakao.com)
- [Business Messaging](https://docs.kakaoi.ai/kakao_i_connect_message/)

## Sources

Research for this integration was based on:
- [Kakao REST API Documentation](https://developers.kakao.com/docs/latest/en/kakaotalk-message/rest-api)
- [Kakao Developer Center](https://developers.kakao.com/docs/latest/ko/message/common)
- [Conversation API Integration Guide](https://developers.sinch.com/docs/conversation/channel-support/kakaotalk)
- [Sendbird KakaoTalk Integration](https://sendbird.com/products/business-messaging/kakaotalk)
- [Infobip KakaoTalk API](https://www.infobip.com/docs/kakaotalk/kakaotalk-over-api)
