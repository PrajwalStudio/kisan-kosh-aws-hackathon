# Design Document: Kisan-Kosh AI Subsidy Watchdog

## Overview

Kisan-Kosh is an agentic accountability system built on AWS that empowers rural citizens to monitor subsidy application processing, understand eligibility, and take action when Service Level Agreements (SLAs) are breached. The system uses a multimodal approach combining OCR, voice interfaces, and AI-powered reasoning to overcome barriers of language, literacy, and legal complexity.

The architecture leverages AWS managed services to provide a scalable, serverless solution that can handle regional language processing, document analysis, and knowledge retrieval with minimal operational overhead.

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Interface Layer                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Voice In   │  │  Image Upload│  │  Voice Out   │         │
│  │  (Transcribe)│  │     (S3)     │  │   (Polly)    │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Orchestration Layer                         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         AWS Lambda (Workflow Controller)                  │  │
│  │  - Request routing                                        │  │
│  │  - State management                                       │  │
│  │  - Error handling                                         │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Processing Layer                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Textract   │  │   Bedrock    │  │  OpenSearch  │         │
│  │  (OCR/Doc    │  │  (Claude 3.5 │  │  (Vector DB) │         │
│  │  Analysis)   │  │   Sonnet)    │  │              │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Data Layer                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   S3 Bucket  │  │   DynamoDB   │  │  Parameter   │         │
│  │  (Documents) │  │  (Sessions)  │  │    Store     │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

### Component Interaction Flow

1. **Voice Input Flow**: User speaks → Transcribe → Lambda → Bedrock (intent understanding) → Response
2. **Document Upload Flow**: User uploads image → S3 → Lambda → Textract → Lambda → DynamoDB
3. **SLA Check Flow**: Lambda → DynamoDB (get submission date) → OpenSearch (get SLA) → Lambda (calculate) → Bedrock (generate alert) → Polly → User
4. **Eligibility Flow**: Textract (extract land data) → Lambda → OpenSearch (query schemes) → Bedrock (rank/explain) → Polly → User

## Components and Interfaces

### 1. Voice Interface Module

**Purpose**: Handle bidirectional voice communication in regional languages

**Components**:
- **Amazon Transcribe**: Converts speech to text with language detection
- **Amazon Polly**: Converts text to speech with neural voices
- **Language Adapter**: Maps regional languages to AWS service codes

**Interfaces**:

```typescript
interface VoiceInput {
  audioStream: Buffer;
  languageCode: string; // 'kn-IN', 'hi-IN', 'mr-IN'
  sessionId: string;
}

interface VoiceOutput {
  text: string;
  languageCode: string;
  voiceId: string; // Polly voice ID for regional language
  audioFormat: 'mp3' | 'ogg';
}

interface TranscriptionResult {
  transcript: string;
  confidence: number;
  languageCode: string;
  alternatives?: string[];
}

interface SynthesisResult {
  audioStream: Buffer;
  contentType: string;
  requestCharacters: number;
}
```

**Key Design Decisions**:
- Use streaming transcription for real-time feedback
- Implement confidence threshold (0.7) for transcription acceptance
- Cache Polly audio for common phrases to reduce latency
- Support fallback to text input if voice quality is poor

### 2. OCR and Document Processing Module

**Purpose**: Extract structured data from images of application receipts and land records

**Components**:
- **Amazon Textract**: OCR and document analysis
- **Date Extractor**: Identifies and validates dates from OCR output
- **Land Record Parser**: Extracts Survey Numbers, area, and category
- **Validation Engine**: Validates extracted data against patterns

**Interfaces**:

```typescript
interface DocumentUpload {
  imageData: Buffer;
  documentType: 'application_screenshot' | 'land_record' | 'government_pdf';
  sessionId: string;
  metadata?: {
    state?: string;
    district?: string;
  };
}

interface OCRResult {
  rawText: string;
  confidence: number;
  blocks: TextBlock[];
  keyValuePairs?: KeyValuePair[];
  tables?: Table[];
}

interface TextBlock {
  text: string;
  confidence: number;
  boundingBox: BoundingBox;
  blockType: 'LINE' | 'WORD' | 'KEY_VALUE_SET' | 'TABLE';
}

interface ApplicationData {
  submissionDate: Date;
  applicationNumber: string;
  department: string;
  schemeType: string;
  confidence: number;
}

interface LandRecordData {
  surveyNumber: string;
  landArea: number;
  landAreaUnit: 'acres' | 'hectares';
  landCategory: string;
  ownerName?: string;
  district?: string;
  confidence: number;
}
```

