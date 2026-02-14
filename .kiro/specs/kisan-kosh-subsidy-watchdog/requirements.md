# Requirements Document: Kisan-Kosh AI Subsidy Watchdog

## Introduction

Kisan-Kosh is an agentic accountability partner designed to bridge the governance gap in rural subsidy processing. The system actively monitors government service delivery against legally defined Service Level Agreements (SLAs) under the Right to Public Services (RTPS) Act, identifies scheme eligibility from land records, and assists citizens in generating grievances through a regional-language voice interface. The system addresses critical challenges including language barriers, legal complexity, strategic information asymmetry, and digital literacy gaps that prevent rural citizens from accessing their entitled benefits.

## Glossary

- **System**: The Kisan-Kosh AI Subsidy Watchdog application
- **User**: A rural citizen (typically a small or marginal farmer) seeking subsidy information or accountability
- **SLA**: Service Level Agreement - legally defined processing timeline for government services under RTPS Act
- **Citizen_Charter**: Official government document listing services, eligibility criteria, and processing timelines
- **OCR_Engine**: Optical Character Recognition component using Amazon Textract
- **Voice_Interface**: Multimodal component using Amazon Transcribe and Amazon Polly
- **RAG_System**: Retrieval-Augmented Generation system using Amazon OpenSearch Vector Engine
- **AI_Agent**: Amazon Bedrock (Claude 3.5 Sonnet) for reasoning and translation
- **Land_Record**: Government documents like Pahani or Satbara containing Survey Number, Land Area, and Category
- **Survey_Number**: Unique identifier for a land parcel in government records
- **Grievance**: Formal complaint document generated when SLA violations are detected
- **Regional_Language**: Local language spoken by the user (e.g., Kannada, Hindi, Marathi)
- **Jargon**: Complex government terminology in official documents

## Requirements

### Requirement 1: Document Image Processing

**User Story:** As a user, I want to upload photos of my application receipts and land records, so that the system can extract relevant information without manual typing.

#### Acceptance Criteria

1. WHEN a user uploads an image file, THE OCR_Engine SHALL extract text content from the image
2. WHEN the extracted text contains a date, THE System SHALL parse and validate the date format
3. WHEN the extracted text contains structured fields (Survey Number, Land Area, Category), THE OCR_Engine SHALL identify and extract these fields with their values
4. IF the image quality is insufficient for extraction, THEN THE System SHALL return an error message requesting a clearer image
5. THE System SHALL support common image formats (JPEG, PNG, PDF)

### Requirement 2: SLA Monitoring and Breach Detection

**User Story:** As a user, I want to know if my subsidy application is delayed beyond the legal timeline, so that I can take appropriate action.

#### Acceptance Criteria

1. WHEN a submission date is extracted from an application receipt, THE System SHALL calculate the elapsed time from submission to current date
2. WHEN the System needs SLA information for a service, THE RAG_System SHALL retrieve the official processing timeline from the Citizen_Charter
3. WHEN the elapsed time exceeds the official SLA, THE System SHALL generate a breach alert
4. THE System SHALL include the breach duration (days overdue) in the alert
5. WHEN multiple applications are tracked, THE System SHALL monitor each application independently

### Requirement 3: Eligibility Matching from Land Records

**User Story:** As a user, I want to know which subsidy schemes I am eligible for based on my land records, so that I don't miss opportunities.

#### Acceptance Criteria

1. WHEN land record data is extracted (Survey Number, Land Area, Category), THE System SHALL store this information in a structured format
2. WHEN eligibility checking is requested, THE RAG_System SHALL retrieve scheme eligibility criteria from the knowledge base
3. WHEN comparing land data against scheme criteria, THE System SHALL identify all matching schemes
4. THE System SHALL return a list of eligible schemes with their names and brief descriptions
5. IF no schemes match the land data, THEN THE System SHALL inform the user that no eligible schemes were found

### Requirement 4: Regional Language Voice Interface

**User Story:** As a user who speaks a regional language, I want to interact with the system using voice in my native language, so that I can use the system without literacy barriers.

#### Acceptance Criteria

1. WHEN a user speaks in a regional language, THE Voice_Interface SHALL transcribe the audio to text using Amazon Transcribe
2. WHEN the System generates a response, THE Voice_Interface SHALL convert text to speech in the user's regional language using Amazon Polly
3. THE System SHALL support at least one regional language (Kannada) at launch
4. WHEN complex government terminology appears in responses, THE AI_Agent SHALL translate it into village-level analogies
5. THE Voice_Interface SHALL maintain conversation context across multiple turns

### Requirement 5: Grievance Document Generation

**User Story:** As a user whose application has breached the SLA, I want the system to generate a formal complaint for me, so that I can file a grievance without legal expertise.

