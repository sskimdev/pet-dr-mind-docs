<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# "Dr. 마음" 백엔드 시스템 아키텍처 다이어그램

## 1. 전체 시스템 아키텍처 다이어그램

```mermaid
graph TB
    subgraph "클라이언트 레이어"
        Mobile["모바일 앱"]
        Web["웹 앱"]
    end

    subgraph "API 게이트웨이 레이어"
        APIGW["API Gateway<br/>(REST API)"]
        WSGW["API Gateway<br/>(WebSocket)"]
    end

    subgraph "인증 레이어"
        Cognito["Amazon Cognito<br/>사용자 인증"]
    end

    subgraph "마이크로서비스 레이어"
        AuthLambda["Auth Service<br/>(Lambda)"]
        ProfileLambda["Profile Service<br/>(Lambda)"]
        ChatLambda["Chat Service<br/>(Lambda)"]
        WSLambda["WebSocket Handler<br/>(Lambda)"]
        InferenceLambda["Inference Service<br/>(Lambda)"]
        SoapLambda["SOAP Generator<br/>(Lambda)"]
        STTLambda["STT Processor<br/>(Lambda)"]
    end

    subgraph "메시징 & 큐"
        SQS["Amazon SQS<br/>추론 요청 큐"]
        DLQ["Dead Letter Queue"]
    end

    subgraph "오케스트레이션"
        StepFunc["Step Functions<br/>RAG 파이프라인"]
    end

    subgraph "캐싱 레이어"
        Redis["ElastiCache<br/>Redis Cluster"]
    end

    subgraph "데이터베이스 레이어"
        ChatDB["DynamoDB<br/>채팅 테이블"]
        ProfileDB["DynamoDB<br/>프로필 테이블"]
        SoapDB["DynamoDB<br/>SOAP 테이블"]
    end

    subgraph "벡터 데이터베이스"
        OpenSearch["OpenSearch<br/>Serverless<br/>벡터 검색"]
    end

    subgraph "스토리지"
        S3["S3 Bucket<br/>오디오 파일"]
    end

    subgraph "외부 서비스"
        OpenAI["OpenAI API"]
        Claude["Claude API"]
        Transcribe["Amazon Transcribe"]
    end

    subgraph "보안"
        Secrets["Secrets Manager<br/>API 키 관리"]
        IAM["IAM Roles<br/>권한 관리"]
    end

    subgraph "모니터링"
        CloudWatch["CloudWatch<br/>로깅 & 메트릭"]
        XRay["X-Ray<br/>분산 추적"]
    end

    %% 연결 관계
    Mobile --> APIGW
    Web --> APIGW
    Mobile --> WSGW
    Web --> WSGW

    APIGW --> Cognito
    APIGW --> AuthLambda
    APIGW --> ProfileLambda
    APIGW --> ChatLambda

    WSGW --> WSLambda

    AuthLambda --> Cognito
    ProfileLambda --> ProfileDB
    ChatLambda --> ChatDB
    ChatLambda --> SQS

    SQS --> InferenceLambda
    SQS --> DLQ

    InferenceLambda --> StepFunc
    InferenceLambda --> Redis
    InferenceLambda --> OpenSearch
    InferenceLambda --> WSGW

    StepFunc --> OpenAI
    StepFunc --> Claude
    StepFunc --> Secrets

    ChatDB --> SoapLambda
    SoapLambda --> SoapDB
    SoapLambda --> OpenSearch

    S3 --> STTLambda
    STTLambda --> Transcribe
    STTLambda --> ChatDB
    STTLambda --> OpenSearch

    %% 모니터링 연결
    AuthLambda -.-> CloudWatch
    ProfileLambda -.-> CloudWatch
    ChatLambda -.-> CloudWatch
    InferenceLambda -.-> CloudWatch
    SoapLambda -.-> CloudWatch
    STTLambda -.-> CloudWatch

    AuthLambda -.-> XRay
    ProfileLambda -.-> XRay
    ChatLambda -.-> XRay
    InferenceLambda -.-> XRay

    %% 스타일링
    classDef lambdaStyle fill:#FF9900,stroke:#232F3E,stroke-width:2px,color:#FFFFFF
    classDef dbStyle fill:#3F48CC,stroke:#232F3E,stroke-width:2px,color:#FFFFFF
    classDef apiStyle fill:#FF4B4B,stroke:#232F3E,stroke-width:2px,color:#FFFFFF
    classDef cacheStyle fill:#C925D1,stroke:#232F3E,stroke-width:2px,color:#FFFFFF
    classDef externalStyle fill:#146EB4,stroke:#232F3E,stroke-width:2px,color:#FFFFFF

    class AuthLambda,ProfileLambda,ChatLambda,WSLambda,InferenceLambda,SoapLambda,STTLambda lambdaStyle
    class ChatDB,ProfileDB,SoapDB,OpenSearch dbStyle
    class APIGW,WSGW apiStyle
    class Redis cacheStyle
    class OpenAI,Claude,Transcribe externalStyle
```


