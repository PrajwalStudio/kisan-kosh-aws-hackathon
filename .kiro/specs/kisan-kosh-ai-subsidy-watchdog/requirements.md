# Requirements Document: Kisan-Kosh AI Subsidy Watchdog

## Introduction

Kisan-Kosh is an agentic accountability partner designed to bridge the governance gap in rural subsidy processing. The system actively monitors governance compliance, identifies scheme eligibility, tracks applications against official Service Level Agreements (SLAs), and assists in grievance generation through a regional-language voice interface. The system addresses the "Black Box" problem where legally defined processing timelines under the Right to Public Services (RTPS) Act are breached without applicant knowledge, while overcoming barriers of language complexity, digital literacy, and strategic leakage by middlemen.

## Glossary

- **System**: The Kisan-Kosh AI Subsidy Watchdog application
- **User**: A rural citizen, typically a small or marginal farmer, seeking subsidy information
- **SLA**: Service Level Agreement - legally defined processing timeline under RTPS Act
- **Citizen_Charter**: Official government document defining service delivery standards and timelines
- **Application_Screenshot**: Digital image of subsidy application submission receipt
- **Land_Record**: Government document (Pahani, Satbara, etc.) containing land ownership details
- **Survey_Number**: Unique identifier for a land parcel in government records
- **Grievance**: Formal complaint filed when SLA is breached
- **Regional_Language**: Local language spoken by the user (e.g., Kannada, Hindi, Marathi)
- **OCR_Engine**: Optical Character Recognition system for extracting text from images
- **RAG_System**: Retrieval-Augmented Generation system for querying knowledge base
- **Voice_Interface**: Speech-to-text and text-to-speech system for voice interaction
- **Scheme**: Government subsidy or benefit program
- **RTPS_Act**: Right to Public Services Act governing service delivery timelines

## Requirements

### Requirement 1: SLA Breach Detection

**User Story:** As a rural citizen, I want to know if my subsidy application processing has exceeded the legal timeline, so that I can take timely action to file a grievance.

#### Acceptance Criteria

1. WHEN a User uploads an Application_Screenshot, THE OCR_Engine SHALL extract the submission date with at least 95% accuracy
2. WHEN the submission date is extracted, THE System SHALL retrieve the applicable SLA from the Citizen_Charter using the RAG_System
3. WHEN the current date exceeds the submission date plus the SLA duration, THE System SHALL generate an SLA breach alert
4. WHEN an SLA breach is detected, THE System SHALL calculate the number of days overdue
5. IF the OCR_Engine cannot extract a valid date, THEN THE System SHALL request manual date input from the User
6. WHEN multiple dates are detected in the Application_Screenshot, THE System SHALL identify the submission date using contextual keywords

### Requirement 2: Regional Language Voice Interface

**User Story:** As a rural citizen with limited literacy, I want to interact with the system using my regional language through voice, so that I can access subsidy information without language barriers.

#### Acceptance Criteria

1. WHEN a User speaks in Regional_Language, THE Voice_Interface SHALL transcribe the speech to text with at least 90% accuracy
2. WHEN the System generates a response, THE Voice_Interface SHALL convert text to speech in the User's Regional_Language
3. WHEN complex government terminology is encountered, THE System SHALL translate it into village-level analogies appropriate for the Regional_Language
4. WHERE the User selects a Regional_Language preference, THE System SHALL maintain that language throughout the session
5. WHEN audio quality is poor, THE System SHALL request the User to repeat the input
6. THE System SHALL support at least three Regional_Languages (Kannada, Hindi, Marathi)

### Requirement 3: Land Record Data Extraction

**User Story:** As a farmer, I want to upload photos of my land records, so that the system can automatically identify which subsidy schemes I am eligible for.

#### Acceptance Criteria

1. WHEN a User uploads a Land_Record image, THE OCR_Engine SHALL extract the Survey_Number with at least 90% accuracy
2. WHEN a Land_Record is processed, THE OCR_Engine SHALL extract the land area in acres or hectares
3. WHEN a Land_Record is processed, THE OCR_Engine SHALL extract the land category (irrigated, dry, forest, etc.)
4. WHEN land data is extracted, THE System SHALL validate the Survey_Number format against regional patterns
5. IF the OCR_Engine cannot extract required fields, THEN THE System SHALL request manual input via Voice_Interface
6. WHEN extraction is complete, THE System SHALL store the extracted data for eligibility matching

### Requirement 4: Scheme Eligibility Matching

**User Story:** As a farmer, I want to know which subsidy schemes I am eligible for based on my land records, so that I can apply for the right benefits.

#### Acceptance Criteria

1. WHEN land data is extracted, THE System SHALL query the RAG_System for applicable schemes based on land area
2. WHEN land data is extracted, THE System SHALL query the RAG_System for applicable schemes based on land category
3. WHEN land data is extracted, THE System SHALL query the RAG_System for applicable schemes based on Survey_Number location
4. WHEN eligible schemes are identified, THE System SHALL rank them by benefit amount in descending order
5. WHEN no schemes match the criteria, THE System SHALL suggest the closest matching schemes with explanation
6. WHEN schemes are presented, THE System SHALL include eligibility criteria, benefit amount, and application process for each scheme

### Requirement 5: Grievance Document Generation

**User Story:** As a rural citizen facing an SLA breach, I want the system to generate a formal grievance document, so that I can file a complaint without needing legal expertise.

#### Acceptance Criteria

