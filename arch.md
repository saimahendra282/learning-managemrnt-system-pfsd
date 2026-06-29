# Project Architecture Documentation

## 1. Tech Stack

### Frontend
- **Languages**: HTML5, CSS3, JavaScript (Vanilla & AJAX)
- **Frameworks/Libraries**: Bootstrap 4, jQuery, Popper.js
- **Icons & UI**: Font Awesome
- **Template Engine**: Django Template Language (DTL)
- **Form Handling**: django-crispy-forms

### Backend
- **Core Framework**: Python with Django (>=3.2)
- **API Layer**: Django REST Framework (DRF)
- **Database ORM**: Django ORM
- **Authentication**: Django Auth, Session Authentication, Custom User Models
- **File Handling**: Pillow (Images), xhtml2pdf/reportlab (PDF Generation)
- **Static Files**: Whitenoise
- **Other Utilities**: django-environ, django-filter, django-cors-headers

### Database
- **Production DB**: PostgreSQL (psycopg2)
- **Development DB**: SQLite3

---

## 2. Architecture Overview

**Architecture Type**: Monolithic Application (Django MTV - Model-Template-View pattern) with RESTful API extensions.

The application follows a traditional server-rendered monolithic structure where the Django backend handles business logic, database operations, and renders HTML templates directly to the client. It also exposes API endpoints for potential decoupled client interactions.

### Frontend Architecture

The frontend is divided into role-based components (Admin, Lecturer, Student), primarily utilizing server-side rendering with conditional blocks based on user permissions.

```mermaid
graph TD
    Client[Browser Client]
    
    subgraph UI Layout
        Base[base.html - Root Layout]
        Nav[navbar.html - Top Navigation & Search]
        Sidebar[aside.html - Role-based Menu]
        Base --> Nav
        Base --> Sidebar
    end
    
    subgraph Core Components
        Auth[Authentication & Profiles]
        Dash[Role-based Dashboards]
        CourseUI[Course Registration & Uploads]
        QuizUI[Quiz Interface & Timer]
        ResultUI[Results & GPA Display]
    end
    
    Client --> Base
    Base --> Auth
    Base --> Dash
    Base --> CourseUI
    Base --> QuizUI
    Base --> ResultUI
```

### Backend Architecture

The backend is modularized into several Django applications, each responsible for a specific domain of the student management system.

```mermaid
graph TD
    Request[HTTP Request] --> Routing[config/urls.py]
    
    subgraph Django Apps
        Accounts[accounts - User Mgmt, Roles]
        Core[core - Sessions, Semesters, News]
        Course[course - Departments, Courses, Allocations, Materials]
        Quiz[quiz - Tests, Questions, Sittings]
        Result[result - Grading, GPA, Taken Courses]
        Search[search - Global Search Engine]
        Payments[payments - Stripe Integration]
    end
    
    Routing --> Accounts
    Routing --> Core
    Routing --> Course
    Routing --> Quiz
    Routing --> Result
    Routing --> Search
    Routing --> Payments
```

### Database Schema

Here are the detailed schemas mapping out the entities across the system.

```mermaid
erDiagram
    %% Accounts App
    USER ||--o| STUDENT : "is"
    USER ||--o| LECTURER : "is"
    USER ||--o| DEPT_HEAD : "is"
    USER ||--o| PARENT : "is"
    
    %% Course App
    PROGRAM ||--o{ DEPARTMENT : "has"
    DEPARTMENT ||--o{ STUDENT : "belongs_to"
    PROGRAM ||--o{ COURSE : "offers"
    COURSE ||--o{ UPLOAD : "has_materials"
    COURSE ||--o{ UPLOAD_VIDEO : "has_videos"
    COURSE }o--o{ COURSE_ALLOCATION : "assigned_via"
    USER }|--o{ COURSE_ALLOCATION : "assigned_to"
    
    %% Core App
    SESSION ||--o{ SEMESTER : "contains"
    SESSION ||--o{ COURSE_ALLOCATION : "for"
    
    %% Quiz App
    COURSE ||--o{ QUIZ : "has"
    QUIZ }|--|{ QUESTION : "contains"
    QUESTION ||--o{ ANSWER : "has_options"
    USER ||--o{ SITTING : "takes"
    QUIZ ||--o{ SITTING : "taken_in"
    USER ||--o{ QUIZ_ATTEMPT_HISTORY : "has"
    
    %% Result App
    STUDENT ||--o{ TAKEN_COURSE : "registers"
    COURSE ||--o{ TAKEN_COURSE : "registered_by"
    STUDENT ||--o{ RESULT : "receives"
    COURSE ||--o{ RESULT : "awarded_for"
    QUIZ ||--o| RESULT : "contributes_to"
```

#### Detailed Schemas:
- **`accounts`**: User (Custom AbstractUser), Student (Level, Dept, Faculty), DepartmentHead, Parent.
- **`core`**: NewsAndEvents, ActivityLog, Session (Academic year), Semester (First/Second).
- **`course`**: Program (Faculty), Department, Course (Code, Credit, Level), CourseAllocation (Lecturer mapping), Upload (Files), UploadVideo (Media).
- **`quiz`**: Quiz, Question (Abstract), MCQuestion, Answer, EssayQuestion, Sitting (Tracks active quiz session, progress, answers), QuizAttemptHistory.
- **`result`**: TakenCourse (Registration), Result (CA, Mid-exam, Quiz, Exam, Total, Grade, Point, CGPA logic).

### Final Full Architecture & Data Flow

This chart represents the holistic view of how a user interacts with the system, how data flows through the monolithic stack, and how the database serves the applications.

```mermaid
graph LR
    User((User:\nStudent/Lecturer/Admin))
    
    subgraph Frontend [Frontend Layer - Browser]
        Templates[Django Templates + Bootstrap]
        AJAX[jQuery / AJAX calls]
    end
    
    subgraph Backend [Backend Layer - Django WSGI]
        URL[URL Router]
        MW[Middleware: Auth, Security, CSRF]
        
        subgraph Controllers [Views / API]
            View[Class-based & Function Views]
            API[DRF API ViewSets]
        end
        
        subgraph Business Logic
            Forms[Forms & Validators]
            Services[Utils & PDF Gen]
        end
        
        ORM[Django ORM]
    end
    
    subgraph Data [Data Layer]
        DB[(PostgreSQL / SQLite)]
        Media[Media Storage: Images/Videos]
        Stripe((Stripe API))
    end
    
    %% Connections
    User -->|HTTP GET/POST| Templates
    Templates -->|Submit Form| MW
    AJAX -->|REST API Calls| MW
    
    MW --> URL
    URL --> View
    URL --> API
    
    View --> Forms
    View --> Services
    API --> Services
    
    Forms --> ORM
    Services --> ORM
    
    ORM -->|Query/Mutate| DB
    View -->|Save Uploads| Media
    View -.->|Payment Processing| Stripe
    
    %% Responses
    View -->|Render HTML| Templates
    API -->|JSON Data| AJAX
    Templates -->|Rendered UI| User
```
