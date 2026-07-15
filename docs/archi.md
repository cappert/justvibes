# JustVibes Architecture

## System Overview

The JustVibes platform is a Netvibes-inspired dashboard application built with:
- **Frontend**: Angular (latest) with RxJS/NgRx
- **Backend**: Spring Boot 3.2 with REST API
- **Real-time**: Kafka for event streaming + WebSocket (STOMP) for live updates
- **Database**: PostgreSQL with Redis caching

---

## High-Level Architecture Diagram

```
FRONTEND (Angular)
    |
    +-- Dashboard View
    +-- Widget Library  
    +-- User Settings
    |
    +-- RxJS State Management (NgRx)
    |
    ===== HTTP + WebSocket (STOMP) =====
    |
SPRING BOOT BACKEND
    |
    +-- Controllers
    |   +-- DashboardController
    |   +-- WidgetController
    |   +-- AuthController
    |
    +-- Services
    |   +-- DashboardService
    |   +-- WidgetService
    |   +-- AuthService
    |   +-- NotificationService
    |
    +-- Repositories (JPA)
    |   +-- DashboardRepository
    |   +-- WidgetRepository
    |   +-- UserRepository
    |
    ===== Three-way Output =====
    |
    +-- PostgreSQL Database
    +-- Kafka Topics
    +-- WebSocket Broadcast
```

---

## Data Model / Entity Relationships

```
USER (1)
  ├─ id (UUID, PK)
  ├─ username (VARCHAR, UNIQUE)
  ├─ email (VARCHAR, UNIQUE)
  ├─ password_hash (VARCHAR)
  ├─ created_at, updated_at
  |
  └─ (1:N) DASHBOARD
       ├─ id (UUID, PK)
       ├─ user_id (UUID, FK)
       ├─ name, description
       ├─ layout (JSON)
       ├─ created_at, updated_at
       |
       └─ (1:N) USER_WIDGET (Junction Table)
            ├─ id (UUID, PK)
            ├─ user_id, widget_id, dashboard_id
            ├─ position_x, position_y
            ├─ width, height
            ├─ settings (JSON)
            ├─ created_at, updated_at
            |
            └─ (N:1) WIDGET
                 ├─ id (UUID, PK)
                 ├─ name, type, description
                 ├─ config (JSON)
                 ├─ created_at, updated_at
```

---

## Request/Response Flow

```
User Action in Angular Component
    |
    v
Service makes HTTP Call
    POST /api/dashboards
    |
    v
Spring Controller receives request
    @PostMapping("/dashboards")
    Authentication Check
    |
    v
Business Service processes logic
    DashboardService.create()
    Validation & Business Rules
    |
    v
Repository saves to database
    dashboardRepo.save()
    |
    +--- PostgreSQL: INSERT
    +--- Kafka: PUBLISH event to dashboard-updates topic
    +--- WebSocket: BROADCAST to /topic/dashboard/{id}
    |
    v
Response returned to client
    |
    v
Connected clients receive update
    Update NgRx Store
    Re-render Angular Components
```

---

## API Endpoints

### Authentication
```
POST   /api/auth/register      - Create user account
POST   /api/auth/login         - Get JWT token
POST   /api/auth/refresh       - Refresh token
```

### Dashboards
```
GET    /api/dashboards         - List all dashboards
POST   /api/dashboards         - Create dashboard
GET    /api/dashboards/{id}    - Get dashboard details
PUT    /api/dashboards/{id}    - Update dashboard
DELETE /api/dashboards/{id}    - Delete dashboard
```

### Widgets
```
GET    /api/widgets                    - List available widgets
POST   /api/dashboards/{id}/widgets    - Add widget to dashboard
PUT    /api/widgets/{id}               - Update widget config
DELETE /api/widgets/{id}               - Remove widget
GET    /api/dashboards/{id}/widgets    - Get dashboard widgets
```

### WebSocket (Real-time)
```
WS     /api/ws                 - STOMP endpoint
SUB    /topic/dashboard/{dashboardId}
SUB    /topic/widget/{widgetId}
SUB    /topic/notifications
```

---

## Angular Component Hierarchy

