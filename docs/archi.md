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
┌────────────────────────────────────────────────────────────────────┐
│                    FRONTEND (Angular)                              │
│                                                                    │
│  Dashboard View  |  Widget Library  |  User Settings              │
│        |              |                      |                    │
│        └──────────────┼──────────────────────┘                    │
│                       |                                            │
│           RxJS State Management (NgRx)                            │
│        Dashboard | Widget | User | Notification                  │
│                       |                                            │
└───────────────────────┼────────────────────────────────────────────┘
                        |
            HTTP + WebSocket (STOMP)
                        |
        ┌───────────────┼───────────────┐
        |               |               |
        v               v               v
┌──────────────────────────────────────────────────┐
│    SPRING BOOT BACKEND (REST API Layer)          │
│                                                  │
│  Controllers:                                    │
│  • DashboardController                           │
│  • WidgetController                              │
│  • AuthController                                │
│           |                                      │
│  Services:                                       │
│  • DashboardService                              │
│  • WidgetService                                 │
│  • AuthService                                   │
│  • NotificationService                           │
│           |                                      │
│  Repositories (JPA):                             │
│  • DashboardRepository                           │
│  • WidgetRepository                              │
│  • UserRepository                                │
│           |                                      │
└───────────┼──────────────────────────────────────┘
            |
    ┌───────┼───────┬──────────┐
    |       |       |          |
    v       v       v          v
PostgreSQL Kafka WebSocket Response
Database   Topics  Broadcast  to Client
```

---

## Data Model

```
User
├─ id (UUID, PK)
├─ username (VARCHAR, UNIQUE)
├─ email (VARCHAR, UNIQUE)
├─ password_hash (VARCHAR)
├─ created_at (TIMESTAMP)
└─ updated_at (TIMESTAMP)
    |
    └─── 1:N ──> Dashboard
         ├─ id (UUID, PK)
         ├─ user_id (UUID, FK)
         ├─ name (VARCHAR)
         ├─ description (TEXT)
         ├─ layout (JSON)
         ├─ created_at (TIMESTAMP)
         └─ updated_at (TIMESTAMP)
              |
              └─── 1:N ──> UserWidget (Junction)
                   ├─ id (UUID, PK)
                   ├─ user_id (UUID, FK)
                   ├─ widget_id (UUID, FK)
                   ├─ dashboard_id (UUID, FK)
                   ├─ position_x (INTEGER)
                   ├─ position_y (INTEGER)
                   ├─ width (INTEGER)
                   ├─ height (INTEGER)
                   ├─ settings (JSON)
                   ├─ created_at (TIMESTAMP)
                   └─ updated_at (TIMESTAMP)
                        |
                        └─── N:1 ──> Widget
                             ├─ id (UUID, PK)
                             ├─ name (VARCHAR)
                             ├─ type (VARCHAR)
                             ├─ description (TEXT)
                             ├─ config (JSON)
                             ├─ created_at (TIMESTAMP)
                             └─ updated_at (TIMESTAMP)
```

---

## Request/Response Flow

```
User Action (Angular Component Click)
        |
        v
Service (HTTP Call)
POST /api/dashboards
        |
        v
Spring Controller
@PostMapping("/dashboards")
Auth Check
        |
        v
Business Service
DashboardService.create()
Validation & Rules
        |
        v
Repository (JPA)
dashboardRepo.save()
        |
    ┌───┼───┬──────────┐
    |   |   |          |
    v   v   v          v
 PostgreSQL Kafka WebSocket Response
 Database   Event   Broadcast
            Topic   to Clients
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

## Component Hierarchy (Angular)

```
AppComponent
├── DashboardModule
│   ├── DashboardListComponent
│   │   ├── DashboardCardComponent
│   │   └── CreateDashboardModalComponent
│   └── DashboardDetailComponent
│       ├── ToolbarComponent
│       ├── GridLayoutComponent
│       │   ├── WidgetComponent (N instances)
│       │   ├── WidgetHeaderComponent
│       │   ├── WidgetContentComponent
│       │   ├── WidgetFooterComponent
│       │   └── AddWidgetComponent
│       └── SettingsComponent
├── SharedModule
│   ├── HeaderComponent
│   ├── SidenavComponent
│   ├── PipesModule
│   └── DirectivesModule
└── Services
    ├── DashboardService
    ├── WidgetService
    ├── AuthService
    ├── WebSocketService
    ├── NotificationService
    ├── ApiService
    └── StorageService
```

---

## Kafka Topics & Events

### Topic: dashboard-updates
```
DASHBOARD_CREATED
  Payload: { dashboardId, userId, name, timestamp }

DASHBOARD_UPDATED
  Payload: { dashboardId, userId, changes, timestamp }

DASHBOARD_DELETED
  Payload: { dashboardId, userId, timestamp }
```

### Topic: widget-data
```
WIDGET_ADDED
  Payload: { widgetId, dashboardId, type, config, timestamp }

WIDGET_UPDATED
  Payload: { widgetId, dashboardId, config, timestamp }

WIDGET_DATA_FETCHED
  Payload: { widgetId, data, timestamp }
```

### Topic: user-events
```
USER_LOGGED_IN
  Payload: { userId, timestamp, ip }

USER_UPDATED_PROFILE
  Payload: { userId, changes, timestamp }

USER_LOGGED_OUT
  Payload: { userId, timestamp }
```