## 2. 채팅 메시지 처리 시퀀스 다이어그램

```mermaid
sequenceDiagram
    participant Client as "클라이언트"
    participant APIGW as "API Gateway"
    participant Auth as "Cognito"
    participant ChatSvc as "Chat Service"
    participant ChatDB as "Chat DynamoDB"
    participant SQS as "SQS Queue"
    participant InfSvc as "Inference Service"
    participant Redis as "Redis Cache"
    participant VectorDB as "Vector DB"
    participant LLMGateway as "LLM Gateway"
    participant LLM as "External LLM"
    participant WebSocket as "WebSocket API"

    Note over Client, WebSocket: "사용자 메시지 전송 플로우"

    Client->>+APIGW: POST /chat/send
    APIGW->>+Auth: "토큰 검증"
    Auth-->>-APIGW: "사용자 정보"
    
    APIGW->>+ChatSvc: "메시지 처리 요청"
    ChatSvc->>+ChatDB: "사용자 메시지 저장"
    ChatDB-->>-ChatSvc: "저장 완료"
    
    ChatSvc->>+SQS: "AI 추론 요청 전송"
    SQS-->>-ChatSvc: "큐 전송 완료"
    
    ChatSvc-->>-APIGW: "처리 중 응답"
    APIGW-->>-Client: "200 OK: 답변 준비 중"

    Note over SQS, WebSocket: "비동기 AI 추론 처리"

    SQS->>+InfSvc: "추론 요청 트리거"
    
    InfSvc->>+Redis: "캐시된 응답 확인"
    Redis-->>-InfSvc: "캐시 결과"
    
    alt "캐시 없음"
        InfSvc->>+VectorDB: "유사도 검색"
        VectorDB-->>-InfSvc: "관련 문서 반환"
        
        InfSvc->>+LLMGateway: "RAG 컨텍스트 + 질문"
        LLMGateway->>+LLM: "LLM API 호출"
        LLM-->>-LLMGateway: "AI 응답"
        LLMGateway-->>-InfSvc: "응답 반환"
        
        InfSvc->>+Redis: "응답 캐싱"
        Redis-->>-InfSvc: "캐싱 완료"
    else "캐시 있음"
        Note over InfSvc: "캐시된 응답 사용"
    end
    
    InfSvc->>+ChatDB: "AI 응답 저장"
    ChatDB-->>-InfSvc: "저장 완료"
    
    InfSvc->>+WebSocket: "실시간 응답 전송"
    WebSocket-->>-Client: "AI 응답 수신"
    
    InfSvc-->>-SQS: "처리 완료"
```


## 3. RAG 파이프라인 상세 플로우 다이어그램

