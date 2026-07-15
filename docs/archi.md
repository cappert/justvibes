================================================================================
                    JUSTVIBES ARCHITECTURE DIAGRAM
                   Netvibes-inspired Dashboard Platform
================================================================================

SYSTEM ARCHITECTURE
================================================================================

┌─────────────────────────────────────────────────────────────────────────┐
│                          FRONTEND (Angular)                              │
│                                                                          │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐     │
│  │  Dashboard View  │  │  Widget Library  │  │  User Settings   │     │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘     │
│           │                     │                     │                 │
│           └─────────────────────┼─────────────────────┘                 │
│                                 │                                        │
│                    ┌────────────▼───────────┐                           │
│                    │  RxJS State (NgRx)     │                           │
│                    │  Dashboard | Widget    │                           │
│                    │  User | Notification   │                           │
│                    └─────────┬──────────────┘                           │
│                              │                                           │
└──────────────────────────────┼───────────────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │  HTTP + WebSocket   │
                    │  REST Client        │
                    │  STOMP Subscriber   │
                    └─────────┬───────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
    ┌──────────────────────────────────────────────────────┐
    │       SPRING BOOT BACKEND (REST API Layer)           │
    │                                                      │
    │  ┌──────────────────────────────────────────────┐  │
    │  │  @RestControllers                           │  │
    │  │  ├─ DashboardController                     │  │
    │  │  │   GET /dashboards                        │  │
    │  │  │   POST /dashboards                       │  │
    │  │  │   PUT /dashboards/{id}                   │  │
    │  │  │   DELETE /dashboards/{id}                │  │
    │  │  │                                          │  │
    │  │  ├─ WidgetController                        │  │
    │  │  │   GET /widgets                           │  │
    │  │  │   POST /dashboards/{id}/widgets          │  │
    │  │  │   PUT /widgets/{id}                      │  │
    │  │  │   DELETE /widgets/{id}                   │  │
    │  │  │                                          │  │
    │  │  └─ AuthController                          │  │
    │  │      POST /auth/register                    │  │
    │  │      POST /auth/login                       │  │
    │  │      POST /auth/refresh                     │  │
    │  └──────────────────────────────────────────────┘  │
    │                          │                         │
    │  ┌──────────────────────▼─────────────────────┐   │
    │  │  Service Layer (Business Logic)            │   │
    │  │  ├─ DashboardService                       │   │
    │  │  │   • CRUD operations                     │   │
    │  │  │   • Layout management                   │   │
    │  │  │   • Share & permissions                 │   │
    │  │  │                                         │   │
    │  │  ├─ WidgetService                          │   │
    │  │  │   • Widget lifecycle                    │   │
    │  │  │   • Data aggregation                    │   │
    │  │  │   • External API calls                  │   │
    │  │  │                                         │   │
    │  │  ├─ AuthService                            │   │
    │  │  │   • JWT generation & validation         │   │
    │  │  │   • Password hashing                    │   │
    │  │  │                                         │   │
    │  │  └─ NotificationService                    │   │
    │  │      • Event broadcasting                  │   │
    │  └──────────────────────┬──────────────────────┘   │
    │                         │                          │
    │  ┌──────────────────────▼──────────────────────┐   │
    │  │  Repository Layer (JPA)                    │   │
    │  │  ├─ DashboardRepository                    │   │
    │  │  ├─ WidgetRepository                       │   │
    │  │  ├─ UserRepository                         │   │
    │  │  └─ UserWidgetRepository                   │   │
    │  └──────────────────────┬──────────────────────┘   │
    └─────────────────────────┼──────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │   PostgreSQL │ │    Kafka     │ │ WebSocket    │
    │   Database   │ │   Producer   │ │   Broadcast  │
    │              │ │              │ │              │
    │  Tables:     │ │  Topics:     │ │  Endpoints:  │
    │  • users     │ │  • dashboard-│ │  /ws (STOMP) │
    │  • dashboards│ │    updates   │ │              │
    │  • widgets   │ │  • widget-   │ │ Subscribe:   │
    │  • config    │ │    data      │ │ /topic/dash/ │
    │  • metadata  │ │  • user-     │ │ {dashId}     │
    └──────────────┘ │    events    │ └──────────────┘
                     └──────┬───────┘
                            │
    ┌───────────────────────┼───────────────────────┐
    │                       │                       │
    │            ┌──────────▼────────────┐          │
    │            │  Kafka Consumers      │          │
    │            │  Event Listeners      │          │
    │            └──────────┬─────────────┘          │
    │                       │                       │
    │            ┌──────────▼────────────┐          │
    │            │ WebSocket Message    │          │
    │            │ Broker (STOMP)       │          │
    │            │ Real-time Pub/Sub    │          │
    │            └──────────┬─────────────┘          │
    │                       │                       │
    └───────────────────────┼───────────────────────┘
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
         ▼                  ▼                  ▼
    ┌─────────┐        ┌─────────┐       ┌─────────┐
    │Frontend │        │Frontend │       │Frontend │
    │  Client │        │  Client │       │  Client │
    │    1    │        │    2    │       │   N     │
    │         │        │         │       │         │
    │  UI     │        │  UI     │       │  UI     │
    │ Updates │        │ Updates │       │ Updates │
    └─────────┘        └─────────┘       └─────────┘


