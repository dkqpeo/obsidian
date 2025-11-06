# EduMentor AI - 전체 시스템 아키텍처

Python FastAPI 서버의 4가지 LangGraph 워크플로우를 포함한 전체 시스템 흐름도

---

## 시스템 전체 아키텍처

```mermaid
graph TB
    subgraph "Client Layer"
        USER[👤 사용자]
        ADMIN[👨‍💼 관리자]
    end

    subgraph "Spring Boot Backend"
        SB[Spring Boot Server<br/>포트: 8080]
        SB_AUTH[인증/권한]
        SB_DB[(MySQL<br/>사용자/학습자료<br/>문제/답변)]
    end

    subgraph "Python FastAPI Server (포트: 8000)"
        direction TB

        subgraph "1️⃣ Upload Pipeline"
            U_START([START])
            U_PARSE[Parse Node<br/>문서 파싱]
            U_EMBED[Embed & Store Node<br/>임베딩 + 저장]
            U_END([END])

            U_START --> U_PARSE
            U_PARSE --> U_EMBED
            U_EMBED --> U_END
        end

        subgraph "2️⃣ QA Pipeline"
            Q_START([START])
            Q_RETRIEVE[Retrieve Node<br/>문서 검색]
            Q_DECISION{관련 문서<br/>있음?}
            Q_FALLBACK[Fallback Node<br/>안내 메시지]
            Q_GENERATE[Generate Node<br/>답변 생성]
            Q_VERIFY[Verify Node<br/>품질 검증]
            Q_END([END])

            Q_START --> Q_RETRIEVE
            Q_RETRIEVE --> Q_DECISION
            Q_DECISION -->|No| Q_FALLBACK
            Q_DECISION -->|Yes| Q_GENERATE
            Q_FALLBACK --> Q_END
            Q_GENERATE --> Q_VERIFY
            Q_VERIFY --> Q_END
        end

        subgraph "3️⃣ Problem Generation Pipeline"
            P_START([START])
            P_ANALYZE[Analyze Node<br/>내용 분석]
            P_BUILD[Build Context Node<br/>컨텍스트 구성]
            P_REFINE[Refine Context Node<br/>컨텍스트 보강]
            P_GENERATE[Generate Node<br/>문제 생성]
            P_VALIDATE[Validate Node<br/>문제 검증]
            P_DECISION{충분한<br/>문제?}
            P_END([END])

            P_START --> P_ANALYZE
            P_ANALYZE --> P_BUILD
            P_BUILD --> P_GENERATE
            P_GENERATE --> P_VALIDATE
            P_VALIDATE --> P_DECISION
            P_DECISION -->|No| P_REFINE
            P_DECISION -->|Yes| P_END
            P_REFINE --> P_GENERATE
        end

        subgraph "4️⃣ Grading Pipeline"
            G_START([START])
            G_VALIDATE[Validate Input Node<br/>입력 검증]
            G_GRADE[Grade Node<br/>채점 실행]
            G_VERIFY[Verify Node<br/>결과 검증]
            G_DECISION{신뢰도<br/>충분?}
            G_END([END])

            G_START --> G_VALIDATE
            G_VALIDATE --> G_GRADE
            G_GRADE --> G_VERIFY
            G_VERIFY --> G_DECISION
            G_DECISION -->|No| G_GRADE
            G_DECISION -->|Yes| G_END
        end
    end

    subgraph "External Services"
        UPSTAGE[☀️ Upstage API<br/>LLM + Embedding]
        CHROMA[(🗄️ ChromaDB<br/>Vector Store)]
    end

    %% User Flow
    USER -->|1. 파일 업로드| SB
    USER -->|2. 질문하기| SB
    USER -->|4. 답변 제출| SB
    ADMIN -->|3. 문제 생성| SB

    %% Spring Boot to FastAPI
    SB -->|POST /qa/upload| U_START
    SB -->|POST /qa/ask| Q_START
    SB -->|POST /problems/generate| P_START
    SB -->|POST /problems/check-answer| G_START

    %% FastAPI to External Services
    U_PARSE --> UPSTAGE
    U_EMBED --> UPSTAGE
    U_EMBED --> CHROMA

    Q_RETRIEVE --> CHROMA
    Q_RETRIEVE --> UPSTAGE
    Q_GENERATE --> UPSTAGE

    P_ANALYZE --> CHROMA
    P_ANALYZE --> UPSTAGE
    P_GENERATE --> UPSTAGE

    G_GRADE --> UPSTAGE

    %% Spring Boot DB
    SB --> SB_AUTH
    SB --> SB_DB

    %% Styling
    style USER fill:#e1f5ff
    style ADMIN fill:#fff4e1
    style SB fill:#ffe5e5
    style UPSTAGE fill:#ffe5e5
    style CHROMA fill:#e5f5e5
```

