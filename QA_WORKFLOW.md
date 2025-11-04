# QA System - LangGraph Workflow

RAG 기반 질의응답 시스템의 LangGraph 워크플로우

---

## Upload Workflow

```mermaid
graph TD
    Start([START]) --> Parse[parse_document_node<br/>문서 파싱<br/><b>⏱️ PDF 5~300초</b>]
    Parse --> EmbedStore[embed_and_store_node<br/>임베딩 및 저장<br/><b>⏱️ 청크당 ~0.1초</b>]
    EmbedStore --> End([END])

    style Parse fill:#e1f5ff
    style EmbedStore fill:#fff4e1
```

### 성능 지표

| 단계 | 소요 시간 | 주요 작업 | 예시 (554페이지 PDF) |
|------|---------|----------|---------------------|
| **Parse** | 5~300초 | Upstage Document Parse API | ~300초 |
| **Embed & Store** | 청크당 ~0.1초 | 임베딩 생성 + ChromaDB 저장 | 2,700청크 × 0.1초 = ~270초 |
| **Total** | **문서 크기 의존** | 파싱 + 임베딩 + 저장 | **~570초 (9.5분)** |

**최적화 포인트**:
- 배치 임베딩으로 API 호출 최소화 (텍스트 배열 한 번에 전송)
- ChromaDB 배치 저장 (BATCH_SIZE=100)으로 HTTP 요청 27회로 감소

### State: UploadState

```mermaid
classDiagram
    class UploadState {
        +int material_id
        +str file_path
        +str file_type
        +List~Dict~ parsed_blocks
        +List~List[float]~ embeddings
        +str status
    }
```

### parse_document_node

```mermaid
flowchart LR
    A[파일 입력] --> B{파일 타입}
    B -->|PDF| C[Upstage<br/>Document Parse API]
    B -->|PPT| D[python-pptx<br/>파서]
    C --> E[파싱된 블록]
    D --> E
    E --> F[parsed_blocks<br/>반환]

    style C fill:#ffe5e5
    style D fill:#e5f5ff
```

### embed_and_store_node (⏱️ 청크당 ~0.1초)

```mermaid
flowchart TD
    A[parsed_blocks] --> B[RecursiveCharacterTextSplitter<br/>청킹<br/><i>~1초</i>]
    B --> C[청크 생성<br/>chunk_size=500<br/>overlap=50<br/><i>~2초</i>]
    C --> D[Upstage Embedding API<br/>배치 임베딩<br/><i>~240초</i>]
    D --> E{배치 처리<br/>BATCH_SIZE=100<br/><i>~27초</i>}
    E --> F[ChromaDB 저장<br/>배치 1/27<br/><i>~1초/배치</i>]
    F --> G{더 저장할<br/>배치?}
    G -->|Yes| F
    G -->|No| H[저장 완료<br/>embeddings 반환]

    style B fill:#fff4e1
    style D fill:#ffe5e5
    style E fill:#e5f5e5
```

**세부 처리 단계** (554페이지 PDF 기준):
1. **텍스트 분할** (~3초): 554개 블록을 2,700개 청크로 분할
2. **임베딩 생성** (~240초): Upstage API로 2,700개 청크 → 4096차원 벡터 변환
3. **배치 저장** (~27초): ChromaDB에 100개씩 27회 저장 (페이로드 제한 회피)
4. **Total**: **~270초 (4.5분)**

#### 청킹 상세

```mermaid
graph LR
    A[페이지 1<br/>5000자] --> B[청크 1<br/>500자]
    A --> C[청크 2<br/>500자]
    A --> D[청크 3<br/>500자]
    A --> E[...]
    A --> F[청크 10<br/>500자]

    B -.overlap 50자.-> C
    C -.overlap 50자.-> D

    style A fill:#ffe5e5
    style B fill:#e5f5ff
    style C fill:#e5f5ff
    style D fill:#e5f5ff
    style F fill:#e5f5ff
```

#### 배치 처리 상세

```mermaid
sequenceDiagram
    participant W as Workflow
    participant C as ChromaClient
    participant DB as ChromaDB

    Note over W: 2700개 청크

    loop 배치 1-27
        W->>W: 청크 100개 선택
        W->>C: add_documents(100개)
        C->>DB: HTTP POST (1.76MB)
        DB-->>C: 200 OK
        C-->>W: 저장 완료
    end

    Note over W: 총 2700개 저장 완료
```

