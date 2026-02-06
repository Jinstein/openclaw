# OpenClaw KakaoTalk Channel Plugin

KakaoTalk messaging platform integration for OpenClaw.

## Overview

This plugin enables OpenClaw to integrate with KakaoTalk, South Korea's dominant messaging platform with over 50 million users. The integration supports sending and receiving messages through KakaoTalk's official APIs.

## Features

- **AlimTalk (알림톡)**: Send informational messages to users
- **FriendTalk (친구톡)**: Send messages to channel friends
- **KakaoTalk Channel API**: Manage channel interactions
- **Message Templates**: Support for default and custom message templates
- **OAuth 2.0 Authentication**: Secure API authentication

## Installation

### From npm (when published)
```bash
openclaw plugins install @openclaw/kakao
```

### From source
```bash
openclaw plugins install ./extensions/kakao
```

### During onboarding
Select **KakaoTalk** in the channel selection and confirm the install prompt.

## Quick Setup

1. Create a KakaoTalk Channel at https://center-pf.kakao.com
2. Register your application at https://developers.kakao.com
3. Obtain API credentials (App Key, REST API Key)
4. Configure the plugin:

```json5
{
  channels: {
    kakao: {
      enabled: true,
      appKey: "your-app-key",
      restApiKey: "your-rest-api-key",
      dmPolicy: "pairing",
    },
  },
}
```

## Message Types

### AlimTalk (Information Messages)
AlimTalk is designed for sending informational messages like notifications, confirmations, and alerts. Messages must follow pre-approved templates.

### FriendTalk (Friend Messages)
FriendTalk allows sending messages to users who have added your KakaoTalk channel as a friend. More flexible content compared to AlimTalk.

### Channel Messages
Direct integration with KakaoTalk channels for 1:1 conversations and channel management.

## API Integration Methods

The plugin supports multiple integration approaches:

1. **Direct REST API**: Direct integration with Kakao's REST API
2. **JavaScript SDK**: Web-based integration using Kakao's JS SDK
3. **Business Messaging APIs**: Integration through Kakao i Connect for business messaging

## Requirements

- KakaoTalk Channel (free to create)
- Registered Kakao Developer application
- Valid API credentials
- Approved message templates (for AlimTalk)

## Configuration

See [KakaoTalk Channel Documentation](/channels/kakao) for detailed configuration options.

## Resources

- [Kakao Developers](https://developers.kakao.com)
- [KakaoTalk Channel](https://center-pf.kakao.com)
- [API Documentation](https://developers.kakao.com/docs/latest/ko/kakaotalk-channel/common)
- [Message Templates](https://developers.kakao.com/docs/latest/ko/kakaotalk-message/rest-api)

## Status

**Experimental** - Active development. Direct messages and channel interactions supported. Group messaging coming soon.

## License

See main OpenClaw repository for license information.