---

## 워크플로우별 상세 흐름

### 1️⃣ Upload Pipeline (파일 업로드 → 벡터 저장)

```mermaid
sequenceDiagram
    participant User as 👤 사용자
    participant SB as Spring Boot
    participant API as FastAPI
    participant Parse as Parse Node
    participant Embed as Embed & Store Node
    participant UP as ☀️ Upstage
    participant DB as 🗄️ ChromaDB

    User->>SB: 파일 업로드 (PDF/PPT)
    SB->>SB: 공유 볼륨에 저장
    SB->>API: POST /qa/upload<br/>{material_id, file_path}

    API->>Parse: 문서 파싱 시작
    Parse->>UP: Document Parse API
    UP-->>Parse: 파싱 결과 (블록)

    Parse->>Embed: parsed_blocks
    Embed->>Embed: 청킹 (500자)
    Embed->>UP: 임베딩 생성 (배치)
    UP-->>Embed: embeddings (4096차원)

    Embed->>DB: 배치 저장 (100개씩)
    DB-->>Embed: 저장 완료

    Embed-->>API: status: completed
    API-->>SB: {page_count, chunk_count}
    SB-->>User: 업로드 성공
```

**소요 시간**: 5~10분 (문서 크기 의존)

---

### 2️⃣ QA Pipeline (질문 → 답변 생성)

```mermaid
sequenceDiagram
    participant User as 👤 사용자
    participant SB as Spring Boot
    participant API as FastAPI
    participant Ret as Retrieve Node
    participant Dec as Decision
    participant Fall as Fallback Node
    participant Gen as Generate Node
    participant Ver as Verify Node
    participant UP as ☀️ Upstage
    participant DB as 🗄️ ChromaDB

    User->>SB: 질문 입력
    SB->>API: POST /qa/ask<br/>{material_id, question}

    API->>Ret: 문서 검색 시작
    Ret->>UP: 질문 임베딩
    UP-->>Ret: query_embedding
    Ret->>DB: 검색 (k=5)
    DB-->>Ret: retrieved_docs + distance

    Ret->>Dec: has_relevant_docs?

    alt 관련 문서 없음 (distance >= 0.5)
        Dec->>Fall: Fallback 응답
        Fall-->>API: "관련 내용을 찾을 수 없습니다"
    else 관련 문서 있음
        Dec->>Gen: 답변 생성
        Gen->>UP: Solar LLM (context + question)
        UP-->>Gen: answer
        Gen->>Ver: 품질 검증
        Ver->>Ver: 길이/키워드 체크
        Ver-->>API: answer + quality
    end

    API-->>SB: {answer, sources, response_time_ms}
    SB-->>User: 답변 표시
```

**소요 시간**: 1.0~1.5초 (normal) / 0.2~0.3초 (fallback)

---

### 3️⃣ Problem Generation Pipeline (문제 자동 생성)