**Key Design Decisions**:
- Use Textract's AnalyzeDocument API with FORMS and TABLES features
- Implement multi-pass extraction: first raw OCR, then structured analysis
- Use regex patterns for Survey Number validation (state-specific)
- Store original images in S3 for manual review if confidence < 0.85

### 3. SLA Monitoring Engine

**Purpose**: Track application timelines and detect SLA breaches

**Components**:
- **Date Calculator**: Computes deadlines excluding holidays
- **Holiday Calendar**: Maintains national and state holidays
- **Breach Detector**: Compares current date against SLA deadline
- **Alert Generator**: Creates breach notifications

**Interfaces**:

```typescript
interface SLAConfiguration {
  schemeName: string;
  department: string;
  processingDays: number;
  dayType: 'working' | 'calendar';
  state: string;
  effectiveDate: Date;
  source: string; // Citizen Charter reference
}

interface ApplicationTracking {
  applicationId: string;
  submissionDate: Date;
  slaDeadline: Date;
  currentStatus: 'pending' | 'breached' | 'completed';
  daysRemaining: number;
  daysOverdue: number;
}

interface BreachAlert {
  applicationId: string;
  submissionDate: Date;
  slaDeadline: Date;
  daysOverdue: number;
  grievanceEligible: boolean;
  nextSteps: string[];
}

interface HolidayCalendar {
  state: string;
  year: number;
  holidays: Holiday[];
}

interface Holiday {
  date: Date;
  name: string;
  type: 'national' | 'state' | 'optional';
}
```

**Key Design Decisions**:
- Store holiday calendars in DynamoDB with state-wise partitioning
- Use Lambda for date calculations to avoid timezone issues
- Implement working day calculation algorithm that handles edge cases
- Cache SLA configurations in memory for frequently accessed schemes

**Date Calculation Algorithm**:
```
function calculateSLADeadline(submissionDate, processingDays, dayType, state):
  if dayType == 'calendar':
    return submissionDate + processingDays
  else:
    currentDate = submissionDate
    daysAdded = 0
    while daysAdded < processingDays:
      currentDate = currentDate + 1 day
      if not isWeekend(currentDate) and not isHoliday(currentDate, state):
        daysAdded += 1
    return currentDate
```

### 4. RAG Knowledge Base Module

**Purpose**: Store and retrieve Citizen Charters, scheme documents, and RTPS Act information

**Components**:
- **Amazon OpenSearch**: Vector database for semantic search
- **Document Indexer**: Processes and indexes documents
- **Query Engine**: Retrieves relevant information
- **Amazon Bedrock**: Generates embeddings and answers

**Interfaces**:

```typescript
interface DocumentIndex {
  documentId: string;
  documentType: 'citizen_charter' | 'scheme_document' | 'rtps_act';
  title: string;
  state: string;
  department: string;
  effectiveDate: Date;
  content: string;
  embedding: number[]; // 1536-dimensional vector
  metadata: Record<string, any>;
}

interface RAGQuery {
  query: string;
  filters?: {
    state?: string;
    department?: string;
    documentType?: string;
  };
  topK: number;
  minScore: number;
}

interface RAGResult {
  documents: RetrievedDocument[];
  answer: string;
  confidence: number;
  sources: string[];
}

interface RetrievedDocument {
  documentId: string;
  title: string;
  excerpt: string;
  relevanceScore: number;
  metadata: Record<string, any>;
}
```

**Key Design Decisions**:
- Use Amazon Bedrock Titan Embeddings for vector generation
- Implement hybrid search (keyword + semantic) for better accuracy
- Store document chunks of 512 tokens with 50-token overlap
- Use metadata filtering to narrow search by state/department
- Implement re-ranking using Claude to improve relevance

**Indexing Pipeline**:
```
PDF Upload → S3 → Lambda Trigger → 
  Textract (extract text) → 
  Chunking (512 tokens) → 
  Bedrock (generate embeddings) → 
  OpenSearch (index with metadata)
```

### 5. AI Reasoning and Translation Module

