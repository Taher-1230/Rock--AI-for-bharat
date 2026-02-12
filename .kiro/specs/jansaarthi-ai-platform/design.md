# Design Document: JanSaarthi AI Platform

## Overview

JanSaarthi AI is a mobile-first web platform that bridges the gap between Indian citizens and government welfare schemes through secure authentication, intelligent filtering, and AI-powered simplification. The platform architecture follows a three-tier model: a React-based frontend for multilingual user interaction, a Node.js/Express backend for business logic and API orchestration, and a PostgreSQL database for persistent storage. The system integrates with DigiLocker OAuth for identity verification and leverages large language models for content simplification in multiple languages.

The design prioritizes mobile responsiveness, data privacy, and scalability to serve underserved communities with varying levels of digital literacy. Key architectural decisions include stateless API design for horizontal scaling, JWT-based session management, and asynchronous AI processing with fallback mechanisms.

## Architecture

### System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Client Layer                          │
│  ┌──────────────────────────────────────────────────────┐  │
│  │   React SPA (Mobile-First Responsive)                │  │
│  │   - Multilingual UI (Gujarati, Hindi, English)       │  │
│  │   - State Management (Redux)                         │  │
│  │   - Responsive Components                            │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ HTTPS/TLS 1.2+
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     Application Layer                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │   API Gateway (Express.js)                           │  │
│  │   - Authentication Middleware                        │  │
│  │   - Rate Limiting                                    │  │
│  │   - Request Validation                               │  │
│  └──────────────────────────────────────────────────────┘  │
│                            │                                 │
│  ┌─────────────┬──────────┴──────────┬──────────────────┐  │
│  │             │                      │                  │  │
│  ▼             ▼                      ▼                  ▼  │
│  ┌──────┐  ┌──────┐  ┌──────────┐  ┌─────────────────┐ │  │
│  │ Auth │  │User  │  │ Scheme   │  │ AI Simplifier   │ │  │
│  │Service│ │Service│ │  Service │  │    Service      │ │  │
│  └──────┘  └──────┘  └──────────┘  └─────────────────┘ │  │
└─────────────────────────────────────────────────────────────┘
       │           │            │                │
       │           │            │                │
       ▼           ▼            ▼                ▼
┌──────────┐ ┌─────────────────────────┐  ┌──────────────┐
│DigiLocker│ │   PostgreSQL Database   │  │  LLM API     │
│   API    │ │  - Users                │  │ (OpenAI/     │
│          │ │  - Schemes              │  │  Gemini)     │
└──────────┘ │  - Sessions             │  └──────────────┘
             │  - Consent Records      │
             └─────────────────────────┘
```

### Technology Stack

**Frontend:**
- React 18+ with TypeScript for type safety
- Redux Toolkit for state management
- React Router for navigation
- i18next for internationalization
- Tailwind CSS for responsive styling
- Axios for HTTP requests

**Backend:**
- Node.js 18+ with Express.js
- TypeScript for type safety
- Passport.js for OAuth integration
- JWT for session management
- Winston for logging
- Joi for request validation

**Database:**
- PostgreSQL 14+ for relational data
- Redis for session caching and rate limiting

**External Services:**
- DigiLocker OAuth API for authentication
- OpenAI GPT-4 or Google Gemini for AI simplification
- AWS S3 for static asset storage (optional)

**Infrastructure:**
- Docker for containerization
- AWS/GCP for cloud hosting
- Nginx as reverse proxy
- Let's Encrypt for SSL certificates

## Components and Interfaces

### 1. Authentication Service

**Responsibilities:**
- Handle DigiLocker OAuth flow
- Manage user sessions
- Validate authentication tokens
- Handle consent management

**Key Methods:**

```typescript
interface AuthService {
  // Initiate DigiLocker OAuth flow
  initiateDigiLockerAuth(): Promise<{ authUrl: string }>;
  
  // Handle OAuth callback and create session
  handleOAuthCallback(code: string): Promise<{ 
    sessionToken: string; 
    user: UserProfile 
  }>;
  
  // Validate session token
  validateSession(token: string): Promise<{ 
    valid: boolean; 
    userId: string 
  }>;
  
  // Logout and invalidate session
  logout(sessionToken: string): Promise<void>;
  
  // Record user consent
  recordConsent(userId: string, consentType: string): Promise<void>;
  