#### Acceptance Criteria

1. WHEN an SLA breach is detected, THE System SHALL generate a formal grievance draft in English
2. THE Grievance SHALL include the service name, submission date, SLA deadline, breach duration, and applicant details
3. WHEN a grievance is generated, THE AI_Agent SHALL create a simplified explanation in the user's regional language
4. THE System SHALL format the grievance according to official complaint templates
5. THE System SHALL provide the grievance in both downloadable text format and as audio explanation

### Requirement 6: Document Summarization and Explanation

**User Story:** As a user, I want to understand complex government documents quickly, so that I can make informed decisions without reading lengthy PDFs.

#### Acceptance Criteria

1. WHEN a user uploads a government PDF document, THE System SHALL extract the text content
2. WHEN summarization is requested, THE AI_Agent SHALL generate a concise summary of the document
3. THE System SHALL convert the summary into a 30-second audio explanation in the user's regional language
4. WHEN Jargon appears in the document, THE AI_Agent SHALL replace it with simple explanations
5. THE System SHALL preserve critical information (dates, amounts, eligibility criteria) in the summary

### Requirement 7: Knowledge Base Management

**User Story:** As a system administrator, I want to maintain an up-to-date repository of Citizen Charters and RTPS documents, so that the system provides accurate SLA information.

#### Acceptance Criteria

1. WHEN a new Citizen Charter document is added, THE RAG_System SHALL process and index it in the vector database
2. THE RAG_System SHALL extract service names, SLA timelines, and eligibility criteria from indexed documents
3. WHEN querying for SLA information, THE RAG_System SHALL retrieve the most relevant and recent document
4. THE System SHALL support versioning of Citizen Charters to track changes over time
5. WHEN a document is updated, THE RAG_System SHALL re-index the content and mark previous versions as superseded

### Requirement 8: Data Storage and Retrieval

**User Story:** As a user, I want my application tracking data to be saved, so that I can check status across multiple sessions.

#### Acceptance Criteria

1. WHEN a user uploads an application receipt, THE System SHALL store the extracted data with a unique identifier
2. WHEN a user returns to the system, THE System SHALL retrieve previously tracked applications
3. THE System SHALL persist land record data for reuse in eligibility checks
4. THE System SHALL store user language preferences for consistent experience
5. WHEN storing user data, THE System SHALL encrypt sensitive information

### Requirement 9: Workflow Orchestration

**User Story:** As a user, I want the system to guide me through the process step-by-step, so that I can complete tasks without confusion.

#### Acceptance Criteria

1. WHEN a user starts a new session, THE System SHALL present available actions (track application, check eligibility, explain document)
2. WHEN a workflow requires multiple steps, THE System SHALL execute them in the correct sequence
3. WHEN an error occurs in a workflow step, THE System SHALL provide a clear error message and recovery options
4. THE System SHALL maintain workflow state across voice interactions
5. WHEN a workflow completes, THE System SHALL summarize the results and offer next actions

### Requirement 10: Date Calculation and Validation

**User Story:** As a user, I want accurate calculation of how long my application has been pending, so that I can trust the breach alerts.

#### Acceptance Criteria

1. WHEN calculating elapsed time, THE System SHALL use the submission date from the receipt and the current date
2. THE System SHALL account for calendar variations (different month lengths, leap years)
3. WHEN comparing against SLA, THE System SHALL use business days if specified in the Citizen Charter
4. THE System SHALL validate that submission dates are not in the future
5. WHEN a date cannot be parsed, THE System SHALL request clarification from the user

### Requirement 11: Multi-Format Input Handling

**User Story:** As a user with limited technical skills, I want to submit information in whatever format is easiest for me, so that I'm not blocked by format requirements.

#### Acceptance Criteria

1. THE System SHALL accept images captured by mobile phone cameras
2. THE System SHALL accept scanned documents in PDF format
3. THE System SHALL accept voice input for queries and responses
4. WHEN a user provides information via voice, THE System SHALL confirm understanding before proceeding
5. THE System SHALL handle images with varying quality, orientation, and lighting conditions

### Requirement 12: Error Handling and User Guidance

**User Story:** As a user, I want clear guidance when something goes wrong, so that I can correct issues and continue.

#### Acceptance Criteria

1. WHEN OCR extraction fails, THE System SHALL explain the issue in simple terms and suggest solutions
2. WHEN no matching schemes are found, THE System SHALL suggest alternative actions or information sources
3. WHEN the RAG system cannot find SLA information, THE System SHALL inform the user and offer manual input options
4. IF the voice interface cannot understand the user, THEN THE System SHALL ask for clarification in the user's language
5. THE System SHALL log errors for system improvement without exposing technical details to users
