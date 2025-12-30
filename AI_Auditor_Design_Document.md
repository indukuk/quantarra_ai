# AI Auditor Design Document
## SaaS Compliance Platform Enhancement

### Executive Summary
This document outlines the design for an AI Auditor feature to be integrated into the existing Bubble-based SaaS compliance platform. The AI Auditor will provide automated evidence review, expert chat assistance, and knowledge base integration using AWS Bedrock.

### Core Capabilities

#### 1. Automated Evidence Auditing
- **Functionality**: AI reviews all uploaded evidence against compliance controls
- **Output**: Detailed feedback and recommendations for each control
- **Trigger**: Customer-initiated audit runs after evidence upload completion

#### 2. Expert Chat Interface
- **Functionality**: Interactive AI assistant for evidence owners
- **Purpose**: Provide auditor expertise and guidance during evidence preparation
- **Context**: Leverages control-specific knowledge base

#### 3. Knowledge Base Integration
- **Scope**: Comprehensive knowledge base for each compliance regime
- **Content**: Control requirements, best practices, common issues
- **Integration**: Powers both audit feedback and chat responses

### Architecture Overview

#### Option 1: Evidence Storage in Bubble (Recommended Approach)

```mermaid
graph TB
    subgraph "Bubble Platform"
        subgraph "Data Layer"
            EDB[(Evidence Database)]
            CDB[(Controls Database)]
            KB[(Knowledge Base)]
            UDB[(Usage Tracking)]
        end
        
        subgraph "Application Layer"
            UI[User Interface]
            AR[Audit Runner]
            CI[Chat Interface]
            UM[Usage Monitor]
        end
        
        subgraph "Integration Layer"
            API[Bubble API Connector]
            FP[File Processor]
            QM[Queue Manager]
        end
    end
    
    subgraph "AWS Bedrock"
        Claude[Claude 3.5 Sonnet]
        Titan[Titan Embeddings]
        Custom[Custom Models]
    end
    
    subgraph "External Systems"
        Billing[Billing System]
        Alerts[Alert System]
    end
    
    UI --> AR
    UI --> CI
    AR --> EDB
    AR --> CDB
    AR --> KB
    CI --> KB
    CI --> EDB
    
    AR --> API
    CI --> API
    API --> Claude
    API --> Titan
    API --> Custom
    
    UM --> UDB
    UM --> Billing
    UM --> Alerts
    
    FP --> EDB
    QM --> AR
    QM --> CI
```

### AI Processing Architecture Decision

#### Option A: Vector Database Approach (Recommended)
```mermaid
sequenceDiagram
    participant User
    participant Bubble as Bubble App
    participant Extract as Content Extractor
    participant Vector as Vector DB
    participant Bedrock as AWS Bedrock
    
    User->>Bubble: Upload Document
    Bubble->>Extract: Process Document
    Extract->>Extract: OCR/Text Extraction
    Extract->>Bedrock: Generate Embeddings
    Bedrock->>Vector: Store Document Vectors
    Vector->>Bubble: Confirm Storage
    
    Note over User,Bubble: Document processed once, stored for reuse
    
    User->>Bubble: Run Audit
    Bubble->>Vector: Query Relevant Evidence
    Vector->>Bubble: Return Matching Documents
    Bubble->>Bedrock: Analyze with Control Context
    Bedrock->>Bubble: Return Audit Results
```

#### Option B: Direct API Approach
```mermaid
sequenceDiagram
    participant User
    participant Bubble as Bubble App
    participant Storage as File Storage
    participant Bedrock as AWS Bedrock
    
    User->>Bubble: Upload Document
    Bubble->>Storage: Store Raw Document
    Storage->>Bubble: Confirm Storage
    
    Note over User,Bubble: Document stored as-is
    
    User->>Bubble: Run Audit
    Bubble->>Storage: Retrieve Documents
    Storage->>Bubble: Return Raw Files
    Bubble->>Bedrock: Process + Analyze (Real-time)
    Bedrock->>Bubble: Return Audit Results
    
    Note over Bedrock,Bubble: Heavy processing on every audit
```

#### Comparison Analysis

| Aspect | Vector DB Approach | Direct API Approach |
|--------|-------------------|-------------------|
| **Initial Processing** | Heavy (once per document) | Light (store only) |
| **Audit Performance** | Fast (pre-processed) | Slow (process each time) |
| **Cost per Audit** | Low (cached embeddings) | High (full reprocessing) |
| **Storage Requirements** | Higher (vectors + raw) | Lower (raw files only) |
| **Scalability** | Excellent | Poor with volume |
| **Multi-language Support** | Better (embedding models) | Depends on multimodal AI |
| **Search Capabilities** | Semantic search enabled | Limited to full document |
| **Complexity** | Higher initial setup | Simpler architecture |

### Technical Implementation

#### Recommended Architecture: Hybrid Vector + Direct Processing