DATA MODEL
================================================================================

User (1) ──── (Many) Dashboards
  │
  ├─ id (PK)
  ├─ username
  ├─ email
  ├─ password_hash
  └─ created_at

Dashboard (1) ──── (Many) UserWidgets
  │
  ├─ id (PK)
  ├─ user_id (FK)
  ├─ name
  ├─ description
  ├─ layout (JSON)
  └─ created_at

Widget (1) ──── (Many) UserWidgets
  │
  ├─ id (PK)
  ├─ name
  ├─ type (news, weather, social, etc)
  ├─ config (JSON)
  └─ created_at

UserWidget (Junction)
  │
  ├─ id (PK)
  ├─ user_id (FK)
  ├─ widget_id (FK)
  ├─ dashboard_id (FK)
  ├─ position (x, y)
  ├─ size (width, height)
  └─ settings (JSON)


REQUEST/RESPONSE FLOW
================================================================================

USER ACTION
     │
     ▼
┌─────────────────────────────┐
│ Angular Component           │
│ Click / Form Submit         │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ Service (HTTP/API Call)     │
│ POST /api/dashboards        │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ Spring Controller           │
│ @PostMapping("/dashboards") │
│ Auth Check                  │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ Business Service            │
│ DashboardService.create()   │
│ Validation & Rules          │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ Repository (JPA)            │
│ dashboardRepo.save()        │
│ Database Insert             │
└────────┬────────────────────┘
         │
         ├──► Database (PostgreSQL)
         │
         ├──► Kafka Producer
         │    "dashboard-updates" topic
         │
         └──► WebSocket Broadcaster
              "/topic/dashboard/{id}"
              Notify all subscribers
         │
         ▼
┌─────────────────────────────┐
│ Connected Frontend Clients  │
│ Receive Message             │
│ Update NgRx Store           │
│ Re-render Component         │
└─────────────────────────────┘


API ENDPOINTS
================================================================================

AUTH
├─ POST   /api/auth/register          - Create user account
├─ POST   /api/auth/login             - Get JWT token
└─ POST   /api/auth/refresh           - Refresh token

DASHBOARDS
├─ GET    /api/dashboards             - List all dashboards
├─ POST   /api/dashboards             - Create dashboard
├─ GET    /api/dashboards/{id}        - Get dashboard details
├─ PUT    /api/dashboards/{id}        - Update dashboard
└─ DELETE /api/dashboards/{id}        - Delete dashboard

WIDGETS
├─ GET    /api/widgets                - List available widgets
├─ POST   /api/dashboards/{id}/widgets - Add widget to dashboard
├─ PUT    /api/widgets/{id}           - Update widget config
├─ DELETE /api/widgets/{id}           - Remove widget
└─ GET    /api/dashboards/{id}/widgets - Get dashboard widgets

REAL-TIME (WebSocket)
└─ WS     /api/ws                     - STOMP endpoint
   ├─ SUB  /topic/dashboard/{dashboardId}
   ├─ SUB  /topic/widget/{widgetId}
   └─ SUB  /topic/notifications


COMPONENT HIERARCHY (Angular)
================================================================================

AppComponent (Shell)
│
├─ DashboardModule
│  ├─ DashboardListComponent
│  │  ├─ DashboardCardComponent
│  │  └─ CreateDashboardModalComponent
│  │
│  └─ DashboardDetailComponent
│     ├─ ToolbarComponent
│     ├─ GridLayoutComponent
│     │  ├─ WidgetComponent (N)
│     │  │  ├─ WidgetHeaderComponent
│     │  │  ├─ WidgetContentComponent
│     │  │  └─ WidgetFooterComponent
│     │  └─ AddWidgetComponent
│     └─ SettingsComponent
│
├─ SharedModule
│  ├─ HeaderComponent
│  ├─ SidenavComponent
│  ├─ PipesModule
│  └─ DirectivesModule
│
└─ Services
   ├─ DashboardService
   ├─ WidgetService
   ├─ AuthService
   ├─ WebSocketService
   ├─ NotificationService
   ├─ ApiService
   └─ StorageService


