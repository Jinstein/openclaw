# 카카오톡 채널 구현 TODO 리스트

## 🔴 구현 전 필수 확인 사항

### 1. OpenClaw 코어 API 이해 필요
**현재 상태**: 문서에 TypeScript 인터페이스는 있지만 실제 사용 예시 부족

**필요한 작업**:
- [ ] `ChannelPlugin` 인터페이스 전체 구조 파악
  - 파일: `/src/channels/plugins/types.ts`
  - Telegram 구현체 참고: `/extensions/telegram/src/channel.ts`
- [ ] `plugin-sdk` export 목록 확인
- [ ] 메시지 envelope 구조 파악 (MessageEnvelope type)
- [ ] ChannelDock 통합 방법 이해

**참고할 파일**:
```
src/plugins/types.ts           # Plugin 타입 정의
src/channels/plugins/types.ts  # ChannelPlugin 타입
extensions/telegram/src/        # Telegram 참고 구현
extensions/zalo/                # 간단한 구현 예시
```

### 2. 카카오 API 실제 테스트 필요
**현재 상태**: 문서만으로 API 스펙 파악, 실제 동작 미확인

**필요한 작업**:
- [ ] 카카오 개발자 계정 생성
- [ ] 테스트 앱 등록 및 API 키 발급
- [ ] REST API 테스트 도구로 실제 API 호출 테스트
  - "나에게 보내기" API 테스트
  - OAuth 인증 플로우 테스트
  - 에러 응답 형식 확인
- [ ] 실제 응답 데이터 구조 확인 (문서와 차이 확인)