```mermaid
graph TD
    subgraph "Document Upload Flow"
        UP[Document Upload] --> PROC[Content Processor]
        PROC --> OCR[OCR/Text Extraction]
        PROC --> META[Metadata Extraction]
        OCR --> CHUNK[Text Chunking]
        CHUNK --> EMB[Generate Embeddings]
        EMB --> VDB[(Vector Database)]
        META --> BDB[(Bubble Database)]
        UP --> RAW[(Raw File Storage)]
    end
    
    subgraph "Audit Processing Flow"
        AUDIT[Run Audit] --> QUERY[Query Vector DB]
        QUERY --> MATCH[Find Relevant Chunks]
        MATCH --> CONTEXT[Build Context]
        CONTEXT --> AI[Bedrock Analysis]
        AI --> RESULT[Audit Results]
        
        AUDIT --> DIRECT[Direct File Analysis]
        DIRECT --> MULTIMODAL[Multimodal AI]
        MULTIMODAL --> VISUAL[Visual Analysis]
        VISUAL --> RESULT
    end
    
    subgraph "Storage Layer"
        VDB --> SEMANTIC[Semantic Search]
        BDB --> METADATA[Metadata Queries]
        RAW --> ORIGINAL[Original Documents]
    end
```

#### Processing Strategy by Document Type

**Text Documents (PDFs, Word, etc.)**
- Extract text → Generate embeddings → Store in vector DB
- Enable semantic search and context retrieval
- Fast audit processing using pre-computed vectors

**Visual Documents (Images, Charts, Diagrams)**
- Store original files → Direct multimodal AI processing
- Use Claude 3.5 Sonnet or GPT-4V for visual analysis
- Process on-demand during audits

**Structured Data (Spreadsheets, JSON)**
- Extract structured data → Store in Bubble database
- Generate embeddings for text fields
- Direct processing for calculations/validations

#### Implementation Details

**Vector Database Options:**
1. **AWS OpenSearch** (Recommended for AWS ecosystem)
2. **Pinecone** (Managed, easy to use)
3. **Chroma** (Open source, self-hosted)
4. **Bubble Plugin** (Vector search plugins available)

**Processing Pipeline:**
```mermaid
graph LR
    DOC[Document] --> TYPE{Document Type}
    
    TYPE -->|Text| TEXT_FLOW[Text Processing]
    TYPE -->|Image| IMG_FLOW[Image Processing]
    TYPE -->|Mixed| HYBRID_FLOW[Hybrid Processing]
    
    TEXT_FLOW --> EXTRACT[Text Extraction]
    EXTRACT --> CHUNK[Chunking Strategy]
    CHUNK --> EMBED[Embeddings Generation]
    EMBED --> STORE_V[(Vector Store)]
    
    IMG_FLOW --> STORE_R[(Raw Storage)]
    STORE_R --> DIRECT[Direct AI Processing]
    
    HYBRID_FLOW --> EXTRACT
    HYBRID_FLOW --> STORE_R
```

**Chunking Strategy:**
- **Semantic Chunking**: Split by paragraphs, sections
- **Overlap**: 20% overlap between chunks for context
- **Size**: 500-1000 tokens per chunk
- **Metadata**: Include document source, page numbers, control mappings

#### Cost Analysis: Vector DB vs Direct API

```mermaid
graph TD
    subgraph "Vector DB Approach Costs"
        VDB_SETUP[Initial Setup: $500-2000]
        VDB_STORAGE[Storage: $0.10/GB/month]
        VDB_EMBED[Embeddings: $0.0001/1K tokens]
        VDB_QUERY[Queries: $0.001/query]
        VDB_TOTAL[Total: Lower long-term cost]
    end
    
    subgraph "Direct API Approach Costs"
        API_SETUP[Initial Setup: $100-500]
        API_STORAGE[Storage: $0.05/GB/month]
        API_PROCESS[Processing: $3-15/1M tokens]
        API_MULTIMODAL[Multimodal: $10-50/1K images]
        API_TOTAL[Total: Higher per-audit cost]
    end
    
    VDB_TOTAL --> BREAK_EVEN[Break-even: ~100 audits]
    API_TOTAL --> BREAK_EVEN
```

#### Performance Comparison

| Metric | Vector DB | Direct API |
|--------|-----------|------------|
| **Document Upload** | 30-60 seconds | 1-5 seconds |
| **Audit Runtime** | 5-15 seconds | 30-120 seconds |
| **Concurrent Audits** | High (cached data) | Limited (processing bottleneck) |
| **Search Accuracy** | Semantic matching | Full document context |
| **Memory Usage** | Low (vector queries) | High (full document processing) |

#### Implementation Decision: Direct API Approach

**Current Strategy: Direct API for MVP**
- Faster initial development (2-4 weeks vs 8-12 weeks)
- Lower upfront complexity and costs
- Perfect for <50 customers
- Validate user workflows before scaling

**Future Migration Path:**
- Migrate to Vector DB when >50 active customers
- Add vector processing for text documents
- Keep direct processing for images/complex visuals

### Direct API Implementation Details

#### Data Storage Strategy
```mermaid
graph TD
    subgraph "Document Upload"
        UP[Upload Document] --> STORE[Store in Bubble File Storage]
        STORE --> META[Extract Basic Metadata]
        META --> DB[(Bubble Database)]
    end
    
    subgraph "Audit Processing"
        AUDIT[Run Audit] --> RETRIEVE[Get Documents for Control]
        RETRIEVE --> AI[Send to Bedrock Multimodal]
        AI --> ANALYSIS[AI Analysis Results]
        ANALYSIS --> DECISION{Store Results?}
        
        DECISION -->|Yes| STORE_RESULTS[Store in Audit Results Table]
        DECISION -->|No| DISPLAY[Display Only]
        
        STORE_RESULTS --> HISTORY[Enable Audit History]
        STORE_RESULTS --> COMPARE[Enable Comparisons]
        STORE_RESULTS --> REPORTS[Generate Reports]
    end
```