1. WHEN an SLA breach is detected, THE System SHALL generate a formal grievance draft in English
2. WHEN a grievance is generated, THE System SHALL include the application submission date, SLA deadline, and days overdue
3. WHEN a grievance is generated, THE System SHALL include relevant RTPS Act sections and Citizen_Charter references
4. WHEN a grievance is generated, THE System SHALL provide a simplified explanation in Regional_Language via Voice_Interface
5. WHEN a grievance is generated, THE System SHALL format it according to official grievance submission standards
6. WHEN the User requests modifications, THE System SHALL allow editing of applicant details while preserving legal language

### Requirement 6: Government Document Summarization

**User Story:** As a rural citizen, I want to understand complex government PDFs about schemes and policies, so that I can make informed decisions without reading lengthy documents.

#### Acceptance Criteria

1. WHEN a User uploads a government PDF, THE System SHALL extract the text content
2. WHEN a PDF is processed, THE System SHALL generate a summary not exceeding 200 words
3. WHEN a summary is generated, THE System SHALL convert it to a 30-second audio explanation in Regional_Language
4. WHEN summarizing, THE System SHALL prioritize eligibility criteria, benefit amounts, and application deadlines
5. WHEN technical terms are encountered, THE System SHALL replace them with village-level analogies
6. WHEN the PDF contains tables or forms, THE System SHALL describe their purpose in simple language

### Requirement 7: Data Persistence and Session Management

**User Story:** As a user, I want my uploaded documents and extracted data to be saved, so that I can return to check my application status without re-uploading everything.

#### Acceptance Criteria

1. WHEN a User uploads an Application_Screenshot, THE System SHALL store it in persistent storage
2. WHEN land data is extracted, THE System SHALL associate it with the User's session identifier
3. WHEN a User returns to the System, THE System SHALL retrieve previously extracted data using the session identifier
4. WHEN sensitive data is stored, THE System SHALL encrypt it at rest
5. WHEN a User requests data deletion, THE System SHALL remove all associated data within 24 hours
6. THE System SHALL retain user data for a maximum of 90 days unless explicitly deleted

### Requirement 8: SLA Calculation and Date Processing

**User Story:** As a user, I want accurate calculation of SLA deadlines considering working days and holidays, so that breach detection is legally correct.

#### Acceptance Criteria

1. WHEN calculating SLA deadlines, THE System SHALL exclude Sundays and national holidays
2. WHEN calculating SLA deadlines, THE System SHALL exclude state-specific holidays based on the User's location
3. WHEN the submission date is extracted, THE System SHALL validate it is not a future date
4. WHEN the SLA duration is retrieved, THE System SHALL convert it to calendar days accounting for non-working days
5. WHEN calculating days overdue, THE System SHALL use the current date in Indian Standard Time (IST)
6. WHEN a Citizen_Charter specifies working days, THE System SHALL interpret it as excluding weekends and holidays

### Requirement 9: Knowledge Base Management

**User Story:** As a system administrator, I want the system to maintain an up-to-date knowledge base of Citizen Charters and scheme documents, so that users receive accurate information.

#### Acceptance Criteria

1. WHEN a new Citizen_Charter is added, THE System SHALL index it in the RAG_System within 1 hour
2. WHEN a Citizen_Charter is updated, THE System SHALL replace the old version and re-index
3. WHEN querying the RAG_System, THE System SHALL retrieve the most relevant document with at least 85% semantic accuracy
4. WHEN multiple Citizen_Charters match a query, THE System SHALL prioritize by document recency
5. THE System SHALL store Citizen_Charters with metadata including state, department, and effective date
6. WHEN a Citizen_Charter is older than 2 years, THE System SHALL flag it for administrator review

### Requirement 10: Error Handling and User Guidance

**User Story:** As a user with limited technical knowledge, I want clear guidance when something goes wrong, so that I can successfully complete my task.

#### Acceptance Criteria

1. WHEN OCR extraction fails, THE System SHALL provide voice guidance in Regional_Language explaining the issue
2. WHEN image quality is insufficient, THE System SHALL suggest specific improvements (better lighting, focus, angle)
3. WHEN the RAG_System cannot find relevant information, THE System SHALL inform the User and suggest alternative actions
4. WHEN network connectivity is lost, THE System SHALL save the current state and resume when connection is restored
5. IF an error occurs during voice transcription, THEN THE System SHALL request the User to repeat more slowly
6. WHEN the System encounters an unexpected error, THE System SHALL log it for administrator review and provide a user-friendly message

### Requirement 11: Multi-Document Processing

**User Story:** As a user applying for multiple schemes, I want to upload multiple land records and application screenshots, so that I can track all my applications in one place.

#### Acceptance Criteria

1. WHEN a User uploads multiple Land_Records, THE System SHALL process each one independently
2. WHEN multiple Land_Records are processed, THE System SHALL aggregate eligibility across all land parcels
3. WHEN a User uploads multiple Application_Screenshots, THE System SHALL track SLA status for each application separately
4. WHEN displaying multiple applications, THE System SHALL sort them by urgency (days until SLA breach)
5. WHEN aggregating land data, THE System SHALL calculate total land area across all records
6. WHEN multiple documents are uploaded, THE System SHALL provide a summary of all extracted data for User verification

### Requirement 12: Accessibility and Usability

**User Story:** As a user with limited digital literacy, I want a simple and intuitive interface, so that I can use the system without technical assistance.

#### Acceptance Criteria

1. WHEN the System starts, THE Voice_Interface SHALL provide audio instructions in Regional_Language
2. WHEN a User completes an action, THE System SHALL provide audio confirmation
3. THE System SHALL limit each voice prompt to a maximum of 15 seconds
4. WHEN presenting options, THE System SHALL limit choices to a maximum of 4 at a time
5. WHEN the User is silent for more than 10 seconds, THE System SHALL repeat the last prompt
6. THE System SHALL use consistent audio cues for success, error, and information messages