```
AppComponent (Root)
  |
  +-- DashboardModule
  |    |
  |    +-- DashboardListComponent
  |    |    +-- DashboardCardComponent
  |    |    +-- CreateDashboardModalComponent
  |    |
  |    +-- DashboardDetailComponent
  |         +-- ToolbarComponent
  |         +-- GridLayoutComponent
  |         |    +-- WidgetComponent (N instances)
  |         |    |    +-- WidgetHeaderComponent
  |         |    |    +-- WidgetContentComponent
  |         |    |    +-- WidgetFooterComponent
  |         |    +-- AddWidgetComponent
  |         +-- SettingsComponent
  |
  +-- SharedModule
  |    +-- HeaderComponent
  |    +-- SidenavComponent
  |    +-- PipesModule
  |    +-- DirectivesModule
  |
  +-- Core Services
       +-- DashboardService
       +-- WidgetService
       +-- AuthService
       +-- WebSocketService
       +-- NotificationService
       +-- ApiService
       +-- StorageService
```

---

## Kafka Topics and Events

### Topic: dashboard-updates
```
Event: DASHBOARD_CREATED
Payload: { dashboardId, userId, name, timestamp }

Event: DASHBOARD_UPDATED
Payload: { dashboardId, userId, changes, timestamp }

Event: DASHBOARD_DELETED
Payload: { dashboardId, userId, timestamp }
```

### Topic: widget-data
```
Event: WIDGET_ADDED
Payload: { widgetId, dashboardId, type, config, timestamp }

Event: WIDGET_UPDATED
Payload: { widgetId, dashboardId, config, timestamp }

Event: WIDGET_DATA_FETCHED
Payload: { widgetId, data, timestamp }
```

### Topic: user-events
```
Event: USER_LOGGED_IN
Payload: { userId, timestamp, ip }

Event: USER_UPDATED_PROFILE
Payload: { userId, changes, timestamp }

Event: USER_LOGGED_OUT
Payload: { userId, timestamp }
```

---

## WebSocket Message Flow (STOMP Protocol)

```
1. Client initiates connection
   ws://localhost:8081/api/ws
   
2. STOMP CONNECT frame
   accept-version: 1.0,1.1,1.2
   
3. Server responds CONNECTED
   version: 1.2
   
4. Client SUBSCRIBE to topic
   destination: /topic/dashboard/123
   id: sub-1
   
5. Server sends MESSAGE
   destination: /topic/dashboard/123
   content-type: application/json
   
   Payload: {
     "type": "DASHBOARD_UPDATED",
     "dashboardId": 123,
     "changes": { ... }
   }
   
6. Client receives and re-renders
   Update NgRx store
   Trigger change detection
```

---

## PostgreSQL Database Schema

### Table: users
```
id              UUID PRIMARY KEY
username        VARCHAR UNIQUE NOT NULL
email           VARCHAR UNIQUE NOT NULL
password_hash   VARCHAR NOT NULL
created_at      TIMESTAMP DEFAULT NOW()
updated_at      TIMESTAMP DEFAULT NOW()
```

### Table: dashboards
```
id              UUID PRIMARY KEY
user_id         UUID FOREIGN KEY REFERENCES users(id)
name            VARCHAR NOT NULL
description     TEXT
layout          JSON
created_at      TIMESTAMP DEFAULT NOW()
updated_at      TIMESTAMP DEFAULT NOW()
```

### Table: widgets
```
id              UUID PRIMARY KEY
name            VARCHAR NOT NULL
type            VARCHAR NOT NULL
description     TEXT
config          JSON
created_at      TIMESTAMP DEFAULT NOW()
updated_at      TIMESTAMP DEFAULT NOW()
```

### Table: user_widgets (Junction)
```
id              UUID PRIMARY KEY
user_id         UUID FOREIGN KEY REFERENCES users(id)
widget_id       UUID FOREIGN KEY REFERENCES widgets(id)
dashboard_id    UUID FOREIGN KEY REFERENCES dashboards(id)
position_x      INTEGER
position_y      INTEGER
width           INTEGER
height          INTEGER
settings        JSON
created_at      TIMESTAMP DEFAULT NOW()
updated_at      TIMESTAMP DEFAULT NOW()
```