#### Recommended Data Model for Direct API

**Documents Table:**
- document_id (Primary Key)
- file_name
- file_type
- file_size
- upload_date
- control_mappings (List of control IDs)
- customer_id
- file_url (Bubble file reference)

**Audit Results Table (Recommended to Store):**
- audit_id (Primary Key)
- control_id
- document_ids (List of analyzed documents)
- ai_findings (JSON with structured results)
- recommendations (Text)
- compliance_status (Pass/Fail/Partial)
- confidence_score (0-100)
- audit_date
- customer_id
- auditor_version (Track AI model versions)

**Benefits of Storing Analysis Results:**
✅ Audit trail and compliance history
✅ Progress tracking over time
✅ Faster re-display of previous results
✅ Comparative analysis between audits
✅ Reporting and analytics capabilities
✅ Reduced API costs for repeat views

#### Processing Flow for Direct API
```mermaid
sequenceDiagram
    participant User
    participant Bubble as Bubble App
    participant Files as File Storage
    participant Bedrock as AWS Bedrock
    participant DB as Results Database
    
    User->>Bubble: Upload Evidence Documents
    Bubble->>Files: Store Files
    Bubble->>DB: Store Document Metadata
    
    User->>Bubble: Run Audit for Control X
    Bubble->>DB: Get Documents for Control X
    Bubble->>Files: Retrieve Document Files
    
    loop For Each Document
        Files->>Bedrock: Send Document + Control Requirements
        Bedrock->>Bedrock: Multimodal Analysis
        Bedrock->>Bubble: Return Analysis Results
    end
    
    Bubble->>Bubble: Aggregate Results
    Bubble->>DB: Store Audit Results (Recommended)
    Bubble->>User: Display Findings & Recommendations
```

#### Code Implementation Approach

**Bubble Workflow for Vector DB:**
```javascript
// Pseudo-code for Bubble backend workflow
When Document is uploaded:
  1. Extract text content
  2. Generate embeddings via Bedrock API
  3. Store vectors in external vector DB
  4. Store metadata in Bubble database
  
When Audit is triggered:
  1. Query vector DB for relevant content
  2. Retrieve top-k similar chunks
  3. Send context + control requirements to Bedrock
  4. Process AI response and store results
```

**Bubble Workflow for Direct API:**
```javascript
// Pseudo-code for direct processing
When Document is uploaded:
  1. Store file in Bubble file storage
  2. Extract basic metadata only
  
When Audit is triggered:
  1. Retrieve all documents for control
  2. Send files directly to Bedrock multimodal
  3. Process response and store results
```

#### Knowledge Base Structure
```mermaid
graph TD
    CR[Compliance Regime] --> CC1[Control Category 1]
    CR --> CC2[Control Category 2]
    CR --> CC3[Control Category N]
    
    CC1 --> IC1[Individual Control 1.1]
    CC1 --> IC2[Individual Control 1.2]
    CC2 --> IC3[Individual Control 2.1]
    
    IC1 --> REQ1[Requirements]
    IC1 --> ET1[Evidence Types]
    IC1 --> CI1[Common Issues]
    IC1 --> BP1[Best Practices]
    
    IC2 --> REQ2[Requirements]
    IC2 --> ET2[Evidence Types]
    IC2 --> CI2[Common Issues]
    IC2 --> BP2[Best Practices]
    
    CC1 --> XR1[Cross-references]
    CC2 --> XR2[Cross-references]
    
    CR --> TEMP[Templates & Examples]
    
    style CR fill:#e1f5fe
    style CC1 fill:#f3e5f5
    style CC2 fill:#f3e5f5
    style CC3 fill:#f3e5f5
    style IC1 fill:#fff3e0
    style IC2 fill:#fff3e0
    style IC3 fill:#fff3e0
```

### Pros and Cons Analysis

#### Pros: Evidence in Bubble
✅ **Unified Data Management**: Single source of truth for all compliance data
✅ **Simplified Architecture**: Reduced complexity with fewer external dependencies
✅ **Native Integration**: Seamless workflow within existing Bubble app
✅ **Cost Efficiency**: No additional storage infrastructure costs
✅ **Security**: Leverages Bubble's existing security model
✅ **User Experience**: Consistent interface and permissions model
✅ **Backup/Recovery**: Included in Bubble's data protection
✅ **Compliance**: Easier to maintain data residency requirements

#### Cons: Evidence in Bubble
❌ **Storage Limitations**: Bubble file storage limits may constrain large evidence files
❌ **Performance**: Large file processing may impact app performance
❌ **Scalability**: Potential bottlenecks with high-volume evidence uploads
❌ **File Processing**: Limited native capabilities for complex document analysis
❌ **Vendor Lock-in**: Increased dependency on Bubble platform
❌ **Cost Scaling**: Bubble storage costs may become significant at scale

### Cost Chargeback Options

#### Cost Model Overview
```mermaid
graph TD
    AWS[AWS Bedrock Costs] --> DEV[Development Markup 40-60%]
    DEV --> SUP[Support Markup 20-30%]
    SUP --> PROFIT[Profit Margin 30-50%]
    PROFIT --> CUSTOMER[Customer Price 2-3x AWS]
    
    subgraph "Usage Tracking"
        AUDIT[Audit Runs]
        CHAT[Chat Messages]
        STORAGE[Evidence Storage]
        KB_ACCESS[Knowledge Base Access]
    end
    
    AUDIT --> CUSTOMER
    CHAT --> CUSTOMER
    STORAGE --> CUSTOMER
    KB_ACCESS --> CUSTOMER
```