  // Revoke user consent
  revokeConsent(userId: string, consentType: string): Promise<void>;
}
```

**DigiLocker Integration:**

```typescript
interface DigiLockerClient {
  // Get authorization URL
  getAuthorizationUrl(redirectUri: string): string;
  
  // Exchange authorization code for access token
  getAccessToken(code: string): Promise<{ 
    accessToken: string; 
    expiresIn: number 
  }>;
  
  // Fetch user profile data
  getUserProfile(accessToken: string): Promise<{
    name: string;
    dob: string;
    gender: string;
    state: string;
  }>;
}
```

### 2. User Service

**Responsibilities:**
- Manage user profiles
- Handle profile updates
- Validate profile data
- Calculate user eligibility context

**Key Methods:**

```typescript
interface UserService {
  // Create user profile from DigiLocker data
  createProfile(digiLockerData: DigiLockerProfile): Promise<UserProfile>;
  
  // Get user profile by ID
  getProfile(userId: string): Promise<UserProfile>;
  
  // Update user profile
  updateProfile(userId: string, updates: Partial<UserProfile>): Promise<UserProfile>;
  
  // Delete user account and data
  deleteAccount(userId: string): Promise<void>;
  
  // Get user eligibility context for filtering
  getEligibilityContext(userId: string): Promise<EligibilityContext>;
  
  // Export user data (DPDPA compliance)
  exportUserData(userId: string): Promise<UserDataExport>;
}

interface UserProfile {
  id: string;
  name: string;
  age: number;
  state: string;
  gender: 'male' | 'female' | 'other';
  incomeBracket?: 'below-1L' | '1L-3L' | '3L-5L' | 'above-5L';
  languagePreference: 'en' | 'gu' | 'hi';
  createdAt: Date;
  updatedAt: Date;
}

interface EligibilityContext {
  age: number;
  state: string;
  gender: string;
  incomeBracket?: string;
}
```

### 3. Scheme Service

**Responsibilities:**
- Manage scheme database
- Filter schemes by eligibility
- Retrieve scheme details
- Track scheme analytics

**Key Methods:**

```typescript
interface SchemeService {
  // Get all schemes (optionally filtered by eligibility)
  getSchemes(
    eligibilityContext?: EligibilityContext,
    filters?: SchemeFilters
  ): Promise<Scheme[]>;
  
  // Get scheme by ID with full details
  getSchemeById(schemeId: string, language: string): Promise<SchemeDetail>;
  
  // Search schemes by keyword
  searchSchemes(query: string, language: string): Promise<Scheme[]>;
  
  // Check if user is eligible for specific scheme
  checkEligibility(
    schemeId: string, 
    eligibilityContext: EligibilityContext
  ): Promise<{ eligible: boolean; reasons: string[] }>;
  
  // Record scheme view for analytics
  recordSchemeView(schemeId: string, userId: string): Promise<void>;
}

interface Scheme {
  id: string;
  name: string;
  shortDescription: string;
  category: string;
  state?: string;
  eligibilityCriteria: EligibilityCriteria;
}

interface SchemeDetail extends Scheme {
  fullDescription: string;
  benefits: string[];
  requiredDocuments: Document[];
  applicationProcess: ApplicationStep[];
  contactInfo: ContactInfo;
  officialUrl?: string;
  lastUpdated: Date;
}

interface EligibilityCriteria {
  minAge?: number;
  maxAge?: number;
  gender?: string[];
  states?: string[];
  incomeBrackets?: string[];
  otherCriteria?: Record<string, any>;
}

interface Document {
  name: string;
  description: string;
  mandatory: boolean;
  availableInDigiLocker: boolean;
  obtainFrom?: string;
}

interface ApplicationStep {
  stepNumber: number;
  instruction: string;
  estimatedTime?: string;
}
```

### 4. AI Simplification Service

**Responsibilities:**
- Simplify complex government text
- Support multiple languages
- Cache simplification results
- Handle API failures gracefully

**Key Methods:**

```typescript
interface AISimplificationService {
  // Simplify text in specified language
  simplifyText(
    text: string, 
    language: 'en' | 'gu' | 'hi',
    context?: string
  ): Promise<{ 
    simplifiedText: string; 
    cached: boolean 
  }>;
  
  // Simplify scheme description
  simplifySchemeDescription(
    scheme: SchemeDetail, 
    language: string
  ): Promise<SimplifiedScheme>;
  
  // Batch simplify multiple texts
  simplifyBatch(
    texts: string[], 
    language: string
  ): Promise<string[]>;
  