---

## QA Workflow

```mermaid
graph TD
    Start([START]) --> Retrieve[retrieve_node<br/>문서 검색<br/><b>⏱️ 0.2~0.3초</b>]
    Retrieve --> Generate[generate_answer_node<br/>답변 생성<br/><b>⏱️ 0.8~1.0초</b>]
    Generate --> End([END])

    style Retrieve fill:#e1f5ff
    style Generate fill:#fff4e1
```

### 성능 지표

| 단계 | 평균 시간 | 주요 작업 |
|------|---------|----------|
| **Retrieve** | 0.2~0.3초 | 질문 임베딩 + ChromaDB 검색 (k=3) |
| **Generate** | 0.8~1.0초 | 컨텍스트 구성 + Solar LLM 응답 생성 |
| **Total** | **1.0~1.3초** | 전체 QA 응답 시간 |

### State: QAState

```mermaid
classDiagram
    class QAState {
        +str question
        +int material_id
        +List~Dict~ retrieved_docs
        +str answer
        +List~Dict~ sources
        +float response_time
    }
```

### retrieve_node (⏱️ 0.2~0.3초)

```mermaid
flowchart TD
    A[질문 입력] --> B[Upstage Embedding API<br/>질문 임베딩<br/><i>~0.1초</i>]
    B --> C[ChromaDB 검색<br/>n_results=3<br/>filter: material_id<br/><i>~0.1초</i>]
    C --> D[유사도 계산<br/>Cosine Distance<br/><i>~0.01초</i>]
    D --> E[상위 3개 청크<br/>retrieved_docs<br/><i>~0.01초</i>]

    style B fill:#ffe5e5
    style C fill:#e5f5e5
    style D fill:#fff4e1
```

**세부 처리 단계**:
1. **질문 임베딩 생성** (~0.1초): Upstage API를 통해 4096차원 벡터 생성
2. **벡터 유사도 검색** (~0.1초): ChromaDB에서 코사인 거리 기반 검색
3. **결과 필터링** (~0.01초): material_id로 필터링, 상위 3개 선택
4. **응답 구성** (~0.01초): 문서 내용, 페이지, 메타데이터 포맷팅

#### 검색 과정

```mermaid
sequenceDiagram
    participant Q as Question
    participant E as Upstage Embedding
    participant DB as ChromaDB
    participant R as Retrieved Docs

    Q->>E: "Iterator 패턴이란?"
    E-->>Q: [0.1, 0.2, ..., 0.9] (4096차원)
    Q->>DB: query(embedding, n=3, filter)
    DB->>DB: 코사인 유사도 계산
    DB-->>R: Top 3 청크
    Note over R: 청크1: distance=0.12<br/>청크2: distance=0.18<br/>청크3: distance=0.24
```

### generate_answer_node (⏱️ 0.8~1.0초)

```mermaid
flowchart TD
    A[retrieved_docs] --> B[컨텍스트 구성<br/>페이지 번호 포함<br/><i>~0.01초</i>]
    B --> C[프롬프트 생성<br/>학습자료 + 질문<br/><i>~0.01초</i>]
    C --> D[Upstage Solar LLM<br/>solar-1-mini-chat<br/>temperature=0.3<br/><i>~0.8초</i>]
    D --> E[답변 생성<br/><i>~0.01초</i>]
    E --> F[출처 정보 구성<br/>excerpt 100자<br/><i>~0.01초</i>]
    F --> G[answer + sources<br/>반환]

    style D fill:#ffe5e5
    style F fill:#fff4e1
```

**세부 처리 단계**:
1. **컨텍스트 구성** (~0.01초): 검색된 3개 청크를 페이지 번호와 함께 포맷팅
2. **프롬프트 빌드** (~0.01초): 시스템 프롬프트 + 컨텍스트 + 질문 결합
3. **LLM 응답 생성** (~0.8초): Solar-1-mini-chat 모델로 답변 생성 (가장 긴 단계)
4. **후처리** (~0.02초): 답변 텍스트 추출 및 출처 정보 구성

#### 답변 생성 과정