#### 1. Usage-Based Pricing Model
```mermaid
graph LR
    subgraph "Customer Usage"
        AU[Audit Runs]
        CM[Chat Messages]
        ES[Evidence Storage]
    end
    
    subgraph "Pricing Tiers"
        AU --> AUP[Per Control: $0.50-2.00]
        CM --> CMP[Per Message: $0.10-0.50]
        ES --> ESP[Per GB/Month: $2-5]
    end
    
    subgraph "Billing"
        AUP --> BILL[Monthly Bill]
        CMP --> BILL
        ESP --> BILL
    end
```

#### 2. Tiered Subscription Model
```mermaid
graph TD
    BASIC[Basic Tier $99/month] --> B1[10 Audit Runs]
    BASIC --> B2[100 Chat Messages]
    BASIC --> B3[5GB Storage]
    
    PRO[Professional Tier $299/month] --> P1[Unlimited Audits]
    PRO --> P2[Unlimited Chat]
    PRO --> P3[50GB Storage]
    PRO --> P4[Priority Support]
    
    ENT[Enterprise Tier Custom] --> E1[Custom Models]
    ENT --> E2[Dedicated Resources]
    ENT --> E3[SLA Guarantees]
    ENT --> E4[Custom Integration]
```

#### 3. Credit-Based System
```mermaid
graph TD
    PURCHASE[Customer Purchases Credits] --> WALLET[Credit Wallet]
    
    WALLET --> AUDIT_COST[Audit: 10-50 Credits]
    WALLET --> CHAT_COST[Chat: 1-5 Credits]
    WALLET --> STORAGE_COST[Storage: 2 Credits/GB/Month]
    
    AUDIT_COST --> USAGE[Usage Tracking]
    CHAT_COST --> USAGE
    STORAGE_COST --> USAGE
    
    USAGE --> DEDUCT[Deduct from Wallet]
    DEDUCT --> ALERT{Credits Low?}
    ALERT -->|Yes| NOTIFY[Notify Customer]
    ALERT -->|No| CONTINUE[Continue Service]
```

#### 4. Hybrid Model (Recommended)
```mermaid
graph TD
    CUSTOMER[Customer] --> TIER{Subscription Tier}
    
    TIER --> BASIC_SUB[Basic: $149/month]
    TIER --> PRO_SUB[Pro: $399/month]
    TIER --> ENT_SUB[Enterprise: Custom]
    
    BASIC_SUB --> BASIC_INCLUDE[Includes: 20 Audits, 500 Chats, 10GB]
    PRO_SUB --> PRO_INCLUDE[Includes: 100 Audits, 2000 Chats, 100GB]
    ENT_SUB --> ENT_INCLUDE[Includes: Custom Limits]
    
    BASIC_INCLUDE --> OVERAGE{Usage > Limit?}
    PRO_INCLUDE --> OVERAGE
    ENT_INCLUDE --> OVERAGE
    
    OVERAGE -->|Yes| OVERAGE_BILL[Overage Charges]
    OVERAGE -->|No| BASE_BILL[Base Subscription Only]
    
    OVERAGE_BILL --> TOTAL[Total Monthly Bill]
    BASE_BILL --> TOTAL
```

### Cost Calculation Framework

#### AWS Bedrock Costs (Estimated)
- **Claude 3.5 Sonnet**: ~$3 per 1M input tokens, ~$15 per 1M output tokens
- **Titan Embeddings**: ~$0.0001 per 1K tokens
- **Knowledge Base**: ~$0.10 per GB per month

#### Markup Strategy
- **Development/Maintenance**: 40-60% markup
- **Support/Infrastructure**: 20-30% markup
- **Profit Margin**: 30-50% markup
- **Total Customer Price**: 2-3x AWS costs

#### Example Pricing Structure
```
Audit Run Pricing:
- Small Control Set (1-10 controls): $5-15 per run
- Medium Control Set (11-50 controls): $20-50 per run
- Large Control Set (50+ controls): $60-150 per run

Chat Interface:
- Per Message: $0.10-0.50
- Per Session (up to 50 messages): $5-15
- Monthly Unlimited: $50-200

Storage:
- Evidence Storage: $2-5 per GB per month
- Knowledge Base Access: $10-30 per regime per month
```

### Implementation Phases

```mermaid
gantt
    title AI Auditor Implementation Timeline
    dateFormat  YYYY-MM-DD
    section Phase 1: Foundation
    AWS Bedrock Setup           :p1a, 2024-01-01, 1w
    Evidence Processing         :p1b, after p1a, 2w
    Knowledge Base Structure    :p1c, after p1a, 2w
    
    section Phase 2: Core Features
    Automated Audit Engine      :p2a, after p1b, 2w
    Chat Interface Development  :p2b, after p1c, 2w
    Usage Tracking System       :p2c, after p2a, 1w
    
    section Phase 3: Enhancement
    Advanced Analytics          :p3a, after p2b, 2w
    Cost Chargeback System      :p3b, after p2c, 2w
    Performance Optimization    :p3c, after p3a, 1w
    
    section Phase 4: Scale & Polish
    Load Testing               :p4a, after p3b, 1w
    Advanced Reporting         :p4b, after p3c, 2w
    Customer Onboarding        :p4c, after p4a, 2w
```