**Purpose**: Provide intelligent responses, translations, and document generation using LLM

**Components**:
- **Amazon Bedrock (Claude 3.5 Sonnet)**: Core reasoning engine
- **Prompt Manager**: Manages system prompts for different tasks
- **Translation Engine**: Converts between English and regional languages
- **Jargon Simplifier**: Replaces technical terms with analogies

**Interfaces**:

```typescript
interface LLMRequest {
  task: 'translate' | 'simplify' | 'generate_grievance' | 'summarize' | 'explain';
  input: string;
  context?: string;
  targetLanguage?: string;
  parameters?: {
    maxTokens?: number;
    temperature?: number;
  };
}

interface LLMResponse {
  output: string;
  confidence: number;
  tokensUsed: number;
  model: string;
}

interface TranslationRequest {
  text: string;
  sourceLanguage: string;
  targetLanguage: string;
  domain: 'legal' | 'agricultural' | 'general';
  simplify: boolean;
}

interface GrievanceRequest {
  applicantName: string;
  applicationNumber: string;
  submissionDate: Date;
  slaDeadline: Date;
  daysOverdue: number;
  schemeName: string;
  department: string;
  citizenCharterReference: string;
}

interface GrievanceDocument {
  englishVersion: string;
  regionalExplanation: string;
  attachments: string[];
  submissionInstructions: string;
}
```

**Key Design Decisions**:
- Use Claude 3.5 Sonnet for complex reasoning and translation
- Implement few-shot prompting with regional language examples
- Create domain-specific glossaries for agricultural and legal terms
- Use temperature=0.3 for factual tasks, 0.7 for creative explanations
- Implement output validation to ensure translations preserve meaning

**Prompt Templates**:

```
System Prompt for Translation:
"You are a translator specializing in Indian government documents. 
Translate the following text from English to {language}, using 
village-level analogies for complex terms. Maintain accuracy while 
ensuring a 5th-grade reading level."

System Prompt for Grievance Generation:
"You are a legal assistant helping rural citizens file grievances 
under the RTPS Act. Generate a formal complaint in English following 
official format, including: applicant details, application timeline, 
SLA breach details, relevant legal sections, and requested action."
```

### 6. Scheme Eligibility Matcher

**Purpose**: Match user land records against scheme eligibility criteria

**Components**:
- **Criteria Parser**: Extracts eligibility rules from scheme documents
- **Matching Engine**: Evaluates user data against criteria
- **Ranking Algorithm**: Prioritizes schemes by benefit amount
- **Explanation Generator**: Creates user-friendly explanations

**Interfaces**:

```typescript
interface SchemeDefinition {
  schemeId: string;
  schemeName: string;
  department: string;
  state: string;
  eligibilityCriteria: EligibilityCriteria;
  benefitAmount: number;
  applicationProcess: string;
  documents: string[];
}

interface EligibilityCriteria {
  minLandArea?: number;
  maxLandArea?: number;
  landCategories?: string[];
  farmerType?: ('small' | 'marginal' | 'large')[];
  additionalConditions?: string[];
}

interface EligibilityMatch {
  scheme: SchemeDefinition;
  matchScore: number;
  matchedCriteria: string[];
  unmatchedCriteria: string[];
  explanation: string;
}

interface EligibilityRequest {
  landRecords: LandRecordData[];
  userLocation: {
    state: string;
    district: string;
  };
}

interface EligibilityResponse {
  eligibleSchemes: EligibilityMatch[];
  ineligibleSchemes: EligibilityMatch[];
  recommendations: string[];
}
```

**Key Design Decisions**:
- Store scheme definitions in DynamoDB with GSI on state
- Use rule-based matching for objective criteria (land area, category)
- Use LLM for subjective criteria evaluation
- Rank schemes by: eligibility match score × benefit amount
- Provide explanations for both eligible and ineligible schemes