**테스트 환경**:
- [Kakao REST API Test Tool](https://developers.kakao.com/docs/latest/en/tool/rest-api-test)
- Postman/Insomnia로 직접 API 테스트

### 3. AlimTalk/FriendTalk 실제 프로세스 확인
**현재 상태**: 개념적 이해만 있음

**불확실한 부분**:
- [ ] AlimTalk 템플릿 승인 프로세스
  - 실제 승인 소요 시간
  - 거부 사유 및 재신청 방법
  - 템플릿 변수 제약사항
- [ ] FriendTalk 친구 관계 확인 방법
- [ ] 비즈니스 채널 전환 조건 및 비용
- [ ] 메시지 전송 제한 (일일/시간당 한도)

### 4. 웹훅 이벤트 구조 확인
**현재 상태**: 웹훅 존재 여부만 확인

**필요한 정보**:
- [ ] 카카오톡 채널 1:1 채팅 웹훅 이벤트 형식
- [ ] 웹훅 서명 검증 방법
- [ ] 웹훅 재시도 정책
- [ ] 이벤트 타입 전체 목록

**참고**: 카카오 공식 문서에서 웹훅 스펙 찾기 어려움. 실제 테스트 필요할 수 있음.

### 5. 토큰 관리 세부사항
**불확실한 부분**:
- [ ] Access Token 정확한 만료 시간 (문서: ~6시간)
- [ ] Refresh Token 만료 시간 (문서: ~2개월)
- [ ] 토큰 갱신 시 새 Refresh Token 발급 여부
- [ ] 동시 접근 시 토큰 race condition 처리

### 6. 에러 코드 전체 목록
**현재 상태**: 주요 HTTP 상태 코드만 파악

**필요한 작업**:
- [ ] Kakao 전용 에러 코드 전체 목록 확인
- [ ] 각 에러별 재시도 가능 여부
- [ ] Rate limit 상세 정책 (앱당/키당/사용자당)

---

## 🟡 구현 중 결정 필요 사항

### 1. 메시지 타입 우선순위
**선택 필요**:
- Option A: 나에게 보내기 + 친구에게 보내기 (개인 사용자용)
- Option B: AlimTalk + FriendTalk (비즈니스용)
- Option C: 카카오톡 채널 1:1 채팅 (고객 지원용)

**추천**: Phase 1에서는 Option A (더 간단, 즉시 테스트 가능)

### 2. OpenClaw 기존 채널과의 차이점 처리
**확인 필요**:
- 카카오는 "친구 관계" 개념이 중요 → 다른 채널과 다른 접근 제어 로직
- 템플릿 사전 승인 필요 → 동적 메시지 제한적
- 웹훅이 제한적일 수 있음 → 폴링 방식 필요?

### 3. 미디어 처리 방식
**결정 필요**:
- 이미지 업로드: 카카오 서버 vs URL 참조
- 최대 파일 크기 제한 (문서: 2MB, 실제는?)
- 지원 이미지 포맷 확인

---

## 🟢 바로 시작 가능한 작업

### Phase 1-A: 기초 구조 (즉시 가능)
- [x] 폴더 구조 생성
- [x] package.json 설정
- [x] plugin metadata 작성
- [ ] TypeScript 컴파일 설정
- [ ] 기본 index.ts 구현 (로딩/언로딩만)

### Phase 1-B: API 클라이언트 스켈레톤
```typescript
// src/api-client.ts
export class KakaoApiClient {
  constructor(private config: { restApiKey: string }) {}

  async sendMemo(accessToken: string, message: string) {
    // TODO: 실제 API 호출 구현
    throw new Error('Not implemented');
  }
}
```

### Phase 1-C: OAuth 핸들러 스켈레톤
```typescript
// src/oauth.ts
export class OAuthHandler {
  getAuthorizationUrl() {
    // TODO: 구현
  }

  async exchangeToken(code: string) {
    // TODO: 구현
  }
}
```

---

## 📋 당신이 준비해야 할 것들

### 필수 준비사항

#### 1. 카카오 개발자 계정 & 앱
**작업 순서**:
1. https://developers.kakao.com 회원가입
2. "애플리케이션 추가하기" 클릭
3. 앱 이름 입력, 앱 생성
4. **앱 키 확보**:
   - JavaScript 키 (클라이언트용)
   - REST API 키 (서버용)
   - Admin 키 (관리용, 선택)

**예상 소요 시간**: 10분

#### 2. 카카오톡 채널 (선택, 비즈니스 기능용)
**작업 순서**:
1. https://center-pf.kakao.com 접속
2. "채널 만들기"
3. 채널 정보 입력 (이름, 설명, 프로필 이미지)
4. **채널 ID 확인** (형식: `_abc123def`)

**예상 소요 시간**: 5-10분

#### 3. 테스트용 카카오톡 계정
- 개발자 본인 카카오톡 계정
- (선택) 테스트용 별도 계정

#### 4. 개발 환경
```bash
# OpenClaw 개발 환경 확인
node --version  # v22 이상
pnpm --version  # 설치 확인

# 프로젝트 빌드 테스트
cd /home/user/openclaw
pnpm install
pnpm build

# Kakao 플러그인 개발 시작
cd extensions/kakao
# 여기서 개발 시작
```

#### 5. API 테스트 도구 (선택)
- [Kakao REST API Test Tool](https://developers.kakao.com/docs/latest/en/tool/rest-api-test)
- 또는 Postman/Insomnia/curl

---

## 🎯 추천 구현 순서

### Step 1: 환경 설정 (1-2시간)
1. 카카오 개발자 계정 생성
2. 테스트 앱 생성 및 키 발급
3. Kakao REST API Test Tool로 "나에게 보내기" 테스트
4. OAuth 플로우 수동 테스트 (브라우저에서)

### Step 2: 최소 구현 (4-8시간)
1. API 클라이언트 구현 (`src/api-client.ts`)
   - `sendMemo` 메서드 (나에게 보내기)
   - 에러 처리
2. OAuth 핸들러 구현 (`src/oauth.ts`)
   - Authorization URL 생성
   - Token 교환
3. 간단한 테스트 작성

### Step 3: OpenClaw 통합 (8-12시간)
1. Channel Provider 구현 (`src/provider.ts`)
2. Message Handler 구현 (`src/messages.ts`)
3. Plugin 등록 (`index.ts`)
4. 로컬에서 게이트웨이 연동 테스트

### Step 4: 고급 기능 (개별 2-4시간씩)
- 템플릿 시스템
- 미디어 처리
- AlimTalk 지원
- 에러 처리 개선

---

## 💡 개발 팁

### 1. Telegram 코드를 많이 참고하세요
```bash
# Telegram 구현이 가장 완성도 높음
extensions/telegram/src/channel.ts  # 플러그인 등록 패턴
src/telegram/bot.ts                  # 메시지 처리
src/telegram/send.ts                 # 메시지 전송
```

### 2. 점진적 구현
- 처음부터 모든 기능 구현하지 말 것
- "나에게 보내기" 1개 기능으로 시작
- 동작 확인 후 기능 추가

### 3. 실제 API 먼저 테스트
```bash
# curl로 직접 API 호출해보기
curl -X POST "https://kapi.kakao.com/v2/api/talk/memo/default/send" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "template_object": {
      "object_type": "text",
      "text": "Hello from curl!",
      "link": {"web_url": "https://openclaw.ai"}
    }
  }'
```

### 4. 로그를 많이 남기세요
```typescript
ctx.logger.info(`Kakao: Sending message to user ${userId}`);
ctx.logger.debug(`Kakao: API response: ${JSON.stringify(response)}`);
ctx.logger.error(`Kakao: Failed to send message: ${error.message}`);
```

---

## 📚 참고 자료

### 공식 문서
- [Kakao Developers](https://developers.kakao.com)
- [REST API Reference](https://developers.kakao.com/docs/latest/en/kakaologin/rest-api)
- [Message API](https://developers.kakao.com/docs/latest/en/kakaotalk-message/rest-api)
- [REST API Test Tool](https://developers.kakao.com/docs/latest/en/tool/rest-api-test)

### OpenClaw 코드 참고
```
extensions/telegram/    # 가장 완전한 구현
extensions/zalo/        # 간단한 구현
extensions/line/        # 아시아 메신저 참고용
src/channels/           # 채널 공통 로직
```

---

## 🚨 주의사항

### 1. 카카오 API 정책
- 개인정보 처리에 주의
- 스팸 발송 금지 (계정 정지 가능)
- 비즈니스 용도는 사업자 등록 필요할 수 있음

### 2. 개발 중 제한
- Access Token 만료 자주 발생 → 갱신 로직 필수
- Rate Limiting 엄격 → 재시도 로직 필수
- 템플릿 승인 1-2일 소요 → AlimTalk 개발 시 시간 여유 필요

### 3. 보안
- API 키 절대 GitHub에 커밋 금지
- `.env` 또는 `config.json`에만 저장
- 토큰은 암호화하여 저장

---

## ✅ 체크리스트

구현 시작 전 확인:
- [ ] 카카오 개발자 계정 생성 완료
- [ ] 테스트 앱 등록 및 API 키 발급 완료
- [ ] "나에게 보내기" API 수동 테스트 성공
- [ ] OpenClaw 개발 환경 세팅 완료
- [ ] Telegram 코드 읽어보기 완료
- [ ] TypeScript 기본 문법 숙지

준비되면 바로 구현 시작 가능합니다! 🚀