### Redis Cache Keys
```
user_sessions:{userId}       - User session data
dashboard_data:{dashboardId} - Dashboard snapshot
widget_data:{widgetId}       - Widget data cache
rate_limits:{userId}         - API rate limiting
```

---

## Backend Service Responsibilities

### DashboardService
- CRUD operations (Create, Read, Update, Delete)
- Layout management (Grid configuration)
- Share dashboards with other users
- Permission checks (View, Edit, Delete)
- Publish events to Kafka

### WidgetService
- Register new widget types
- Add/Remove widgets from dashboards
- Fetch external data (News API, Weather API, etc)
- Cache widget data in Redis
- Aggregate data from multiple sources
- Publish widget updates to Kafka

### AuthService
- User registration and validation
- Login and JWT token generation
- Token refresh and validation
- Password hashing (BCrypt)
- CORS and CSRF protection

### NotificationService
- Build notification messages
- Publish to Kafka topics
- Broadcast via WebSocket
- User notification preferences

---

## Frontend Service Responsibilities

### DashboardService
- HTTP calls to /api/dashboards
- Cache dashboard list in memory
- Manage optimistic updates
- Error handling and retry logic

### WidgetService
- HTTP calls to /api/widgets
- Manage widget configurations
- Handle widget-specific data
- Performance optimization

### WebSocketService
- Establish STOMP connection
- Manage subscriptions
- Handle reconnection logic
- Message serialization/deserialization
- Broadcast received messages to subscribers

### AuthService
- Handle login/register forms
- Store JWT token (localStorage)
- HTTP interceptor for Authorization header
- Redirect on 401/403 errors
- Token refresh on expiry

---

## Security Architecture

### Authentication Flow
```
User enters credentials
    |
    v
Backend validates with BCrypt
    |
    v
Generates JWT token (24-hour expiry)
    |
    v
Client stores in localStorage
    |
    v
Subsequent requests include JWT in header
    |
    v
Token refresh endpoint extends session (7-day window)
```

### Authorization
- Role-based access control (@PreAuthorize)
- Resource-level permission checks
- Dashboard owner validation
- Custom annotations for fine-grained control

### CORS Configuration
```
Allowed Origins: http://localhost:4200, http://localhost:3000
Allowed Methods: GET, POST, PUT, DELETE, OPTIONS, PATCH
Allow Credentials: true
Max Age: 3600 seconds
```

### WebSocket Security
- Requires JWT token in header
- Validates token on connection
- Isolated subscriptions per user
- Message validation and sanitization

---

## Error Handling Strategy

### HTTP Status Codes
```
400 Bad Request       - Invalid input validation
401 Unauthorized      - Missing or invalid JWT
403 Forbidden        - Insufficient permissions
404 Not Found        - Resource doesn't exist
409 Conflict         - Resource already exists
500 Internal Server  - Unexpected errors
503 Unavailable      - Kafka/DB connection issues
```

### WebSocket Error Handling
- Connection failures: Auto-reconnect with exponential backoff
- Message delivery errors: Client-side retry logic
- Subscription failures: Fallback to HTTP polling
- Session timeout: Re-authenticate with new token

### Frontend Error Handling
- HTTP Interceptor catches all errors
- Global error toast notifications
- Logging to console and backend
- User-friendly error messages

---

## Scalability & Performance

### Horizontal Scaling
- Multiple Spring Boot instances behind load balancer
- Session state in Redis (shared across instances)
- Kafka brokers for distributed event streaming
- Database connection pooling (HikariCP)
- CDN for static assets

### Performance Optimization
- Lazy loading of dashboards and widgets
- Pagination for large lists
- Widget data caching (Redis, 5-minute TTL)
- Database query optimization (indexes, eager loading)
- Frontend state optimization (OnPush change detection)
- Compression (gzip) for HTTP responses

### Monitoring and Observability
```
Spring Actuator      - Health checks (/actuator/health)
Prometheus           - Metrics export (/actuator/prometheus)
Kafka Monitoring     - Consumer lag tracking
Error Tracking       - Sentry integration
Centralized Logging  - ELK stack (Elasticsearch, Logstash, Kibana)
```

