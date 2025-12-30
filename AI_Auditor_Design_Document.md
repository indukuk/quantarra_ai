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

#### Recommended Implementation Strategy

**Phase 1: Start with Direct API**
- Faster initial development
- Lower upfront complexity
- Validate user workflows and requirements

**Phase 2: Migrate to Vector DB**
- Implement when you have >50 active customers
- Add vector processing for text documents
- Keep direct processing for images/complex visuals

**Phase 3: Hybrid Optimization**
- Smart routing based on document type
- Caching layer for frequently accessed content
- Advanced semantic search capabilities

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

#### Code Implementation Approaches

**Bubble Workflow for Vector DB:**
```javascript
// Pseudo-code for Bubble backend workflow
When Document is uploaded:
  1. Extract text content using OCR/parsing
  2. Generate embeddings via Bedrock Titan
  3. Store vectors in external vector DB (OpenSearch/Pinecone)
  4. Store metadata in Bubble database
  
When Audit is triggered:
  1. Query vector DB for relevant content by control
  2. Retrieve top-k similar chunks (k=5-10)
  3. Build context with control requirements
  4. Send to Bedrock Claude for analysis
  5. Process AI response and store results
```

**Bubble Workflow for Direct API:**
```javascript
// Pseudo-code for direct processing
When Document is uploaded:
  1. Store file in Bubble file storage
  2. Extract basic metadata (filename, size, type)
  
When Audit is triggered:
  1. Retrieve all documents for specific control
  2. Send files directly to Bedrock multimodal API
  3. Include control requirements in prompt
  4. Process response and store results
```

**Hybrid Approach Implementation:**
```javascript
// Smart routing based on document characteristics
When Document is uploaded:
  if (document.type === 'text' || document.type === 'pdf') {
    // Vector DB processing
    processForVectorStorage(document)
  } else if (document.type === 'image' || document.hasVisualElements) {
    // Direct multimodal processing
    storeForDirectProcessing(document)
  } else {
    // Default to vector processing
    processForVectorStorage(document)
  }

When Audit is triggered:
  textEvidence = queryVectorDB(control.requirements)
  visualEvidence = getDirectProcessingFiles(control.id)
  
  results = combineAnalysis(
    analyzeTextEvidence(textEvidence),
    analyzeVisualEvidence(visualEvidence)
  )
```