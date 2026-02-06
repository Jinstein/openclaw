---
title: OpenClaw 기능 단위 분석
description: OpenClaw 코드베이스의 기능별 상세 분석 문서
---

# OpenClaw 기능 단위 분석

OpenClaw는 여러 메시징 플랫폼과 통합되어 로컬에서 실행되는 개인용 AI 어시스턴트입니다. 이 문서는 코드베이스를 기능 단위로 분석한 것입니다.

## 목차

1. [프로젝트 개요](#프로젝트-개요)
2. [핵심 아키텍처](#핵심-아키텍처)
3. [기능별 모듈 분석](#기능별-모듈-분석)
4. [통합 및 확장성](#통합-및-확장성)
5. [배포 및 플랫폼](#배포-및-플랫폼)

---

## 프로젝트 개요

**OpenClaw** (v2026.2.4)는 다음을 제공하는 통합 AI 어시스턴트 플랫폼입니다:

- **로컬 실행**: 사용자의 기기에서 직접 실행되는 자체 호스팅 솔루션
- **다중 채널 지원**: WhatsApp, Telegram, Discord, Slack, Signal 등 다양한 메시징 플랫폼 통합
- **다중 AI 제공자**: Anthropic Claude, OpenAI, Google Gemini, AWS Bedrock 등 지원
- **확장 가능**: 플러그인 및 스킬 시스템을 통한 기능 확장
- **크로스 플랫폼**: CLI, macOS 앱, iOS/Android 모바일 앱

### 주요 기술 스택

- **런타임**: Node.js 22+ / Bun
- **언어**: TypeScript (ESM)
- **프레임워크**:
  - WhatsApp: Baileys (웹 클라이언트)
  - Telegram: grammY
  - Discord: discord.js
  - Slack: Bolt SDK
  - 웹소켓 서버: ws
- **모바일**: Swift (iOS/macOS), Kotlin (Android)
- **테스트**: Vitest (70% 커버리지 목표)

---

## 핵심 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                     사용자 인터페이스                         │
│  (Telegram, Discord, WhatsApp, Slack, CLI, 모바일 앱)        │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                     채널 레이어                               │
│  (메시지 수신, 변환, 라우팅, 응답 포매팅)                      │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                   게이트웨이 서버                             │
│  (WebSocket 제어 평면, 세션 관리, 노드 레지스트리)            │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                   AI 에이전트 런타임                          │
│  (대화 처리, 도구 실행, 스트리밍, 컨텍스트 관리)               │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                AI 제공자 레이어                               │
│  (Anthropic, OpenAI, Google, Bedrock, 로컬 모델)            │
└─────────────────────────────────────────────────────────────┘
```

---

## 기능별 모듈 분석

### 1. AI 에이전트 시스템 (`src/agents/`)

#### 1.1 에이전트 런타임 핵심

**위치**: `src/agents/`

**주요 컴포넌트**:

- **에이전트 세션 관리**: Pi 에이전트 인스턴스 생성 및 생명주기 관리
- **도구 실행 엔진**: 50+ 빌트인 도구 (Bash, 파일 작업, 웹 검색, 브라우저 자동화 등)
- **스트리밍 응답**: 실시간 토큰 스트리밍 및 채널별 포맷팅
- **컨텍스트 관리**: 대화 이력, 압축, 컨텍스트 윈도우 가드
- **사고 과정 처리**: 사고(thinking) 블록의 표시 수준 제어

**핵심 기능**:

```typescript
// 에이전트 세션 생성
// src/agents/create-agent.ts
interface AgentOptions {
  model: string;           // 모델 선택 (claude-opus-4, gpt-4o 등)
  authProfile: string;     // 인증 프로필
  streaming: boolean;      // 스트리밍 응답 여부
  tools: ToolConfig[];     // 사용 가능한 도구 목록
  systemPrompt?: string;   // 시스템 프롬프트
}
```

#### 1.2 인증 프로필 시스템

**위치**: `src/agents/auth/`

**기능**:
- OAuth 및 API 키 관리
- 다중 프로필 지원 (프로필별 독립적인 인증)
- 제공자별 Failover (여러 인증 프로필 간 자동 전환)
- 환경 변수 및 설정 파일 통합

**지원 제공자**:
- Anthropic (OAuth + API 키)
- OpenAI (API 키)
- Google (OAuth + API 키)
- AWS Bedrock (IAM)
- GitHub Copilot (OAuth)
- Minimax, XAI, Groq 등

**설정 예시**:
```yaml
# ~/.openclaw/config.yaml
agent:
  authProfiles:
    - name: primary
      provider: anthropic
      oauth: true
    - name: backup
      provider: openai
      apiKey: ${OPENAI_API_KEY}
```

#### 1.3 도구 시스템

**위치**: `src/agents/tools/`

**카테고리별 도구**:

1. **시스템 도구**:
   - `Bash`: 셸 명령 실행
   - `Read`: 파일 읽기 (텍스트, 이미지, PDF, Jupyter 노트북)
   - `Write`: 파일 쓰기
   - `Edit`: 파일 편집
   - `Glob`: 파일 패턴 매칭
   - `Grep`: 코드 검색 (ripgrep 기반)

2. **웹 도구**:
   - `WebSearch`: 웹 검색 (Brave Search)
   - `WebFetch`: URL 콘텐츠 가져오기
   - `Browser`: Playwright 기반 브라우저 자동화

3. **채널 도구**:
   - `SendMessage`: 특정 채널/사용자에게 메시지 전송
   - `PairChannel`: 채널 페어링 관리
   - `ListChannels`: 활성 채널 목록 조회

4. **세션 도구**:
   - `SessionRead`: 다른 세션 읽기
   - `SessionWrite`: 세션에 메시지 추가
   - `SessionDelete`: 세션 삭제

5. **노드 도구** (모바일/데스크톱 통합):
   - `CameraSnap`: 카메라 캡처
   - `ScreenSnap`: 화면 캡처
   - `CanvasUpdate`: 캔버스 렌더링

#### 1.4 컨텍스트 관리 및 압축

**위치**: `src/agents/context/`

**기능**:
- **대화 이력 관리**: 메시지 히스토리 저장 및 로드
- **자동 압축**: 컨텍스트 윈도우 초과 시 이전 대화 요약
- **컨텍스트 가드**: 토큰 한도 모니터링 및 경고
- **세션 영속성**: JSONL 형식으로 세션 저장

**압축 전략**:
```typescript
interface CompactionConfig {
  enabled: boolean;          // 압축 활성화
  threshold: number;         // 압축 시작 임계값 (토큰 수)
  targetRatio: number;       // 목표 압축 비율 (0.5 = 50%)
  preserveRecent: number;    // 최근 메시지 보존 수
}
```

---

### 2. 게이트웨이 서버 (`src/gateway/`)

#### 2.1 WebSocket 제어 평면

**위치**: `src/gateway/server.ts`

**기능**:
- 양방향 WebSocket 통신 (에이전트 ↔ 채널 ↔ 노드)
- 프로토콜 기반 메시지 라우팅
- 연결 상태 관리 및 재연결 처리
- 인증 및 세션 관리

**프로토콜 메시지 타입**:
- `agent:message`: 에이전트 메시지 요청
- `channel:message`: 채널 메시지 수신
- `node:command`: 노드 명령 (카메라, 화면 캡처 등)
- `session:update`: 세션 상태 업데이트
- `system:health`: 헬스 체크

#### 2.2 세션 관리

**위치**: `src/gateway/sessions/`

**기능**:
- 대화 세션 추적 (사용자별, 채널별, 피어별)
- 세션 키 유도 (메인, 피어별, 채널-피어별 스코프)
- 세션 영속성 (파일 시스템)
- 세션 프루닝 (오래된 세션 자동 정리)

**세션 스코프**:
```typescript
enum SessionScope {
  MAIN = 'main',                    // 단일 메인 세션
  PER_PEER = 'per-peer',            // 사용자별 독립 세션
  PER_CHANNEL_PEER = 'per-channel-peer'  // 채널-사용자별 세션
}
```

#### 2.3 노드 레지스트리

**위치**: `src/gateway/nodes/`

**기능**:
- 연결된 장치(iOS, Android, macOS) 관리
- 노드 기능 등록 (카메라, 화면, 마이크, 캔버스)
- 노드 명령 라우팅 및 승인 워크플로
- 노드 상태 모니터링

**노드 기능**:
- `camera`: 카메라 접근
- `screen`: 화면 캡처
- `microphone`: 오디오 녹음
- `canvas`: 캔버스 렌더링
- `voice-wake`: 음성 웨이크 워드

#### 2.4 Cron 서비스

**위치**: `src/gateway/cron/`

**기능**:
- 스케줄된 작업 실행
- Cron 표현식 지원 (분, 시, 일, 월, 요일)
- 에이전트 통합 (정기적인 자동 메시지)
- 작업 이력 및 로깅

**설정 예시**:
```yaml
cron:
  jobs:
    - name: daily-summary
      schedule: "0 9 * * *"      # 매일 오전 9시
      channel: telegram
      message: "오늘의 요약을 만들어줘"
```

#### 2.5 디스커버리 및 페어링

**위치**: `src/gateway/discovery/`

**기능**:
- Bonjour/mDNS 기반 로컬 네트워크 디스커버리
- QR 코드 기반 페어링
- 게이트웨이 자동 발견 (모바일 앱용)
- 보안 페어링 토큰

#### 2.6 OpenResponses HTTP API

**위치**: `src/gateway/openresponses/`

**기능**:
- OpenAI 호환 HTTP API (`/v1/chat/completions`)
- 외부 클라이언트 통합 (Cursor, Continue 등)
- 스트리밍 및 비스트리밍 응답
- API 키 인증

---

### 3. 메시징 채널 통합 (`src/channels/`, `src/telegram/`, `src/discord/` 등)

#### 3.1 채널 아키텍처

**위치**: `src/channels/registry.ts`

**핵심 개념**:
- **채널 플러그인**: 각 메시징 플랫폼은 독립적인 플러그인
- **통합 인터페이스**: 모든 채널은 공통 API 구현
- **메시지 정규화**: 플랫폼별 메시지 → 내부 통합 포맷
- **응답 포매팅**: 내부 포맷 → 플랫폼별 메시지

**채널 플러그인 인터페이스**:
```typescript
interface ChannelPlugin {
  name: string;                      // 채널 이름 (예: 'telegram')
  start(): Promise<void>;            // 채널 시작
  stop(): Promise<void>;             // 채널 중지
  sendMessage(params): Promise<void>;  // 메시지 전송
  onMessage(handler): void;          // 메시지 수신 핸들러
  supports: ChannelCapabilities;     // 지원 기능
}
```

#### 3.2 지원 채널

**코어 채널** (`src/` 내장):

1. **WhatsApp** (`src/whatsapp/`)
   - Baileys 웹 클라이언트 사용
   - QR 코드 로그인
   - 그룹 및 DM 지원
   - 미디어 전송/수신

2. **Telegram** (`src/telegram/`)
   - grammY 프레임워크
   - 봇 API 사용
   - 인라인 키보드, 콜백 쿼리
   - 그룹 관리 권한

3. **Discord** (`src/discord/`)
   - discord.js 라이브러리
   - 길드, 채널, 스레드 지원
   - 슬래시 커맨드
   - 역할 기반 권한

4. **Slack** (`src/slack/`)
   - Bolt SDK
   - Socket Mode (WebSocket)
   - 슬래시 커맨드
   - 스레드 응답

5. **Signal** (`src/signal/`)
   - signal-cli 래퍼
   - E2E 암호화
   - 그룹 및 DM

6. **iMessage** (`src/imessage/`)
   - 레거시 imsg 프로토콜
   - macOS 전용

**확장 채널** (`extensions/`):

- Microsoft Teams (`extensions/msteams/`)
- Matrix (`extensions/matrix/`)
- LINE (`extensions/line/`)
- Feishu/Lark (`extensions/feishu/`)
- Mattermost (`extensions/mattermost/`)
- Nextcloud Talk (`extensions/nextcloud-talk/`)
- Nostr (`extensions/nostr/`)
- Twitch (`extensions/twitch/`)
- Zalo (`extensions/zalo/`, `extensions/zalouser/`)
- BlueBubbles (`extensions/bluebubbles/`)
- Voice Call (`extensions/voice-call/`)

#### 3.3 메시지 처리 파이프라인

```
수신 메시지 → 채널 플러그인 → 정규화 → 라우팅 → 에이전트 →
도구 실행 → 응답 생성 → 포매팅 → 채널 플러그인 → 전송
```

**주요 단계**:

1. **수신**: 채널별 API/웹훅에서 메시지 수신
2. **정규화**: 통합 메시지 포맷으로 변환
3. **라우팅**: 세션 키 유도 및 에이전트 라우팅
4. **처리**: 에이전트가 메시지 처리 및 응답 생성
5. **포매팅**: 채널별 포맷으로 변환 (마크다운, HTML, 플레인 텍스트)
6. **전송**: 채널 API를 통해 응답 전송
7. **확인**: 리액션/체크마크로 완료 표시

#### 3.4 그룹 메시지 처리

**위치**: `src/routing/group-handling.ts`

**기능**:
- **멘션 게이팅**: 봇이 멘션될 때만 응답
- **DM vs 그룹 구분**: 다른 처리 로직
- **스레드 지원**: Discord, Slack 스레드에서 컨텍스트 유지
- **그룹 권한 관리**: 특정 그룹/채널만 허용

**설정 예시**:
```yaml
routing:
  groupMessages:
    requireMention: true          # 멘션 필수
    allowedGroups:                # 허용된 그룹 목록
      - "telegram:group:123456"
      - "discord:guild:789012"
```

---

### 4. 라우팅 및 멀티 에이전트 (`src/routing/`)

#### 4.1 라우팅 시스템

**위치**: `src/routing/router.ts`

**기능**:
- 채널/계정/피어 기반 라우팅
- 세션 키 유도 (채널, 사용자, 대화별)
- 에이전트 워크스페이스 격리
- 바인딩 시스템 (복잡한 라우팅 규칙)

**세션 키 유도 로직**:
```typescript
// 세션 스코프에 따른 키 생성
function deriveSessionKey(
  scope: SessionScope,
  channel: string,
  peer: string,
  guild?: string
): string {
  switch (scope) {
    case 'main':
      return 'main';
    case 'per-peer':
      return `${channel}:${peer}`;
    case 'per-channel-peer':
      return `${channel}:${guild || 'dm'}:${peer}`;
  }
}
```

#### 4.2 멀티 에이전트 격리

**개념**:
- 각 사용자/그룹/채널은 독립적인 에이전트 인스턴스
- 세션 간 데이터 격리
- 동시 처리 지원 (여러 에이전트 병렬 실행)

**장점**:
- **프라이버시**: 사용자 간 대화 격리
- **커스터마이제이션**: 사용자별 다른 설정/모델
- **확장성**: 병렬 처리로 성능 향상

#### 4.3 허용 목록 및 페어링

**위치**: `src/routing/allowlist.ts`

**기능**:
- DM 페어링: 최초 사용자는 페어링 코드 필요
- 허용된 사용자 목록 관리
- 자동 페어링 (설정 시)
- 페어링 만료 및 갱신

**보안 고려사항**:
- 기본적으로 DM은 페어링 필요 (무단 접근 방지)
- 페어링 코드는 일회용 및 시간 제한
- 관리자가 수동으로 허용 목록 관리 가능

---

### 5. 미디어 파이프라인 (`src/media/`)

#### 5.1 미디어 처리

**위치**: `src/media/pipeline.ts`

**기능**:
- 이미지/오디오/비디오 다운로드 및 저장
- 포맷 변환 (Sharp, FFmpeg)
- 크기 제한 및 압축
- 임시 파일 생명주기 관리

**지원 포맷**:
- **이미지**: PNG, JPEG, WebP, GIF
- **오디오**: MP3, OGG, WAV, M4A
- **비디오**: MP4, WebM, AVI

#### 5.2 미디어 이해

**위치**: `src/media-understanding/`

**기능**:
- 이미지 분석 (비전 모델)
- 오디오 전사 (Whisper 등)
- 비디오 프레임 추출 및 분석
- OCR (텍스트 추출)

**통합**:
- 에이전트에 자동으로 미디어 컨텍스트 추가
- 사용자가 이미지/오디오 전송 시 자동 분석
- 전사 텍스트는 대화 컨텍스트에 포함

#### 5.3 미디어 저장

**위치**: `src/media/storage.ts`

**전략**:
- 임시 파일: `/tmp/openclaw/media/`
- 영구 파일: `~/.openclaw/media/`
- 자동 정리 (설정된 시간 후 삭제)
- 캐싱 (동일 미디어 재다운로드 방지)

---

### 6. 인프라 및 시스템 관리 (`src/infra/`)

#### 6.1 네트워킹

**위치**: `src/infra/network/`

**기능**:
- **포트 관리**: 자동 포트 할당 및 충돌 감지
- **Tailscale 통합**: VPN을 통한 보안 원격 접근
- **SSH 터널링**: 원격 게이트웨이 접근
- **Wide Area DNS**: 공용 DNS를 통한 디스커버리

**Tailscale 통합**:
```yaml
gateway:
  network:
    tailscale:
      enabled: true
      hostname: openclaw-gateway
```

#### 6.2 업데이트 시스템

**위치**: `src/infra/updates/`

**기능**:
- 버전 체크 (npm registry)
- 자동 업데이트 (설정 시)
- 릴리스 채널 관리 (stable, beta, dev)
- 변경 로그 표시

**릴리스 채널**:
- **stable**: 태그된 릴리스만 (`vYYYY.M.D`)
- **beta**: 프리릴리스 (`vYYYY.M.D-beta.N`)
- **dev**: 메인 브랜치 최신 커밋

#### 6.3 샌드박싱

**위치**: `src/infra/sandbox/`

**기능**:
- Docker 컨테이너에서 명령 실행
- 격리된 환경 (파일 시스템, 네트워크)
- 리소스 제한 (CPU, 메모리)
- 보안 정책

**사용 사례**:
- 에이전트 도구 실행 (Bash 등)
- 신뢰할 수 없는 코드 실행
- 개발 환경 격리

#### 6.4 실행 승인

**위치**: `src/infra/exec-approvals/`

**기능**:
- 명령 실행 전 승인 요청
- 승인 워크플로 (자동/수동)
- 명령 화이트리스트
- 감사 로그

**설정 예시**:
```yaml
security:
  execApprovals:
    enabled: true
    autoApprove:              # 자동 승인 명령
      - "ls"
      - "cat"
    requireApproval:          # 승인 필요 명령
      - "rm"
      - "sudo"
```

#### 6.5 제공자 사용량 추적

**위치**: `src/infra/provider-usage/`

**기능**:
- 토큰 사용량 추적 (입력/출력)
- 비용 추정 (모델별 가격)
- 사용량 통계 (일별, 월별)
- 사용량 제한 및 경고

#### 6.6 상태 마이그레이션

**위치**: `src/infra/migrations/`

**기능**:
- 설정 파일 버전 관리
- 자동 마이그레이션 (이전 버전 → 최신)
- 백업 및 복구
- 호환성 유지

---

### 7. CLI 및 명령어 (`src/cli/`, `src/commands/`)

#### 7.1 CLI 구조

**위치**: `src/cli/program/build-program.ts`

**아키텍처**:
- Commander.js 기반
- 계층적 명령 구조
- 플러그인 명령 자동 등록
- 도움말 및 자동 완성

**주요 명령 그룹**:
- `agent`: 에이전트 실행 및 테스트
- `gateway`: 게이트웨이 서버 관리
- `channels`: 채널 관리 및 상태
- `config`: 설정 관리
- `onboard`: 초기 설정 마법사
- `status`: 시스템 상태 조회
- `doctor`: 진단 및 수리

#### 7.2 주요 명령어

**에이전트 명령** (`openclaw agent`):
```bash
# 대화형 모드
openclaw agent

# 단일 메시지
openclaw agent --message "안녕하세요"

# RPC 모드 (JSON 입출력)
openclaw agent --mode rpc --json
```

**게이트웨이 명령** (`openclaw gateway`):
```bash
# 게이트웨이 시작
openclaw gateway run

# 특정 포트/바인딩
openclaw gateway run --port 8080 --bind 0.0.0.0

# 게이트웨이 중지
openclaw gateway stop

# 상태 확인
openclaw gateway status
```

**채널 명령** (`openclaw channels`):
```bash
# 채널 상태 조회
openclaw channels status

# 프로브 포함 상세 상태
openclaw channels status --probe

# 메시지 전송
openclaw message send --channel telegram --peer @username --text "안녕"
```

**설정 명령** (`openclaw config`):
```bash
# 설정 조회
openclaw config get agent.model

# 설정 변경
openclaw config set agent.model claude-opus-4

# 설정 파일 편집
openclaw config edit

# 설정 검증
openclaw config validate
```

**온보딩** (`openclaw onboard`):
```bash
# 대화형 설정
openclaw onboard

# 빠른 설정 (기본값)
openclaw onboard --quick
```

**진단** (`openclaw doctor`):
```bash
# 시스템 진단
openclaw doctor

# 자동 수리
openclaw doctor --fix
```

#### 7.3 진행 상황 표시

**위치**: `src/cli/progress.ts`

**기능**:
- 스피너 (단일 작업)
- 진행률 바 (다단계 작업)
- 다중 작업 진행률
- ANSI 색상 및 이모지

**사용 예시**:
```typescript
import { spinner } from './cli/progress';

const s = spinner('데이터 로딩 중...');
await loadData();
s.success('로딩 완료!');
```

---

### 8. 플러그인 시스템 (`src/plugins/`, `extensions/`)

#### 8.1 플러그인 아키텍처

**위치**: `src/plugins/loader.ts`

**개념**:
- NPM 패키지 기반 플러그인
- 독립적인 의존성 관리
- 자동 발견 및 로드
- 플러그인 SDK 제공

**플러그인 타입**:
1. **채널 플러그인**: 새로운 메시징 플랫폼 추가
2. **인증 제공자 플러그인**: 새로운 AI 제공자 추가
3. **메모리 백엔드 플러그인**: 벡터 DB 통합
4. **도구 플러그인**: 에이전트 도구 추가
5. **진단 플러그인**: 모니터링 및 로깅

#### 8.2 플러그인 SDK

**위치**: `src/plugin-sdk/`

**제공 API**:
```typescript
// 플러그인 진입점
export interface OpenClawPlugin {
  name: string;
  version: string;
  type: 'channel' | 'auth' | 'memory' | 'tool' | 'diagnostic';

  // 초기화
  init(context: PluginContext): Promise<void>;

  // 정리
  cleanup(): Promise<void>;

  // 플러그인별 인터페이스
  [key: string]: any;
}

// 플러그인 컨텍스트
export interface PluginContext {
  config: any;                   // 플러그인 설정
  logger: Logger;                // 로거 인스턴스
  gateway: GatewayClient;        // 게이트웨이 클라이언트
  storage: StorageProvider;      // 스토리지 제공자
}
```

#### 8.3 주요 플러그인

**채널 플러그인**:
- `@openclaw/plugin-msteams`: Microsoft Teams
- `@openclaw/plugin-matrix`: Matrix 프로토콜
- `@openclaw/plugin-bluebubbles`: BlueBubbles iMessage

**인증 제공자 플러그인**:
- `@openclaw/plugin-google-antigravity`: Google AI Studio
- `@openclaw/plugin-minimax`: Minimax AI
- `@openclaw/plugin-qwen`: Alibaba Qwen

**메모리 플러그인**:
- `@openclaw/plugin-lancedb`: LanceDB 벡터 DB
- `@openclaw/plugin-memory-core`: 코어 메모리 백엔드

**진단 플러그인**:
- `@openclaw/plugin-otel`: OpenTelemetry 통합

#### 8.4 플러그인 설치 및 관리

```bash
# 플러그인 목록
openclaw plugins list

# 플러그인 설치
openclaw plugins install @openclaw/plugin-matrix

# 플러그인 활성화
openclaw plugins enable matrix

# 플러그인 비활성화
openclaw plugins disable matrix

# 플러그인 제거
openclaw plugins uninstall @openclaw/plugin-matrix
```

---

### 9. 스킬 시스템 (`skills/`)

#### 9.1 스킬 아키텍처

**위치**: `skills/`

**개념**:
- 스킬 = 특화된 에이전트 도구 번들
- 각 스킬은 독립적인 디렉토리
- 메타데이터 파일 (`skill.json`)
- 도구, 프롬프트, 리소스 포함

**스킬 구조**:
```
skills/
  skill-name/
    skill.json          # 메타데이터
    prompt.md           # 시스템 프롬프트
    tools/              # 스킬 전용 도구
    resources/          # 리소스 파일
```

#### 9.2 빌트인 스킬

**생산성 스킬**:
- `1password`: 1Password CLI 통합
- `apple-notes`: Apple Notes 관리
- `apple-reminders`: Apple Reminders 관리
- `bear-notes`: Bear Notes 통합

**커뮤니케이션 스킬**:
- `discord`: Discord 서버 관리
- `github`: GitHub 이슈/PR 관리
- `bluebubbles`: iMessage 통합

**미디어 스킬**:
- `camera-snap`: 카메라 캡처
- `gif-search`: GIF 검색 및 전송
- `canvas`: 라이브 캔버스 렌더링

**개발 스킬**:
- `coding-agent`: 코드 작성 및 디버깅
- `model-usage`: AI 모델 사용량 추적

**시스템 스킬**:
- `healthcheck`: 시스템 헬스 체크
- `nano-banana-pro`: 시스템 진단

#### 9.3 스킬 사용

```bash
# 스킬 목록
openclaw skills list

# 스킬 활성화
openclaw skills enable 1password

# 스킬 비활성화
openclaw skills disable 1password

# 스킬 설정
openclaw config set skills.1password.enabled true
```

**에이전트에서 스킬 사용**:
```yaml
agent:
  skills:
    - 1password
    - github
    - camera-snap
```

---

### 10. 모바일 및 데스크톱 앱 (`apps/`)

#### 10.1 macOS 앱

**위치**: `apps/macos/`

**기능**:
- 메뉴바 앱
- 게이트웨이 제어 (시작/중지)
- 캔버스 렌더링 (A2UI)
- 음성 웨이크 워드 (Siri 단축어)
- 시스템 트레이 알림

**기술 스택**:
- Swift / SwiftUI
- AppKit
- Combine
- WebSocket 클라이언트

**빌드**:
```bash
# 패키징
pnpm mac:package

# 실행
pnpm mac:open

# 재시작
pnpm mac:restart
```

#### 10.2 iOS 앱

**위치**: `apps/ios/`

**기능**:
- 모바일 노드 (카메라, 화면)
- 대화 모드 (실시간 음성)
- 푸시 알림
- 게이트웨이 디스커버리 (mDNS)
- 설정 관리

**기술 스택**:
- Swift / SwiftUI
- UIKit
- AVFoundation (카메라/오디오)
- WebSocket 클라이언트

**빌드**:
```bash
# Xcode 프로젝트 생성
pnpm ios:gen

# Xcode 열기
pnpm ios:open

# 빌드 및 실행
pnpm ios:run
```

#### 10.3 Android 앱

**위치**: `apps/android/`

**기능**:
- 모바일 노드 (카메라, 화면)
- 푸시 알림
- 게이트웨이 디스커버리
- 설정 관리

**기술 스택**:
- Kotlin
- Jetpack Compose
- CameraX
- OkHttp (WebSocket)

**빌드**:
```bash
# 빌드
pnpm android:assemble

# 설치
pnpm android:install

# 실행
pnpm android:run
```

#### 10.4 공유 프레임워크

**위치**: `apps/shared/OpenClawKit/`

**기능**:
- 공통 모델 및 프로토콜
- WebSocket 클라이언트
- 인증 로직
- 네트워크 유틸리티

**사용**:
- iOS 및 macOS 앱에서 공유
- Swift Package Manager

---

## 통합 및 확장성

### 외부 시스템 통합

#### GitHub 통합

**기능**:
- OAuth 인증
- GitHub Copilot 모델 사용
- PR/이슈 관리 도구
- 저장소 검색

**설정**:
```yaml
agent:
  authProfiles:
    - name: github-copilot
      provider: github-copilot
      oauth: true
```

#### 브라우저 자동화

**위치**: `src/browser/`

**기능**:
- Playwright 기반
- 상태 영속성 (쿠키, localStorage)
- 스크린샷 및 PDF 생성
- JavaScript 실행

**에이전트 도구**:
- `BrowserNavigate`: URL 탐색
- `BrowserClick`: 요소 클릭
- `BrowserType`: 텍스트 입력
- `BrowserScreenshot`: 스크린샷 캡처

#### Tailscale VPN

**기능**:
- 보안 원격 접근
- NAT 통과
- 암호화된 연결

**사용 사례**:
- 외부에서 게이트웨이 접근
- 여러 장치 간 프라이빗 네트워크

#### Docker 샌드박스

**기능**:
- 격리된 명령 실행
- 커스텀 이미지 지원
- 볼륨 마운트
- 네트워크 제어

**설정**:
```yaml
sandbox:
  enabled: true
  image: ubuntu:22.04
  network: bridge
  volumes:
    - /tmp:/tmp
```

---

## 배포 및 플랫폼

### 설치 방법

**Linux/macOS**:
```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

**Windows (PowerShell)**:
```powershell
irm https://openclaw.ai/install.ps1 | iex
```

**npm (크로스 플랫폼)**:
```bash
npm install -g openclaw
```

### 시스템 요구사항

- **OS**: Linux, macOS, Windows (WSL)
- **Node.js**: 22.12.0 이상
- **메모리**: 최소 2GB RAM
- **디스크**: 500MB (설치) + 1GB (세션 데이터)
- **네트워크**: 인터넷 연결 (AI 제공자용)

### 배포 모드

#### 로컬 모드

- 단일 기기에서 실행
- 모든 컴포넌트 로컬
- 가장 간단한 설정

#### 원격 게이트웨이 모드

- 게이트웨이는 서버에서 실행
- 클라이언트는 게이트웨이에 연결
- 여러 장치에서 공유 사용

#### 분산 모드

- 여러 게이트웨이 인스턴스
- 로드 밸런싱
- 고가용성

---

## 성능 및 최적화

### 성능 특성

- **응답 시간**: 스트리밍 응답으로 즉각적인 피드백
- **동시성**: 여러 세션 병렬 처리
- **메모리 사용량**: 세션당 약 100MB
- **토큰 사용량**: 컨텍스트 압축으로 최소화

### 최적화 전략

1. **컨텍스트 압축**: 자동 대화 요약
2. **캐싱**: 미디어 및 응답 캐싱
3. **배치 처리**: 여러 도구 호출 묶기
4. **스트리밍**: 실시간 토큰 스트리밍
5. **프로토콜 최적화**: 바이너리 프로토콜 (WebSocket)

---

## 보안 고려사항

### 인증 및 권한

- OAuth 2.0 (Anthropic, OpenAI, Google, GitHub)
- API 키 보안 저장 (`~/.openclaw/credentials/`)
- 채널별 페어링 및 허용 목록
- 명령 실행 승인 워크플로

### 데이터 보호

- 로컬 데이터 저장 (클라우드 없음)
- E2E 암호화 (Signal)
- 세션 격리
- 민감 정보 필터링

### 네트워크 보안

- Tailscale VPN 통합
- HTTPS/WSS 프로토콜
- 방화벽 친화적 (Socket Mode)
- mTLS 지원 (선택 사항)

---

## 테스트 전략

### 테스트 카테고리

1. **단위 테스트** (`*.test.ts`):
   - 개별 함수/클래스 테스트
   - 70% 커버리지 목표

2. **통합 테스트**:
   - 모듈 간 상호작용 테스트
   - 채널 통합 테스트

3. **E2E 테스트** (`*.e2e.test.ts`):
   - 전체 워크플로 테스트
   - Docker 환경에서 실행

4. **라이브 테스트**:
   - 실제 API 키로 테스트
   - `LIVE=1 pnpm test:live`

### 테스트 명령

```bash
# 전체 테스트
pnpm test

# 커버리지
pnpm test:coverage

# E2E
pnpm test:e2e

# 라이브 테스트 (실제 키 필요)
pnpm test:live

# Docker 테스트
pnpm test:docker:all
```

---

## 개발 워크플로

### 코드 스타일

- TypeScript (ESM)
- Oxlint + Oxfmt
- 파일당 ~500 LOC 권장
- 간결한 코드 코멘트

### 커밋 가이드라인

- Conventional Commits
- 스코프 기반 메시지 (예: `CLI: add verbose flag`)
- `scripts/committer` 사용 권장

### PR 워크플로

1. Feature 브랜치 생성
2. 커밋 (의미 있는 메시지)
3. 테스트 실행
4. PR 생성
5. 리뷰 및 머지
6. 변경 로그 업데이트

### 릴리스 프로세스

1. 버전 범프 (`package.json`, `Info.plist` 등)
2. 변경 로그 업데이트
3. 빌드 및 테스트
4. npm 퍼블리시
5. GitHub 릴리스 생성
6. macOS 앱 공증 및 배포

---

## 결론

OpenClaw는 다음과 같은 특징을 가진 종합적인 AI 어시스턴트 플랫폼입니다:

- **모듈화**: 플러그인과 스킬로 확장 가능
- **다중 채널**: 10+ 메시징 플랫폼 지원
- **다중 제공자**: 5+ AI 제공자 통합
- **보안**: 로컬 실행, 데이터 격리, 승인 워크플로
- **크로스 플랫폼**: CLI, 웹, macOS, iOS, Android
- **개발자 친화적**: 명확한 API, 풍부한 문서, 활발한 커뮤니티

이 문서는 코드베이스의 기능 단위별 개요를 제공하며, 각 모듈의 세부 사항은 해당 소스 코드 및 개별 문서를 참조하십시오.

---

## 관련 문서

- [아키텍처 개요](/concepts/architecture)
- [CLI 참조](/cli/index)
- [채널 가이드](/channels/index)
- [플러그인 개발](/plugins/development)
- [릴리스 가이드](/reference/RELEASING)
- [테스트 가이드](/testing)

---

**마지막 업데이트**: 2026-02-06
**버전**: 2026.2.4