  // Check if simplification is available
  isAvailable(): Promise<boolean>;
}

interface SimplifiedScheme {
  id: string;
  simplifiedDescription: string;
  simplifiedEligibility: string;
  simplifiedBenefits: string[];
  simplifiedSteps: string[];
}
```

**LLM Integration:**

```typescript
interface LLMClient {
  // Generate simplified text using LLM
  generateSimplification(
    prompt: string,
    systemPrompt: string,
    maxTokens: number
  ): Promise<string>;
  
  // Check API health and rate limits
  checkHealth(): Promise<{ available: boolean; rateLimitRemaining: number }>;
}
```

### 5. Session Management

**Responsibilities:**
- Create and validate JWT tokens
- Manage session expiration
- Handle session refresh
- Implement session security

**Key Methods:**

```typescript
interface SessionManager {
  // Create new session
  createSession(userId: string, metadata: SessionMetadata): Promise<string>;
  
  // Validate session token
  validateToken(token: string): Promise<{ 
    valid: boolean; 
    userId?: string;
    expiresAt?: Date;
  }>;
  
  // Refresh session
  refreshSession(token: string): Promise<string>;
  
  // Invalidate session
  invalidateSession(token: string): Promise<void>;
  
  // Clean up expired sessions
  cleanupExpiredSessions(): Promise<number>;
}

interface SessionMetadata {
  ipAddress: string;
  userAgent: string;
  loginTime: Date;
}
```

### 6. Multilingual Service

**Responsibilities:**
- Manage translations
- Handle language switching
- Provide fallback content
- Support RTL languages (future)

**Key Methods:**

```typescript
interface MultilingualService {
  // Get translation for key
  translate(key: string, language: string, params?: Record<string, any>): string;
  
  // Get all translations for language
  getTranslations(language: string): Promise<Record<string, string>>;
  
  // Detect user language from browser
  detectLanguage(acceptLanguageHeader: string): string;
  
  // Check if translation exists
  hasTranslation(key: string, language: string): boolean;
}
```

## Data Models

### Database Schema

**Users Table:**
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  digilocker_id VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(255) NOT NULL,
  age INTEGER CHECK (age > 0 AND age <= 120),
  state VARCHAR(100) NOT NULL,
  gender VARCHAR(20) NOT NULL,
  income_bracket VARCHAR(50),
  language_preference VARCHAR(10) DEFAULT 'en',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP NULL
);

CREATE INDEX idx_users_digilocker_id ON users(digilocker_id);
CREATE INDEX idx_users_state ON users(state);
```

**Sessions Table:**
```sql
CREATE TABLE sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash VARCHAR(255) UNIQUE NOT NULL,
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  expires_at TIMESTAMP NOT NULL,
  invalidated_at TIMESTAMP NULL
);

CREATE INDEX idx_sessions_user_id ON sessions(user_id);
CREATE INDEX idx_sessions_token_hash ON sessions(token_hash);
CREATE INDEX idx_sessions_expires_at ON sessions(expires_at);
```

**Consent Records Table:**
```sql
CREATE TABLE consent_records (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  consent_type VARCHAR(100) NOT NULL,
  granted BOOLEAN NOT NULL,
  granted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  revoked_at TIMESTAMP NULL
);

CREATE INDEX idx_consent_user_id ON consent_records(user_id);
```

**Schemes Table:**
```sql
CREATE TABLE schemes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name_en VARCHAR(500) NOT NULL,
  name_gu VARCHAR(500),
  name_hi VARCHAR(500),
  short_description_en TEXT NOT NULL,
  short_description_gu TEXT,
  short_description_hi TEXT,
  full_description_en TEXT NOT NULL,
  full_description_gu TEXT,
  full_description_hi TEXT,
  category VARCHAR(100) NOT NULL,
  state VARCHAR(100),
  min_age INTEGER,
  max_age INTEGER,
  eligible_genders TEXT[], -- Array of genders
  eligible_states TEXT[], -- Array of states
  eligible_income_brackets TEXT[], -- Array of income brackets
  official_url TEXT,
  last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_schemes_category ON schemes(category);
CREATE INDEX idx_schemes_state ON schemes(state);
CREATE INDEX idx_schemes_age_range ON schemes(min_age, max_age);
```

