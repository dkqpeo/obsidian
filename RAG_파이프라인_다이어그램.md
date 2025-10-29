# RAG 파이프라인 다이어그램

**프로젝트**: EduMentor AI
**목적**: 팀별 RAG 파이프라인 상세 구조 및 데이터 흐름
**작성일**: 2025-10-28

---

## 📋 목차

1. [전체 시스템 RAG 아키텍처](#전체-시스템-rag-아키텍처)
2. [팀1 QA 시스템 RAG 파이프라인](#팀1-qa-시스템-rag-파이프라인)
3. [팀2 문제 생성 RAG 파이프라인](#팀2-문제-생성-rag-파이프라인)
4. [성능 최적화 전략](#성능-최적화-전략)

---

## 🏗️ 전체 시스템 RAG 아키텍처

### 시스템 전체 흐름 (최적화된 파일 업로드)

```mermaid
graph TB
    subgraph "사용자 인터페이스"
        A[Frontend] -->|PDF 업로드| B[Spring Boot Backend]
        A -->|질문| B
        A -->|문제 생성 요청| B
    end

    subgraph "Spring Boot Backend - Port 8080"
        B -->|1. 파일 저장| SV[공유 볼륨<br/>/app/shared/uploads]
        B -->|2. 메타데이터 저장| C[PostgreSQL]
        B -->|3. 파일 경로만 전달| D[Python AI Engine]
    end

    subgraph "Python AI Engine - Port 8000"
        D -->|경로에서 파일 읽기| SV
        D -->|/qa/upload| E1[자료 파싱]
        D -->|/qa/ask| E2[QA RAG]
        D -->|/problems/generate| E3[문제 생성 RAG]
    end

    subgraph "RAG 인프라"
        E1 -->|임베딩 저장| F1[ChromaDB]
        E2 -->|벡터 검색| F1
        E3 -->|벡터 검색| F1
        E2 -->|LLM 호출| F2[Upstage Solar]
        E3 -->|LLM 호출| F2
    end

    style B fill:#e1f5ff
    style SV fill:#fff4e1
    style E1 fill:#ffe1e1
    style E2 fill:#e1ffe1
    style E3 fill:#e1f5ff
    style F1 fill:#f0e1ff
    style F2 fill:#ffe1f0
```

**핵심 최적화**:
- ✨ 파일은 Frontend → Spring Boot로 1회만 전송
- ✨ Spring Boot가 공유 볼륨에 저장
- ✨ Python은 파일 경로만 받아 직접 읽기 (네트워크 전송 X)

---

## 📚 팀1: QA 시스템 RAG 파이프라인

### 1. 자료 업로드 및 파싱 파이프라인 (최적화)

```mermaid
graph LR
    subgraph "1. 업로드 단계 - Spring Boot"
        A[Frontend<br/>PDF 파일] -->|multipart/form-data| B[Spring Boot<br/>/api/materials/upload]
        B -->|파일 검증| B1[파일 검증<br/>크기/확장자]
        B1 -->|저장| C[공유 볼륨<br/>/app/shared/uploads]
        B -->|메타데이터| D[PostgreSQL<br/>PENDING]
    end

    subgraph "2. 파싱 요청 - Spring → Python"
        B -->|파일 경로 전달| E[Python<br/>/qa/upload]
        E -->|경로에서 읽기| C
    end

    subgraph "3. 파싱 단계 - Python"
        E --> F[Upstage Document<br/>Parse API]
        F --> G[구조화된 텍스트<br/>+ 페이지 정보]
    end

    subgraph "4. 청킹 단계"
        G --> H[RecursiveCharacter<br/>TextSplitter]
        H -->|chunk_size=500<br/>overlap=50| I[텍스트 청크 생성]
    end

    subgraph "5. 임베딩 단계"
        I --> J[Upstage Embeddings<br/>API]
        J -->|solar-embedding<br/>-1-large| K[1024-dim 벡터]
    end

    subgraph "6. 저장 단계"
        K --> L[ChromaDB<br/>Collection]
        L -->|metadata: page,<br/>material_id| M[(Vector Store)]
        E -->|완료 응답| D
        D -->|COMPLETED| N[PostgreSQL<br/>상태 업데이트]
    end

    style B fill:#e1f5ff
    style C fill:#fff4e1
    style F fill:#ffe1e1
    style J fill:#f0e1ff
    style M fill:#e1ffe1
```

**주요 구성 요소**:

| 단계 | 담당 | 도구 | 설정 | 목적 |
|------|------|------|------|------|
| **파일 수신** | Spring Boot | FileStorageService | 50MB 제한, PDF만 | 파일 검증 및 저장 |
| **경로 전달** | Spring Boot | WebClient | 파일 경로만 | 네트워크 최적화 |
| **파싱** | Python | Upstage Document Parse | API v1 | PDF 구조 분석 |
| **청킹** | Python | RecursiveCharacterTextSplitter | size=500, overlap=50 | 의미 단위 분할 |
| **임베딩** | Python | Upstage Embeddings | solar-embedding-1-large | 1024차원 벡터화 |
| **저장** | Python | ChromaDB | cosine similarity | 벡터 검색 |

### 1-1. 자료 업로드 상세 데이터 흐름 (Sequence Diagram)

```mermaid
sequenceDiagram
    participant User as 사용자
    participant FE as Frontend
    participant Spring as Spring Boot
    participant Vol as 공유 볼륨
    participant Py as Python /qa/upload
    participant Upstage as Upstage API
    participant Chroma as ChromaDB
    participant DB as PostgreSQL

    User->>FE: PDF 파일 선택
    FE->>Spring: POST /api/materials/upload<br/>(multipart/form-data)

    Note over Spring: 1. 파일 검증
    Spring->>Spring: validateFile()<br/>- 크기 < 50MB<br/>- 확장자 = pdf<br/>- 경로 조작 방지

    Note over Spring: 2. 파일 저장
    Spring->>Vol: 파일 저장<br/>/app/shared/uploads/uuid.pdf
    Spring->>DB: INSERT Material<br/>status=PENDING

    Note over Spring: 3. Python에 경로만 전달
    Spring->>Py: POST /qa/upload<br/>{material_id, file_path}

    Note over Py: 4. 파일 읽기
    Py->>Vol: open(file_path, 'rb')
    Vol-->>Py: PDF 파일 데이터

    Note over Py: 5. Upstage Parse
    Py->>Upstage: Document Parse API
    Upstage-->>Py: parsed_content<br/>(텍스트 + 페이지 정보)

    Note over Py: 6. 청킹
    Py->>Py: RecursiveCharacterTextSplitter<br/>chunk_size=500, overlap=50

    Note over Py: 7. 임베딩
    Py->>Upstage: Embeddings API<br/>(chunks)
    Upstage-->>Py: embeddings [1024-dim]

    Note over Py: 8. ChromaDB 저장
    Py->>Chroma: add_documents()<br/>(texts, embeddings, metadata)
    Chroma-->>Py: 저장 완료

    Note over Py: 9. 완료 응답
    Py-->>Spring: {status: "completed",<br/>page_count, chunk_count}

    Note over Spring: 10. 상태 업데이트
    Spring->>DB: UPDATE Material<br/>status=COMPLETED
    Spring-->>FE: 업로드 성공 응답
    FE-->>User: "업로드 완료!"
```

**성능 지표**:
- **파일 전송**: Frontend → Spring Boot (3초, 10MB 기준)
- **파일 저장**: Spring Boot → 공유 볼륨 (< 0.5초)
- **경로 전달**: Spring Boot → Python (< 0.1초) ✨ 최적화!
- **파싱**: Upstage Document Parse (3~5초)
- **청킹 + 임베딩**: 2~3초
- **ChromaDB 저장**: 1~2초
- **총 소요 시간**: 약 8~10초 (50MB PDF 기준)

**기존 대비 개선**:
- 파일 전송 2회 → 1회 (27% 속도 향상)
- 네트워크 부하 50% 감소

### 2. QA RAG 파이프라인 (1-2초 응답)

```mermaid
graph TB
    subgraph "입력 단계"
        A[사용자 질문] --> B[Question Preprocessing]
        B -->|불용어 제거| B1[정제된 질문]
    end

    subgraph "Retrieve 단계 - 0.2~0.3초"
        B1 --> C[Query Embedding]
        C -->|Upstage Embeddings| D[Query Vector]
        D --> E[ChromaDB 검색]
        E -->|k=3, cosine similarity| F[Top-3 청크 검색]
        F --> G{유사도 체크}
        G -->|score > 0.7| H[관련 컨텍스트]
        G -->|score ≤ 0.7| I[컨텍스트 부족 처리]
    end

    subgraph "Generate 단계 - 0.8~1.0초"
        H --> J[Prompt Template]
        I --> J
        J --> K[LangChain LCEL Chain]
        K --> L[Upstage Solar Chat]
        L -->|solar-1-mini-chat| M[답변 생성]
    end

    subgraph "후처리 단계"
        M --> N[출처 정보 추가]
        N --> O[응답 포맷팅]
        O --> P{응답 품질 검증}
        P -->|품질 OK| Q[최종 답변 + 출처]
        P -->|품질 불량| R[재생성 또는 안내]
    end

    style C fill:#ffe1e1
    style E fill:#fff4e1
    style L fill:#f0e1ff
    style Q fill:#e1ffe1
```

### 3. QA RAG 상세 데이터 흐름

```mermaid
sequenceDiagram
    participant User as 사용자
    participant FE as Frontend
    participant Spring as Spring Boot
    participant Py as Python /qa/ask
    participant Workflow as QA Workflow
    participant Embed as Upstage Embeddings
    participant Chroma as ChromaDB
    participant LLM as Upstage Solar Chat
    participant Cache as Response Cache
    participant DB as PostgreSQL

    User->>FE: 질문 입력
    FE->>Spring: POST /api/qa/ask<br/>{material_id, question}
    Spring->>Py: POST /qa/ask<br/>{material_id, question}
    Py->>Cache: 캐시 조회 (질문 해시)

    alt 캐시 Hit
        Cache-->>Py: 캐싱된 답변 반환
        Py-->>Spring: 즉시 응답 (< 0.1초)
        Spring->>DB: QA 세션 저장
        Spring-->>FE: 응답 전달
        FE-->>User: 답변 표시
    else 캐시 Miss
        Py->>Workflow: invoke({question, material_id})

        Note over Workflow: State: RetrieveState
        Workflow->>Embed: 질문 임베딩 요청
        Embed-->>Workflow: query_vector [1024]

        Workflow->>Chroma: similarity_search(vector, k=3)
        Chroma-->>Workflow: Top-3 청크 + 메타데이터

        Note over Workflow,Chroma: ✅ Retrieve 완료 (0.2~0.3초)

        Note over Workflow: State: GenerateState
        Workflow->>Workflow: build_prompt(question, context)

        Workflow->>LLM: invoke(prompt)
        LLM-->>Workflow: answer + reasoning

        Note over Workflow,LLM: ✅ Generate 완료 (0.8~1.0초)

        Workflow->>Workflow: format_response(answer, sources)
        Workflow-->>Py: {answer, sources, time_ms}

        Py->>Cache: 결과 캐싱
        Py-->>Spring: 최종 응답 (1~2초)
        Spring->>DB: QA 세션 저장
        Spring-->>FE: 응답 전달
        FE-->>User: 답변 + 출처 표시
    end
```

**성능 지표**:
- **Retrieve**: 0.2~0.3초 (ChromaDB 벡터 검색)
- **Generate**: 0.8~1.0초 (LLM 응답 생성)
- **Total**: 1.0~1.3초 (캐싱 없을 때)
- **Cached**: < 0.1초 (동일 질문 반복 시)

### 4. QA 프롬프트 템플릿

```python
QA_PROMPT_TEMPLATE = """
당신은 학습 자료 기반 QA 전문가입니다.

**컨텍스트**:
{context}

**질문**: {question}

**지침**:
1. 제공된 컨텍스트만을 사용하여 답변하세요
2. 컨텍스트에 없는 내용은 "제공된 자료에서 찾을 수 없습니다"라고 명확히 하세요
3. 답변은 명확하고 구체적으로 작성하세요
4. 가능하면 예제나 코드를 포함하세요
5. 출처를 명시하세요 (페이지 번호 등)

**답변**:
"""
```

---

## 🎯 팀2: 문제 생성 RAG 파이프라인

### 1. 문제 생성 전체 파이프라인

```mermaid
graph TB
    subgraph "입력 단계"
        A[문제 생성 요청] -->|material_id, difficulty, count| B[Request Validation]
    end

    subgraph "학습 내용 분석 - Retrieve 단계"
        B --> C[학습 자료 로드]
        C --> D[ChromaDB 전체 검색]
        D -->|k=10| E[학습 내용 청크들]
        E --> F[내용 분석 및 주제 추출]
        F --> G{주제 충분?}
        G -->|Yes| H[주제 리스트]
        G -->|No| I[추가 검색]
        I --> D
    end

    subgraph "문제 생성 - Generate 단계"
        H --> J{난이도 라우팅}
        J -->|BEGINNER| K1[초급 Generator]
        J -->|INTERMEDIATE| K2[중급 Generator]
        J -->|ADVANCED| K3[고급 Generator]

        K1 --> L1[문제 초안 생성]
        K2 --> L2[문제 초안 생성]
        K3 --> L3[문제 초안 생성]
    end

    subgraph "검증 및 재생성"
        L1 & L2 & L3 --> M[문제 검증]
        M --> N{검증 통과?}
        N -->|Pass| O[최종 문제]
        N -->|Fail| P[재생성 요청]
        P --> J
        O --> Q[힌트 생성]
        Q --> R[테스트 케이스 생성]
    end

    subgraph "후처리"
        R --> S[난이도 점수 계산]
        S --> T[메타데이터 추가]
        T --> U[최종 문제 세트]
    end

    style D fill:#fff4e1
    style K1 fill:#e1f5ff
    style K2 fill:#ffe1e1
    style K3 fill:#f0e1ff
    style M fill:#ffe1e1
    style U fill:#e1ffe1
```

### 2. 난이도별 Generator 상세

```mermaid
graph LR
    subgraph "초급 Generator"
        A1[학습 내용] --> B1[개념 정의 문제]
        A1 --> B2[코드 읽기 문제]
        A1 --> B3[빈칸 채우기]
        B1 & B2 & B3 --> C1[기본 문제 세트]
    end

    subgraph "중급 Generator"
        A2[학습 내용] --> D1[코드 작성 문제]
        A2 --> D2[디버깅 문제]
        A2 --> D3[설계 문제]
        D1 & D2 & D3 --> C2[중급 문제 세트]
    end

    subgraph "고급 Generator"
        A3[학습 내용] --> E1[시스템 설계]
        A3 --> E2[최적화 문제]
        A3 --> E3[복합 시나리오]
        E1 & E2 & E3 --> C3[고급 문제 세트]
    end

    style C1 fill:#e1f5ff
    style C2 fill:#ffe1e1
    style C3 fill:#f0e1ff
```

### 3. 문제 생성 Workflow (LangGraph)

```mermaid
sequenceDiagram
    participant User as 사용자
    participant FE as Frontend
    participant Spring as Spring Boot
    participant Py as Python /problems/generate
    participant Workflow as Problem Workflow
    participant Chroma as ChromaDB
    participant LLM as Upstage Solar
    participant Validator as 문제 검증기
    participant DB as PostgreSQL

    User->>FE: 문제 생성 요청
    FE->>Spring: POST /api/problems/generate<br/>{material_id, difficulty, count}
    Spring->>Py: POST /problems/generate<br/>{material_id, difficulty, count}
    Py->>Workflow: invoke(request)

    Note over Workflow: State: AnalyzeState
    Workflow->>Chroma: get_all_chunks(material_id)
    Chroma-->>Workflow: 전체 학습 내용

    Workflow->>LLM: 주제 추출 요청
    LLM-->>Workflow: 주제 리스트 [5~10개]

    loop count만큼 반복
        Note over Workflow: State: GenerateState
        Workflow->>Workflow: 난이도 확인

        alt BEGINNER
            Workflow->>LLM: 초급 문제 생성 프롬프트
        else INTERMEDIATE
            Workflow->>LLM: 중급 문제 생성 프롬프트
        else ADVANCED
            Workflow->>LLM: 고급 문제 생성 프롬프트
        end

        LLM-->>Workflow: 문제 초안

        Note over Workflow: State: ValidateState
        Workflow->>Validator: validate(problem)

        alt 검증 통과
            Validator-->>Workflow: ✅ PASS
            Workflow->>Workflow: add_to_final_set
        else 검증 실패
            Validator-->>Workflow: ❌ FAIL (이유)
            Note over Workflow: 재생성 (최대 3회)
        end
    end

    Note over Workflow: State: EnrichState
    Workflow->>LLM: 힌트 생성
    LLM-->>Workflow: 힌트 리스트

    Workflow->>LLM: 테스트 케이스 생성
    LLM-->>Workflow: 입출력 예제

    Workflow->>Workflow: calculate_difficulty_score
    Workflow-->>Py: 최종 문제 세트
    Py-->>Spring: {problems, metadata}
    Spring->>DB: 문제 저장
    Spring-->>FE: 응답 전달
    FE-->>User: 생성된 문제 표시
```

### 4. 문제 검증 로직

```mermaid
graph TB
    A[생성된 문제] --> B{1. 길이 검사}
    B -->|question > 20자| C{2. 구조 검사}
    B -->|question ≤ 20자| FAIL1[❌ 문제가 너무 짧음]

    C -->|필수 필드 존재| D{3. 난이도 점수}
    C -->|필수 필드 누락| FAIL2[❌ 구조 불완전]

    D{난이도별 점수 검증}
    D -->|BEGINNER: 1-3| E{4. 중복 검사}
    D -->|INTERMEDIATE: 4-6| E
    D -->|ADVANCED: 7-10| E
    D -->|범위 벗어남| FAIL3[❌ 난이도 불일치]

    E -->|기존 문제와 다름| F{5. 품질 검사}
    E -->|유사 문제 존재| FAIL4[❌ 중복 문제]

    F{답안 존재 & 힌트 적절}
    F -->|모두 충족| PASS[✅ 검증 통과]
    F -->|조건 미충족| FAIL5[❌ 품질 기준 미달]

    style PASS fill:#e1ffe1
    style FAIL1 fill:#ffe1e1
    style FAIL2 fill:#ffe1e1
    style FAIL3 fill:#ffe1e1
    style FAIL4 fill:#ffe1e1
    style FAIL5 fill:#ffe1e1
```

**검증 기준**:

| 검증 항목 | 기준 | 실패 시 처리 |
|-----------|------|--------------|
| **최소 길이** | question > 20자 | 재생성 |
| **필수 필드** | question, answer, hints 존재 | 재생성 |
| **난이도 점수** | 초급:1-3, 중급:4-6, 고급:7-10 | 재생성 |
| **중복 검사** | 기존 문제와 유사도 < 0.8 | 재생성 |
| **품질 검사** | 답안 완성도, 힌트 유용성 | 재생성 |

### 5. 난이도별 프롬프트 템플릿

#### 초급 (BEGINNER)

```python
BEGINNER_PROMPT = """
**학습 내용**: {context}
**주제**: {topic}

초급 학습자를 위한 기본 개념 확인 문제를 생성하세요.

**문제 유형**:
1. 개념 정의 (이 용어의 의미는?)
2. 코드 읽기 (이 코드의 실행 결과는?)
3. 빈칸 채우기 (다음 코드의 빈칸을 채우세요)

**요구사항**:
- 난이도 점수: 1-3
- 학습 내용에 직접 명시된 내용만 사용
- 명확한 정답 존재
- 3개의 힌트 제공
- 간단한 코드 예제 포함 (선택)

출력 형식: JSON
{
  "question": "문제 내용",
  "answer": "정답",
  "hints": ["힌트1", "힌트2", "힌트3"],
  "difficulty_score": 2,
  "problem_type": "CONCEPT"
}
"""
```

#### 중급 (INTERMEDIATE)

```python
INTERMEDIATE_PROMPT = """
**학습 내용**: {context}
**주제**: {topic}

중급 학습자를 위한 응용 문제를 생성하세요.

**문제 유형**:
1. 코드 작성 (특정 기능 구현)
2. 디버깅 (오류가 있는 코드 수정)
3. 설계 (간단한 시스템 설계)

**요구사항**:
- 난이도 점수: 4-6
- 학습 내용을 바탕으로 응용
- 여러 개념 통합
- 단계별 힌트 제공
- 테스트 케이스 포함

출력 형식: JSON
{
  "question": "문제 내용 (상황 설명)",
  "answer": "완전한 답안 코드",
  "hints": ["힌트1", "힌트2", "힌트3", "힌트4"],
  "difficulty_score": 5,
  "problem_type": "CODING",
  "test_cases": [
    {"input": "...", "expected": "..."},
    {"input": "...", "expected": "..."}
  ]
}
"""
```

#### 고급 (ADVANCED)

```python
ADVANCED_PROMPT = """
**학습 내용**: {context}
**주제**: {topic}

고급 학습자를 위한 심화 문제를 생성하세요.

**문제 유형**:
1. 시스템 설계 (아키텍처 설계)
2. 최적화 (성능 개선)
3. 복합 시나리오 (실무 상황 해결)

**요구사항**:
- 난이도 점수: 7-10
- 여러 주제 통합
- 설계 및 구현 모두 요구
- Trade-off 고려 필요
- 확장성 및 성능 고려
- 상세한 테스트 케이스

출력 형식: JSON
{
  "question": "복합 시나리오 문제",
  "answer": "설계 + 구현 + 설명",
  "hints": ["힌트1", "힌트2", "힌트3", "힌트4", "힌트5"],
  "difficulty_score": 8,
  "problem_type": "SYSTEM_DESIGN",
  "test_cases": [
    {"scenario": "...", "expected": "..."},
    {"scenario": "...", "expected": "..."}
  ],
  "evaluation_criteria": ["확장성", "성능", "코드 품질"]
}
"""
```

---

## ⚡ 성능 최적화 전략

### 1. QA 시스템 최적화

```mermaid
graph LR
    subgraph "검색 최적화"
        A1[Query Cache] --> B1[중복 질문 제거]
        A2[Vector Cache] --> B2[임베딩 재사용]
        A3[Index Optimization] --> B3[ChromaDB HNSW]
    end

    subgraph "생성 최적화"
        C1[Prompt Caching] --> D1[시스템 프롬프트 재사용]
        C2[Model Selection] --> D2[mini-chat 사용]
        C3[Context Window] --> D3[최소 컨텍스트]
    end

    subgraph "전체 최적화"
        E1[병렬 처리] --> F1[임베딩 + 검색 동시]
        E2[Batch Processing] --> F2[여러 질문 일괄]
        E3[Connection Pool] --> F3[API 연결 재사용]
    end

    style B1 fill:#e1f5ff
    style B3 fill:#fff4e1
    style D2 fill:#f0e1ff
    style F1 fill:#e1ffe1
```

**목표 성능**:
- **QA 응답**: < 2초 (95 percentile)
- **문제 생성**: < 30초 (3문제 기준)
- **자료 파싱**: < 10초 (10페이지 PDF)

### 2. 문제 생성 최적화

| 최적화 기법 | 방법 | 효과 |
|-------------|------|------|
| **병렬 생성** | 여러 문제 동시 생성 | 3배 속도 향상 |
| **재생성 제한** | 최대 3회 재시도 | 무한 루프 방지 |
| **주제 캐싱** | 학습 내용 분석 재사용 | 50% 시간 절약 |
| **템플릿 최적화** | 명확한 지시사항 | 재생성률 감소 |

### 3. ChromaDB 최적화

```yaml
# chromadb 설정
collection_metadata:
  hnsw:space: "cosine"
  hnsw:construction_ef: 200
  hnsw:search_ef: 100
  hnsw:M: 16

# 인덱싱 전략
- 자료별 별도 컬렉션
- 주기적 인덱스 재구성
- 메모리 캐싱 활성화
```

---

## 📊 데이터 흐름 요약

### 자료 업로드 시스템 (최적화)

```
사용자 (PDF 선택)
    ↓
Frontend (파일 전송)
    ↓
Spring Boot (파일 검증 + 공유 볼륨 저장) - 3초
    ↓
Spring Boot → Python (파일 경로만 전달) - < 0.1초 ✨
    ↓
Python (경로에서 파일 읽기)
    ↓
Upstage Document Parse - 3~5초
    ↓
청킹 + 임베딩 - 2~3초
    ↓
ChromaDB 저장 - 1~2초
    ↓
PostgreSQL 상태 업데이트
    ↓
사용자 (업로드 완료, 총 8~10초)
```

### QA 시스템

```
사용자 질문
    ↓
Frontend → Spring Boot → Python - < 0.1초
    ↓
질문 임베딩 (0.1초)
    ↓
ChromaDB 검색 (0.2초)
    ↓
프롬프트 구성 (0.1초)
    ↓
LLM 생성 (0.8초)
    ↓
Python → Spring Boot (DB 저장) → Frontend
    ↓
답변 + 출처 (총 1.2초)
```

### 문제 생성 시스템

```
생성 요청
    ↓
Frontend → Spring Boot → Python
    ↓
학습 내용 로드 (1초)
    ↓
주제 분석 (3초)
    ↓
문제 생성 (각 5초 × 3 = 15초)
    ↓
검증 + 힌트 생성 (5초)
    ↓
Python → Spring Boot (DB 저장) → Frontend
    ↓
최종 문제 세트 (총 24초)
```

---

## 🔗 관련 문서

- [팀1_QA시스템_구현가이드.md](./팀1_QA시스템_구현가이드.md)
- [팀2_문제생성_구현가이드.md](./팀2_문제생성_구현가이드.md)
- [전체_시스템_아키텍처.md](./전체_시스템_아키텍처.md)
- [Python_통합_구현가이드.md](./Python_통합_구현가이드.md)
- [파일업로드_최적화_구현가이드.md](OLD_파일업로드_최적화_구현가이드.md) - ✨ 파일 업로드 최적화 상세

---

**작성일**: 2025-10-28
**최종 수정**: 2025-10-28 (Spring Boot 플로우 반영, PPT 제거, 최적화 추가)
