# JustVibes Architecture

## System Overview

The JustVibes platform is a Netvibes-inspired dashboard application built with:
- **Frontend**: Angular (latest) with RxJS/NgRx
- **Backend**: Spring Boot 3.2 with REST API
- **Real-time**: Kafka for event streaming + WebSocket (STOMP) for live updates
- **Database**: PostgreSQL with Redis caching

## High-Level Architecture

```
FRONTEND (Angular)
    |
    | HTTP + WebSocket
    |
SPRING BOOT BACKEND (REST API)
    |
    +-- Database (PostgreSQL)
    +-- Message Broker (Kafka)
    +-- WebSocket Server (STOMP)
```

## Components

### Frontend (Angular)
- Dashboard Views
- Widget Library
- User Settings
- RxJS State Management (NgRx)

### Backend (Spring Boot)
- REST Controllers
- Business Logic Services
- JPA Repository Layer
- Kafka Producers/Consumers

### Data Storage
- PostgreSQL: User data, dashboards, widgets
- Redis: Sessions, caching, rate limiting

### Real-time Communication
- Kafka: Event streaming between services
- WebSocket (STOMP): Push updates to clients

## API Routes

### Authentication
- `POST /api/auth/register` - Register user
- `POST /api/auth/login` - Login
- `POST /api/auth/refresh` - Refresh JWT token

### Dashboards
- `GET /api/dashboards` - List dashboards
- `POST /api/dashboards` - Create dashboard
- `GET /api/dashboards/{id}` - Get dashboard
- `PUT /api/dashboards/{id}` - Update dashboard
- `DELETE /api/dashboards/{id}` - Delete dashboard

### Widgets
- `GET /api/widgets` - List available widgets
- `POST /api/dashboards/{id}/widgets` - Add widget to dashboard
- `PUT /api/widgets/{id}` - Update widget
- `DELETE /api/widgets/{id}` - Remove widget

### WebSocket (Real-time)
- `WS /api/ws` - STOMP connection
- `SUB /topic/dashboard/{dashboardId}` - Subscribe to dashboard updates
- `SUB /topic/widget/{widgetId}` - Subscribe to widget updates
- `SUB /topic/notifications` - Subscribe to notifications

## Database Schema

### Users Table
- id (UUID, PK)
- username (VARCHAR, UNIQUE)
- email (VARCHAR, UNIQUE)
- password_hash (VARCHAR)
- created_at, updated_at

### Dashboards Table
- id (UUID, PK)
- user_id (UUID, FK)
- name (VARCHAR)
- description (TEXT)
- layout (JSON)
- created_at, updated_at

### Widgets Table
- id (UUID, PK)
- name (VARCHAR)
- type (VARCHAR)
- description (TEXT)
- config (JSON)
- created_at, updated_at

### UserWidgets Table (Junction)
- id (UUID, PK)
- user_id (UUID, FK)
- widget_id (UUID, FK)
- dashboard_id (UUID, FK)
- position_x, position_y
- width, height
- settings (JSON)
- created_at, updated_at

## Kafka Topics

### dashboard-updates
- DASHBOARD_CREATED
- DASHBOARD_UPDATED
- DASHBOARD_DELETED

### widget-data
- WIDGET_ADDED
- WIDGET_UPDATED
- WIDGET_DATA_FETCHED

### user-events
- USER_LOGGED_IN
- USER_UPDATED_PROFILE
- USER_LOGGED_OUT

## Request Flow

1. User action in Angular component
2. Service makes HTTP call to backend API
3. Spring Controller receives request
4. Business Service processes logic
5. Repository saves to database
6. Event published to Kafka
7. WebSocket broadcaster sends to all subscribers
8. Connected clients receive update and re-render

## Security

### Authentication
- Username/Email + Password (BCrypt hashing)
- JWT tokens (24-hour expiry)
- Token refresh window (7 days)

### Authorization
- Role-based access control
- Resource-level permission checks
- Dashboard owner validation

### CORS
- Allowed origins: localhost:4200, localhost:3000
- Methods: GET, POST, PUT, DELETE, OPTIONS, PATCH

## Services

### Backend Services
- **DashboardService**: CRUD, layout management, sharing
- **WidgetService**: Widget lifecycle, data aggregation, external APIs
- **AuthService**: Registration, login, JWT management
- **NotificationService**: Event broadcasting

### Frontend Services
- **DashboardService**: HTTP calls, local caching, optimistic updates
- **WidgetService**: Widget management and configuration
- **AuthService**: Auth forms, token storage, JWT interceptor
- **WebSocketService**: STOMP connection, subscriptions

## Performance & Scalability

### Caching
- Redis for sessions and temporary data
- 5-minute TTL for widget data
- Rate limiting per user

### Database
- Connection pooling (HikariCP)
- Query optimization with indexes
- Eager loading for related entities

### Frontend
- Lazy loading modules
- OnPush change detection
- Pagination for lists

### Deployment
- Multiple Spring Boot instances behind load balancer
- Kafka cluster for distributed streaming
- PostgreSQL replication

## Monitoring

- Spring Actuator endpoints
- Prometheus metrics export
- Kafka consumer lag monitoring
- Error tracking (Sentry)
- ELK stack for logging