KAFKA TOPICS & EVENTS
================================================================================

Topic: dashboard-updates
├─ Event: DASHBOARD_CREATED
│  └─ Payload: {dashboardId, userId, name, timestamp}
│
├─ Event: DASHBOARD_UPDATED
│  └─ Payload: {dashboardId, userId, changes, timestamp}
│
└─ Event: DASHBOARD_DELETED
   └─ Payload: {dashboardId, userId, timestamp}

Topic: widget-data
├─ Event: WIDGET_ADDED
│  └─ Payload: {widgetId, dashboardId, type, config, timestamp}
│
├─ Event: WIDGET_UPDATED
│  └─ Payload: {widgetId, dashboardId, config, timestamp}
│
└─ Event: WIDGET_DATA_FETCHED
   └─ Payload: {widgetId, data, timestamp}

Topic: user-events
├─ Event: USER_LOGGED_IN
│  └─ Payload: {userId, timestamp, ip}
│
├─ Event: USER_UPDATED_PROFILE
│  └─ Payload: {userId, changes, timestamp}
│
└─ Event: USER_LOGGED_OUT
   └─ Payload: {userId, timestamp}


WEBSOCKET MESSAGE FLOW (STOMP)
================================================================================

SUBSCRIPTION:
┌─────────────────────────────────────────────────────────┐
│ Client connects to: ws://localhost:8081/api/ws          │
│                                                         │
│ STOMP CONNECT                                           │
│ CONNECT                                                 │
│ accept-version:1.0,1.1,1.2                              │
│                                                         │
│ Server responds: CONNECTED                              │
│                                                         │
│ Client subscribes:                                      │
│ SUBSCRIBE                                               │
│ id:sub-1                                                │
│ destination:/topic/dashboard/123                        │
│                                                         │
│ Server sends updates:                                   │
│ MESSAGE                                                 │
│ destination:/topic/dashboard/123                        │
│ content-type:application/json                           │
│                                                         │
│ {"type":"DASHBOARD_UPDATED","dashboardId":123,...}     │
└─────────────────────────────────────────────────────────┘


BACKEND SERVICE RESPONSIBILITIES
================================================================================

DashboardService
├─ CRUD operations (Create, Read, Update, Delete)
├─ Layout management (Grid configuration)
├─ Share dashboards with other users
├─ Permission checks (View, Edit, Delete)
└─ Publish events to Kafka

WidgetService
├─ Register new widget types
├─ Add/Remove widgets from dashboards
├─ Fetch external data (News API, Weather API, etc)
├─ Cache widget data (Redis)
├─ Aggregate data from multiple sources
└─ Publish widget updates to Kafka

AuthService
├─ User registration & validation
├─ Login & JWT token generation
├─ Token refresh & validation
├─ Password hashing (BCrypt)
└─ CORS & CSRF protection

NotificationService
├─ Build notification messages
├─ Publish to Kafka topics
├─ Broadcast via WebSocket
└─ User notification preferences


FRONTEND SERVICE RESPONSIBILITIES
================================================================================

DashboardService
├─ HTTP calls to /api/dashboards
├─ Cache dashboard list in memory
├─ Manage optimistic updates
└─ Error handling & retry logic

WidgetService
├─ HTTP calls to /api/widgets
├─ Manage widget configurations
├─ Handle widget-specific data
└─ Performance optimization

WebSocketService
├─ Establish STOMP connection
├─ Manage subscriptions
├─ Handle reconnection logic
├─ Message serialization/deserialization
└─ Broadcast received messages to subscribers

AuthService
├─ Handle login/register forms
├─ Store JWT token (localStorage)
├─ HTTP interceptor for Authorization header
├─ Redirect on 401/403 errors
└─ Token refresh on expiry


DATA PERSISTENCE STRATEGY
================================================================================