```mermaid
flowchart TD
    Start(["추론 요청 시작"]) --> CacheCheck["캐시 확인"]
    
    CacheCheck --> CacheHit{{"캐시 존재?"}}
    CacheHit -->|"있음"| ReturnCached["캐시된 응답 반환"]
    CacheHit -->|"없음"| GetContext["사용자 컨텍스트 수집"]
    
    GetContext --> Profile["프로필 정보 조회"]
    GetContext --> History["대화 기록 조회"]
    
    Profile --> VectorSearch["벡터 유사도 검색"]
    History --> VectorSearch
    
    VectorSearch --> EmbeddingGen["질문 임베딩 생성"]
    EmbeddingGen --> SearchExec["OpenSearch 검색 실행"]
    SearchExec --> FilterResults["결과 필터링 & 스코어링"]
    
    FilterResults --> ContextBuild["RAG 컨텍스트 구성"]
    ContextBuild --> SystemPrompt["시스템 프롬프트 추가"]
    ContextBuild --> UserProfile["사용자 프로필 추가"]
    ContextBuild --> RecentChat["최근 대화 추가"]
    ContextBuild --> KnowledgeBase["지식베이스 정보 추가"]
    
    SystemPrompt --> LLMRoute["LLM 라우팅 결정"]
    UserProfile --> LLMRoute
    RecentChat --> LLMRoute
    KnowledgeBase --> LLMRoute
    
    LLMRoute --> ModelSelect{{"모델 선택"}}
    ModelSelect -->|"Claude"| ClaudeAPI["Claude API 호출"]
    ModelSelect -->|"OpenAI"| OpenAIAPI["OpenAI API 호출"]
    
    ClaudeAPI --> ResponseProcess["응답 처리"]
    OpenAIAPI --> ResponseProcess
    
    ResponseProcess --> SourceExtract["출처 정보 추출"]
    SourceExtract --> ResponseValidate["응답 검증"]
    
    ResponseValidate --> ValidResponse{{"응답 유효?"}}
    ValidResponse -->|"유효"| CacheStore["응답 캐싱"]
    ValidResponse -->|"무효"| Fallback["폴백 모델 시도"]
    
    Fallback --> ModelSelect
    
    CacheStore --> DBStore["DynamoDB 저장"]
    ReturnCached --> WebSocketSend["WebSocket 전송"]
    DBStore --> WebSocketSend
    
    WebSocketSend --> End(["처리 완료"])
    
    %% 에러 처리
    CacheCheck --> CacheError{{"캐시 에러"}}
    VectorSearch --> SearchError{{"검색 에러"}}
    LLMRoute --> LLMError{{"LLM 에러"}}
    
    CacheError -->|"에러"| GetContext
    SearchError -->|"에러"| DefaultContext["기본 컨텍스트 사용"]
    LLMError -->|"에러"| ErrorResponse["에러 응답 생성"]
    
    DefaultContext --> LLMRoute
    ErrorResponse --> WebSocketSend
    
    %% 스타일링
    classDef startEnd fill:#4CAF50,stroke:#2E7D32,stroke-width:2px,color:#FFFFFF
    classDef process fill:#2196F3,stroke:#1565C0,stroke-width:2px,color:#FFFFFF
    classDef decision fill:#FF9800,stroke:#EF6C00,stroke-width:2px,color:#FFFFFF
    classDef error fill:#F44336,stroke:#C62828,stroke-width:2px,color:#FFFFFF
    classDef external fill:#9C27B0,stroke:#6A1B9A,stroke-width:2px,color:#FFFFFF
    
    class Start,End startEnd
    class CacheCheck,GetContext,Profile,History,VectorSearch,EmbeddingGen,SearchExec,FilterResults,ContextBuild,SystemPrompt,UserProfile,RecentChat,KnowledgeBase,ResponseProcess,SourceExtract,ResponseValidate,CacheStore,DBStore,WebSocketSend,DefaultContext process
    class CacheHit,ModelSelect,ValidResponse decision
    class CacheError,SearchError,LLMError,Fallback,ErrorResponse error
    class ClaudeAPI,OpenAIAPI,ReturnCached external
```


## 4. K-SOAP 생성 워크플로우 다이어그램