**Scheme Documents Table:**
```sql
CREATE TABLE scheme_documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scheme_id UUID NOT NULL REFERENCES schemes(id) ON DELETE CASCADE,
  name_en VARCHAR(255) NOT NULL,
  name_gu VARCHAR(255),
  name_hi VARCHAR(255),
  description_en TEXT,
  description_gu TEXT,
  description_hi TEXT,
  mandatory BOOLEAN DEFAULT true,
  available_in_digilocker BOOLEAN DEFAULT false,
  obtain_from_en TEXT,
  obtain_from_gu TEXT,
  obtain_from_hi TEXT,
  display_order INTEGER
);

CREATE INDEX idx_scheme_documents_scheme_id ON scheme_documents(scheme_id);
```

**Application Steps Table:**
```sql
CREATE TABLE application_steps (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scheme_id UUID NOT NULL REFERENCES schemes(id) ON DELETE CASCADE,
  step_number INTEGER NOT NULL,
  instruction_en TEXT NOT NULL,
  instruction_gu TEXT,
  instruction_hi TEXT,
  estimated_time VARCHAR(50)
);

CREATE INDEX idx_application_steps_scheme_id ON application_steps(scheme_id);
```

**Scheme Views Table (Analytics):**
```sql
CREATE TABLE scheme_views (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scheme_id UUID NOT NULL REFERENCES schemes(id) ON DELETE CASCADE,
  user_id UUID REFERENCES users(id) ON DELETE SET NULL,
  viewed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  language VARCHAR(10)
);

CREATE INDEX idx_scheme_views_scheme_id ON scheme_views(scheme_id);
CREATE INDEX idx_scheme_views_viewed_at ON scheme_views(viewed_at);
```

**Simplification Cache Table:**
```sql
CREATE TABLE simplification_cache (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  content_hash VARCHAR(64) UNIQUE NOT NULL,
  original_text TEXT NOT NULL,
  simplified_text TEXT NOT NULL,
  language VARCHAR(10) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  expires_at TIMESTAMP NOT NULL
);

CREATE INDEX idx_simplification_cache_hash ON simplification_cache(content_hash);
CREATE INDEX idx_simplification_cache_expires ON simplification_cache(expires_at);
```

### API Endpoints

**Authentication Endpoints:**
```
POST   /api/auth/digilocker/initiate    - Initiate DigiLocker OAuth
GET    /api/auth/digilocker/callback    - Handle OAuth callback
POST   /api/auth/logout                 - Logout user
GET    /api/auth/session                - Validate session
```

**User Endpoints:**
```
GET    /api/users/profile               - Get current user profile
PUT    /api/users/profile               - Update user profile
DELETE /api/users/account               - Delete user account
GET    /api/users/data-export           - Export user data
POST   /api/users/consent               - Grant consent
DELETE /api/users/consent/:type         - Revoke consent
```

**Scheme Endpoints:**
```
GET    /api/schemes                     - List schemes (with filters)
GET    /api/schemes/:id                 - Get scheme details
GET    /api/schemes/search              - Search schemes
POST   /api/schemes/:id/check-eligibility - Check eligibility
POST   /api/schemes/:id/view            - Record scheme view
```

**Utility Endpoints:**
```
GET    /api/health                      - Health check
GET    /api/translations/:language      - Get translations
POST   /api/feedback                    - Submit user feedback
```


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property Reflection

After analyzing all acceptance criteria, I identified several areas of redundancy:

1. **Display properties (7.1-7.5, 8.1-8.5, 9.1-9.5)** can be consolidated into comprehensive properties about scheme detail rendering rather than separate properties for each field
2. **Analytics tracking (14.1-14.4)** can be combined into a single property about event tracking
3. **Session management (12.1-12.5)** properties are distinct and necessary
4. **Error handling (13.1-13.5)** can be consolidated into properties about error message display and logging
5. **Simplification properties (5.1-5.3)** can be combined since they all test the same simplification behavior on different content types

The following properties represent the minimal, non-redundant set that provides comprehensive coverage.

### Authentication and Session Properties

**Property 1: DigiLocker OAuth Redirect**
*For any* user initiating DigiLocker login, the system should redirect to a valid DigiLocker OAuth authorization URL containing the correct client ID and redirect URI.
**Validates: Requirements 1.2**

**Property 2: Session Creation on Authentication**
*For any* valid DigiLocker authentication token, the system should create a user session with a unique session ID, expiration time of 24 hours, and secure storage.
**Validates: Requirements 1.4, 1.6, 12.1**

**Property 3: Session Invalidation on Logout**
*For any* active session, when logout is triggered, the session should be immediately invalidated and removed from storage.
**Validates: Requirements 12.4**