**Matching Algorithm**:
```
function matchSchemes(landRecords, schemes):
  totalLandArea = sum(record.landArea for record in landRecords)
  landCategories = unique(record.landCategory for record in landRecords)
  
  matches = []
  for scheme in schemes:
    score = 0
    matched = []
    unmatched = []
    
    if scheme.minLandArea and totalLandArea >= scheme.minLandArea:
      score += 1
      matched.append("Minimum land area")
    elif scheme.minLandArea:
      unmatched.append("Minimum land area")
    
    if scheme.landCategories and any(cat in scheme.landCategories for cat in landCategories):
      score += 1
      matched.append("Land category")
    elif scheme.landCategories:
      unmatched.append("Land category")
    
    matches.append({
      scheme: scheme,
      matchScore: score / totalCriteria,
      matchedCriteria: matched,
      unmatchedCriteria: unmatched
    })
  
  return sorted(matches, key=lambda x: x.matchScore * x.scheme.benefitAmount, reverse=True)
```

### 7. Session and State Management

**Purpose**: Maintain user context across interactions

**Components**:
- **DynamoDB**: Persistent session storage
- **Session Manager**: CRUD operations for sessions
- **State Machine**: Tracks conversation flow

**Interfaces**:

```typescript
interface UserSession {
  sessionId: string;
  userId?: string;
  languagePreference: string;
  createdAt: Date;
  lastAccessedAt: Date;
  expiresAt: Date;
  state: SessionState;
  data: SessionData;
}

interface SessionState {
  currentFlow: 'sla_check' | 'eligibility' | 'grievance' | 'document_explain';
  currentStep: string;
  awaitingInput: boolean;
  inputType?: 'voice' | 'image' | 'text';
}

interface SessionData {
  applications?: ApplicationTracking[];
  landRecords?: LandRecordData[];
  uploadedDocuments?: string[]; // S3 keys
  eligibleSchemes?: EligibilityMatch[];
  grievances?: GrievanceDocument[];
}
```

**Key Design Decisions**:
- Use sessionId as partition key in DynamoDB
- Set TTL to 90 days for automatic cleanup
- Implement session encryption for sensitive data
- Use optimistic locking to prevent concurrent updates
- Cache active sessions in Lambda memory (5-minute TTL)

## Data Models

### DynamoDB Tables

**Table: UserSessions**
```
Partition Key: sessionId (String)
Attributes:
  - userId (String, optional)
  - languagePreference (String)
  - createdAt (Number, timestamp)
  - lastAccessedAt (Number, timestamp)
  - expiresAt (Number, TTL)
  - state (Map)
  - data (Map)
```

**Table: ApplicationTracking**
```
Partition Key: sessionId (String)
Sort Key: applicationId (String)
Attributes:
  - submissionDate (Number, timestamp)
  - slaDeadline (Number, timestamp)
  - currentStatus (String)
  - daysRemaining (Number)
  - daysOverdue (Number)
  - schemeType (String)
  - department (String)
  - documentS3Key (String)
GSI: statusIndex (currentStatus, slaDeadline)
```

**Table: SchemeDefinitions**
```
Partition Key: schemeId (String)
Attributes:
  - schemeName (String)
  - department (String)
  - state (String)
  - eligibilityCriteria (Map)
  - benefitAmount (Number)
  - applicationProcess (String)
  - documents (List)
  - lastUpdated (Number, timestamp)
GSI: stateIndex (state, benefitAmount)
```

**Table: HolidayCalendar**
```
Partition Key: stateYear (String, format: "STATE-YYYY")
Attributes:
  - holidays (List of Maps)
    - date (String, ISO format)
    - name (String)
    - type (String)
```

### S3 Bucket Structure

```
kisan-kosh-documents/
├── uploads/
│   ├── {sessionId}/
│   │   ├── application-screenshots/
│   │   │   └── {timestamp}-{filename}
│   │   ├── land-records/
│   │   │   └── {timestamp}-{filename}
│   │   └── government-pdfs/
│   │       └── {timestamp}-{filename}
├── knowledge-base/
│   ├── citizen-charters/
│   │   └── {state}/{department}/{filename}
│   ├── scheme-documents/
│   │   └── {state}/{scheme-id}/{filename}
│   └── rtps-documents/
│       └── {state}/{filename}
└── generated/
    └── grievances/
        └── {sessionId}/{applicationId}-grievance.pdf
```

### OpenSearch Index Schema