```mermaid
flowchart TD
    Start(["대화 세션 종료"]) --> StreamTrigger["DynamoDB Stream 트리거"]
    
    StreamTrigger --> CheckEvent{{"이벤트 타입 확인"}}
    CheckEvent -->|"REMOVE"| GetSessionID["세션 ID 추출"]
    CheckEvent -->|"기타"| Skip["처리 건너뛰기"]
    
    GetSessionID --> QueryMessages["대화 내용 조회"]
    QueryMessages --> ValidateMessages{{"메시지 존재?"}}
    
    ValidateMessages -->|"없음"| Skip
    ValidateMessages -->|"있음"| AnalyzeConversation["대화 분석 시작"]
    
    AnalyzeConversation --> BuildPrompt["분석 프롬프트 구성"]
    BuildPrompt --> ConversationText["대화 텍스트 구성"]
    ConversationText --> LLMAnalysis["LLM 분석 요청"]
    
    LLMAnalysis --> ParseResult["JSON 결과 파싱"]
    ParseResult --> ExtractSOAP{{"SOAP 요소 추출"}}
    
    ExtractSOAP --> SubjectiveData["주관적 소견<br/>(Subjective)"]
    ExtractSOAP --> ObjectiveData["객관적 소견<br/>(Objective)"]  
    ExtractSOAP --> AssessmentData["평가<br/>(Assessment)"]
    ExtractSOAP --> PlanData["계획<br/>(Plan)"]
    
    SubjectiveData --> GenerateSOAP["SOAP 노트 생성"]
    ObjectiveData --> GenerateSOAP
    AssessmentData --> GenerateSOAP
    PlanData --> GenerateSOAP
    
    GenerateSOAP --> FormatSOAP["마크다운 포맷팅"]
    FormatSOAP --> CreateMetadata["메타데이터 구성"]
    
    CreateMetadata --> SaveDynamoDB["DynamoDB 저장"]
    SaveDynamoDB --> GenerateEmbedding["벡터 임베딩 생성"]
    
    GenerateEmbedding --> PrepareVectorData["벡터 데이터 준비"]
    PrepareVectorData --> StoreVectorDB["벡터 DB 저장"]
    
    StoreVectorDB --> Success["처리 성공"]
    
    %% 에러 처리
    QueryMessages --> QueryError{{"조회 에러"}}
    LLMAnalysis --> AnalysisError{{"분석 에러"}}
    ParseResult --> ParseError{{"파싱 에러"}}
    SaveDynamoDB --> SaveError{{"저장 에러"}}
    StoreVectorDB --> VectorError{{"벡터 저장 에러"}}
    
    QueryError --> LogError["에러 로깅"]
    AnalysisError --> DefaultSOAP["기본 SOAP 구조 생성"]
    ParseError --> DefaultSOAP
    SaveError --> LogError
    VectorError --> PartialSuccess["부분 성공<br/>(SOAP만 저장)"]
    
    DefaultSOAP --> FormatSOAP
    LogError --> End
    PartialSuccess --> End
    Success --> End(["완료"])
    Skip --> End
    
    %% 스타일링
    classDef startEnd fill:#4CAF50,stroke:#2E7D32,stroke-width:2px,color:#FFFFFF
    classDef process fill:#2196F3,stroke:#1565C0,stroke-width:2px,color:#FFFFFF
    classDef decision fill:#FF9800,stroke:#EF6C00,stroke-width:2px,color:#FFFFFF
    classDef error fill:#F44336,stroke:#C62828,stroke-width:2px,color:#FFFFFF
    classDef soap fill:#9C27B0,stroke:#6A1B9A,stroke-width:2px,color:#FFFFFF
    
    class Start,End startEnd
    class StreamTrigger,GetSessionID,QueryMessages,AnalyzeConversation,BuildPrompt,ConversationText,LLMAnalysis,ParseResult,GenerateSOAP,FormatSOAP,CreateMetadata,SaveDynamoDB,GenerateEmbedding,PrepareVectorData,StoreVectorDB,Success,DefaultSOAP,PartialSuccess,LogError process
    class CheckEvent,ValidateMessages,ExtractSOAP,QueryError,AnalysisError,ParseError,SaveError,VectorError decision
    class SubjectiveData,ObjectiveData,AssessmentData,PlanData soap
    class Skip error
```


## 5. STT 처리 워크플로우 다이어그램