**Property 4: Session Expiration Cleanup**
*For any* expired session (past expiration time), the session data should be deleted from storage.
**Validates: Requirements 11.4**

**Property 5: Single Active Session Per User**
*For any* user account, at most one session should be active at any given time. Creating a new session should invalidate any existing session for that user.
**Validates: Requirements 12.5**

**Property 6: OAuth Failure Handling**
*For any* failed DigiLocker authorization (invalid code, network error, or API error), the system should display an error message and allow the user to retry without crashing.
**Validates: Requirements 1.5**

### Consent and Privacy Properties

**Property 7: Consent Recording with Timestamp**
*For any* user granting consent, the system should record the consent type, user ID, granted status, and timestamp in the database.
**Validates: Requirements 2.3**

**Property 8: Data Deletion on Consent Revocation**
*For any* consent revocation, all DigiLocker-fetched data associated with that consent should be deleted within 24 hours.
**Validates: Requirements 2.6**

**Property 9: Manual Entry on Consent Denial**
*For any* user denying DigiLocker data access consent, the system should provide manual profile entry fields for all required information.
**Validates: Requirements 2.4**

### User Profile Properties

**Property 10: DigiLocker Data Fetch on Consent**
*For any* user granting DigiLocker consent, the system should fetch name, age (calculated from DOB), state, and gender from DigiLocker API and populate the user profile.
**Validates: Requirements 3.1**

**Property 11: Age Validation**
*For any* age input (manual or calculated), the system should accept only positive integers between 1 and 120, rejecting all other values with a validation error.
**Validates: Requirements 3.3**

**Property 12: State Validation**
*For any* state input, the system should accept only values from the predefined list of Indian states and union territories, rejecting all other values.
**Validates: Requirements 3.4**

**Property 13: Profile Completeness Prompts**
*For any* user profile missing required fields (age, state, gender), the system should display prompts indicating which fields need completion.
**Validates: Requirements 3.5**

**Property 14: Scheme Refresh on Profile Update**
*For any* profile update that changes eligibility-relevant fields (age, state, gender, income bracket), the system should refresh the scheme recommendations list.
**Validates: Requirements 3.7**

### Eligibility Filtering Properties

**Property 15: Eligibility Matching**
*For any* user profile and scheme, the eligibility filter should return true if and only if the user's age, state, gender, and income bracket all match the scheme's criteria (where criteria are specified).
**Validates: Requirements 4.1, 4.2, 4.3**

**Property 16: Empty Results Handling**
*For any* user profile where no schemes match eligibility criteria, the system should return an empty list and display a "no eligible schemes found" message.
**Validates: Requirements 4.5 (edge case)**

### AI Simplification Properties

**Property 17: Content Simplification**
*For any* scheme content (description, eligibility criteria, or benefits) in a supported language, the AI simplifier should return simplified text that is shorter or uses simpler vocabulary than the original.
**Validates: Requirements 5.1, 5.2, 5.3**

**Property 18: Simplification Fallback**
*For any* simplification request that fails (API error, timeout, or invalid response), the system should display the original unsimplified content with a warning message.
**Validates: Requirements 5.5**

**Property 19: Simplification Caching**
*For any* text that has been simplified before (same content hash and language), the system should return the cached result without calling the AI API again.
**Validates: Implicit performance requirement**

### Multilingual Properties

**Property 20: Language Detection**
*For any* first-time user with a browser Accept-Language header containing 'gu', 'hi', or 'en', the system should set the interface language to match the browser preference.
**Validates: Requirements 6.2**

**Property 21: Language Switching**
*For any* language change action, all interface text should immediately update to the selected language without requiring page reload.
**Validates: Requirements 6.4**

**Property 22: Content Language Matching**
*For any* scheme displayed, the content (name, description, documents, steps) should be shown in the user's selected language if available.
**Validates: Requirements 6.5**

**Property 23: Translation Fallback**
*For any* content where translation is unavailable in the selected language, the system should display the English version with a notification indicating translation is unavailable.
**Validates: Requirements 6.6**

**Property 24: Language Persistence**
*For any* language selection, the preference should be stored and restored in subsequent sessions for the same user.
**Validates: Requirements 6.7**

### Scheme Display Properties

**Property 25: Scheme Detail Completeness**
*For any* scheme selected by a user, the displayed detail page should include all available fields: name, description, benefits, eligibility criteria, required documents, application steps, and contact information.
**Validates: Requirements 7.1, 7.2, 7.3, 7.4, 7.5**