**Index: citizen_charters**
```json
{
  "mappings": {
    "properties": {
      "document_id": {"type": "keyword"},
      "title": {"type": "text"},
      "content": {"type": "text"},
      "embedding": {
        "type": "knn_vector",
        "dimension": 1536
      },
      "state": {"type": "keyword"},
      "department": {"type": "keyword"},
      "effective_date": {"type": "date"},
      "document_type": {"type": "keyword"},
      "sla_days": {"type": "integer"},
      "scheme_name": {"type": "text"}
    }
  }
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*


### Property 1: OCR Date Extraction Accuracy

*For any* application screenshot containing a valid submission date, when processed by the OCR Engine, the extracted date should match the actual date with at least 95% accuracy across a large sample set.

**Validates: Requirements 1.1**

### Property 2: SLA Retrieval Correctness

*For any* extracted submission date and scheme type, the RAG System should retrieve the applicable SLA from the correct Citizen Charter for that scheme and state.

**Validates: Requirements 1.2**

### Property 3: SLA Breach Detection Correctness

*For any* submission date and SLA duration, when the current date exceeds the submission date plus the SLA duration (accounting for working days), the System should generate an SLA breach alert with the correct number of days overdue.

**Validates: Requirements 1.3, 1.4**

### Property 4: OCR Failure Fallback

*For any* image where the OCR Engine cannot extract a valid date (confidence below threshold or no date detected), the System should request manual date input from the User.

**Validates: Requirements 1.5**

### Property 5: Date Disambiguation

*For any* application screenshot containing multiple dates, the System should correctly identify the submission date using contextual keywords (e.g., "submitted on", "application date").

**Validates: Requirements 1.6**

### Property 6: Voice Transcription Accuracy

*For any* voice input in a supported Regional Language, the Voice Interface should transcribe the speech to text with at least 90% accuracy across a large sample set.

**Validates: Requirements 2.1**

### Property 7: Speech Synthesis Capability

*For any* text response in a supported Regional Language, the Voice Interface should successfully convert it to speech audio.

**Validates: Requirements 2.2**

### Property 8: Language Preference Persistence

*For any* user session where a Regional Language preference is set, all subsequent interactions in that session should maintain the same language preference.

**Validates: Requirements 2.4**

### Property 9: Audio Quality Detection

*For any* voice input with poor audio quality (high noise, low volume, or low confidence score), the System should request the User to repeat the input.

**Validates: Requirements 2.5**

### Property 10: Land Record Data Extraction Completeness

*For any* land record image, when processed by the OCR Engine, the System should extract the Survey Number, land area, and land category, and store all extracted fields with their confidence scores.

**Validates: Requirements 3.1, 3.2, 3.3, 3.6**

### Property 11: Survey Number Format Validation

*For any* extracted Survey Number, the System should validate it against the regional pattern for the user's state, accepting valid formats and rejecting invalid ones.

**Validates: Requirements 3.4**

### Property 12: Extraction Failure Fallback

*For any* land record where required fields cannot be extracted (confidence below threshold), the System should request manual input via the Voice Interface.

**Validates: Requirements 3.5**

### Property 13: Multi-Criteria Scheme Querying

*For any* set of land records with extracted data (area, category, location), the System should query the RAG System for applicable schemes matching any of these criteria.

**Validates: Requirements 4.1, 4.2, 4.3**

### Property 14: Scheme Ranking by Benefit

*For any* set of eligible schemes, the System should rank them in descending order by benefit amount.

**Validates: Requirements 4.4**

### Property 15: Fallback Scheme Suggestions

*For any* land record data where no schemes exactly match the eligibility criteria, the System should suggest the closest matching schemes with explanations of unmet criteria.

**Validates: Requirements 4.5**

### Property 16: Scheme Information Completeness

*For any* scheme presented to the user, the System should include eligibility criteria, benefit amount, and application process information.

**Validates: Requirements 4.6**

### Property 17: Grievance Generation on Breach

*For any* detected SLA breach, the System should generate a formal grievance draft in English.

**Validates: Requirements 5.1**

### Property 18: Grievance Content Completeness

*For any* generated grievance, the document should include the application submission date, SLA deadline, days overdue, relevant RTPS Act sections, and Citizen Charter references.

**Validates: Requirements 5.2, 5.3**

### Property 19: Grievance Regional Explanation

*For any* generated grievance, the System should provide a simplified explanation in the user's Regional Language via the Voice Interface.

**Validates: Requirements 5.4**

### Property 20: Grievance Format Compliance

*For any* generated grievance, the document should conform to official grievance submission format standards.

**Validates: Requirements 5.5**

### Property 21: Selective Grievance Editing

*For any* grievance being edited, when the user modifies applicant details, the legal language sections should remain unchanged.

**Validates: Requirements 5.6**

### Property 22: PDF Text Extraction

*For any* government PDF uploaded by the user, the System should successfully extract the text content.

**Validates: Requirements 6.1**

### Property 23: Summary Length Constraint

*For any* processed government PDF, the generated summary should not exceed 200 words.

**Validates: Requirements 6.2**

### Property 24: Audio Summary Duration Constraint

*For any* generated summary, when converted to audio in Regional Language, the duration should not exceed 30 seconds.

**Validates: Requirements 6.3**

### Property 25: Summary Content Prioritization

*For any* government PDF containing eligibility criteria, benefit amounts, or application deadlines, the generated summary should include these elements.

**Validates: Requirements 6.4**

### Property 26: Table Description Generation

*For any* government PDF containing tables or forms, the System should generate descriptions of their purpose in the summary.

**Validates: Requirements 6.6**

### Property 27: Document Storage Persistence

*For any* uploaded application screenshot, the System should store it in persistent storage (S3) and return a storage reference.

**Validates: Requirements 7.1**

### Property 28: Session Data Association

*For any* extracted land data, the System should associate it with the user's session identifier in the database.

**Validates: Requirements 7.2**

### Property 29: Session Data Retrieval Round-Trip

*For any* user session, data stored during the session should be retrievable using the session identifier, and the retrieved data should match the stored data.

**Validates: Requirements 7.3**

### Property 30: Data Encryption at Rest

*For any* sensitive data stored in the system (application screenshots, land records, personal information), the data should be encrypted at rest in storage.

**Validates: Requirements 7.4**

### Property 31: Data Deletion Timing

*For any* user data deletion request, all associated data should be removed from the system within 24 hours.

**Validates: Requirements 7.5**

### Property 32: Data Retention TTL

*For any* user data stored in the system, if not explicitly deleted, it should be automatically removed after 90 days.

**Validates: Requirements 7.6**

### Property 33: Working Day SLA Calculation

*For any* submission date, SLA duration in working days, and user location (state), the calculated SLA deadline should exclude Sundays, national holidays, and state-specific holidays.

**Validates: Requirements 8.1, 8.2, 8.6**

### Property 34: Future Date Validation

*For any* extracted submission date, if the date is in the future, the System should reject it and request correction.

**Validates: Requirements 8.3**

### Property 35: SLA Duration Conversion

*For any* SLA duration specified in working days, the System should convert it to calendar days by accounting for non-working days (weekends and holidays).

**Validates: Requirements 8.4**

### Property 36: Timezone Consistency

*For any* date calculation (SLA deadline, days overdue), the System should use the current date in Indian Standard Time (IST).

**Validates: Requirements 8.5**

### Property 37: Document Indexing Performance

*For any* new Citizen Charter added to the knowledge base, the System should index it in the RAG System within 1 hour.

**Validates: Requirements 9.1**

### Property 38: Document Update and Re-indexing

*For any* Citizen Charter update, the System should replace the old version in the knowledge base and re-index the new version.

**Validates: Requirements 9.2**

### Property 39: RAG Retrieval Accuracy

*For any* query to the RAG System with a known relevant document, the System should retrieve the most relevant document with at least 85% semantic accuracy across a large sample set.

**Validates: Requirements 9.3**

### Property 40: Document Recency Prioritization

*For any* RAG query that matches multiple Citizen Charters, the System should prioritize results by document recency (effective date).

**Validates: Requirements 9.4**

### Property 41: Document Metadata Completeness

*For any* Citizen Charter stored in the knowledge base, the System should store metadata including state, department, and effective date.

**Validates: Requirements 9.5**

### Property 42: Document Age Flagging

*For any* Citizen Charter in the knowledge base older than 2 years, the System should flag it for administrator review.

**Validates: Requirements 9.6**

### Property 43: Error Guidance in Regional Language

*For any* error condition (OCR failure, transcription error, unexpected error), the System should provide voice guidance in the user's Regional Language explaining the issue and suggesting next steps.

**Validates: Requirements 10.1, 10.5, 10.6**

### Property 44: Image Quality Feedback

*For any* uploaded image with insufficient quality (blur, poor lighting, wrong angle), the System should provide specific improvement suggestions.

**Validates: Requirements 10.2**

### Property 45: RAG Fallback Messaging

*For any* RAG query that returns no relevant results, the System should inform the user and suggest alternative actions.

**Validates: Requirements 10.3**

### Property 46: State Persistence on Network Failure

*For any* network connectivity loss during a session, the System should save the current state, and when connection is restored, the session should resume from the saved state.

**Validates: Requirements 10.4**

### Property 47: Independent Multi-Document Processing

*For any* set of multiple uploaded documents (land records or application screenshots), the System should process each document independently and track results separately.

**Validates: Requirements 11.1, 11.3**

### Property 48: Land Data Aggregation

*For any* set of multiple land records, the System should aggregate eligibility across all parcels and calculate the total land area.

**Validates: Requirements 11.2, 11.5**

### Property 49: Application Urgency Sorting

*For any* set of multiple tracked applications, when displayed to the user, they should be sorted by urgency (days until SLA breach, with breached applications first).

**Validates: Requirements 11.4**

### Property 50: Multi-Document Extraction Summary

*For any* set of multiple uploaded documents, after processing, the System should provide a summary of all extracted data for user verification.

**Validates: Requirements 11.6**

### Property 51: Action Confirmation Feedback

*For any* completed user action (upload, query, generation), the System should provide audio confirmation in the user's Regional Language.

**Validates: Requirements 12.2**

### Property 52: Voice Prompt Duration Limit

*For any* voice prompt generated by the System, the audio duration should not exceed 15 seconds.

**Validates: Requirements 12.3**

### Property 53: Option Count Limit

*For any* presentation of choices to the user, the System should limit the number of options to a maximum of 4 at a time.

**Validates: Requirements 12.4**

### Property 54: Silence Timeout Handling

*For any* user interaction where the user is silent for more than 10 seconds, the System should repeat the last prompt.

**Validates: Requirements 12.5**

### Property 55: Audio Cue Consistency

*For any* system message (success, error, information), the System should use consistent audio cues for each message type across all interactions.

**Validates: Requirements 12.6**

## Error Handling

### Error Categories

The system implements comprehensive error handling across four categories:

1. **Input Errors**: Invalid or poor-quality user inputs
2. **Processing Errors**: Failures in OCR, transcription, or AI processing
3. **System Errors**: Infrastructure failures (network, storage, service limits)
4. **Data Errors**: Missing or invalid data in knowledge base

### Error Handling Strategy

**Graceful Degradation**:
- When OCR confidence is low, request manual input rather than failing
- When voice transcription fails, offer text input alternative
- When RAG retrieval finds no results, suggest related schemes

**User-Friendly Messaging**:
- All error messages delivered via voice in Regional Language
- Avoid technical jargon in error explanations
- Provide actionable next steps with every error

**Retry Logic**:
- Automatic retry for transient failures (network, service throttling)
- Exponential backoff for AWS service calls
- Maximum 3 retry attempts before requesting user intervention

**Error Logging**:
- All errors logged to CloudWatch with context
- Critical errors trigger SNS notifications to administrators
- User-facing errors include correlation IDs for support

### Specific Error Scenarios

**OCR Extraction Failure**:
```
Error: Low confidence in date extraction
User Message: "मुझे आपकी रसीद पर तारीख स्पष्ट नहीं दिख रही है। कृपया तारीख बोलें।"
(I cannot clearly see the date on your receipt. Please speak the date.)
Fallback: Voice input for manual date entry
```

**Voice Transcription Failure**:
```
Error: Audio quality too poor for transcription
User Message: "मुझे आपकी आवाज़ ठीक से सुनाई नहीं दी। कृपया धीरे-धीरे दोहराएं।"
(I could not hear you clearly. Please repeat slowly.)
Fallback: Request slower speech or offer text input
```

**SLA Not Found**:
```
Error: No matching SLA in knowledge base
User Message: "मुझे इस योजना के लिए समय सीमा नहीं मिली। क्या आप योजना का नाम फिर से बता सकते हैं?"
(I could not find the timeline for this scheme. Can you tell me the scheme name again?)
Fallback: Request scheme clarification or manual SLA input
```

**Network Failure**:
```
Error: Connection lost during processing
System Action: Save current state to DynamoDB
User Message: "कनेक्शन टूट गया। आपका डेटा सुरक्षित है। कृपया दोबारा कोशिश करें।"
(Connection lost. Your data is safe. Please try again.)
Recovery: Resume from saved state when connection restored
```

**Service Limit Exceeded**:
```
Error: Bedrock throttling limit reached
System Action: Queue request with exponential backoff
User Message: "बहुत सारे लोग अभी सिस्टम का उपयोग कर रहे हैं। कृपया 30 सेकंड प्रतीक्षा करें।"
(Many people are using the system now. Please wait 30 seconds.)
Recovery: Automatic retry after backoff period
```

## Testing Strategy

### Dual Testing Approach

The system requires both unit testing and property-based testing for comprehensive coverage:

**Unit Tests**: Focus on specific examples, edge cases, and integration points
- Specific date formats and edge cases (leap years, month boundaries)
- Known Survey Number patterns for different states
- Sample grievance documents with known content
- Integration between Lambda functions and AWS services
- Error conditions with specific inputs

**Property-Based Tests**: Verify universal properties across all inputs
- OCR accuracy across randomly generated screenshots
- Date calculation correctness for random submission dates
- Language consistency across random session flows
- Data persistence round-trips for random data
- Scheme matching for random land record combinations

### Property-Based Testing Configuration

**Framework**: Use `fast-check` for TypeScript/JavaScript Lambda functions

**Test Configuration**:
- Minimum 100 iterations per property test
- Each test tagged with feature name and property number
- Tag format: `Feature: kisan-kosh-ai-subsidy-watchdog, Property {N}: {property_text}`

**Example Property Test Structure**:
```typescript
import fc from 'fast-check';