```mermaid
flowchart TD
    Start(["오디오 파일 업로드"]) --> S3Trigger["S3 이벤트 트리거"]
    
    S3Trigger --> ExtractInfo["파일 정보 추출"]
    ExtractInfo --> ParsePath["파일 경로 파싱"]
    ParsePath --> GetMetadata["S3 메타데이터 조회"]
    
    GetMetadata --> ValidateFile{{"오디오 파일 검증"}}
    ValidateFile -->|"유효하지 않음"| Skip["처리 건너뛰기"]
    ValidateFile -->|"유효함"| CreateJob["Transcribe 작업 생성"]
    
    CreateJob --> JobConfig["작업 설정 구성"]
    JobConfig --> SetLanguage["언어 설정<br/>(ko-KR)"]
    JobConfig --> SetSpeakers["화자 분리 설정"]
    JobConfig --> SetFormat["미디어 포맷 설정"]
    
    SetLanguage --> StartTranscribe["Transcribe 작업 시작"]
    SetSpeakers --> StartTranscribe
    SetFormat --> StartTranscribe
    
    StartTranscribe --> PollStatus["작업 상태 폴링"]
    
    PollStatus --> CheckStatus{{"작업 상태 확인"}}
    CheckStatus -->|"IN_PROGRESS"| Wait["5초 대기"]
    CheckStatus -->|"COMPLETED"| DownloadResult["결과 다운로드"]
    CheckStatus -->|"FAILED"| TranscribeError["전사 실패"]
    
    Wait --> PollStatus
    
    DownloadResult --> ParseTranscript["전사 텍스트 파싱"]
    ParseTranscript --> ExtractText["텍스트 추출"]
    ExtractText --> ValidateText{{"텍스트 유효성 검증"}}
    
    ValidateText -->|"유효하지 않음"| TranscribeError
    ValidateText -->|"유효함"| BuildChatItem["채팅 아이템 구성"]
    
    BuildChatItem --> SetSessionID["세션 ID 설정"]
    BuildChatItem --> SetUserID["사용자 ID 설정"]
    BuildChatItem --> SetType["타입: audio_transcript"]
    BuildChatItem --> SetMetadata["메타데이터 설정"]
    
    SetSessionID --> SaveChat["채팅 DB 저장"]
    SetUserID --> SaveChat
    SetType --> SaveChat
    SetMetadata --> SaveChat
    
    SaveChat --> GenerateEmbedding["벡터 임베딩 생성"]
    GenerateEmbedding --> PrepareVector["벡터 데이터 준비"]
    PrepareVector --> SaveVector["벡터 DB 저장"]
    
    SaveVector --> CleanupJob["Transcribe 작업 정리"]
    CleanupJob --> Success["처리 성공"]
    
    %% 에러 처리
    ExtractInfo --> MetadataError{{"메타데이터 에러"}}
    SaveChat --> SaveError{{"저장 에러"}}
    SaveVector --> VectorError{{"벡터 저장 에러"}}
    
    MetadataError --> DefaultMetadata["기본 메타데이터 사용"]
    SaveError --> LogError["에러 로깅"]
    VectorError --> PartialSuccess["부분 성공<br/>(채팅만 저장)"]
    TranscribeError --> LogError
    
    DefaultMetadata --> CreateJob
    LogError --> End
    PartialSuccess --> CleanupJob
    Success --> End(["완료"])
    Skip --> End
    
    %% 스타일링
    classDef startEnd fill:#4CAF50,stroke:#2E7D32,stroke-width:2px,color:#FFFFFF
    classDef process fill:#2196F3,stroke:#1565C0,stroke-width:2px,color:#FFFFFF
    classDef decision fill:#FF9800,stroke:#EF6C00,stroke-width:2px,color:#FFFFFF
    classDef error fill:#F44336,stroke:#C62828,stroke-width:2px,color:#FFFFFF
    classDef transcribe fill:#9C27B0,stroke:#6A1B9A,stroke-width:2px,color:#FFFFFF
    classDef config fill:#607D8B,stroke:#37474F,stroke-width:2px,color:#FFFFFF
    
    class Start,End startEnd
    class S3Trigger,ExtractInfo,ParsePath,GetMetadata,CreateJob,StartTranscribe,PollStatus,DownloadResult,ParseTranscript,ExtractText,BuildChatItem,SaveChat,GenerateEmbedding,PrepareVector,SaveVector,CleanupJob,Success,DefaultMetadata,PartialSuccess,LogError process
    class ValidateFile,CheckStatus,ValidateText,MetadataError,SaveError,VectorError decision
    class TranscribeError,Skip error
    class JobConfig,SetLanguage,SetSpeakers,SetFormat,SetSessionID,SetUserID,SetType,SetMetadata config
    class Wait transcribe
```


## 6. CI/CD 파이프라인 다이어그램