```mermaid
sequenceDiagram
    participant Admin as 👨‍💼 관리자
    participant SB as Spring Boot
    participant API as FastAPI
    participant Ana as Analyze Node
    participant Build as Build Context Node
    participant Gen as Generate Node
    participant Val as Validate Node
    participant Dec as Decision
    participant Ref as Refine Context Node
    participant UP as ☀️ Upstage
    participant DB as 🗄️ ChromaDB

    Admin->>SB: 문제 생성 요청
    SB->>API: POST /problems/generate<br/>{material_id, difficulty, count}

    API->>Ana: 학습 내용 분석
    Ana->>UP: 토픽 추출 (필요시)
    UP-->>Ana: learning_topics
    Ana->>DB: 토픽별 문서 검색
    DB-->>Ana: documents_by_topic

    Ana->>Build: 컨텍스트 구성
    Build->>Build: 라운드 로빈 균등 분배
    Build->>Gen: context (3000 토큰)

    loop 재생성 (최대 5회)
        Gen->>UP: Solar LLM 문제 생성
        UP-->>Gen: generated_problems
        Gen->>Val: 문제 검증
        Val->>Val: 난이도/필드/중복 체크
        Val->>Dec: validated + rejected

        alt 문제 부족
            Dec->>Ref: 컨텍스트 보강
            Ref->>Ref: 새 문서 추가 (4000 토큰)
            Ref->>Gen: enhanced_context + needed_count
        else 충분
            Dec-->>API: validated_problems
        end
    end

    API-->>SB: {problems, generated_count, rejected_count}
    SB-->>Admin: 생성된 문제 표시
```

**소요 시간**: 13~20초 (재생성 시 +7초/회)

---

### 4️⃣ Grading Pipeline (답변 채점)

```mermaid
sequenceDiagram
    participant User as 👤 사용자
    participant SB as Spring Boot
    participant API as FastAPI
    participant ValIn as Validate Input Node
    participant Grade as Grade Node
    participant Ver as Verify Node
    participant Dec as Decision
    participant Short as ShortAnswerGrader
    participant Code as CodingGrader
    participant UP as ☀️ Upstage

    User->>SB: 답변 제출
    SB->>API: POST /problems/check-answer<br/>{problem, user_answer}

    API->>ValIn: 입력 검증

    alt 빈 답변 또는 잘못된 타입
        ValIn-->>API: score=0, is_verified=True
    else 정상 입력
        ValIn->>Grade: 채점 시작

        alt SHORT_ANSWER
            Grade->>Short: LLM 채점
            Short->>UP: 의미적 유사도 평가
            UP-->>Short: similarity_score
            Short-->>Grade: grading_result + confidence
        else CODING
            Grade->>Code: 코드 실행
            Code->>Code: 테스트 케이스 검증
            Code-->>Grade: grading_result + confidence
        end

        Grade->>Ver: 결과 검증
        Ver->>Dec: confidence >= 0.3?

        alt 신뢰도 낮음 & retry < 2
            Dec->>Grade: 재채점
        else 신뢰도 충분 또는 최대 재시도
            Dec-->>API: grading_result
        end
    end

    API-->>SB: {score, is_correct, feedback, confidence_score}
    SB-->>User: 채점 결과 표시
```

**소요 시간**: 0.5~2.5초 (재채점 시 +0.5초)

---

## 데이터 흐름 요약

```mermaid
graph LR
    subgraph "데이터 저장"
        A[파일 업로드] --> B[파싱]
        B --> C[청킹]
        C --> D[임베딩]
        D --> E[(ChromaDB)]
    end

    subgraph "데이터 활용"
        E -->|검색| F[QA 답변]
        E -->|검색| G[문제 생성]
    end

    subgraph "학습 평가"
        G --> H[문제 출제]
        H --> I[학생 답변]
        I --> J[채점]
    end

    style E fill:#e5f5e5
    style F fill:#e1f5ff
    style G fill:#fff4e1
    style J fill:#ffe5e5
```

---

## 시스템 구성 요소