// Feature: kisan-kosh-ai-subsidy-watchdog, Property 3: SLA Breach Detection Correctness
test('SLA breach detection calculates days overdue correctly', () => {
  fc.assert(
    fc.property(
      fc.date(), // submission date
      fc.integer({ min: 1, max: 90 }), // SLA duration in days
      fc.integer({ min: 0, max: 180 }), // days elapsed
      (submissionDate, slaDays, daysElapsed) => {
        const currentDate = new Date(submissionDate.getTime() + daysElapsed * 24 * 60 * 60 * 1000);
        const result = checkSLABreach(submissionDate, slaDays, currentDate);
        
        if (daysElapsed > slaDays) {
          expect(result.breached).toBe(true);
          expect(result.daysOverdue).toBe(daysElapsed - slaDays);
        } else {
          expect(result.breached).toBe(false);
        }
      }
    ),
    { numRuns: 100 }
  );
});
```

### Test Coverage Requirements

**Unit Test Coverage**:
- All Lambda function handlers: 100%
- Business logic functions: 90%
- Error handling paths: 85%
- Integration points: 80%

**Property Test Coverage**:
- All 55 correctness properties must have corresponding property tests
- Each property test must run minimum 100 iterations
- Property tests must use appropriate generators for domain data

### Testing AWS Services

**Mocking Strategy**:
- Use AWS SDK mocks for unit tests
- Use LocalStack for integration tests
- Use actual AWS services for end-to-end tests in staging

**Service-Specific Testing**:
- **Textract**: Test with sample images of varying quality
- **Bedrock**: Test with mock responses for deterministic tests
- **OpenSearch**: Test with in-memory vector store for unit tests
- **Transcribe/Polly**: Test with pre-recorded audio samples
- **DynamoDB**: Test with DynamoDB Local for integration tests

### Performance Testing

**Load Testing**:
- Simulate 100 concurrent users
- Test OCR processing time (target: < 5 seconds per image)
- Test RAG query response time (target: < 2 seconds)
- Test end-to-end flow (target: < 15 seconds)

**Accuracy Testing**:
- OCR accuracy: Test with 1000 sample screenshots (target: 95%)
- Voice transcription: Test with 500 audio samples (target: 90%)
- RAG retrieval: Test with 200 queries (target: 85% relevance)

### Continuous Testing

**CI/CD Pipeline**:
1. Unit tests run on every commit
2. Property tests run on every pull request
3. Integration tests run on merge to main
4. End-to-end tests run before deployment
5. Performance tests run weekly in staging

**Test Data Management**:
- Maintain repository of sample documents (anonymized)
- Generate synthetic test data for property tests
- Update test data when new states/schemes are added
- Version control test data alongside code