---

## WebSocket Message Flow (STOMP)

```
Client connects to: ws://localhost:8081/api/ws

STOMP Frame Sequence:
┌──────────────────────────────┐
│ 1. Client CONNECT            │
│    accept-version:1.0,1.1,1.2│
├──────────────────────────────┤
│ 2. Server CONNECTED          │
│    version:1.2               │
├──────────────────────────────┤
│ 3. Client SUBSCRIBE          │
│    destination:/topic/dash/123
│    id:sub-1                  │
├──────────────────────────────┤
│ 4. Server MESSAGE            │
│    destination:/topic/dash/123
│    {type:UPDATED,id:123,...}│
└──────────────────────────────┘
```

---

## Database Schema

### PostgreSQL Tables

**users**
```
id (UUID, PK)
username (VARCHAR, UNIQUE)
email (VARCHAR, UNIQUE)
password_hash (VARCHAR)
created_at (TIMESTAMP)
updated_at (TIMESTAMP)
```

**dashboards**
```
id (UUID, PK)
user_id (UUID, FK → users)
name (VARCHAR)
description (TEXT)
layout (JSON)
created_at (TIMESTAMP)
updated_at (TIMESTAMP)
```

**widgets**
```
id (UUID, PK)
name (VARCHAR)
type (VARCHAR)
description (TEXT)
config (JSON)
created_at (TIMESTAMP)
updated_at (TIMESTAMP)
```

**user_widgets** (Junction)
```
id (UUID, PK)
user_id (UUID, FK → users)
widget_id (UUID, FK → widgets)
dashboard_id (UUID, FK → dashboards)
position_x (INTEGER)
position_y (INTEGER)
width (INTEGER)
height (INTEGER)
settings (JSON)
created_at (TIMESTAMP)
updated_at (TIMESTAMP)
```

### Redis Cache
```
user_sessions:{userId}        - Session data
dashboard_data:{dashboardId}   - Dashboard snapshot
widget_data:{widgetId}        - Widget data cache
rate_limits:{userId}          - API rate limiting
```

---

## Service Responsibilities

### Backend Services

**DashboardService**
- CRUD operations (Create, Read, Update, Delete)
- Layout management (Grid configuration)
- Share dashboards with other users
- Permission checks (View, Edit, Delete)
- Publish events to Kafka

**WidgetService**
- Register new widget types
- Add/Remove widgets from dashboards
- Fetch external data (News API, Weather API, etc)
- Cache widget data in Redis
- Aggregate data from multiple sources
- Publish widget updates to Kafka

**AuthService**
- User registration and validation
- Login and JWT token generation
- Token refresh and validation
- Password hashing (BCrypt)
- CORS and CSRF protection

**NotificationService**
- Build notification messages
- Publish to Kafka topics
- Broadcast via WebSocket
- User notification preferences

### Frontend Services

**DashboardService**
- HTTP calls to /api/dashboards
- Cache dashboard list in memory
- Manage optimistic updates
- Error handling and retry logic

**WidgetService**
- HTTP calls to /api/widgets
- Manage widget configurations
- Handle widget-specific data
- Performance optimization

**WebSocketService**
- Establish STOMP connection
- Manage subscriptions
- Handle reconnection logic
- Message serialization/deserialization
- Broadcast received messages to subscribers

**AuthService**
- Handle login/register forms
- Store JWT token (localStorage)
- HTTP interceptor for Authorization header
- Redirect on 401/403 errors
- Token refresh on expiry

---

## Security Architecture

### Authentication
- Registration: Username/Email + Password (BCrypt)
- Login: Credentials → JWT Token (24h expiry)
- Refresh: Old token → New token (7 days refresh window)
- Logout: Token blacklist (Redis)

### Authorization
- @PreAuthorize("hasRole('USER')")
- @PreAuthorize("@dashboardService.canAccess(#id)")
- Custom annotations for permission checks

### CORS
- Allowed Origins: localhost:4200, localhost:3000
- Allowed Methods: GET, POST, PUT, DELETE, OPTIONS, PATCH
- Credentials: true
- Max Age: 3600 seconds

### WebSocket Security
- Requires JWT token in header
- Validates token on connection
- Isolated subscriptions per user
- Message validation and sanitization

---

## Error Handling Strategy

### HTTP Error Responses
```
400 Bad Request         - Invalid input validation
401 Unauthorized        - Missing or invalid JWT
403 Forbidden          - Insufficient permissions
404 Not Found          - Resource doesn't exist
409 Conflict           - Resource already exists
500 Internal Server    - Unexpected errors
503 Service Unavailable - Kafka/DB issues
```

### WebSocket Error Handling
- Connection failures → Auto-reconnect with exponential backoff
- Message delivery errors → Client-side retry logic
- Subscription failures → Fallback to HTTP polling
- Session timeout → Re-authenticate

### Frontend Error Handling
- HTTP Interceptor catches all errors
- Global error toast notifications
- Logging to console/backend
- User-friendly error messages

---

## Scalability Considerations

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
- Spring Actuator for health checks
- Prometheus metrics export
- Kafka consumer lag monitoring
- Frontend error tracking (Sentry)
- ELK stack for centralized logging
