# QA System - LangGraph Workflow

RAG 기반 질의응답 시스템의 LangGraph 워크플로우

---

## Upload Workflow

```mermaid
graph TD
    Start([START]) --> Parse[parse_document_node<br/>문서 파싱]
    Parse --> EmbedStore[embed_and_store_node<br/>임베딩 및 저장]
    EmbedStore --> End([END])

    style Parse fill:#e1f5ff
    style EmbedStore fill:#fff4e1
```

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

### embed_and_store_node

```mermaid
flowchart TD
    A[parsed_blocks] --> B[RecursiveCharacterTextSplitter<br/>청킹]
    B --> C[청크 생성<br/>chunk_size=500<br/>overlap=50]
    C --> D[Upstage Embedding API<br/>배치 임베딩]
    D --> E{배치 처리<br/>BATCH_SIZE=100}
    E --> F[ChromaDB 저장<br/>배치 1/27]
    F --> G{더 저장할<br/>배치?}
    G -->|Yes| F
    G -->|No| H[저장 완료<br/>embeddings 반환]

    style B fill:#fff4e1
    style D fill:#ffe5e5
    style E fill:#e5f5e5
```

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
    Start([START]) --> Retrieve[retrieve_node<br/>문서 검색]
    Retrieve --> Generate[generate_answer_node<br/>답변 생성]
    Generate --> End([END])

    style Retrieve fill:#e1f5ff
    style Generate fill:#fff4e1
```

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

### retrieve_node

```mermaid
flowchart TD
    A[질문 입력] --> B[Upstage Embedding API<br/>질문 임베딩]
    B --> C[ChromaDB 검색<br/>n_results=3<br/>filter: material_id]
    C --> D[유사도 계산<br/>Cosine Distance]
    D --> E[상위 3개 청크<br/>retrieved_docs]

    style B fill:#ffe5e5
    style C fill:#e5f5e5
    style D fill:#fff4e1
```

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

### generate_answer_node

```mermaid
flowchart TD
    A[retrieved_docs] --> B[컨텍스트 구성<br/>페이지 번호 포함]
    B --> C[프롬프트 생성<br/>학습자료 + 질문]
    C --> D[Upstage Solar LLM<br/>solar-1-mini-chat<br/>temperature=0.3]
    D --> E[답변 생성]
    E --> F[출처 정보 구성<br/>excerpt 100자]
    F --> G[answer + sources<br/>반환]

    style D fill:#ffe5e5
    style F fill:#fff4e1
```

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
