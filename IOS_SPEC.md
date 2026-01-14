# StudyBuddy iOS App - Technical Specification

**Version:** 1.0
**Date:** January 13, 2026
**Platform:** iOS 17.0+
**Based on:** Web version (React 19 + FastAPI)

---

## Executive Summary

StudyBuddy iOS is a native educational app that helps parents guide children through curriculum-aligned practice questions with adaptive difficulty. The app features two learning modes: **Practice Mode** (adaptive, infinite sessions) and **Quiz Mode** (timed, fixed questions), with comprehensive progress tracking and standards-based content (Eureka Math for mathematics, Common Core for other subjects).

**Key Differentiators:**
- Adaptive difficulty engine that responds to child performance in real-time
- 1,275+ structured subtopics for granular learning progression
- Dual-mode learning (practice vs. quiz) for different pedagogical approaches
- Offline-first with intelligent sync (local CoreData + backend sync)
- Family-friendly design with multiple child profiles per parent account

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Technology Stack](#technology-stack)
3. [Features & User Flows](#features--user-flows)
4. [Data Models](#data-models)
5. [API Integration](#api-integration)
6. [UI/UX Design Guidelines](#uiux-design-guidelines)
7. [State Management](#state-management)
8. [Offline Support](#offline-support)
9. [Testing Strategy](#testing-strategy)
10. [Development Phases](#development-phases)
11. [Security & Privacy](#security--privacy)
12. [Performance Requirements](#performance-requirements)

---

## Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────┐
│                 iOS Application                      │
│  ┌────────────┐  ┌─────────────┐  ┌──────────────┐ │
│  │  SwiftUI   │  │  Combine    │  │  Core Data   │ │
│  │  Views     │  │  Publishers │  │  (Offline)   │ │
│  └────────────┘  └─────────────┘  └──────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │         View Models (MVVM Pattern)             │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │   Services   │  │   Network    │  │  Sync     │ │
│  │   Layer      │  │   Manager    │  │  Engine   │ │
│  └──────────────┘  └──────────────┘  └───────────┘ │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
         ┌────────────────────────────────┐
         │     FastAPI Backend (Existing) │
         │    - Authentication            │
         │    - Question Management       │
         │    - Progress Tracking         │
         │    - Quiz Sessions             │
         └────────────────────────────────┘
                          │
                          ▼
              ┌──────────────────────┐
              │   PostgreSQL         │
              │   (Supabase)         │
              └──────────────────────┘
```

### Design Patterns

1. **MVVM (Model-View-ViewModel)**: Separation of UI from business logic
2. **Repository Pattern**: Abstraction layer for data access (local + remote)
3. **Coordinator Pattern**: Navigation flow management
4. **Factory Pattern**: Object creation for questions, sessions, etc.
5. **Observer Pattern**: Real-time updates via Combine publishers

---

## Technology Stack

### Core Frameworks
- **Language**: Swift 5.9+
- **UI Framework**: SwiftUI (iOS 17.0+)
- **Reactive Programming**: Combine
- **Networking**: URLSession + async/await
- **Local Storage**: Core Data (with CloudKit sync optional)
- **Keychain**: Secure token storage

### Third-Party Dependencies (Minimal)
- **Alamofire** (Optional): If advanced networking features needed
- **KeychainAccess**: Simplified keychain operations
- **Charts** (Swift Charts): Progress visualization
- **SDWebImage**: Image caching (if question images supported)

### Development Tools
- **Xcode 15.0+**: IDE
- **Swift Package Manager**: Dependency management
- **XCTest + XCUITest**: Testing frameworks
- **SwiftLint**: Code style enforcement
- **Fastlane**: CI/CD automation

### Backend (Existing)
- **API**: FastAPI (Python 3.11+)
- **Database**: PostgreSQL via Supabase
- **AI**: OpenAI GPT-4 for question generation

---

## Features & User Flows

### 1. Authentication Flow

#### 1.1 Sign Up
**User Story**: As a parent, I want to create an account so I can track my children's progress.

**Flow:**
1. Launch app → Welcome screen with "Sign Up" and "Login" buttons
2. Tap "Sign Up" → Navigate to registration form
3. Enter email and password (min 6 characters)
4. Validate inputs:
   - Email format (regex: `^[^@\s]+@[^@\s]+\.[^@\s]+$`)
   - Password length (≥6 characters)
5. Tap "Create Account" → POST `/auth/signup`
6. On success:
   - Store access token in Keychain
   - Store parent info in UserDefaults
   - Navigate to "Add First Child" screen
7. On failure:
   - Display error toast (e.g., "Email already exists")

**API Endpoint:**
```swift
POST /auth/signup
Request: { email: String, password: String }
Response: {
  access_token: String,
  token_type: "bearer",
  parent: { id: String, email: String, created_at: Date }
}
```

#### 1.2 Login
**User Story**: As a returning parent, I want to log in to access my account.

**Flow:**
1. Launch app → Welcome screen
2. Tap "Login" → Navigate to login form
3. Enter email and password
4. Tap "Sign In" → POST `/auth/login`
5. On success:
   - Store token securely
   - Fetch children list
   - Navigate to Dashboard
6. On failure:
   - Display error (e.g., "Invalid credentials")

#### 1.3 Token Management
- **Token Storage**: Keychain (encrypted, device-locked)
- **Token Expiry**: Backend handles (no expiry currently)
- **Auto-Login**: Check for valid token on app launch
- **Logout**: Clear token + local cache, return to Welcome

---

### 2. Child Profile Management

#### 2.1 Add Child
**User Story**: As a parent, I want to add my child's profile with their grade level.

**Flow:**
1. Dashboard → Tap "Add Child" button
2. Present modal form with fields:
   - Name (required, text field)
   - Grade (required, picker 0-12)
   - Birthdate (optional, date picker)
   - ZIP Code (optional, for pacing features)
3. Tap "Save" → POST `/children`
4. On success:
   - Add to local Core Data
   - Refresh children list
   - Auto-select new child
   - Dismiss modal
5. On failure:
   - Display error inline

**API Endpoint:**
```swift
POST /children
Headers: { Authorization: "Bearer <token>" }
Request: {
  name: String,
  grade: Int (0-12),
  birthdate?: Date,
  zip?: String
}
Response: {
  id: String,
  parent_id: String,
  name: String,
  grade: Int,
  birthdate?: Date,
  zip?: String,
  created_at: Date
}
```

#### 2.2 Edit Child
**Flow:**
1. Dashboard → Tap child card → Tap "Edit" (...)
2. Present form with pre-filled data
3. Modify fields
4. Tap "Save" → PATCH `/children/{id}`
5. Update local cache + UI

#### 2.3 Delete Child
**Flow:**
1. Swipe left on child card → "Delete"
2. Confirm with alert ("Are you sure?")
3. DELETE `/children/{id}`
4. Remove from local storage
5. If current child deleted, select first available or show empty state

#### 2.4 Select Child
**Flow:**
1. Tap child card → Highlight with border
2. Update selected child in app state
3. Load progress for selected child
4. Enable practice/quiz buttons

---

### 3. Practice Mode (Adaptive)

#### 3.1 Start Practice Session
**User Story**: As a parent, I want my child to practice questions that adapt to their skill level.

**Flow:**
1. Dashboard → Ensure child selected → Tap "Start Practice"
2. Present subject picker (Math, Reading, Science, Social Studies)
3. (Optional) Select topic from dropdown
4. (Optional) Select subtopic from dropdown
5. Choose difficulty preference:
   - 🎯 **Adaptive (Recommended)** - System adjusts automatically
   - Easy - Manual override
   - Medium - Manual override
   - Hard - Manual override
6. Tap "Begin Practice" → POST `/questions/fetch`
7. Display first question with:
   - Question stem (large, readable font)
   - Multiple choice options (A, B, C, D buttons)
   - Subtopic badge (if available)
   - Session stats (questions answered, accuracy)

**API Endpoint:**
```swift
POST /questions/fetch
Request: {
  child_id: String,
  subject: String,
  topic?: String,
  subtopic?: String,
  limit: Int (default 5, max 20)
}
Response: {
  questions: [Question],
  selected_subtopic?: String,
  session_id?: String
}
```

#### 3.2 Answer Question
**Flow:**
1. User taps option button (A/B/C/D)
2. Disable all buttons
3. Start timer (track `time_spent_ms`)
4. Tap "Submit Answer" → POST `/attempts`
5. Backend validates answer
6. Display feedback:
   - **Correct**: Green checkmark, "Correct!" message, confetti animation
   - **Incorrect**: Red X, show correct answer, display rationale
7. Show "Next Question" button
8. Update session stats (accuracy, streak)

**API Endpoint:**
```swift
POST /attempts
Request: {
  child_id: String,
  question_id: String,
  selected: String,
  time_spent_ms: Int (≥0)
}
Response: {
  attempt_id: String,
  correct: Bool,
  expected: String
}
```

#### 3.3 Adaptive Difficulty Logic (Simplified Baseline)
**Current Implementation:**
- **High Mastery** (accuracy ≥95%, ≥10 attempts) → Hard questions
- **Progressive** (accuracy ≥80%) → Medium questions
- **Foundational** (accuracy <80%) → Easy questions

**Future Enhancement (CLAUDE_PLAN4):**
- Session-based adjustments
- Streak modulation (every 3 correct → +1 level)
- Recovery mechanisms (2 consecutive incorrect → -1 level)
- Cooldown (one level change per 2 attempts)

#### 3.4 End Session
**Flow:**
1. Tap "End Session" button (or auto-end after 20 questions)
2. Navigate to Session Summary screen with:
   - Questions attempted
   - Questions correct
   - Accuracy percentage
   - Total time
   - Average time per question
   - Subjects practiced
3. Show "Continue Practicing" or "Back to Dashboard"

---

### 4. Quiz Mode (Timed, Non-Adaptive)

#### 4.1 Create Quiz
**User Story**: As a parent, I want to give my child a timed quiz to test knowledge under pressure.

**Flow:**
1. Dashboard → Tap "Start Quiz"
2. Present quiz setup modal:
   - **Subject**: Picker (Math, Reading, etc.)
   - **Topic**: Picker (e.g., "Addition", "Fractions")
   - **Subtopic**: (Optional) Picker
   - **Question Count**: Slider (5-30, default 10)
   - **Duration**: Slider (5 min - 2 hours, auto-calculated or manual)
     - Formula: `(easy_count × 60s) + (medium_count × 90s) + (hard_count × 120s)`
   - **Difficulty Mix**: Preset selector
     - Balanced (default): 30% easy, 50% medium, 20% hard
     - Custom sliders (advanced)
3. Display duration preview: "Estimated time: 15 minutes"
4. Check for existing active quiz → Show warning if exists
5. Tap "Create Quiz" → POST `/quiz/sessions`
6. Navigate to Quiz screen with timer

**API Endpoint:**
```swift
POST /quiz/sessions
Request: {
  child_id: String,
  subject: String,
  topic: String,
  subtopic?: String,
  question_count: Int (5-30),
  duration_sec: Int (300-7200),
  difficulty_mix?: {
    easy: Float (0-1),
    medium: Float (0-1),
    hard: Float (0-1)  // Must sum to 1.0
  }
}
Response: {
  session: QuizSession,
  questions: [QuizQuestionDisplay],  // No answers/explanations
  time_remaining_sec: Int
}
```

#### 4.2 Take Quiz
**Flow:**
1. Display all questions at once (scrollable list or paginated)
2. Show sticky timer at top (counts down, syncs with backend every 10s)
3. For each question:
   - Radio button selection (single choice)
   - Visual indicator for answered vs. unanswered
4. "Jump to Question" navigation for 10+ questions
5. Tap "Submit Quiz" → Warn if unanswered questions remain
6. Confirm submission → POST `/quiz/sessions/{id}/submit`
7. Navigate to Results screen

**Timer Behavior:**
- **Local countdown**: Updates every second
- **Backend sync**: Fetch time remaining every 10 seconds
- **Auto-submit**: When timer reaches 0:00
- **Persistence**: On app backgrounding/foregrounding, recalculate from server timestamp

#### 4.3 Quiz Results
**Flow:**
1. Display summary card:
   - Score (percentage)
   - Correct / Total
   - Time taken
2. Show incorrect items only:
   - Question stem
   - User's answer (highlighted red)
   - Correct answer (highlighted green)
   - Explanation/rationale
3. Action buttons:
   - "Start New Quiz"
   - "Review All Questions" (optional)
   - "Back to Dashboard"

**API Endpoint:**
```swift
POST /quiz/sessions/{quiz_id}/submit
Request: {
  answers: [{ question_id: String, selected_choice: String }]
}
Response: {
  session_id: String,
  score: Int (0-100),
  correct_count: Int,
  total_questions: Int,
  time_taken_sec: Int,
  incorrect_items: [QuizIncorrectItem],
  submitted_at: Date
}
```

#### 4.4 Quiz History
**Flow:**
1. Progress tab → Toggle "Quizzes" view
2. List past quizzes with:
   - Date/time
   - Subject and topic
   - Score (percentage)
   - Time taken
3. Tap quiz → Navigate to results view (read-only)

---

### 5. Progress Tracking

#### 5.1 Progress Overview
**User Story**: As a parent, I want to see my child's overall learning progress.

**Flow:**
1. Dashboard → Progress section (or dedicated tab)
2. Display overview cards:
   - **Current Streak**: 🔥 "5 Day Streak!"
   - **Overall Accuracy**: 87%
   - **Questions Practiced**: 120
   - **Questions Correct**: 104
3. Subject-level breakdown:
   - Math: 90% (45/50) - Green progress bar
   - Reading: 85% (40/47) - Blue progress bar
   - Science: 80% (15/19) - Orange progress bar
4. Refresh on pull-to-refresh → GET `/progress/{child_id}`

**API Endpoint:**
```swift
GET /progress/{child_id}
Response: {
  attempted: Int,
  correct: Int,
  accuracy: Int (0-100),
  current_streak: Int,
  by_subject: {
    "math": { correct: Int, total: Int, accuracy: Int },
    "reading": { correct: Int, total: Int, accuracy: Int },
    ...
  }
}
```

#### 5.2 Session History
**Flow:**
1. Progress tab → "Recent Sessions" list
2. Display sessions with:
   - Date/time
   - Subject
   - Questions answered
   - Accuracy
   - Duration
3. Tap session → Session detail view

**API Endpoint:**
```swift
GET /sessions?child_id={child_id}
Response: [Session]
```

---

### 6. Standards Reference

#### 6.1 Browse Standards
**User Story**: As a parent, I want to understand what my child is learning.

**Flow:**
1. Settings tab → "Learning Standards"
2. Filter by:
   - Subject (Math, Reading, etc.)
   - Grade level (0-12)
3. Display standards in grouped list:
   - **Subject → Grade → Domain → Standards**
4. Tap standard → Show full description
5. "See Questions" → Filter practice by this standard

**API Endpoint:**
```swift
GET /standards
Query params: { subject?: String, grade?: Int }
Response: [Standard]

Standard: {
  subject: String,
  grade: Int,
  domain?: String,
  sub_domain?: String,
  standard_ref: String,
  title?: String,
  description?: String
}
```

---

## Data Models

### Core Data Entities (Offline Storage)

#### 1. ParentEntity
```swift
@Model
class ParentEntity {
    @Attribute(.unique) var id: String
    var email: String
    var createdAt: Date

    @Relationship(deleteRule: .cascade)
    var children: [ChildEntity]

    // Computed
    var isLoggedIn: Bool { /* check keychain for token */ }
}
```

#### 2. ChildEntity
```swift
@Model
class ChildEntity {
    @Attribute(.unique) var id: String
    var parentId: String
    var name: String
    var grade: Int  // 0-12
    var birthdate: Date?
    var zip: String?
    var createdAt: Date
    var lastSyncedAt: Date?

    @Relationship(deleteRule: .cascade)
    var attempts: [AttemptEntity]

    @Relationship(deleteRule: .cascade)
    var sessions: [SessionEntity]

    @Relationship(deleteRule: .cascade)
    var quizSessions: [QuizSessionEntity]
}
```

#### 3. QuestionEntity
```swift
@Model
class QuestionEntity {
    @Attribute(.unique) var id: String
    var standardRef: String?
    var subject: String
    var grade: Int?
    var topic: String?
    var subTopic: String?
    var difficulty: String?  // easy, medium, hard
    var stem: String
    var optionsJSON: Data  // [String]
    var correctAnswer: String
    var rationale: String?
    var source: String?
    var hash: String
    var createdAt: Date

    // Computed
    var options: [String] {
        try? JSONDecoder().decode([String].self, from: optionsJSON)
    }
}
```

#### 4. AttemptEntity
```swift
@Model
class AttemptEntity {
    @Attribute(.unique) var id: String
    var childId: String
    var questionId: String
    var selected: String
    var correct: Bool
    var timeSpentMs: Int
    var createdAt: Date
    var syncStatus: SyncStatus  // .pending, .synced, .failed

    @Relationship
    var child: ChildEntity?

    @Relationship
    var question: QuestionEntity?
}

enum SyncStatus: String, Codable {
    case pending
    case synced
    case failed
}
```

#### 5. SessionEntity
```swift
@Model
class SessionEntity {
    @Attribute(.unique) var id: String
    var childId: String
    var subject: String?
    var topic: String?
    var subtopic: String?
    var startedAt: Date
    var endedAt: Date?
    var createdAt: Date
    var updatedAt: Date
    var syncStatus: SyncStatus

    @Relationship
    var child: ChildEntity?
}
```

#### 6. QuizSessionEntity
```swift
@Model
class QuizSessionEntity {
    @Attribute(.unique) var id: String
    var childId: String
    var subject: String
    var topic: String
    var subtopic: String?
    var status: String  // active, completed, expired
    var durationSec: Int
    var difficultyMixJSON: Data  // {easy: Float, medium: Float, hard: Float}
    var startedAt: Date
    var submittedAt: Date?
    var score: Int?  // 0-100
    var totalQuestions: Int
    var createdAt: Date
    var syncStatus: SyncStatus

    @Relationship(deleteRule: .cascade)
    var questions: [QuizQuestionEntity]

    @Relationship
    var child: ChildEntity?
}
```

#### 7. QuizQuestionEntity
```swift
@Model
class QuizQuestionEntity {
    @Attribute(.unique) var id: String
    var quizSessionId: String
    var questionId: String
    var index: Int
    var correctChoice: String
    var explanation: String
    var selectedChoice: String?
    var isCorrect: Bool?

    @Relationship
    var quizSession: QuizSessionEntity?

    @Relationship
    var question: QuestionEntity?
}
```

#### 8. SubtopicEntity
```swift
@Model
class SubtopicEntity {
    @Attribute(.unique) var id: String
    var subject: String
    var grade: Int
    var topic: String
    var subtopic: String
    var description: String?
    var sequenceOrder: Int
    var createdAt: Date
}
```

---

## API Integration

### Network Layer Architecture

#### 1. NetworkManager (Singleton)
```swift
class NetworkManager {
    static let shared = NetworkManager()

    private let baseURL = "https://api.studybuddy.com"  // Or configured
    private let session: URLSession

    private init() {
        let config = URLSessionConfiguration.default
        config.timeoutIntervalForRequest = 30
        config.timeoutIntervalForResource = 300
        self.session = URLSession(configuration: config)
    }

    func request<T: Decodable>(
        endpoint: Endpoint,
        method: HTTPMethod = .get,
        body: Encodable? = nil
    ) async throws -> T {
        // Build URLRequest
        // Add Authorization header if token exists
        // Execute request with async/await
        // Decode response
        // Handle errors (401 → logout, 500 → retry)
    }
}
```

#### 2. Endpoint Enum
```swift
enum Endpoint {
    case signup
    case login
    case children
    case childDetail(id: String)
    case fetchQuestions
    case submitAttempt
    case progress(childId: String)
    case sessions
    case sessionDetail(id: String)
    case createQuiz
    case quizDetail(id: String)
    case submitQuiz(id: String)
    case standards

    var path: String {
        switch self {
        case .signup: return "/auth/signup"
        case .login: return "/auth/login"
        case .children: return "/children"
        case .childDetail(let id): return "/children/\(id)"
        case .fetchQuestions: return "/questions/fetch"
        case .submitAttempt: return "/attempts"
        case .progress(let childId): return "/progress/\(childId)"
        case .sessions: return "/sessions"
        case .sessionDetail(let id): return "/sessions/\(id)"
        case .createQuiz: return "/quiz/sessions"
        case .quizDetail(let id): return "/quiz/sessions/\(id)"
        case .submitQuiz(let id): return "/quiz/sessions/\(id)/submit"
        case .standards: return "/standards"
        }
    }
}
```

#### 3. APIService (Type-Safe)
```swift
protocol APIServiceProtocol {
    func signup(email: String, password: String) async throws -> AuthResponse
    func login(email: String, password: String) async throws -> AuthResponse
    func fetchChildren() async throws -> [Child]
    func createChild(_ request: ChildCreate) async throws -> Child
    func updateChild(id: String, _ request: ChildUpdate) async throws -> Child
    func deleteChild(id: String) async throws
    func fetchQuestions(_ request: QuestionRequest) async throws -> QuestionResponse
    func submitAttempt(_ request: AttemptSubmission) async throws -> AttemptResult
    func fetchProgress(childId: String) async throws -> ProgressResponse
    func fetchSessions(childId: String) async throws -> [Session]
    func fetchSessionDetail(id: String) async throws -> SessionSummary
    func createQuiz(_ request: QuizCreateRequest) async throws -> QuizSessionResponse
    func fetchQuizDetail(id: String) async throws -> QuizSessionResponse
    func submitQuiz(id: String, _ request: QuizSubmitRequest) async throws -> QuizResult
    func fetchStandards(subject: String?, grade: Int?) async throws -> [Standard]
}

class APIService: APIServiceProtocol {
    private let networkManager = NetworkManager.shared

    // Implementation using networkManager.request()
}
```

#### 4. Error Handling
```swift
enum APIError: Error, LocalizedError {
    case unauthorized  // 401
    case forbidden     // 403
    case notFound      // 404
    case conflict      // 409 (e.g., active quiz exists)
    case serverError   // 500
    case networkError(Error)
    case decodingError(Error)
    case invalidResponse

    var errorDescription: String? {
        switch self {
        case .unauthorized:
            return "Your session has expired. Please log in again."
        case .conflict:
            return "An active quiz already exists for this topic."
        case .serverError:
            return "Something went wrong. Please try again."
        case .networkError:
            return "Check your internet connection."
        default:
            return "An unexpected error occurred."
        }
    }
}
```

---

## UI/UX Design Guidelines

### Design System (iOS Native)

#### 1. Color Palette
```swift
extension Color {
    // Brand Colors
    static let brandPrimary = Color("BrandPrimary")  // Vibrant blue
    static let brandSecondary = Color("BrandSecondary")  // Accent color

    // Semantic Colors
    static let success = Color.green
    static let error = Color.red
    static let warning = Color.orange
    static let info = Color.blue

    // Subject Colors (match web)
    static let mathColor = Color("MathColor")  // Blue
    static let readingColor = Color("ReadingColor")  // Purple
    static let scienceColor = Color("ScienceColor")  // Green
    static let socialStudiesColor = Color("SocialStudiesColor")  // Orange

    // Neutral Colors
    static let background = Color("Background")  // Light gray
    static let surface = Color.white
    static let textPrimary = Color("TextPrimary")  // Dark gray
    static let textSecondary = Color("TextSecondary")  // Medium gray
    static let divider = Color("Divider")  // Light gray
}
```

#### 2. Typography
```swift
extension Font {
    // Headings
    static let heading1 = Font.system(size: 28, weight: .bold)
    static let heading2 = Font.system(size: 22, weight: .semibold)
    static let heading3 = Font.system(size: 18, weight: .semibold)

    // Body
    static let bodyLarge = Font.system(size: 17, weight: .regular)
    static let bodyMedium = Font.system(size: 15, weight: .regular)
    static let bodySmall = Font.system(size: 13, weight: .regular)

    // Special
    static let buttonLabel = Font.system(size: 17, weight: .semibold)
    static let caption = Font.system(size: 12, weight: .regular)
}
```

#### 3. Spacing
```swift
extension CGFloat {
    static let spacing4: CGFloat = 4
    static let spacing8: CGFloat = 8
    static let spacing12: CGFloat = 12
    static let spacing16: CGFloat = 16
    static let spacing24: CGFloat = 24
    static let spacing32: CGFloat = 32
    static let spacing48: CGFloat = 48
}
```

#### 4. Animations
```swift
extension Animation {
    static let quick = Animation.easeOut(duration: 0.15)
    static let standard = Animation.easeInOut(duration: 0.3)
    static let slow = Animation.easeInOut(duration: 0.5)

    // Spring animations for natural feel
    static let spring = Animation.spring(response: 0.3, dampingFraction: 0.7)
}
```

### Key Screens

#### 1. Welcome Screen
- Large app logo
- Tagline: "Adaptive learning for K-12 students"
- "Sign Up" button (primary)
- "Login" button (secondary)
- Background: Subtle gradient

#### 2. Dashboard
- **Top**: Sticky header with parent info, selected child, logout
- **Main Content**:
  - Child selector (horizontal scroll, cards)
  - Practice Mode card ("Start Practice")
  - Quiz Mode card ("Start Quiz")
  - Progress summary (streak, accuracy)
- **Bottom**: Tab bar (Dashboard, Progress, Settings)

#### 3. Practice Session
- **Top**: Timer, question count, accuracy
- **Middle**: Question stem (large font), subtopic badge
- **Options**: 4 buttons (A/B/C/D) with tap animation
- **Bottom**: "Submit Answer" button (disabled until option selected)
- **Feedback**: Full-screen overlay with confetti (correct) or shake (incorrect)

#### 4. Quiz Session
- **Top**: Sticky timer (countdown with warning at 1 min)
- **Middle**: Scrollable question list (all visible)
- **Radio Buttons**: Single selection per question
- **Jump Navigation**: Numbered buttons (1, 2, 3...) for quick access
- **Bottom**: "Submit Quiz" button (sticky)

#### 5. Results Screen
- **Summary Card**: Score badge, time, breakdown
- **Incorrect List**: Expandable sections per question
- **Actions**: "Start New Quiz", "Back to Dashboard"

---

## State Management

### MVVM Architecture

#### 1. ViewModels
```swift
@MainActor
class AuthViewModel: ObservableObject {
    @Published var isAuthenticated = false
    @Published var parent: Parent?
    @Published var isLoading = false
    @Published var errorMessage: String?

    private let apiService: APIServiceProtocol
    private let keychainService: KeychainServiceProtocol

    func signup(email: String, password: String) async {
        isLoading = true
        defer { isLoading = false }

        do {
            let response = try await apiService.signup(email: email, password: password)
            keychainService.saveToken(response.access_token)
            parent = response.parent
            isAuthenticated = true
        } catch {
            errorMessage = error.localizedDescription
        }
    }

    func logout() {
        keychainService.deleteToken()
        parent = nil
        isAuthenticated = false
    }
}

@MainActor
class ChildrenViewModel: ObservableObject {
    @Published var children: [Child] = []
    @Published var selectedChild: Child?
    @Published var isLoading = false
    @Published var errorMessage: String?

    private let apiService: APIServiceProtocol

    func fetchChildren() async {
        // ...
    }

    func addChild(_ request: ChildCreate) async {
        // ...
    }
}

@MainActor
class PracticeViewModel: ObservableObject {
    @Published var currentQuestion: Question?
    @Published var questions: [Question] = []
    @Published var selectedOption: String?
    @Published var isSubmitting = false
    @Published var feedback: Feedback?
    @Published var sessionStats: SessionStats = .init()

    func fetchQuestions(request: QuestionRequest) async {
        // ...
    }

    func submitAnswer(selected: String, timeSpentMs: Int) async {
        // ...
    }
}
```

#### 2. Combine Publishers
```swift
class SyncService {
    // Auto-sync every 30 seconds when online
    let syncPublisher = Timer.publish(every: 30, on: .main, in: .common)
        .autoconnect()
        .eraseToAnyPublisher()

    // Network reachability observer
    let reachabilityPublisher: AnyPublisher<Bool, Never>
}
```

---

## Offline Support

### Strategy: Offline-First with Intelligent Sync

#### 1. Data Persistence
- **Core Data**: Primary local storage
- **CloudKit**: Optional for cross-device sync
- **Keychain**: Secure token storage

#### 2. Sync Logic
```swift
class SyncManager {
    private let coreDataStack: CoreDataStack
    private let apiService: APIServiceProtocol

    func sync() async throws {
        // 1. Upload pending attempts (syncStatus == .pending)
        try await uploadPendingAttempts()

        // 2. Fetch latest progress from backend
        try await fetchAndMergeProgress()

        // 3. Download new questions for prefetch
        try await prefetchQuestions()

        // 4. Update lastSyncedAt timestamp
        updateLastSyncTimestamp()
    }

    private func uploadPendingAttempts() async throws {
        let pendingAttempts = fetchPendingAttempts()
        for attempt in pendingAttempts {
            do {
                _ = try await apiService.submitAttempt(attempt.toRequest())
                attempt.syncStatus = .synced
            } catch {
                attempt.syncStatus = .failed
            }
        }
        try coreDataStack.save()
    }
}
```

#### 3. Conflict Resolution
- **Server Wins**: For progress data (accuracy, streaks)
- **Merge**: For attempts (deduplicate by attempt_id)
- **Last Write Wins**: For child profile updates

#### 4. Prefetching
- Download 20 questions per subject when online
- Store in Core Data for offline practice
- Trigger refetch when question count < 5

#### 5. Network Status Indicator
- Banner at top: "Offline - Your progress will sync when online"
- Green checkmark: "Synced" (auto-hide after 2s)
- Yellow warning: "Sync pending" (show count)

---

## Testing Strategy

### 1. Unit Tests (XCTest)
**Target Coverage: 80%+**

```swift
// Example: ViewModel Tests
class AuthViewModelTests: XCTestCase {
    var sut: AuthViewModel!
    var mockAPIService: MockAPIService!
    var mockKeychainService: MockKeychainService!

    override func setUp() {
        mockAPIService = MockAPIService()
        mockKeychainService = MockKeychainService()
        sut = AuthViewModel(
            apiService: mockAPIService,
            keychainService: mockKeychainService
        )
    }

    func testSignupSuccess() async {
        // Given
        let email = "test@example.com"
        let password = "password123"
        mockAPIService.signupResult = .success(mockAuthResponse)

        // When
        await sut.signup(email: email, password: password)

        // Then
        XCTAssertTrue(sut.isAuthenticated)
        XCTAssertNotNil(sut.parent)
        XCTAssertNil(sut.errorMessage)
        XCTAssertTrue(mockKeychainService.savedToken != nil)
    }
}
```

### 2. UI Tests (XCUITest)
**Test Critical Paths:**
- Authentication flow (signup, login, logout)
- Add child → Start practice → Answer question → View progress
- Create quiz → Take quiz → Submit → View results

```swift
class PracticeFlowUITests: XCTestCase {
    func testCompletePracticeFlow() {
        let app = XCUIApplication()
        app.launch()

        // Login
        app.buttons["Login"].tap()
        app.textFields["Email"].tap()
        app.textFields["Email"].typeText("parent@example.com")
        app.secureTextFields["Password"].tap()
        app.secureTextFields["Password"].typeText("password123")
        app.buttons["Sign In"].tap()

        // Select child
        app.scrollViews.otherElements.buttons["Child Card: Alex"].tap()

        // Start practice
        app.buttons["Start Practice"].tap()
        app.buttons["Subject: Math"].tap()
        app.buttons["Begin Practice"].tap()

        // Answer question
        waitForElement(app.staticTexts["Question Stem"])
        app.buttons["Option A"].tap()
        app.buttons["Submit Answer"].tap()

        // Verify feedback
        XCTAssertTrue(app.staticTexts["Correct!"].exists || app.staticTexts["Incorrect"].exists)
    }
}
```

### 3. Integration Tests
- API service with mock URLSession
- Core Data stack with in-memory store
- Sync logic with mock network responses

### 4. Snapshot Tests (Optional)
- Use **SnapshotTesting** library
- Capture screenshots of key screens
- Detect visual regressions

---

## Development Phases

### Phase 0: Project Setup (Week 1)
**Goal**: Bootstrap iOS project with architecture foundations

- [ ] Create new Xcode project (iOS App, SwiftUI, SwiftData)
- [ ] Configure Swift Package Manager dependencies
- [ ] Set up Core Data schema (entities defined above)
- [ ] Implement NetworkManager + APIService
- [ ] Create design system (colors, fonts, spacing)
- [ ] Set up MVVM folder structure:
  ```
  StudyBuddy/
    ├── Models/
    ├── ViewModels/
    ├── Views/
    │   ├── Authentication/
    │   ├── Dashboard/
    │   ├── Practice/
    │   ├── Quiz/
    │   ├── Progress/
    │   └── Components/
    ├── Services/
    │   ├── APIService.swift
    │   ├── CoreDataStack.swift
    │   ├── KeychainService.swift
    │   └── SyncManager.swift
    ├── Utilities/
    └── Resources/
  ```
- [ ] Configure SwiftLint rules
- [ ] Set up CI/CD with Fastlane (optional)

**Deliverables:**
- Empty project builds and runs
- Network layer can make authenticated requests
- Core Data schema created
- Design tokens accessible via `Color.brandPrimary`, etc.

---

### Phase 1: Authentication & Child Management (Week 2-3)
**Goal**: Users can sign up, log in, and manage child profiles

#### Backend (Reuse Existing)
- ✅ `/auth/signup`, `/auth/login` endpoints
- ✅ `/children` CRUD endpoints
- ✅ Token-based authentication

#### iOS Implementation
- [ ] **WelcomeView**: Sign up / Login buttons
- [ ] **AuthenticationView**: Email + password form with validation
- [ ] **AuthViewModel**: Signup, login, logout, token persistence
- [ ] **KeychainService**: Secure token storage
- [ ] **DashboardView**: Empty state when no children
- [ ] **ChildrenView**: List, add, edit, delete child profiles
- [ ] **ChildrenViewModel**: Manage children list + selection
- [ ] **Unit Tests**: AuthViewModel, ChildrenViewModel
- [ ] **UI Tests**: Signup flow, login flow, add child

**Acceptance Criteria:**
- User can sign up with email/password
- User can log in and token persists across app launches
- User can add/edit/delete child profiles
- Selected child persists in app state
- Logout clears token and returns to Welcome screen

---

### Phase 2: Practice Mode (Adaptive) (Week 4-6)
**Goal**: Core adaptive practice functionality

#### Backend (Reuse Existing)
- ✅ `/questions/fetch` endpoint (adaptive difficulty)
- ✅ `/attempts` endpoint (log attempts)
- ✅ Subtopic selection logic
- ✅ Adaptive difficulty engine

#### iOS Implementation
- [ ] **PracticeSetupView**: Subject/topic/subtopic pickers, difficulty selector
- [ ] **PracticeSessionView**: Question display, MCQ options, timer
- [ ] **PracticeViewModel**: Fetch questions, submit attempts, track session
- [ ] **FeedbackView**: Correct/incorrect overlay with animations
- [ ] **SessionSummaryView**: End-of-session stats
- [ ] **Timer Logic**: Track time per question (time_spent_ms)
- [ ] **Confetti Animation**: Celebrate correct answers
- [ ] **Unit Tests**: PracticeViewModel, question selection logic
- [ ] **UI Tests**: Complete practice flow

**Acceptance Criteria:**
- User can select subject/topic/subtopic
- Questions load from backend (adaptive difficulty)
- User can answer questions and see immediate feedback
- Correct answers show green checkmark + confetti
- Incorrect answers show correct answer + rationale
- Session stats update in real-time
- Time tracking accurate (±100ms)

---

### Phase 3: Progress Tracking (Week 7-8)
**Goal**: Visualize child's learning progress

#### Backend (Reuse Existing)
- ✅ `/progress/{child_id}` endpoint
- ✅ `/sessions` endpoints

#### iOS Implementation
- [ ] **ProgressView**: Overview cards (streak, accuracy, questions)
- [ ] **SubjectBreakdownView**: Subject-level progress bars
- [ ] **SessionHistoryView**: List past sessions
- [ ] **ProgressViewModel**: Fetch and display progress data
- [ ] **Charts**: Use Swift Charts for visualizations
- [ ] **Unit Tests**: ProgressViewModel
- [ ] **UI Tests**: Navigate to progress, refresh data

**Acceptance Criteria:**
- Progress overview displays current streak, accuracy, totals
- Subject breakdown shows per-subject stats with color-coded bars
- Session history lists past practice sessions
- Data refreshes on pull-to-refresh
- Charts render correctly (if implemented)

---

### Phase 4: Quiz Mode (Week 9-11)
**Goal**: Timed, non-adaptive quiz functionality

#### Backend (Reuse Existing)
- ✅ `/quiz/sessions` endpoints (create, get, submit)
- ✅ Quiz selection logic (difficulty mix)
- ✅ Quiz grading service

#### iOS Implementation
- [ ] **QuizSetupView**: Subject/topic, question count slider, duration slider, difficulty mix
- [ ] **QuizSessionView**: All questions visible, timer, radio buttons
- [ ] **TimerView**: Countdown with sync logic (local + backend every 10s)
- [ ] **QuizResultsView**: Summary + incorrect items table
- [ ] **QuizViewModel**: Create, fetch, submit quiz
- [ ] **Jump Navigation**: Quick access to questions (for 10+)
- [ ] **Warning Modals**: Unanswered questions, time expiring
- [ ] **Unit Tests**: QuizViewModel, timer sync logic
- [ ] **UI Tests**: Create quiz → Take quiz → Submit → View results

**Acceptance Criteria:**
- User can configure quiz (subject, topic, count, duration, mix)
- Timer counts down and syncs with backend every 10s
- Timer persists across app backgrounding/foregrounding
- Auto-submit triggers at 0:00
- Quiz results show score + incorrect items with explanations
- Quiz history accessible from Progress tab

---

### Phase 5: Offline Support & Sync (Week 12-13)
**Goal**: App works offline with intelligent sync

#### iOS Implementation
- [ ] **SyncManager**: Upload pending attempts, fetch progress, prefetch questions
- [ ] **Network Reachability**: Monitor connection status
- [ ] **Offline Banner**: Display sync status at top
- [ ] **Conflict Resolution**: Implement merge strategies
- [ ] **Prefetch Logic**: Download 20 questions per subject when online
- [ ] **Core Data Sync**: Track syncStatus (.pending, .synced, .failed)
- [ ] **Unit Tests**: SyncManager, conflict resolution
- [ ] **Integration Tests**: Offline → Online sync scenarios

**Acceptance Criteria:**
- App functions offline (practice with cached questions)
- Pending attempts upload when connection restored
- Network status banner displays correctly
- No data loss during offline usage
- Prefetch keeps 20+ questions available per subject

---

### Phase 6: Polish & Testing (Week 14-15)
**Goal**: Production-ready app

#### Tasks
- [ ] **Accessibility**: VoiceOver support, Dynamic Type, color contrast
- [ ] **Localization**: Prepare for multiple languages (i18n)
- [ ] **Animations**: Refine transitions, micro-interactions
- [ ] **Error Handling**: User-friendly error messages
- [ ] **Onboarding**: First-time user tutorial
- [ ] **Performance**: Optimize Core Data queries, image caching
- [ ] **Test Coverage**: Achieve 80%+ unit test coverage
- [ ] **UI Tests**: All critical paths covered
- [ ] **Beta Testing**: TestFlight distribution to 10+ parents

**Deliverables:**
- App Store-ready build
- All tests passing (unit + UI)
- Accessibility audit complete
- Beta feedback incorporated

---

### Phase 7: App Store Launch (Week 16)
**Goal**: Submit to App Store

#### Tasks
- [ ] **App Store Assets**: Screenshots (6.5", 5.5"), app icon, preview video
- [ ] **App Store Listing**: Description, keywords, category (Education)
- [ ] **Privacy Policy**: COPPA compliance (children's app)
- [ ] **Terms of Service**: Legal agreements
- [ ] **App Review**: Submit for review, respond to feedback
- [ ] **Marketing**: Landing page, social media, parent forums

**Deliverables:**
- App live on App Store
- Monitoring/analytics in place (Crashlytics, Analytics)
- Support email configured

---

## Security & Privacy

### 1. Authentication
- **Token Storage**: Keychain with kSecAttrAccessibleWhenUnlocked
- **Token Transmission**: HTTPS only (TLS 1.3)
- **No Token Expiry**: Backend handles (future: implement refresh tokens)
- **Logout**: Clear token + Core Data cache

### 2. Data Encryption
- **At Rest**: Core Data encryption (NSFileProtectionComplete)
- **In Transit**: TLS 1.3 for all API calls
- **Keychain**: iOS system encryption

### 3. Privacy Compliance
- **COPPA**: Parental consent for children under 13
  - Parent creates account (gatekeeper)
  - No direct child data collection
  - No advertising or third-party tracking
- **Privacy Policy**: Disclose data collection (email, child grade, progress)
- **Data Retention**: Delete account → cascade delete all child data

### 4. Permissions
- **No Camera**: Questions are text-based (images optional future)
- **No Location**: ZIP code optional, entered manually
- **No Contacts**: No social features

### 5. App Transport Security
```xml
<!-- Info.plist -->
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <false/>
</dict>
```

---

## Performance Requirements

### 1. Response Times
- **App Launch**: < 2 seconds to dashboard (warm launch)
- **API Calls**: < 500ms average (P95 < 2s)
- **Question Load**: < 1 second (with prefetch)
- **Answer Submit**: < 300ms (optimistic UI update)
- **Progress Refresh**: < 500ms

### 2. Offline Performance
- **Question Cache**: 20+ questions per subject
- **Sync Latency**: Upload pending attempts < 5s
- **Prefetch**: Background task when app idle

### 3. Memory Management
- **Question Images**: SDWebImage for caching (if applicable)
- **Core Data Faulting**: Lazy load large objects
- **Background Modes**: Sync in background (fetch API)

### 4. Battery Efficiency
- **No Constant Polling**: Use Combine timers (pause when inactive)
- **Sync Trigger**: Only on foreground, every 30s if active
- **Background Sync**: Use BGTaskScheduler for daily prefetch

### 5. App Size
- **Target**: < 50 MB (initial download)
- **Assets**: Use asset catalogs with slicing
- **Compression**: HEIC for images, enable bitcode

---

## Analytics & Monitoring

### 1. Crash Reporting
- **Tool**: Firebase Crashlytics or Sentry
- **Metrics**: Crash-free rate > 99.5%

### 2. User Analytics
- **Tool**: Firebase Analytics or Mixpanel
- **Events to Track**:
  - User signup/login
  - Child created/deleted
  - Practice session started/completed
  - Quiz session started/completed
  - Question answered (correct/incorrect)
  - Sync completed/failed

### 3. Performance Monitoring
- **Tool**: Firebase Performance or New Relic
- **Metrics**:
  - API response times (P50, P95, P99)
  - App startup time
  - Screen load times
  - Network success rate

### 4. User Feedback
- **In-App**: Shake to send feedback (debug builds)
- **App Store**: Monitor reviews and ratings
- **Support**: Email support@studybuddy.com

---

## Open Questions & Future Enhancements

### Open Questions
1. **CloudKit Sync**: Should we support cross-device sync (iPad, iPhone)?
2. **Offline Quiz**: Allow quiz creation offline? (Complex timer sync)
3. **Question Images**: Support images in questions? (Storage implications)
4. **Localization**: Which languages to support first? (Spanish, French?)
5. **iPad Support**: Optimize UI for larger screens? (Split view, landscape)

### Future Enhancements (Post-MVP)
1. **Parent Dashboard**: Multi-child overview, weekly reports
2. **Gamification**: Badges, achievements, avatars
3. **Social Features**: Leaderboards (privacy-safe)
4. **Apple Watch**: Quick practice sessions on watch
5. **Widgets**: Home screen widget with daily streak
6. **Siri Shortcuts**: "Start math practice"
7. **Advanced Adaptive**: Implement full CLAUDE_PLAN4 (session-based, mastery detection)
8. **Teacher Mode**: Classroom management, bulk child accounts
9. **Custom Quizzes**: Parents create custom question sets
10. **Spaced Repetition**: Schedule review based on forgetting curve

---

## Appendix

### A. API Endpoint Summary

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/auth/signup` | POST | No | Create parent account |
| `/auth/login` | POST | No | Authenticate parent |
| `/children` | GET | Yes | List children |
| `/children` | POST | Yes | Create child profile |
| `/children/{id}` | PATCH | Yes | Update child profile |
| `/children/{id}` | DELETE | Yes | Delete child profile |
| `/questions/fetch` | POST | Yes | Fetch adaptive questions |
| `/attempts` | POST | Yes | Submit question attempt |
| `/progress/{child_id}` | GET | Yes | Fetch progress data |
| `/sessions` | GET | Yes | List practice sessions |
| `/sessions/{id}` | GET | Yes | Session detail |
| `/quiz/sessions` | POST | Yes | Create quiz session |
| `/quiz/sessions` | GET | Yes | List quiz sessions |
| `/quiz/sessions/{id}` | GET | Yes | Quiz detail + questions |
| `/quiz/sessions/{id}/submit` | POST | Yes | Submit quiz answers |
| `/standards` | GET | Yes | List learning standards |

### B. Core Data Schema ERD

```
┌─────────────┐
│   Parent    │
│────────────│
│ id          │
│ email       │
│ created_at  │
└─────────────┘
       │
       │ 1:N
       ▼
┌─────────────┐
│    Child    │
│────────────│
│ id          │
│ parent_id   │
│ name        │
│ grade       │
│ birthdate   │
│ zip         │
└─────────────┘
       │
       ├─────────────┬──────────────┐
       │ 1:N         │ 1:N          │ 1:N
       ▼             ▼              ▼
┌──────────┐  ┌──────────┐  ┌────────────┐
│ Attempt  │  │ Session  │  │ QuizSession│
└──────────┘  └──────────┘  └────────────┘
```

### C. SwiftUI Component Examples

#### Example: Question Card
```swift
struct QuestionCardView: View {
    let question: Question
    @Binding var selectedOption: String?
    let onSelect: (String) -> Void

    var body: some View {
        VStack(alignment: .leading, spacing: .spacing16) {
            // Subtopic Badge
            if let subtopic = question.sub_topic {
                Text(subtopic)
                    .font(.caption)
                    .padding(.horizontal, .spacing8)
                    .padding(.vertical, .spacing4)
                    .background(Color.brandPrimary.opacity(0.1))
                    .foregroundColor(.brandPrimary)
                    .cornerRadius(4)
            }

            // Question Stem
            Text(question.stem)
                .font(.heading2)
                .foregroundColor(.textPrimary)

            // Options
            ForEach(question.options.indices, id: \.self) { index in
                OptionButton(
                    label: question.options[index],
                    isSelected: selectedOption == question.options[index],
                    onTap: { onSelect(question.options[index]) }
                )
            }
        }
        .padding(.spacing16)
        .background(Color.surface)
        .cornerRadius(12)
        .shadow(radius: 4)
    }
}
```

---

## Conclusion

This specification provides a comprehensive blueprint for building the StudyBuddy iOS app with feature parity to the existing web application. The phased approach ensures incremental delivery of value while maintaining high code quality and user experience standards.

**Total Estimated Timeline: 16 weeks (4 months)**

**Key Success Metrics:**
- 1,000+ active parent accounts in first 6 months
- 80%+ session completion rate (practice mode)
- 4.5+ App Store rating
- < 1% crash rate
- 95%+ test coverage

**Next Steps:**
1. Review and approve this specification
2. Set up iOS project (Phase 0)
3. Begin Phase 1 development (Authentication & Child Management)
4. Weekly sprint reviews with stakeholder demos

---

**Document Version:** 1.0
**Last Updated:** January 13, 2026
**Author:** Claude Code
**Status:** Draft - Pending Review