#### Phase Details Flow
```mermaid
graph TD
    subgraph "Phase 1: Foundation (Weeks 1-4)"
        P1A[AWS Bedrock Integration]
        P1B[Basic Evidence Processing]
        P1C[Knowledge Base Structure]
    end
    
    subgraph "Phase 2: Core Features (Weeks 5-8)"
        P2A[Automated Audit Functionality]
        P2B[Chat Interface]
        P2C[Usage Tracking]
    end
    
    subgraph "Phase 3: Enhancement (Weeks 9-12)"
        P3A[Advanced Analytics]
        P3B[Cost Chargeback System]
        P3C[Performance Optimization]
    end
    
    subgraph "Phase 4: Scale & Polish (Weeks 13-16)"
        P4A[Load Testing]
        P4B[Advanced Reporting]
        P4C[Customer Onboarding Tools]
    end
    
    P1A --> P2A
    P1B --> P2A
    P1C --> P2B
    P2A --> P3A
    P2B --> P3A
    P2C --> P3B
    P3A --> P4B
    P3B --> P4C
    P3C --> P4A
```

### Risk Mitigation

#### Technical Risks
- **Bubble Performance**: Implement file processing queues
- **AI Accuracy**: Continuous model training and validation
- **Data Security**: Encryption at rest and in transit

#### Business Risks
- **Cost Overruns**: Implement usage caps and alerts
- **Customer Adoption**: Phased rollout with feedback loops
- **Compliance**: Regular security and compliance audits

### Success Metrics

#### KPI Dashboard Overview
```mermaid
graph TD
    subgraph "Technical KPIs"
        ACC[Audit Accuracy >90%]
        RT[Response Time <3s]
        UP[Uptime >99.5%]
        PROC[Processing Speed]
    end
    
    subgraph "Business KPIs"
        ADOPT[Customer Adoption Rate]
        REV[Revenue per Customer]
        COST[Cost Recovery Ratio]
        SAT[Customer Satisfaction]
    end
    
    subgraph "Usage Metrics"
        AU[Audit Runs per Month]
        CM[Chat Messages per User]
        ST[Storage Growth Rate]
        RET[Customer Retention]
    end
    
    ACC --> DASH[KPI Dashboard]
    RT --> DASH
    UP --> DASH
    ADOPT --> DASH
    REV --> DASH
    COST --> DASH
    AU --> DASH
    CM --> DASH
    ST --> DASH
```

#### Success Measurement Flow
```mermaid
graph LR
    DATA[Data Collection] --> ANALYSIS[Analysis Engine]
    ANALYSIS --> ALERTS{Threshold Met?}
    ALERTS -->|No| ACTION[Corrective Action]
    ALERTS -->|Yes| REPORT[Success Report]
    ACTION --> MONITOR[Continue Monitoring]
    REPORT --> STAKEHOLDER[Stakeholder Update]
    MONITOR --> DATA
```

### Next Steps

1. **Stakeholder Review**: Present design to key stakeholders
2. **Technical Validation**: Prototype core AI integration
3. **Cost Modeling**: Detailed financial projections
4. **Customer Research**: Validate pricing with target customers
5. **Development Planning**: Detailed sprint planning and resource allocation

---

*This document serves as the foundation for implementing the AI Auditor feature. Regular updates will be made as requirements evolve and implementation progresses.*

#### Chat Interface Implementation with Vector Context

```mermaid
sequenceDiagram
    participant User
    participant Chat as Chat Interface
    participant Vector as Vector DB
    participant KB as Knowledge Base
    participant Bedrock as AWS Bedrock
    participant Usage as Usage Tracker
    
    User->>Chat: Ask Question about Evidence
    Chat->>Vector: Semantic Search for Relevant Evidence
    Chat->>KB: Retrieve Control Requirements
    Vector->>Chat: Return Matching Evidence Chunks
    KB->>Chat: Return Control Context
    
    Chat->>Bedrock: Send Query + Evidence + Control Context
    Bedrock->>Bedrock: Generate Expert Response
    Bedrock->>Chat: Return Guidance with Citations
    
    Chat->>Usage: Log Usage Metrics
    Chat->>User: Display Response with Evidence References
    
    alt Follow-up Question
        User->>Chat: Follow-up Query
        Chat->>Bedrock: Send with Full Session Context
        Bedrock->>Chat: Contextual Response
        Chat->>User: Display Response
    end
```

#### Bubble Implementation for Direct API

**Workflow Setup in Bubble:**

1. **Document Upload Workflow:**
```javascript
// Bubble Backend Workflow
When File Uploader's value is changed:
  Step 1: Create new Document record
    - Set file_name = File Uploader's file name
    - Set file_type = File Uploader's file type  
    - Set file_url = File Uploader's value
    - Set customer_id = Current User's Customer
    - Set upload_date = Current date/time
  
  Step 2: Extract basic metadata (optional)
    - Set file_size = File Uploader's file size
    - Set control_mappings = (user selected controls)
```