**Property 26: Document Information Completeness**
*For any* required document in a scheme, the display should include document name, description, mandatory status, and DigiLocker availability indicator.
**Validates: Requirements 8.1, 8.2, 8.3, 8.5**

**Property 27: Application Step Numbering**
*For any* scheme with application steps, the steps should be displayed in sequential order with consecutive numbering starting from 1.
**Validates: Requirements 9.1**

**Property 28: Official Portal Links**
*For any* scheme with an official application portal URL, the URL should be displayed as a clickable link in the application process section.
**Validates: Requirements 9.3**

### Mobile Responsiveness Properties

**Property 29: Touch Target Sizing**
*For any* interactive element (button, link, input field), the tap target size should be at least 44px × 44px to ensure touch-friendly interaction.
**Validates: Requirements 10.2**

**Property 30: Vertical Scroll Priority**
*For any* page on mobile viewport (width < 768px), the content should fit within the viewport width without requiring horizontal scrolling.
**Validates: Requirements 10.3**

### Security Properties

**Property 31: Password Hashing**
*For any* user password stored in the database, it should be hashed using bcrypt with a minimum of 10 salt rounds, never storing plaintext passwords.
**Validates: Requirements 11.2**

**Property 32: Token Cleanup on Session End**
*For any* session that ends (logout or expiration), all associated DigiLocker authentication tokens should be deleted from storage immediately.
**Validates: Requirements 11.3**

**Property 33: Data Access Logging**
*For any* access to user profile data, the system should create a log entry containing user ID, accessed resource, timestamp, and action type.
**Validates: Requirements 11.5**

### Session Timeout Properties

**Property 34: Inactivity Warning**
*For any* user session inactive for 30 minutes, the system should display a timeout warning dialog before expiring the session.
**Validates: Requirements 12.2**

**Property 35: Session Extension on Interaction**
*For any* user interaction (click, input, navigation) after a timeout warning is displayed, the session expiration time should be extended by the full session duration.
**Validates: Requirements 12.3**

### Error Handling Properties

**Property 36: Localized Error Messages**
*For any* error that occurs, the error message displayed to the user should be in the user's currently selected language.
**Validates: Requirements 13.1**

**Property 37: Service Unavailability Handling**
*For any* DigiLocker API call that fails due to service unavailability (5xx errors or timeouts), the system should display a service unavailability message and not crash.
**Validates: Requirements 13.3**

**Property 38: Network Error Handling**
*For any* API request that fails due to network connectivity issues, the system should display a connectivity error message and allow retry.
**Validates: Requirements 13.4**

**Property 39: Error Logging**
*For any* error that occurs, the system should create a log entry containing error type, severity level, timestamp, user context, and stack trace.
**Validates: Requirements 13.5**

### Analytics Properties

**Property 40: Event Tracking**
*For any* trackable event (scheme view, login attempt, language change, simplification request), the system should record the event type, timestamp, and non-PII metadata in the analytics database.
**Validates: Requirements 14.1, 14.2, 14.3, 14.4**

**Property 41: PII Exclusion from Analytics**
*For any* analytics event recorded, the data should not contain personally identifiable information such as names, email addresses, phone numbers, or DigiLocker IDs.
**Validates: Requirements 14.5**

## Error Handling

### Error Categories

**1. Authentication Errors**
- DigiLocker OAuth failures (invalid code, expired token)
- Session expiration or invalidation
- Unauthorized access attempts

**Handling Strategy:**
- Redirect to login page with error message
- Clear invalid session data
- Log security-relevant events
- Provide clear user guidance

**2. Validation Errors**
- Invalid profile data (age out of range, invalid state)
- Missing required fields
- Malformed input data

**Handling Strategy:**
- Display field-level validation messages
- Highlight invalid fields in UI
- Prevent form submission until valid
- Preserve valid field values

**3. External Service Errors**
- DigiLocker API unavailability
- AI simplification API failures
- Network connectivity issues

**Handling Strategy:**
- Implement retry logic with exponential backoff
- Provide fallback behavior (original content for AI failures)
- Display user-friendly error messages
- Cache successful responses to reduce API dependency

**4. Database Errors**
- Connection failures
- Query timeouts
- Constraint violations

**Handling Strategy:**
- Implement connection pooling and retry logic
- Log errors with full context
- Display generic error message to users
- Alert monitoring systems for critical failures