```mermaid
sequenceDiagram
    participant D as Retrieved Docs
    participant P as Prompt Builder
    participant L as Solar LLM
    participant A as Answer

    D->>P: 3개 청크
    Note over P: [페이지 552]<br/>Iterator 패턴은...<br/>---<br/>[페이지 316]<br/>...
    P->>L: 프롬프트
    Note over L: 학습자료 내용<br/>+ 학생 질문<br/>+ 답변 규칙
    L-->>A: LLM 응답
    A->>A: 출처 정보 추가
    Note over A: answer<br/>+ sources (100자 excerpt)
```

---

## 전체 시스템 아키텍처

```mermaid
graph TB
    subgraph "Upload Pipeline"
        U1[PDF/PPT] --> U2[Parse]
        U2 --> U3[Chunk<br/>500자]
        U3 --> U4[Embed]
        U4 --> U5[ChromaDB<br/>배치 저장]
    end

    subgraph "QA Pipeline"
        Q1[질문] --> Q2[Embed]
        Q2 --> Q3[ChromaDB<br/>검색]
        Q3 --> Q4[LLM<br/>답변 생성]
        Q4 --> Q5[답변]
    end

    U5 -.자료 저장.-> Q3

    style U3 fill:#fff4e1
    style U4 fill:#ffe5e5
    style Q2 fill:#ffe5e5
    style Q4 fill:#ffe5e5
```

---

## 성능 최적화

### 청킹 전략 비교

```mermaid
graph LR
    subgraph "개선 전 (페이지 단위)"
        A1[페이지 15<br/>5000자] --> A2[임베딩 1개]
        A2 --> A3[검색 정확도 낮음]
    end

    subgraph "개선 후 (청크 단위)"
        B1[페이지 15<br/>5000자] --> B2[청크 10개<br/>각 500자]
        B2 --> B3[임베딩 10개]
        B3 --> B4[검색 정확도 높음]
    end

    style A1 fill:#ffe5e5
    style A3 fill:#ffe5e5
    style B2 fill:#e5f5ff
    style B4 fill:#e5f5e5
```

### 배치 처리 효과

```mermaid
graph TD
    A[2700개 청크] --> B{배치 처리}

    B -->|개선 전| C[1회 요청<br/>47.5MB]
    C --> D[❌ 413 Payload Too Large]

    B -->|개선 후| E[27회 요청<br/>각 1.76MB]
    E --> F[✅ 성공]

    style C fill:#ffe5e5
    style D fill:#ffe5e5
    style E fill:#e5f5ff
    style F fill:#e5f5e5
```

---

## 데이터 흐름

### Upload Flow

```mermaid
stateDiagram-v2
    [*] --> Parsing: file_path, material_id
    Parsing --> Chunking: parsed_blocks
    Chunking --> Embedding: chunks (2700개)
    Embedding --> Storing: embeddings
    Storing --> [*]: status: completed

    note right of Chunking
        RecursiveCharacterTextSplitter
        chunk_size=500
        overlap=50
    end note

    note right of Storing
        배치 처리
        BATCH_SIZE=100
        27회 저장
    end note
```

### QA Flow

```mermaid
stateDiagram-v2
    [*] --> Question: question, material_id
    Question --> Embedding: 질문 텍스트
    Embedding --> Search: query_embedding
    Search --> LLM: retrieved_docs (3개)
    LLM --> Answer: context + prompt
    Answer --> [*]: answer, sources

    note right of Search
        ChromaDB 검색
        n_results=3
        filter: material_id
    end note

    note right of LLM
        Solar LLM
        temperature=0.3
    end note
```

---

## 컴포넌트 다이어그램

```mermaid
graph TB
    subgraph "External Services"
        US[Upstage API<br/>- Document Parse<br/>- Embedding<br/>- Solar LLM]
        CH[ChromaDB<br/>Vector Store]
    end

    subgraph "paper_qa Module"
        WF[workflow.py<br/>LangGraph]
        PA[parsers/<br/>pdf_parser.py<br/>ppt_parser.py]
        API[api.py<br/>FastAPI Routes]
    end

    subgraph "shared Module"
        UC[upstage_client.py<br/>Singleton]
        CC[chroma_client.py<br/>Singleton]
    end

    API --> WF
    WF --> PA
    WF --> UC
    WF --> CC
    UC --> US
    CC --> CH

    style US fill:#ffe5e5
    style CH fill:#e5f5e5
    style WF fill:#fff4e1
```