2. **Audit Processing Workflow:**
```javascript
// Bubble Backend Workflow  
When "Run Audit" button is clicked:
  Step 1: Get documents for control
    - Search Documents where control_mappings contains Current Control
  
  Step 2: For each document, call Bedrock API
    - API Connector: AWS Bedrock
    - Send: Document file + Control requirements + Prompt template
    - Receive: AI analysis results
  
  Step 3: Store results (Recommended)
    - Create Audit Result record
    - Set ai_findings = API response
    - Set compliance_status = Parsed from AI response
    - Set recommendations = Extracted recommendations
  
  Step 4: Display to user
    - Show aggregated results
    - Highlight gaps and recommendations
```

**API Connector Configuration:**
- **Endpoint**: AWS Bedrock Claude 3.5 Sonnet
- **Authentication**: AWS IAM credentials
- **Request Format**: Multimodal (text + file)
- **Response Parsing**: Extract structured findings

**Prompt Template for Bedrock:**
```
You are an expert compliance auditor. Analyze the provided evidence document against the following control requirement:

CONTROL: [Control ID and Description]
REQUIREMENTS: [Specific requirements to check]

DOCUMENT: [Attached file]

Provide your analysis in this JSON format:
{
  "compliance_status": "Pass|Fail|Partial",
  "confidence_score": 85,
  "findings": [
    {
      "requirement": "Specific requirement",
      "status": "Met|Not Met|Partially Met", 
      "evidence_found": "Description of evidence",
      "gaps": "What's missing if any"
    }
  ],
  "recommendations": [
    "Specific actionable recommendation 1",
    "Specific actionable recommendation 2"
  ],
  "summary": "Overall assessment summary"
}
```

### AWS Technical Architecture

#### AWS Infrastructure Overview
```mermaid
graph TB
    subgraph "Bubble Platform"
        UI[User Interface]
        BDB[(Bubble Database)]
        BFS[Bubble File Storage]
        API_CONN[API Connector]
    end
    
    subgraph "AWS Cloud Infrastructure"
        subgraph "API Gateway & Authentication"
            APIGW[API Gateway]
            IAM[IAM Roles & Policies]
            COGNITO[Cognito - Optional]
        end
        
        subgraph "AWS Bedrock"
            BEDROCK[Bedrock Service]
            CLAUDE[Claude 3.5 Sonnet]
            TITAN[Titan Embeddings - Future]
        end
        
        subgraph "Storage & Processing"
            S3[S3 Bucket - Document Cache]
            LAMBDA[Lambda Functions]
            CW[CloudWatch Logs]
        end
        
        subgraph "Monitoring & Security"
            XRAY[X-Ray Tracing]
            KMS[KMS Encryption]
            VPC[VPC - Optional]
        end
    end
    
    UI --> API_CONN
    API_CONN --> APIGW
    APIGW --> IAM
    IAM --> BEDROCK
    BEDROCK --> CLAUDE
    
    BFS --> S3
    LAMBDA --> BEDROCK
    BEDROCK --> CW
    BEDROCK --> XRAY
    
    S3 --> KMS
    BEDROCK --> KMS
```

#### AWS Bedrock Integration Architecture
```mermaid
graph TD
    subgraph "Bubble Application"
        UPLOAD[Document Upload]
        TRIGGER[Audit Trigger]
        DISPLAY[Results Display]
    end
    
    subgraph "AWS API Gateway"
        GATEWAY[API Gateway Endpoint]
        AUTH[Request Authentication]
        THROTTLE[Rate Limiting]
    end
    
    subgraph "AWS Bedrock"
        BEDROCK_API[Bedrock API]
        MODEL_ROUTER[Model Router]
        CLAUDE_35[Claude 3.5 Sonnet]
        GUARDRAILS[Bedrock Guardrails]
    end
    
    subgraph "AWS Supporting Services"
        S3_TEMP[S3 - Temp Document Storage]
        LAMBDA_PROC[Lambda - Document Processor]
        CW_LOGS[CloudWatch Logs]
        CW_METRICS[CloudWatch Metrics]
    end
    
    UPLOAD --> GATEWAY
    TRIGGER --> GATEWAY
    GATEWAY --> AUTH
    AUTH --> THROTTLE
    THROTTLE --> BEDROCK_API
    
    BEDROCK_API --> MODEL_ROUTER
    MODEL_ROUTER --> CLAUDE_35
    CLAUDE_35 --> GUARDRAILS
    
    GATEWAY --> S3_TEMP
    S3_TEMP --> LAMBDA_PROC
    LAMBDA_PROC --> BEDROCK_API
    
    BEDROCK_API --> CW_LOGS
    BEDROCK_API --> CW_METRICS
    
    GUARDRAILS --> DISPLAY
```

#### Data Flow Architecture
```mermaid
sequenceDiagram
    participant Bubble as Bubble App
    participant S3 as AWS S3
    participant Gateway as API Gateway
    participant Bedrock as AWS Bedrock
    participant Claude as Claude 3.5
    participant CW as CloudWatch
    
    Note over Bubble,CW: Document Upload & Processing Flow
    
    Bubble->>S3: Upload Document (Optional Caching)
    Bubble->>Gateway: POST /analyze-document
    Gateway->>Gateway: Authenticate Request
    Gateway->>Bedrock: Forward Request
    
    Bedrock->>Claude: Send Document + Prompt
    Claude->>Claude: Multimodal Analysis
    Claude->>Bedrock: Return Structured Results
    
    Bedrock->>CW: Log Request Metrics
    Bedrock->>Gateway: Return Analysis Results
    Gateway->>Bubble: JSON Response
    
    Bubble->>Bubble: Store Results in Database
    Bubble->>Bubble: Display to User
```