**5. Application Errors**
- Unexpected exceptions
- Logic errors
- Resource exhaustion

**Handling Strategy:**
- Catch and log all unhandled exceptions
- Display generic error page to users
- Preserve user session when possible
- Alert development team for investigation

### Error Response Format

All API errors follow a consistent JSON format:

```typescript
interface ErrorResponse {
  error: {
    code: string;           // Machine-readable error code
    message: string;        // Human-readable message (localized)
    details?: any;          // Additional error context
    timestamp: string;      // ISO 8601 timestamp
    requestId: string;      // Unique request identifier for tracing
  };
}
```

### Logging Strategy

**Log Levels:**
- ERROR: System errors requiring immediate attention
- WARN: Recoverable errors or degraded functionality
- INFO: Significant events (login, logout, scheme views)
- DEBUG: Detailed diagnostic information

**Log Structure:**
```typescript
interface LogEntry {
  level: 'ERROR' | 'WARN' | 'INFO' | 'DEBUG';
  timestamp: string;
  requestId: string;
  userId?: string;
  message: string;
  context: Record<string, any>;
  stackTrace?: string;
}
```

**Sensitive Data Handling:**
- Never log passwords or authentication tokens
- Mask PII in logs (show only first/last characters)
- Sanitize user input before logging
- Implement log retention policies

## Testing Strategy

### Dual Testing Approach

The JanSaarthi AI platform requires both unit testing and property-based testing to ensure comprehensive coverage and correctness.

**Unit Tests** focus on:
- Specific examples of correct behavior
- Edge cases and boundary conditions
- Error handling scenarios
- Integration points between components
- Mock external dependencies (DigiLocker, AI APIs)

**Property-Based Tests** focus on:
- Universal properties that hold for all inputs
- Comprehensive input coverage through randomization
- Invariants that must be maintained
- Round-trip properties (e.g., serialize/deserialize)
- Metamorphic properties (e.g., filtering reduces or maintains list size)

Both approaches are complementary and necessary. Unit tests catch specific bugs and validate concrete examples, while property tests verify general correctness across a wide range of inputs.

### Property-Based Testing Configuration

**Framework Selection:**
- **JavaScript/TypeScript**: fast-check library
- Minimum 100 iterations per property test (due to randomization)
- Configurable seed for reproducible failures
- Shrinking enabled to find minimal failing examples

**Test Tagging:**
Each property-based test must include a comment tag referencing the design document property:

```typescript
// Feature: jansaarthi-ai-platform, Property 11: Age Validation
test('age validation accepts only values between 1 and 120', () => {
  fc.assert(
    fc.property(fc.integer(), (age) => {
      const result = validateAge(age);
      if (age >= 1 && age <= 120) {
        expect(result.valid).toBe(true);
      } else {
        expect(result.valid).toBe(false);
      }
    }),
    { numRuns: 100 }
  );
});
```

### Test Coverage Requirements

**Unit Test Coverage:**
- Minimum 80% code coverage for business logic
- 100% coverage for critical paths (authentication, eligibility filtering)
- All error handling branches tested
- All validation rules tested with valid and invalid inputs

**Property Test Coverage:**
- Each correctness property implemented as a property-based test
- Minimum 100 iterations per property test
- Custom generators for domain objects (User, Scheme, Session)
- Edge cases included in generators (empty strings, boundary values, null/undefined)

### Testing Layers

**1. Unit Tests**
- Individual function and method testing
- Component testing with mocked dependencies
- Validation logic testing
- Utility function testing

**2. Integration Tests**
- API endpoint testing with test database
- Service integration testing
- External API mocking (DigiLocker, AI services)
- Database transaction testing

**3. Property-Based Tests**
- Authentication flow properties
- Eligibility filtering properties
- Session management properties
- Data validation properties
- Error handling properties

**4. End-to-End Tests**
- Critical user flows (login → profile → scheme discovery)
- Cross-browser compatibility testing
- Mobile responsiveness testing
- Performance testing under load

### Test Data Management

**Generators for Property Tests:**