### Python FastAPI Server

| 모듈 | 역할 | 주요 파일 |
|------|------|----------|
| **paper_qa** | QA 시스템 | `workflow.py`, `api.py` |
| **paper_problem** | 문제 생성/채점 | `workflow.py`, `grading_workflow.py`, `api.py` |
| **shared** | 공유 클라이언트 | `upstage_client.py`, `chroma_client.py` |

### External Services

| 서비스 | 용도 | 연결 |
|--------|------|------|
| **Upstage API** | LLM + Embedding + 문서파싱 | Solar-1-mini-chat, Embedding API |
| **ChromaDB** | Vector Database | localhost:8001 |
| **Spring Boot** | 백엔드 서버 | localhost:8080 |

---

## API 엔드포인트 맵

```mermaid
graph TB
    subgraph "FastAPI Endpoints"
        E1[POST /qa/upload<br/>파일 업로드]
        E2[POST /qa/ask<br/>질문하기]
        E3[POST /problems/generate<br/>문제 생성]
        E4[POST /problems/check-answer<br/>답변 채점]
        E5[GET /qa/data/:material_id<br/>저장 데이터 조회]
        E6[GET /problems/difficulties<br/>난이도 정보]
    end

    E1 -.-> W1[Upload Workflow]
    E2 -.-> W2[QA Workflow]
    E3 -.-> W3[Problem Generation Workflow]
    E4 -.-> W4[Grading Workflow]

    style E1 fill:#e1f5ff
    style E2 fill:#fff4e1
    style E3 fill:#ffe5e5
    style E4 fill:#e5f5e5
```

---

## 성능 특성

### 응답 시간

| 워크플로우 | 초기 실행 | 재시도 포함 | 병목 구간 |
|-----------|---------|-----------|----------|
| Upload | 5~10분 | N/A | Upstage Document Parse (5~300초) |
| QA | 1.0~1.5초 | N/A | Solar LLM 생성 (0.8초) |
| Problem Generation | 13초 | 13~20초 | Solar LLM 생성 (8초) |
| Grading | 0.7초 | 0.7~2.5초 | LLM 채점 (0.5초) / 코드 실행 (2초) |

### 토큰 사용량 (추정)

| 워크플로우 | Input 토큰 | Output 토큰 | 비고 |
|-----------|-----------|------------|------|
| Upload | ~500K | ~100K | 554페이지 PDF 기준 |
| QA | ~1K | ~200 | 컨텍스트 3000자 + 답변 |
| Problem Generation | ~3K | ~1K | 문제 3개 생성 기준 |
| Grading | ~500 | ~300 | LLM 채점 기준 |

---

## 확장성 및 개선 방향

### 현재 구현
- ✅ 조건부 분기를 통한 품질 관리
- ✅ 재시도 로직으로 안정성 확보
- ✅ 배치 처리로 효율성 향상

### 향후 개선 가능 사항
- 🔄 캐싱 계층 추가 (Redis)
- 🔄 병렬 문제 생성 (여러 난이도 동시)
- 🔄 실시간 피드백 (WebSocket)
- 🔄 A/B 테스트 (프롬프트 최적화)

---

## 모니터링 포인트

```mermaid
graph TD
    subgraph "품질 지표"
        M1[QA 답변 품질<br/>answer_quality]
        M2[문제 생성 성공률<br/>validated/generated]
        M3[채점 신뢰도<br/>confidence_score]
    end

    subgraph "성능 지표"
        M4[응답 시간<br/>response_time_ms]
        M5[재시도 횟수<br/>retry_count]
        M6[ChromaDB 검색 시간<br/>retrieve_time]
    end

    subgraph "비용 지표"
        M7[Upstage API 호출 수]
        M8[토큰 사용량]
        M9[ChromaDB 저장 용량]
    end

    style M1 fill:#e5f5e5
    style M4 fill:#e1f5ff
    style M7 fill:#fff4e1
```