```mermaid
flowchart TD
    subgraph "소스 관리"
        Dev["개발자"]
        GitHub["GitHub Repository"]
    end
    
    subgraph "CI/CD 파이프라인"
        Pipeline["CodePipeline"]
        
        subgraph "소스 스테이지"
            Source["Source Action<br/>GitHub Integration"]
        end
        
        subgraph "빌드 스테이지"
            Build["CodeBuild Project"]
            subgraph "빌드 작업"
                Test["유닛 테스트<br/>pytest"]
                SAMBuild["SAM Build<br/>컨테이너 빌드"]
                Validate["템플릿 검증<br/>sam validate"]
                Package["패키징<br/>CloudFormation"]
            end
        end
        
        subgraph "개발 배포 스테이지"
            DevDeploy["Development Deploy"]
            subgraph "개발 환경 배포"
                CreateCS["Change Set 생성"]
                ExecuteCS["Change Set 실행"]
                DevStack["dev 스택 배포"]
            end
        end
        
        subgraph "승인 스테이지"
            Approval["Manual Approval<br/>수동 승인"]
        end
        
        subgraph "운영 배포 스테이지"
            ProdDeploy["Production Deploy"]
            subgraph "운영 환경 배포"
                CreateProdCS["Change Set 생성"]
                ExecuteProdCS["Change Set 실행"]
                ProdStack["prod 스택 배포"]
            end
        end
    end
    
    subgraph "인프라 자원"
        DevInfra["개발 환경<br/>AWS 리소스"]
        ProdInfra["운영 환경<br/>AWS 리소스"]
    end
    
    subgraph "모니터링"
        CloudWatch["CloudWatch<br/>로깅 & 메트릭"]
        XRay["X-Ray<br/>분산 추적"]
        Alarms["CloudWatch Alarms<br/>알람 & 알림"]
    end
    
    %% 워크플로우 연결
    Dev --> GitHub
    GitHub --> Source
    Source --> Build
    
    Build --> Test
    Test --> SAMBuild
    SAMBuild --> Validate
    Validate --> Package
    
    Package --> DevDeploy
    DevDeploy --> CreateCS
    CreateCS --> ExecuteCS
    ExecuteCS --> DevStack
    
    DevStack --> Approval
    Approval --> ProdDeploy
    
    ProdDeploy --> CreateProdCS
    CreateProdCS --> ExecuteProdCS
    ExecuteProdCS --> ProdStack
    
    %% 배포 대상 연결
    DevStack --> DevInfra
    ProdStack --> ProdInfra
    
    %% 모니터링 연결
    DevInfra -.-> CloudWatch
    ProdInfra -.-> CloudWatch
    DevInfra -.-> XRay
    ProdInfra -.-> XRay
    CloudWatch --> Alarms
    
    %% 실패 시 롤백 (점선)
    ExecuteCS -.->|"실패 시"| DevStack
    ExecuteProdCS -.->|"실패 시"| ProdStack
    
    %% 피드백 루프 (점선)
    Alarms -.->|"알람 발생"| Dev
    
    %% 스타일링
    classDef developer fill:#4CAF50,stroke:#2E7D32,stroke-width:2px,color:#FFFFFF
    classDef pipeline fill:#2196F3,stroke:#1565C0,stroke-width:2px,color:#FFFFFF
    classDef build fill:#FF9800,stroke:#EF6C00,stroke-width:2px,color:#FFFFFF
    classDef deploy fill:#9C27B0,stroke:#6A1B9A,stroke-width:2px,color:#FFFFFF
    classDef infra fill:#607D8B,stroke:#37474F,stroke-width:2px,color:#FFFFFF
    classDef monitor fill:#F44336,stroke:#C62828,stroke-width:2px,color:#FFFFFF
    
    class Dev,GitHub developer
    class Pipeline,Source pipeline
    class Build,Test,SAMBuild,Validate,Package build
    class DevDeploy,CreateCS,ExecuteCS,DevStack,ProdDeploy,CreateProdCS,ExecuteProdCS,ProdStack,Approval deploy
    class DevInfra,ProdInfra infra
    class CloudWatch,XRay,Alarms monitor
```


## 7. 사용자 여정 플로우 다이어그램

```mermaid
journey
    title "Dr. 마음 사용자 여정"
    
    section "회원가입 & 로그인"
        "앱 다운로드": 5: 사용자
        "회원가입": 4: 사용자
        "이메일 인증": 3: 사용자, Cognito
        "로그인": 5: 사용자, Cognito
    
    section "프로필 설정"
        "반려동물 등록": 4: 사용자, ProfileService
        "기본 정보 입력": 4: 사용자
        "의료 기록 입력": 3: 사용자, ProfileService
    
    section "채팅 상담"
        "증상 설명": 5: 사용자, ChatService
        "사진 업로드": 4: 사용자, S3
        "AI 답변 대기": 2: 사용자
        "AI 답변 수신": 5: 사용자, InferenceService
        "추가 질문": 4: 사용자, ChatService
        "병원 추천": 5: 사용자, InferenceService
    
    section "음성 상담"
        "음성 녹음": 4: 사용자, STTService
        "음성 전사": 3: 사용자, Transcribe
        "텍스트 확인": 4: 사용자
        "AI 분석": 5: 사용자, InferenceService
    
    section "상담 종료"
        "세션 종료": 4: 사용자
        "SOAP 생성": 3: SOAPService
        "상담 기록 저장": 5: VectorDB
        "만족도 평가": 4: 사용자
```