#### AWS Security Architecture
```mermaid
graph TB
    subgraph "Security Layers"
        subgraph "Network Security"
            VPC[VPC - Optional]
            SG[Security Groups]
            NACL[Network ACLs]
        end
        
        subgraph "Identity & Access"
            IAM_ROLE[IAM Service Role]
            IAM_POLICY[IAM Policies]
            API_KEY[API Keys]
        end
        
        subgraph "Data Protection"
            KMS_KEY[KMS Customer Keys]
            ENCRYPT_TRANSIT[TLS 1.2+ Encryption]
            ENCRYPT_REST[S3 Encryption at Rest]
        end
        
        subgraph "Monitoring & Compliance"
            CLOUDTRAIL[CloudTrail Audit Logs]
            CONFIG[AWS Config Rules]
            GUARD_DUTY[GuardDuty - Optional]
        end
    end
    
    subgraph "Bubble Integration"
        BUBBLE_APP[Bubble Application]
        API_CONNECTOR[Bubble API Connector]
    end
    
    BUBBLE_APP --> API_CONNECTOR
    API_CONNECTOR --> API_KEY
    API_KEY --> IAM_ROLE
    IAM_ROLE --> IAM_POLICY
    
    IAM_POLICY --> VPC
    VPC --> SG
    
    API_CONNECTOR --> ENCRYPT_TRANSIT
    ENCRYPT_TRANSIT --> KMS_KEY
    KMS_KEY --> ENCRYPT_REST
    
    IAM_ROLE --> CLOUDTRAIL
    CLOUDTRAIL --> CONFIG
```

#### Option A: Store Analysis Results (Recommended)

**What to Store:**
```mermaid
graph TD
    AI[AI Analysis] --> STRUCT[Structured Results]
    STRUCT --> FINDINGS[Individual Findings per Requirement]
    STRUCT --> RECS[Actionable Recommendations] 
    STRUCT --> STATUS[Compliance Status]
    STRUCT --> CONF[Confidence Scores]
    STRUCT --> SUMMARY[Executive Summary]
    
    FINDINGS --> DB[(Audit Results Table)]
    RECS --> DB
    STATUS --> DB
    CONF --> DB
    SUMMARY --> DB
    
    DB --> HISTORY[Audit History]
    DB --> COMPARE[Progress Tracking]
    DB --> REPORTS[Compliance Reports]
    DB --> ANALYTICS[Trend Analysis]
```

**Benefits:**
✅ **Audit Trail**: Complete history of all assessments
✅ **Progress Tracking**: See improvements over time
✅ **Cost Efficiency**: Don't re-analyze for repeat views
✅ **Reporting**: Generate compliance reports instantly
✅ **Analytics**: Track trends and patterns
✅ **User Experience**: Instant access to previous results

**Storage Requirements:**
- ~1-5KB per document analysis
- ~10-50KB per complete control audit
- Minimal storage cost impact

#### Option B: Real-time Only (Not Recommended)

**Process:**
- Analyze documents on-demand
- Display results immediately
- Don't store analysis results

**Drawbacks:**
❌ **Cost**: Re-analyze every time (expensive)
❌ **Speed**: Slow user experience
❌ **No History**: Can't track progress
❌ **No Reporting**: Limited compliance reporting

### Recommended Data Structure for Stored Results

```json
{
  "audit_id": "audit_123",
  "control_id": "SOC2_CC6.1", 
  "control_name": "Logical Access Controls",
  "audit_date": "2024-01-15T10:30:00Z",
  "overall_status": "Partial",
  "overall_confidence": 78,
  "documents_analyzed": [
    {
      "document_id": "doc_456",
      "document_name": "Access Control Policy.pdf",
      "analysis": {
        "status": "Pass",
        "confidence": 85,
        "findings": [
          {
            "requirement": "Written access control policy exists",
            "status": "Met",
            "evidence": "Policy document dated 2023-12-01 with proper approval signatures",
            "page_references": [1, 2]
          }
        ]
      }
    }
  ],
  "aggregated_findings": [
    {
      "requirement": "Access control policy documented",
      "status": "Met",
      "supporting_documents": ["doc_456"]
    },
    {
      "requirement": "Regular access reviews conducted", 
      "status": "Not Met",
      "gap": "No evidence of quarterly access reviews"
    }
  ],
  "recommendations": [
    "Implement quarterly access review process",
    "Document access review procedures"
  ],
  "next_steps": [
    "Upload quarterly access review reports",
    "Provide access review procedure documentation"
  ]
}
```

### Implementation Priority for Direct API Approach

**Phase 1 (Weeks 1-2): Basic Processing**
- Document upload and storage
- Direct API calls to Bedrock
- Display results in real-time

**Phase 2 (Weeks 3-4): Store Results** 
- Create audit results data model
- Store AI analysis for history
- Basic reporting capabilities

**Phase 3 (Weeks 5-6): Enhanced Features**
- Progress tracking over time
- Comparative analysis
- Advanced reporting