PostgreSQL Tables:
├─ users
│  ├─ id (UUID, PK)
│  ├─ username (VARCHAR, UNIQUE)
│  ├─ email (VARCHAR, UNIQUE)
│  ├─ password_hash (VARCHAR)
│  ├─ created_at (TIMESTAMP)
│  └─ updated_at (TIMESTAMP)
│
├─ dashboards
│  ├─ id (UUID, PK)
│  ├─ user_id (UUID, FK -> users)
│  ├─ name (VARCHAR)
│  ├─ description (TEXT)
│  ├─ layout (JSON) - Grid configuration
│  ├─ created_at (TIMESTAMP)
│  └─ updated_at (TIMESTAMP)
│
├─ widgets
│  ├─ id (UUID, PK)
│  ├─ name (VARCHAR)
│  ├─ type (VARCHAR) - news, weather, social, etc
│  ├─ description (TEXT)
│  ├─ config (JSON) - Default configuration schema
│  ├─ created_at (TIMESTAMP)
│  └─ updated_at (TIMESTAMP)
│
├─ user_widgets (Junction table)
│  ├─ id (UUID, PK)
│  ├─ user_id (UUID, FK -> users)
│  ├─ widget_id (UUID, FK -> widgets)
│  ├─ dashboard_id (UUID, FK -> dashboards)
│  ├─ position_x (INTEGER)
│  ├─ position_y (INTEGER)
│  ├─ width (INTEGER)
│  ├─ height (INTEGER)
│  ├─ settings (JSON) - Widget-specific settings
│  ├─ created_at (TIMESTAMP)
│  └─ updated_at (TIMESTAMP)
│
└─ dashboard_layouts
   ├─ id (UUID, PK)
   ├─ dashboard_id (UUID, FK -> dashboards)
   ├─ grid_columns (INTEGER)
   ├─ grid_rows (INTEGER)
   ├─ mobile_layout (JSON)
   ├─ created_at (TIMESTAMP)
   └─ updated_at (TIMESTAMP)

Redis Cache:
├─ user_sessions:{userId} - User session data
├─ dashboard_data:{dashboardId} - Dashboard snapshot
├─ widget_data:{widgetId} - Widget data cache
└─ rate_limits:{userId} - API rate limiting


SECURITY ARCHITECTURE
================================================================================

Authentication:
├─ Registration: Username/Email + Password (BCrypt)
├─ Login: Credentials → JWT Token (Expires in 24h)
├─ Refresh: Old token → New token (7 days refresh window)
└─ Logout: Token blacklist (Redis)

Authorization:
├─ @PreAuthorize("hasRole('USER')")
├─ @PreAuthorize("@dashboardService.canAccess(#id)")
└─ Custom annotations for permission checks

CORS:
├─ Allowed Origins: http://localhost:4200, http://localhost:3000
├─ Allowed Methods: GET, POST, PUT, DELETE, OPTIONS, PATCH
├─ Credentials: true
└─ Max Age: 3600 seconds

WebSocket Security:
├─ Requires JWT token in header
├─ Validates token on connection
├─ Isolated subscriptions per user
└─ Message validation & sanitization


ERROR HANDLING STRATEGY
================================================================================

HTTP Error Responses:
├─ 400 Bad Request - Invalid input validation
├─ 401 Unauthorized - Missing or invalid JWT
├─ 403 Forbidden - Insufficient permissions
├─ 404 Not Found - Resource doesn't exist
├─ 409 Conflict - Resource already exists
├─ 500 Internal Server Error - Unexpected errors
└─ 503 Service Unavailable - Kafka/DB issues

WebSocket Error Handling:
├─ Connection failures → Auto-reconnect with exponential backoff
├─ Message delivery errors → Client-side retry logic
├─ Subscription failures → Fallback to HTTP polling
└─ Session timeout → Re-authenticate

Frontend Error Handling:
├─ HTTP Interceptor catches all errors
├─ Global error toast notifications
├─ Logging to console/backend
└─ User-friendly error messages


SCALABILITY CONSIDERATIONS
================================================================================

Horizontal Scaling:
├─ Multiple Spring Boot instances behind load balancer
├─ Session state in Redis (shared across instances)
├─ Kafka brokers for distributed event streaming
├─ Database connection pooling (HikariCP)
└─ CDN for static assets

Performance Optimization:
├─ Lazy loading of dashboards & widgets
├─ Pagination for large lists
├─ Widget data caching (Redis, 5-minute TTL)
├─ Database query optimization (indexes, eager loading)
├─ Frontend state optimization (OnPush change detection)
└─ Compression (gzip) for HTTP responses

Monitoring & Observability:
├─ Spring Actuator for health checks
├─ Prometheus metrics export
├─ Kafka consumer lag monitoring
├─ Frontend error tracking (Sentry)
└─ ELK stack for centralized logging


================================================================================
END OF ARCHITECTURE DOCUMENT
================================================================================