## 8. 데이터 플로우 다이어그램

```mermaid
graph LR
    subgraph "입력 데이터"
        UserInput["사용자 입력<br/>(텍스트/음성)"]
        AudioFile["음성 파일<br/>(S3)"]
        UserProfile["사용자 프로필<br/>(DynamoDB)"]
    end
    
    subgraph "실시간 처리"
        ChatService["채팅 서비스"]
        STTService["STT 서비스"]
        Queue["SQS 큐"]
    end
    
    subgraph "AI 추론"
        Cache["Redis 캐시"]
        VectorSearch["벡터 검색"]
        RAGPipeline["RAG 파이프라인"]
        LLMGateway["LLM 게이트웨이"]
    end
    
    subgraph "지식 베이스"
        VectorDB["벡터 데이터베이스<br/>(OpenSearch)"]
        KnowledgeBase["의료 지식<br/>임베딩"]
        SOAPHistory["SOAP 기록<br/>임베딩"]
    end
    
    subgraph "출력 데이터"
        AIResponse["AI 응답"]
        SOAPNote["K-SOAP 노트"]
        ChatHistory["채팅 기록"]
    end
    
    subgraph "저장소"
        ChatDB["채팅 DB<br/>(DynamoDB)"]
        ProfileDB["프로필 DB<br/>(DynamoDB)"]
        SOAPDB["SOAP DB<br/>(DynamoDB)"]
    end
    
    %% 데이터 플로우
    UserInput --> ChatService
    AudioFile --> STTService
    
    ChatService --> Queue
    STTService --> ChatDB
    Queue --> Cache
    
    Cache -->|"캐시 미스"| VectorSearch
    VectorSearch --> VectorDB
    VectorDB --> KnowledgeBase
    VectorDB --> SOAPHistory
    
    KnowledgeBase --> RAGPipeline
    SOAPHistory --> RAGPipeline
    UserProfile --> RAGPipeline
    
    RAGPipeline --> LLMGateway
    LLMGateway --> AIResponse
    
    AIResponse --> ChatDB
    AIResponse --> Cache
    
    ChatDB --> SOAPNote
    SOAPNote --> SOAPDB
    SOAPNote --> VectorDB
    
    ChatDB --> ChatHistory
    UserProfile --> ProfileDB
    
    %% 스타일링
    classDef input fill:#4CAF50,stroke:#2E7D32,stroke-width:2px,color:#FFFFFF
    classDef process fill:#2196F3,stroke:#1565C0,stroke-width:2px,color:#FFFFFF
    classDef ai fill:#FF9800,stroke:#EF6C00,stroke-width:2px,color:#FFFFFF
    classDef knowledge fill:#9C27B0,stroke:#6A1B9A,stroke-width:2px,color:#FFFFFF
    classDef output fill:#F44336,stroke:#C62828,stroke-width:2px,color:#FFFFFF
    classDef storage fill:#607D8B,stroke:#37474F,stroke-width:2px,color:#FFFFFF
    
    class UserInput,AudioFile,UserProfile input
    class ChatService,STTService,Queue process
    class Cache,VectorSearch,RAGPipeline,LLMGateway ai
    class VectorDB,KnowledgeBase,SOAPHistory knowledge
    class AIResponse,SOAPNote,ChatHistory output
    class ChatDB,ProfileDB,SOAPDB storage
```

이러한 다이어그램들은 "Dr. 마음" 백엔드 시스템의 전체 아키텍처와 데이터 플로우를 시각적으로 명확하게 보여줍니다. 각 다이어그램은 시스템의 다른 관점을 제공하며, 개발팀과 스테이크홀더들이 시스템을 이해하고 의사소통하는데 도움이 됩니다.