```typescript
// User profile generator
const userProfileGen = fc.record({
  age: fc.integer({ min: 1, max: 120 }),
  state: fc.constantFrom(...INDIAN_STATES),
  gender: fc.constantFrom('male', 'female', 'other'),
  incomeBracket: fc.option(
    fc.constantFrom('below-1L', '1L-3L', '3L-5L', 'above-5L')
  ),
  languagePreference: fc.constantFrom('en', 'gu', 'hi')
});

// Scheme generator
const schemeGen = fc.record({
  name: fc.string({ minLength: 5, maxLength: 100 }),
  minAge: fc.option(fc.integer({ min: 0, max: 100 })),
  maxAge: fc.option(fc.integer({ min: 0, max: 120 })),
  eligibleStates: fc.option(fc.array(fc.constantFrom(...INDIAN_STATES))),
  eligibleGenders: fc.option(fc.array(fc.constantFrom('male', 'female', 'other')))
});

// Session generator
const sessionGen = fc.record({
  userId: fc.uuid(),
  expiresAt: fc.date({ min: new Date(), max: new Date(Date.now() + 86400000) }),
  ipAddress: fc.ipV4(),
  userAgent: fc.string()
});
```

**Test Database:**
- Separate test database instance
- Automated schema migrations
- Seed data for integration tests
- Cleanup after each test suite

### Continuous Integration

**CI Pipeline:**
1. Lint and format checks
2. Unit tests with coverage reporting
3. Property-based tests (100 iterations)
4. Integration tests with test database
5. Build and bundle optimization
6. Security vulnerability scanning
7. Performance benchmarking

**Quality Gates:**
- All tests must pass
- Code coverage ≥ 80%
- No high-severity security vulnerabilities
- Build size within limits
- Performance benchmarks met

### Manual Testing

**Accessibility Testing:**
- Screen reader compatibility (NVDA, JAWS)
- Keyboard navigation
- Color contrast validation
- WCAG 2.1 Level AA compliance

**Usability Testing:**
- User testing with target personas
- Task completion rate measurement
- System Usability Scale (SUS) scoring
- Feedback collection and iteration

**Localization Testing:**
- Native speaker review for Gujarati and Hindi
- RTL layout testing (future)
- Cultural appropriateness validation
- Translation accuracy verification

## Deployment and Operations

### Deployment Architecture

**Production Environment:**
- Cloud hosting (AWS/GCP) with auto-scaling
- Load balancer for traffic distribution
- CDN for static assets
- Database with read replicas
- Redis cluster for session caching

**Staging Environment:**
- Mirror of production configuration
- Separate database instance
- Used for pre-production validation
- Automated deployment from main branch

**Development Environment:**
- Local Docker containers
- Mock external services
- Hot reload for rapid development
- Seeded test data

### Monitoring and Observability

**Application Monitoring:**
- Request/response logging
- Error rate tracking
- API latency monitoring
- User session analytics

**Infrastructure Monitoring:**
- Server resource utilization (CPU, memory, disk)
- Database performance metrics
- Network traffic and bandwidth
- Auto-scaling triggers

**Alerting:**
- Error rate thresholds
- API latency thresholds
- Service unavailability
- Security incidents

### Security Considerations

**Authentication Security:**
- OAuth 2.0 with PKCE for DigiLocker
- JWT tokens with short expiration
- Secure session storage with HttpOnly cookies
- CSRF protection on all state-changing operations

**Data Security:**
- TLS 1.2+ for all communications
- Encryption at rest for sensitive data
- Database access controls and auditing
- Regular security audits and penetration testing

**API Security:**
- Rate limiting per user and IP
- Input validation and sanitization
- SQL injection prevention (parameterized queries)
- XSS prevention (content security policy)

### Scalability Considerations

**Horizontal Scaling:**
- Stateless API servers for easy scaling
- Session storage in Redis (shared across instances)
- Database connection pooling
- Load balancer health checks

**Caching Strategy:**
- Redis for session data (TTL: 24 hours)
- Redis for AI simplification cache (TTL: 7 days)
- CDN for static assets (TTL: 30 days)
- Database query result caching for scheme lists

**Performance Optimization:**
- Database indexing on frequently queried fields
- Lazy loading for scheme details
- Image optimization and compression
- Code splitting and bundle optimization
- Asynchronous AI processing with job queue

### Maintenance and Updates

**Database Migrations:**
- Version-controlled migration scripts
- Automated migration on deployment
- Rollback capability for failed migrations
- Zero-downtime migrations for production

**Scheme Data Updates:**
- Automated scraping or API integration for scheme data
- Manual review and approval workflow
- Versioning for scheme content changes
- Notification to users for relevant scheme updates

**Dependency Management:**
- Regular security updates
- Automated vulnerability scanning
- Semantic versioning for dependencies
- Testing before production deployment