**Recommendation: Implement result storage from Phase 1** - it's minimal additional work but provides significant value for users and reduces long-term costs.
#### AWS Cost Optimization Architecture
```mermaid
graph TD
    subgraph "Cost Management Strategy"
        subgraph "Request Optimization"
            BATCH[Batch Processing]
            CACHE[Response Caching]
            COMPRESS[Request Compression]
        end
        
        subgraph "Resource Management"
            LAMBDA_SIZING[Right-sized Lambda]
            S3_LIFECYCLE[S3 Lifecycle Policies]
            CW_RETENTION[Log Retention Policies]
        end
        
        subgraph "Monitoring & Alerts"
            COST_EXPLORER[Cost Explorer]
            BUDGETS[AWS Budgets]
            BILLING_ALERTS[Billing Alerts]
        end
    end
    
    BATCH --> LAMBDA_SIZING
    CACHE --> S3_LIFECYCLE
    COMPRESS --> CW_RETENTION
    
    LAMBDA_SIZING --> COST_EXPLORER
    S3_LIFECYCLE --> BUDGETS
    CW_RETENTION --> BILLING_ALERTS
```

### AWS Service Configuration Details

#### 1. AWS Bedrock Configuration
```yaml
# Bedrock Model Configuration
Model: anthropic.claude-3-5-sonnet-20241022-v2:0
Region: us-east-1 (or us-west-2 for lower latency)
Max Tokens: 4096
Temperature: 0.1 (for consistent analysis)
Top P: 0.9

# Request Limits
Max Request Size: 20MB (for document uploads)
Timeout: 300 seconds
Retry Policy: Exponential backoff (3 retries)
```

#### 2. API Gateway Configuration
```yaml
# API Gateway Setup
Type: REST API
Authentication: IAM
Throttling: 
  - Burst Limit: 100 requests
  - Rate Limit: 50 requests/second
CORS: Enabled for Bubble domain
Request Validation: Enabled
Response Caching: 5 minutes (for identical requests)

# Endpoints
POST /analyze-document
POST /chat-query
GET /health-check
```

#### 3. IAM Policies
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": [
        "arn:aws:bedrock:*::foundation-model/anthropic.claude-3-5-sonnet-*"
      ]
    },
    {
      "Effect": "Allow", 
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::your-compliance-docs-bucket/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream", 
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

#### 4. S3 Bucket Configuration
```yaml
# S3 Bucket for Document Caching (Optional)
Bucket Name: compliance-ai-documents-[environment]
Region: Same as Bedrock
Encryption: AES-256 (S3 Managed) or KMS
Versioning: Disabled (temporary storage)
Lifecycle Policy:
  - Delete objects after 7 days
  - Transition to IA after 1 day (if keeping longer)

# Access Policy
Public Access: Blocked
Access: IAM role only
CORS: Configured for Bubble domain
```

#### 5. CloudWatch Configuration
```yaml
# Logging Configuration
Log Groups:
  - /aws/apigateway/compliance-ai-api
  - /aws/bedrock/model-invocations
  - /aws/lambda/document-processor

# Metrics & Alarms
Custom Metrics:
  - API Response Time
  - Bedrock Token Usage
  - Error Rates
  - Cost per Request

Alarms:
  - High Error Rate (>5%)
  - High Latency (>30 seconds)
  - Daily Cost Threshold ($100)
```

### AWS Deployment Architecture Options

#### Option 1: Simple Direct Integration (Recommended for MVP)
```mermaid
graph LR
    BUBBLE[Bubble App] --> APIGW[API Gateway]
    APIGW --> BEDROCK[AWS Bedrock]
    BEDROCK --> CLAUDE[Claude 3.5]
    
    style BUBBLE fill:#e1f5fe
    style BEDROCK fill:#fff3e0
```

#### Option 2: Enhanced with Caching & Processing
```mermaid
graph LR
    BUBBLE[Bubble App] --> APIGW[API Gateway]
    APIGW --> LAMBDA[Lambda Processor]
    LAMBDA --> S3[S3 Cache]
    LAMBDA --> BEDROCK[AWS Bedrock]
    BEDROCK --> CLAUDE[Claude 3.5]
    
    style BUBBLE fill:#e1f5fe
    style LAMBDA fill:#f3e5f5
    style BEDROCK fill:#fff3e0
```

#### Option 3: Enterprise with Full Monitoring
```mermaid
graph TB
    BUBBLE[Bubble App] --> APIGW[API Gateway]
    APIGW --> LAMBDA[Lambda Processor]
    LAMBDA --> S3[S3 Cache]
    LAMBDA --> BEDROCK[AWS Bedrock]
    BEDROCK --> CLAUDE[Claude 3.5]
    
    LAMBDA --> CW[CloudWatch]
    APIGW --> XRAY[X-Ray Tracing]
    BEDROCK --> COST[Cost Explorer]
    
    style BUBBLE fill:#e1f5fe
    style LAMBDA fill:#f3e5f5
    style BEDROCK fill:#fff3e0
    style CW fill:#e8f5e8
```

### Implementation Phases for AWS Integration

#### Phase 1: Basic Bedrock Integration (Week 1-2)
- Set up AWS Bedrock access
- Configure API Gateway
- Create basic IAM roles
- Test document analysis

#### Phase 2: Production Hardening (Week 3-4)
- Add error handling and retries
- Implement request validation
- Set up CloudWatch monitoring
- Configure security policies

#### Phase 3: Optimization & Scaling (Week 5-6)
- Add response caching
- Implement cost monitoring
- Set up automated alerts
- Performance optimization

### Key Decision: Should You Store AI Analysis Results?