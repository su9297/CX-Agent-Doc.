# CXAgent 개발자 문서

> **작성일**: 2025-12-07  
> **대상 독자**: 백엔드/프론트엔드 개발자 (인수인계용)  
> **프로젝트**: SAP CRM AI Assistant

---

## 목차

1. [개요](#1-개요)
2. [시스템 아키텍처](#2-시스템-아키텍처)
3. [기술 스택 및 의존성](#3-기술-스택-및-의존성)
4. [음성 전사 처리 흐름](#4-음성-전사-처리-흐름)
5. [코드 구조 및 주요 모듈 설명](#5-코드-구조-및-주요-모듈-설명)
6. [설정 및 배포](#6-설정-및-배포)
7. [에러 처리 및 로깅](#7-에러-처리-및-로깅)
8. [향후 개선 사항](#8-향후-개선-사항)

---

## 1. 개요

### 1.1 프로젝트 목적

CXAgent는 **SAP Cloud for Customer(C4C)** 시스템과 연동된 AI 기반 CRM 어시스턴트입니다. 영업 담당자가 음성 또는 텍스트로 명령을 내리면, AI가 자동으로:

- 회의록을 생성하거나
- 방문 계획(Visit)을 수립하거나
- 업무(Task)를 할당하거나
- 견적/기회 데이터를 조회합니다.

### 1.2 비즈니스 요구사항

| 요구사항 | 설명 |
|----------|------|
| **음성 회의록 자동화** | 녹음 파일을 업로드하면 전사 후 구조화된 회의록 생성 |
| **자연어 명령 처리** | "삼성전자 김철수와 내일 2시 미팅" → 방문 계획 자동 생성 |
| **CRM 데이터 연동** | SAP C4C에서 견적/기회/계정 데이터 조회 및 생성 준비 |
| **권한 기반 접근 제어** | PRD 환경에서 SALES 역할 사용자만 접근 가능 |
| **실시간 처리 상태 표시** | SSE를 통한 음성 처리 진행률 실시간 전송 |

### 1.3 주요 기능 요약

```
┌─────────────────────────────────────────────────────────────┐
│                      CXAgent 주요 기능                       │
├─────────────────────────────────────────────────────────────┤
│  🎤 음성 처리                                                │
│     - Whisper 기반 한국어/영어 음성 전사                      │
│     - 한국어 특화 7단계 전처리 파이프라인                      │
│     - GPT-4o-mini 기반 회의록 자동 생성                       │
├─────────────────────────────────────────────────────────────┤
│  📅 일정 관리                                                │
│     - 자연어 → Visit/Task 생성                               │
│     - 하이브리드 날짜 파싱 (규칙 90% + LLM 10%)               │
├─────────────────────────────────────────────────────────────┤
│  📊 데이터 조회                                              │
│     - Sales Quote(견적) 조회                                 │
│     - Opportunity(기회) 조회                                 │
│     - 다양한 필터 조건 지원                                   │
├─────────────────────────────────────────────────────────────┤
│  🔐 보안 및 인증                                             │
│     - SAP XSUAA OAuth 2.0 인증                               │
│     - 환경별(DEV/PRD) 자동 감지 및 권한 분기                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 시스템 아키텍처

### 2.1 전체 시스템 구성

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           클라이언트 레이어                              │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  flask_audio_upload_fixed.html (테스트 UI)                        │  │
│  │  - 음성 파일 업로드                                               │  │
│  │  - SSE 진행률 수신                                                │  │
│  │  - 텍스트 질의 입력                                               │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────┬───────────────────────────────────────┘
                                  │ HTTP/SSE
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          애플리케이션 레이어                             │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  main.py (Quart 비동기 웹 서버)                                   │  │
│  │  ├─ /question          : 텍스트 질의 처리                         │  │
│  │  ├─ /ai-api/upload     : 음성 파일 업로드                         │  │
│  │  ├─ /ai-api/stream/:id : SSE 진행률 스트리밍                      │  │
│  │  ├─ /getDestinationURL : Destination URL 조회                    │  │
│  │  └─ /test              : 헬스체크                                 │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                  │                                      │
│  ┌──────────────────┐  ┌────────┴────────┐  ┌─────────────────────┐   │
│  │  LangGraph       │  │  Audio Pipeline │  │  date_processing/  │   │
│  │  워크플로우 엔진  │  │  음성 처리 모듈  │  │  날짜 파싱 모듈     │   │
│  └──────────────────┘  └─────────────────┘  └─────────────────────┘   │
└─────────────────────────────────┬───────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           외부 서비스 레이어                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐ │
│  │  SAP AI Core    │  │  SAP C4C        │  │  SAP BTP               │ │
│  │  ├─ GPT-4o-mini │  │  ├─ 계정 조회    │  │  ├─ Destination 서비스 │ │
│  │  └─ Whisper-1   │  │  ├─ 견적 조회    │  │  ├─ XSUAA 인증        │ │
│  │  (OpenAI Proxy) │  │  └─ 기회 조회    │  │  └─ 서비스 바인딩      │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 모듈 간 데이터 흐름

#### 텍스트 질의 처리 흐름

```
사용자 입력 ("삼성전자 견적 조회해줘")
        │
        ▼
┌─────────────────┐
│  /question API  │
└────────┬────────┘
         │ JWT 토큰 검증 (XSUAA)
         ▼
┌─────────────────┐
│  graph_or_not   │ ← GraphQL 도구 사용 여부 판단
└────────┬────────┘
         │ "route_question" 또는 "graph"
         ▼
┌─────────────────┐
│ route_question  │ ← 의도 분류 (create_task/display/speech/exception)
└────────┬────────┘
         │
    ┌────┼────┬────────┬──────────┐
    ▼    ▼    ▼        ▼          ▼
 create  display  create_task  speech  exception
    │       │         │           │         │
    └───────┴─────────┴───────────┴─────────┘
                      │
                      ▼
              SAP C4C API 호출
                      │
                      ▼
                 JSON 응답 반환
```

#### 음성 처리 데이터 흐름

```
음성 파일 업로드 (POST /ai-api/upload)
        │
        ▼
┌────────────────────────┐
│  세션 ID 생성          │ ← UUID 기반
│  임시 파일 저장         │ ← 현재 작업 디렉터리
└───────────┬────────────┘
            │ asyncio.create_task()
            ▼
┌────────────────────────┐          ┌─────────────────────────┐
│  백그라운드 처리       │─────────▶│  SSE 스트리밍           │
│  process_audio_bg()   │  진행률   │  /ai-api/stream/:id     │
└───────────┬────────────┘  업데이트 └─────────────────────────┘
            │
    ┌───────┴───────┐
    ▼               ▼
┌──────────┐   ┌───────────┐
│ 전처리    │   │ Whisper   │
│ FFmpeg    │   │ STT 전사  │
│ VAD/AGC   │   │           │
└─────┬────┘   └─────┬─────┘
      └──────┬───────┘
             ▼
      ┌──────────────┐
      │  프롬프트     │
      │  분석         │
      └──────┬───────┘
             │
    ┌────────┼────────┐
    ▼        ▼        ▼
 Visit    Task     회의록
 생성     생성     생성
    │        │        │
    └────────┴────────┘
             │
             ▼
      최종 결과 반환 (SSE)
```

---

## 3. 기술 스택 및 의존성

### 3.1 사용 언어 및 프레임워크

| 구분 | 기술 | 버전 정보 |
|------|------|----------|
| **런타임** | Python | 3.10 (`runtime.txt`) |
| **웹 프레임워크** | Quart | 비동기 Flask 호환 |
| **AI 오케스트레이션** | LangGraph + LangChain | 워크플로우 상태 관리 |
| **프론트엔드** | HTML/JS | 테스트 UI 전용 |

### 3.2 주요 라이브러리 및 역할

```python
# requirements.txt 기준

# === 웹 프레임워크 ===
Flask                    # Quart와 호환용
quart                    # 비동기 웹 서버 (메인)
quart-cors               # CORS 처리

# === AI/ML ===
generative-ai-hub-sdk[all]  # SAP AI Core 연동 SDK
langgraph                   # LangChain 워크플로우 엔진
langchain                   # LLM 오케스트레이션

# === 음성 처리 ===
faster-whisper              # Whisper STT (로컬 실행)
pydub>=0.25.0               # 오디오 포맷 변환
soundfile>=0.12.0           # 오디오 파일 I/O
scipy>=1.10.0               # 신호 처리 (필터링)

# === SAP BTP 연동 ===
cfenv                       # Cloud Foundry 환경 변수
sap-xssec                   # SAP XSUAA 인증

# === 유틸리티 ===
pydantic                    # 데이터 검증
httpx                       # 비동기 HTTP 클라이언트
tenacity                    # 재시도 로직
numpy                       # 수치 계산
```

### 3.3 시스템 의존성

```yaml
# apt.yml
packages:
  - ffmpeg  # 오디오 코덱 변환 필수
```

> ⚠️ **중요**: FFmpeg가 없으면 음성 전사 기능이 작동하지 않습니다. Cloud Foundry 배포 시 apt-buildpack이 먼저 실행되어야 합니다.

### 3.4 외부 서비스 의존성

| 서비스 | 용도 | 필수 여부 |
|--------|------|----------|
| `agent-uaa` | XSUAA 인증 | 필수 |
| `aicore` | GPT-4o-mini + Whisper API | 필수 |
| `hrm-destination-service` | C4C/GraphQL 연결 정보 | 필수 |

---

## 4. 음성 전사 처리 흐름

### 4.1 전체 처리 단계 요약

```
입력 → 전처리 → STT → 후처리 분기 → 결과 생성 → 응답
```

### 4.2 단계별 상세 설명

#### 📥 1단계: 파일 업로드 및 세션 생성

**엔드포인트**: `POST /ai-api/upload`

```python
# main.py:3886-3986

async def upload_start():
    # 1. 인증 검증 (XSUAA JWT)
    access_token = request.headers.get('authorization')[7:]
    security_context = xssec.create_security_context(access_token, uaa_service)
    
    # 2. PRD 환경이면 SALES 역할 확인
    if CURRENT_ENV == 'prd':
        await check_user_authorization(email)
    
    # 3. 파일 저장
    temp_path = await save_uploaded_file(file)  # UUID 기반 파일명
    
    # 4. 세션 생성 및 백그라운드 처리 시작
    session_id = str(uuid.uuid4())
    asyncio.create_task(process_audio_background(session_id))
    
    return {'session_id': session_id, 'success': True}
```

**검증 사항**:
- 파일 확장자: `.mp3`, `.wav`, `.m4a`, `.ogg`, `.flac`
- 최대 크기: 25MB (`MAX_UPLOAD_SIZE`)
- 최대 파일 수: 5개

---

#### 🔧 2단계: 한국어 최적화 전처리

**함수**: `korean_optimized_preprocessing()` (main.py:1041-1114)

```
원본 오디오
    │
    ├─▶ [1] FFmpeg 기본 변환
    │       - 16kHz 샘플레이트
    │       - 모노 채널
    │       - PCM 16-bit
    │       - 볼륨 1.5배 증폭
    │
    ├─▶ [2] 무음 제거 (remove_silence_advanced)
    │       - 400ms 이상 무음만 제거
    │       - 200ms 어절 간 쉼 보존
    │       - threshold: -35dB
    │
    ├─▶ [3] Pre-emphasis 필터 (apply_korean_preemphasis)
    │       - 계수: 0.95
    │       - 한국어 자음(ㅅ,ㅊ,ㅋ,ㅌ,ㅍ,ㅎ) 고주파 강화
    │
    ├─▶ [4] 대역통과 필터 (apply_korean_bandpass_filter)
    │       - 주파수 범위: 300-7500Hz
    │       - 4차 버터워스 필터
    │
    ├─▶ [5] 적응형 게인 컨트롤 (apply_adaptive_gain_control)
    │       - 목표 RMS: 0.15
    │       - 50ms 프레임 단위 처리
    │
    ├─▶ [6] VAD (apply_vad_preprocessing)
    │       - 500ms 이상 무음 감지
    │       - threshold: -40dBFS
    │
    └─▶ [7] 최종 정규화 (apply_normalization)
            - pydub.effects.normalize() 사용
```

> 💡 **성능 고려사항**: 전처리 단계는 파일 크기에 비례하여 처리 시간이 증가합니다. 5MB 이상 파일은 "대용량 파일"로 감지되어 SSE keep-alive 간격이 짧아집니다.

---

#### 🎤 3단계: Whisper STT 전사

**함수**: `transcribe_audio()` (main.py:1142-1330)

```python
# 모델 설정
model_size = "base"       # 속도-정확도 균형
compute_type = "int8"     # CPU 최적화
device = "cpu"            # Cloud Foundry 환경

# 전사 설정
segments, info = whisper_model.transcribe(
    file_path,
    language="ko",           # 한국어 강제 지정
    word_timestamps=False,   # 단어별 타임스탬프 비활성화 (속도)
    beam_size=1,             # 빔 검색 최소화 (속도)
    best_of=1,               # 샘플링 최소화 (속도)
    temperature=0.0          # 결정론적 출력
)
```

**반환 데이터 구조**:
```python
{
    "success": True,
    "text": "전사된 텍스트 전체",
    "language": "ko",
    "duration": 120.5,  # 초 단위
    "segments": [
        {"text": "...", "start": 0.0, "end": 5.2},
        {"text": "...", "start": 5.2, "end": 10.1}
    ],
    "model_used": "faster-whisper-base",
    "fallback": False  # 에러 시 True
}
```

---

#### 🔀 4단계: 후처리 분기

**함수**: `process_audio_background()` (main.py:4156-4436)

프롬프트 내용에 따라 3가지 경로로 분기:

| 프롬프트 키워드 | 처리 경로 | 처리 함수 |
|----------------|-----------|-----------|
| `방문`, `visit`, `미팅`, `meeting` | Visit 생성 | `extract_visit_info_from_transcript()` → `_handle_create_task("visit")` |
| `task`, `태스크`, `업무`, `할일` | Task 생성 | `extract_task_info_from_transcript()` → `_handle_create_task("task")` |
| 그 외 | 회의록 생성 | `generate_meeting_minutes()` |

---

#### 📝 5단계: 결과 생성

##### 5-A. Visit/Task 생성 흐름

```python
# 1. 전사 텍스트에서 정보 추출 (LLM 사용)
extraction_result = extract_visit_info_from_transcript(transcript, prompt)
# → account_name, contact_person, visit_start_date, location 등

# 2. 날짜 파싱 (하이브리드 시스템)
date_result = date_parser.parse_visit_date("내일 오후 2시")
# → {"start_datetime": "2025-12-08T14:00:00Z", "end_datetime": "..."}

# 3. C4C API로 계정/담당자 조회
accounts = _try_get(f"{ACC_URL}?$filter=formattedName ct '삼성전자'")
employees = _try_get(f"{EMP_URL}?$filter=workplaceAddress/eMail eq '...'")

# 4. 데이터 변환 및 응답 생성
transformed_data = transform_visit_create_data(results)
```

##### 5-B. 회의록 생성 흐름

```python
# 1. 콘텐츠 안전화 (Azure 콘텐츠 필터 대응)
safe_transcript = aggressive_sanitize_korean(transcript)

# 2. 긴 텍스트는 청크 분할 (500자 이상)
chunks = chunk_transcript(safe_transcript, max_chunk_size=200)

# 3. 청크별 요약 생성
for chunk in chunks:
    summary = safe_llm_call(chunk_prompt)

# 4. 최종 통합 회의록 생성
meeting_minutes = safe_llm_call(final_prompt)
```

> ⚠️ **콘텐츠 필터 방어**: Azure OpenAI의 콘텐츠 필터가 한국어 음성 인식 결과를 오탐하는 경우가 있어, `aggressive_sanitize_korean()` 함수로 90개 이상의 단어를 사전 치환합니다.

---

#### 📤 6단계: 결과 반환 및 정리

**SSE 스트리밍**: `GET /ai-api/stream/:session_id`

```javascript
// 클라이언트 SSE 수신 예시
const eventSource = new EventSource(`/ai-api/stream/${sessionId}`);

eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    
    if (data.event === 'keepalive') return;  // keep-alive 무시
    if (data.final) {
        // 최종 결과
        console.log(data.result);
        eventSource.close();
    } else {
        // 진행률 업데이트
        updateProgress(data.progress, data.status);
    }
};
```

**임시 파일 정리**:
- 처리 완료 즉시: `cleanup_temp_file()` 호출
- 세션 만료 후 10분: `cleanup_session()` 비동기 실행
- 서버 종료 시: `cleanup_all_temp_files()` 전체 정리

---

## 5. 코드 구조 및 주요 모듈 설명

### 5.1 디렉터리별 책임

```
cxagent/
├── main.py                       # 메인 애플리케이션 (4,577줄)
│   ├── 웹 서버 설정 및 라우팅
│   ├── LangGraph 워크플로우 정의
│   ├── 음성 처리 함수들
│   ├── SAP API 연동 함수들
│   └── SSE 스트리밍 로직
│
├── BTPservices.py                # BTP Destination 서비스 연동 (107줄)
│   └── getDestination()          # OAuth 토큰 + Destination 정보 조회
│
├── date_processing/              # 날짜 파싱 모듈
│   ├── hybrid_parser.py          # 하이브리드 파서 (메인 진입점)
│   ├── rule_parser.py            # 규칙 기반 파서 (90% 케이스)
│   ├── llm_parser.py             # LLM 기반 파서 (복잡한 케이스)
│   ├── patterns.py               # 정규식 패턴 정의
│   └── utils.py                  # 유틸리티 함수
│
├── flask_audio_upload_fixed.html # 테스트 UI
│
└── [배포 설정 파일들]
```

### 5.2 핵심 클래스/함수 역할

#### main.py 주요 함수

| 함수명 | 라인 | 역할 |
|--------|------|------|
| `transcribe_audio()` | 1142-1330 | Whisper STT 전사 (에러 폴백 포함) |
| `korean_optimized_preprocessing()` | 1041-1114 | 한국어 특화 7단계 전처리 |
| `generate_meeting_minutes()` | 1546-2023 | 회의록 생성 (5중 콘텐츠 필터 방어) |
| `extract_visit_info_from_transcript()` | 1332-1442 | 방문 정보 추출 |
| `extract_task_info_from_transcript()` | 1444-1544 | 업무 정보 추출 |
| `call_sap_api()` | 2449-2641 | 견적/기회 조회 API 호출 |
| `_handle_create_task()` | 2643-2907 | Visit/Task 생성 통합 처리 |
| `process_audio_background()` | 4156-4436 | 백그라운드 음성 처리 |
| `stream_progress()` | 4012-4154 | SSE 진행률 스트리밍 |

#### LangGraph 노드

```python
# main.py:3761-3794

workflow = StateGraph(GraphState)

# 노드 정의
workflow.add_node("route_question", route_question)  # 의도 분류
workflow.add_node("create", create)                  # 엔티티 생성 안내
workflow.add_node("create_task", create_task)        # Visit/Task 생성
workflow.add_node("display", display)                # 데이터 조회
workflow.add_node("speech", speech)                  # 음성 처리 안내
workflow.add_node("exception", exception)            # 일반 질문 응답
workflow.add_node("graph", graph)                    # GraphQL 호출
workflow.add_node("generate", generate)              # GraphQL 결과 생성
workflow.add_node("tools", ToolNode(graph_tool))     # 도구 실행

# 엣지 정의 (조건부 분기)
workflow.set_conditional_entry_point(graph_or_not, {"graph": "graph", "route_question": "route_question"})
workflow.add_conditional_edges("route_question", route_condition, [...])
```

#### date_processing 모듈

| 클래스 | 파일 | 역할 |
|--------|------|------|
| `HybridDateParser` | hybrid_parser.py | 메인 진입점, 규칙/LLM 결합 |
| `RuleBasedDateParser` | rule_parser.py | 정규식 기반 날짜 파싱 |
| `LLMDateParser` | llm_parser.py | GPT 기반 복잡한 표현 파싱 |
| `DatePatterns` | patterns.py | 패턴 상수 및 매핑 정의 |

```python
# 사용 예시
date_parser = HybridDateParser(llm=llm, llm_enabled=True)
result = date_parser.parse_visit_date("추석 연휴 다음 첫 업무일 오전 10시")
# → {"start_datetime": "2025-10-10T10:00:00Z", "confidence": 0.85, "method_used": "llm"}
```

### 5.3 주요 API 엔드포인트

| 메서드 | 경로 | 인증 | 설명 |
|--------|------|------|------|
| `POST` | `/question` | JWT 필수 | 텍스트 질의 처리 |
| `POST` | `/ai-api/upload` | JWT 필수 | 음성 파일 업로드 |
| `GET` | `/ai-api/stream/:id` | - | SSE 진행률 스트림 |
| `GET` | `/ai-api/get-progress/:id` | - | 폴링용 진행률 조회 |
| `GET` | `/getDestinationURL` | - | Destination URL 조회 |
| `GET` | `/test` | - | 헬스체크 |
| `GET` | `/GetToken` | JWT 필수 | 토큰 확인 (디버그) |
| `GET` | `/flask_audio_upload_fixed` | - | 테스트 UI 서빙 |

#### /question 요청/응답 예시

**Request**:
```json
POST /question
Content-Type: application/json
Authorization: Bearer eyJ...

{
    "user_query": "삼성전자 김철수와 내일 오후 2시 미팅 계획 세워줘"
}
```

**Response (Visit 생성)**:
```json
{
    "event": "create_task",
    "flag": true,
    "value": "visit",
    "message": "2025년 12월 8일 오후 2시부터 2025년 12월 8일 오후 3시까지 삼성전자의 위치 미정에서 김철수와 방문 계획을 생성하시겠습니까?",
    "data": {
        "mapping": [
            {"key": "account_id", "value": "계정 ID"},
            {"key": "account_name", "value": "계정명"},
            ...
        ],
        "realValue": [{
            "account_id": "123ABC",
            "account_name": "삼성전자",
            "contact_name": "김철수",
            "visit_start_date": "2025-12-08T05:00:00Z",
            "visit_end_date": "2025-12-08T06:00:00Z",
            ...
        }]
    }
}
```

---

## 6. 설정 및 배포

### 6.1 환경 변수 목록

#### 필수 환경 변수 (Cloud Foundry 자동 설정)

| 변수명 | 설명 | 예시 |
|--------|------|------|
| `VCAP_APPLICATION` | CF 앱 정보 (JSON) | `{"space_name": "JMCKRAI", ...}` |
| `VCAP_SERVICES` | 바인딩 서비스 정보 | `{"aicore": [...], ...}` |
| `PORT` | 서버 포트 | `8080` |

#### 애플리케이션 환경 변수 (manifest.yml 설정)

```yaml
env:
  CF_DEPLOYMENT: "true"              # CF 환경 감지
  WHISPER_CACHE_DIR: "/tmp"          # 모델 다운로드 위치
  FFMPEG_BINARY: "/usr/bin/ffmpeg"   # FFmpeg 경로
  
  # 타임아웃 설정 (10분)
  HTTP_TIMEOUT: "600"
  REQUEST_TIMEOUT: "600"
  GUNICORN_TIMEOUT: "600"
  UPLOAD_TIMEOUT: "600"
  
  # SSE 최적화
  CF_ENABLE_SSE: "true"
  X_ACCEL_BUFFERING: "no"            # Nginx 버퍼링 비활성화
  
  # 성능 최적화
  OMP_NUM_THREADS: "4"
  OPENBLAS_NUM_THREADS: "4"
```

#### 선택적 환경 변수 (코드에서 참조)

| 변수명 | 기본값 | 설명 |
|--------|--------|------|
| `ENVIRONMENT` | 자동감지 | `dev` 또는 `prd` |
| `MAX_UPLOAD_SIZE` | 25MB | 최대 업로드 크기 |
| `AUDIO_WORKERS` | 2 | ThreadPoolExecutor 워커 수 |
| `ENABLE_KOREAN_FEATURES` | true | 한국어 전처리 사용 여부 |

### 6.2 설정 파일 설명

#### manifest.yml

```yaml
applications:
- name: cxagent
  disk_quota: 8G              # Whisper 모델 + 임시 파일용
  memory: 4G                  # STT 처리에 필요
  buildpacks:
  - https://github.com/cloudfoundry/apt-buildpack.git    # 1순위: FFmpeg 설치
  - https://github.com/cloudfoundry/python-buildpack.git # 2순위: Python
  command: python main.py
  timeout: 600                # 스테이징 타임아웃 (10분)
  services:
  - agent-uaa                 # 인증
  - aicore                    # AI 서비스
  - hrm-destination-service   # 외부 연결
```

#### mta.yaml

MTA(Multi-Target Application) 빌드/배포 정의. 서비스 인스턴스 생성 및 바인딩 자동화.

#### xs-security.json

```json
{
  "xsappname": "cxagent",
  "tenant-mode": "dedicated",
  "oauth2-configuration": {
    "redirect-uris": ["https://*.cfapps.jp10.hana.ondemand.com/**"]
  }
}
```

### 6.3 로컬 개발 환경 설정

```bash
# 1. Python 가상환경 생성
python -m venv venv
.\venv\Scripts\activate  # Windows

# 2. 의존성 설치
pip install -r requirements.txt

# 3. FFmpeg 설치 (Windows)
choco install ffmpeg

# 4. 환경 변수 설정 (.env 파일 또는 직접)
# main.py에서 로컬용 주석 해제 필요:
#   - AICORE_CLIENT_ID
#   - AICORE_CLIENT_SECRET
#   - AICORE_AUTH_URL
#   - AICORE_BASE_URL

# 5. 서버 실행
python main.py
# → http://localhost:5000
```

> ⚠️ **로컬 개발 시 주의**: main.py에서 "로컬용 주석"과 "배포용 주석" 블록을 전환해야 합니다. 인증 및 서비스 바인딩 부분이 영향을 받습니다.

### 6.4 Cloud Foundry 배포 절차

#### MTA 배포 (권장)

```bash
# 1. MTA 빌드
mbt build

# 2. 배포
cf deploy mta_archives/cxagent_1.0.0.mtar
```

#### 직접 배포

```bash
# 1. 서비스 생성 (최초 1회)
cf create-service destination lite hrm-destination-service
cf create-service xsuaa application agent-uaa -c xs-security.json
cf create-service aicore extended aicore-service

# 2. 앱 푸시
cf push -f manifest.yml

# 3. 로그 확인
cf logs cxagent --recent
```

---

## 7. 에러 처리 및 로깅

### 7.1 주요 예외 케이스

#### 인증 관련

| 상황 | HTTP 코드 | 메시지 |
|------|-----------|--------|
| Authorization 헤더 없음 | 403 | (abort 처리) |
| JWT 검증 실패 | 403 | (xssec 예외) |
| SALES 역할 없음 (PRD) | 403 | "접근 권한이 없습니다." |

```python
# main.py:3820-3833
if 'authorization' not in request.headers:
    abort(403)
    
if CURRENT_ENV == 'prd':
    try:
        await check_user_authorization(email)
    except Exception as auth_error:
        abort(403)
```

#### 파일 업로드 관련

| 상황 | HTTP 코드 | 응답 |
|------|-----------|------|
| 파일 크기 초과 (25MB) | 413 | `{"error": "File size too large", ...}` |
| 파일 없음 | 400 | `{"error": "업로드된 파일이 없습니다."}` |
| 빈 파일 | 400 | `{"error": "업로드된 파일이 비어있습니다."}` |

#### STT 처리 관련

| 상황 | 처리 방식 |
|------|-----------|
| FFmpeg 미설치 | 폴백 텍스트 반환 + `fallback: true` |
| Whisper 모델 로드 실패 | 에러 메시지 반환 |
| 전사 실패 | 원본 경로로 재시도 후 폴백 |

```python
# main.py:1319-1330 (폴백 처리)
return {
    "success": True,
    "text": f"안녕하세요. Whisper 처리 중 일시적인 문제가 발생...",
    "fallback": True,
    "note": "Whisper 처리 중 오류가 발생했지만 의미있는 결과를 제공합니다."
}
```

#### 콘텐츠 필터 관련

Azure OpenAI 콘텐츠 필터 트리거 시:

```python
# main.py:1735-1824 (safe_llm_call)
except Exception as e:
    if "content_filter" in error_msg or "ResponsibleAIPolicyViolation" in error_msg:
        if attempt < 3:
            # 재시도: 더 강한 전처리 적용
            sanitized_prompt = aggressive_sanitize_korean(prompt)
            return safe_llm_call(sanitized_prompt, attempt + 1)
        else:
            # 최종 폴백: 기본 회의록 템플릿 반환
            return f"""## 📋 업무 회의록
            ### 회의 개요
            - 일시: {datetime.now().strftime('%Y년 %m월 %d일')}
            ...
            """
```

### 7.2 로그 포맷

현재 코드베이스는 **표준 print문**을 사용합니다.

```python
print(f"LLM initialized successfully")
print(f"⚠️ Destination 서비스 연결 실패, 로컬 폴백 값 사용: {e}")
print(f"사용자 롤: {user_roles}")
```

> 📝 **추가 명세 필요**: 구조화된 로깅(logging 모듈) 및 외부 로그 수집 시스템 연동은 현재 구현되어 있지 않습니다.

### 7.3 장애 대응 관점에서 알아야 할 사항

#### 세션 관리

- 세션은 `processing_status` 딕셔너리에 인메모리 저장
- SSE 연결 종료 후 10분 뒤 자동 정리
- **주의**: 서버 재시작 시 모든 진행 중인 세션 유실

```python
# main.py:4126-4141 (세션 정리)
async def cleanup_session():
    await asyncio.sleep(600)  # 10분 후
    if session_id in processing_status:
        # 임시 파일도 함께 정리
        del processing_status[session_id]
```

#### 타임아웃 설정

| 항목 | 값 | 위치 |
|------|-----|------|
| STT 처리 타임아웃 | 30분 (폴링), 60분 (SSE 대용량) | stream_progress() |
| HTTP 요청 | 600초 (10분) | manifest.yml |
| C4C API 호출 | 15초 | call_sap_api() |
| FFmpeg 변환 | 30초 | preprocess_audio_for_speed() |

#### 임시 파일 관리

```bash
# 임시 파일 패턴
whisper_audio_*.mp3      # 원본 업로드
whisper_audio_*_preprocessed.wav
whisper_audio_*_vad.wav
whisper_audio_*_normalized.wav
```

정리 시점:
1. 처리 완료 직후
2. 세션 만료 (10분)
3. 서버 시작/종료 시 (`cleanup_all_temp_files()`)

---

## 8. 향후 개선 사항

### 8.1 코드 상에서 보이는 기술 부채

#### 🔴 높은 우선순위

| 항목 | 현재 상태 | 개선 방향 |
|------|-----------|-----------|
| **main.py 크기** | 4,577줄 단일 파일 | 모듈 분리 (routes/, services/, models/) |
| **로컬/배포 코드 전환** | 주석 토글 방식 | 환경 변수 기반 자동 분기 |
| **인메모리 세션 저장** | 서버 재시작 시 유실 | Redis 또는 DB 기반 세션 저장 |
| **하드코딩된 값** | 폴백 URL, 비밀번호, 이메일 | 환경 변수 또는 secrets 관리 |

```python
# 예: 하드코딩된 폴백 값 (main.py:3720-3722)
C4C_BASE_URL = "https://my1001291.au1.test.crm.cloud.sap/sap/c4c/api/v1"
C4C_AUTH_USER = "Interface"
C4C_AUTH_PASSWORD = "Welcome1!@Welcome1!@..."  # ⚠️ 보안 위험
```

#### 🟡 중간 우선순위

| 항목 | 현재 상태 | 개선 방향 |
|------|-----------|-----------|
| **로깅** | print문 사용 | Python logging + 구조화 로그 |
| **에러 핸들링** | 일관성 부족 | 커스텀 예외 클래스 정의 |
| **테스트 코드** | 없음 (코드 기준 판단) | pytest 기반 단위/통합 테스트 |
| **API 문서화** | README만 존재 | OpenAPI/Swagger 스펙 생성 |

### 8.2 성능 관점 개선 아이디어

| 항목 | 현재 방식 | 개선 아이디어 |
|------|-----------|---------------|
| **Whisper 모델** | 요청 시 지연 로딩 | 서버 시작 시 워밍업 |
| **동시 처리** | ThreadPoolExecutor (2개) | 워커 수 동적 조정 또는 Celery |
| **파일 처리** | 동기 I/O | aiofiles 비동기 I/O |
| **SSE 타임아웃** | 하드코딩된 간격 | 클라이언트 기반 적응형 간격 |

### 8.3 확장성 관점 개선 아이디어

| 항목 | 현재 한계 | 개선 아이디어 |
|------|-----------|---------------|
| **수평 확장** | 세션 공유 불가 | Redis 세션 스토어 + 스티키 세션 |
| **파일 저장** | 로컬 디스크 | S3/Object Storage |
| **백그라운드 작업** | asyncio.create_task | 메시지 큐 (RabbitMQ, Redis Queue) |
| **모델 업그레이드** | 코드 수정 필요 | 모델 버전 환경 변수화 |

### 8.4 보안 관점 개선 아이디어

| 항목 | 현재 상태 | 권장 조치 |
|------|-----------|-----------|
| **하드코딩 비밀번호** | 코드에 존재 | Secrets Manager 사용 |
| **파일 업로드** | 확장자만 검증 | MIME 타입 + 바이러스 스캔 |
| **입력 검증** | 부분적 | 모든 사용자 입력 sanitize |
| **로그 마스킹** | 없음 | 민감 정보 자동 마스킹 |

```python
# 필요한 개선 예시: 비밀번호 하드코딩 제거
# Before (현재)
C4C_AUTH_PASSWORD = "Welcome1!@Welcome1!@..."

# After (개선)
C4C_AUTH_PASSWORD = os.environ.get('C4C_AUTH_PASSWORD') or get_secret('c4c-password')
```

---

## 📌 부록: 빠른 참조

### 자주 사용하는 명령어

```bash
# 로컬 실행
python main.py

# Cloud Foundry 배포
cf push -f manifest.yml

# 로그 확인
cf logs cxagent --recent

# 앱 상태 확인
cf app cxagent

# 환경 변수 확인
cf env cxagent
```

### 트러블슈팅 체크리스트

- [ ] FFmpeg 설치 확인 (`/test` 엔드포인트)
- [ ] AICORE 서비스 바인딩 확인
- [ ] Destination 서비스 연결 확인
- [ ] 메모리 사용량 확인 (4GB 권장)
- [ ] 임시 파일 누적 여부 확인

### 연락처

> 추가 명세 필요: 담당자 연락처 정보는 코드에 포함되어 있지 않습니다.

---

*이 문서는 코드 분석을 기반으로 자동 생성되었습니다. 운영 환경 특화 정보나 비즈니스 컨텍스트가 필요한 경우 기존 담당자에게 문의하세요.*
