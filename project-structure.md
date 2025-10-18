# Project Structure

## Root Directory Structure

```
url-shortener/
├── README.md
├── docker-compose.yml
├── docker-compose.prod.yml
├── .env.example
├── .gitignore
├── Makefile
├── docs/
│   ├── api.md
│   ├── deployment.md
│   └── development.md
├── frontend/
│   ├── Dockerfile
│   ├── Dockerfile.prod
│   ├── package.json
│   ├── tsconfig.json
│   ├── tailwind.config.js
│   ├── vite.config.ts
│   ├── public/
│   │   ├── index.html
│   │   └── favicon.ico
│   └── src/
│       ├── main.tsx
│       ├── App.tsx
│       ├── index.css
│       ├── components/
│       │   ├── common/
│       │   │   ├── Header.tsx
│       │   │   ├── Footer.tsx
│       │   │   ├── Loading.tsx
│       │   │   └── ErrorBoundary.tsx
│       │   ├── url/
│       │   │   ├── UrlShortenerForm.tsx
│       │   │   ├── UrlResult.tsx
│       │   │   ├── UrlHistory.tsx
│       │   │   └── UrlDetails.tsx
│       │   ├── auth/
│       │   │   ├── LoginForm.tsx
│       │   │   ├── RegisterForm.tsx
│       │   │   └── ProfileForm.tsx
│       │   ├── analytics/
│       │   │   ├── Dashboard.tsx
│       │   │   ├── ClickChart.tsx
│       │   │   └── StatsCard.tsx
│       │   └── qr/
│       │       └── QRCodeDisplay.tsx
│       ├── pages/
│       │   ├── Home.tsx
│       │   ├── Dashboard.tsx
│       │   ├── History.tsx
│       │   ├── Login.tsx
│       │   ├── Register.tsx
│       │   └── Profile.tsx
│       ├── hooks/
│       │   ├── useAuth.ts
│       │   ├── useUrls.ts
│       │   └── useAnalytics.ts
│       ├── services/
│       │   ├── api.ts
│       │   ├── auth.ts
│       │   └── url.ts
│       ├── utils/
│       │   ├── validation.ts
│       │   ├── format.ts
│       │   └── constants.ts
│       ├── types/
│       │   ├── auth.ts
│       │   ├── url.ts
│       │   └── analytics.ts
│       └── store/
│           ├── authStore.ts
│           └── urlStore.ts
└── backend/
    ├── Dockerfile
    ├── Dockerfile.prod
    ├── go.mod
    ├── go.sum
    ├── main.go
    ├── config/
    │   ├── config.go
    │   └── database.go
    ├── cmd/
    │   └── server/
    │       └── main.go
    ├── internal/
    │   ├── api/
    │   │   ├── handlers/
    │   │   │   ├── auth.go
    │   │   │   ├── url.go
    │   │   │   └── analytics.go
    │   │   ├── middleware/
    │   │   │   ├── auth.go
    │   │   │   ├── cors.go
    │   │   │   ├── ratelimit.go
    │   │   │   └── logging.go
    │   │   └── routes/
    │   │       └── routes.go
    │   ├── models/
    │   │   ├── user.go
    │   │   ├── url.go
    │   │   ├── click.go
    │   │   └── analytics.go
    │   ├── services/
    │   │   ├── auth.go
    │   │   ├── url.go
    │   │   ├── analytics.go
    │   │   └── qr.go
    │   ├── repository/
    │   │   ├── user.go
    │   │   ├── url.go
    │   │   ├── click.go
    │   │   └── analytics.go
    │   └── utils/
    │       ├── hash.go
    │       ├── validation.go
    │       └── response.go
    ├── migrations/
    │   ├── 001_create_users_table.sql
    │   ├── 002_create_urls_table.sql
    │   ├── 003_create_clicks_table.sql
    │   └── 004_create_analytics_table.sql
    └── tests/
        ├── integration/
        ├── unit/
        └── fixtures/
```

## Frontend Structure Details

### Components Organization

- **common/**: Reusable UI components used across the application
- **url/**: Components specific to URL shortening functionality
- **auth/**: Authentication-related components
- **analytics/**: Dashboard and analytics visualization components
- **qr/**: QR code generation and display components

### State Management

- **hooks/**: Custom React hooks for API calls and state management
- **services/**: API service layer for backend communication
- **store/**: Global state management (using Zustand or Context API)
- **types/**: TypeScript type definitions

### Pages Structure

Each page represents a major route in the application:
- **Home**: Main URL shortening interface
- **Dashboard**: User dashboard with analytics overview
- **History**: User's URL history with search and filtering
- **Login/Register**: Authentication pages
- **Profile**: User profile management

## Backend Structure Details

### Clean Architecture Approach

The backend follows a clean architecture pattern with clear separation of concerns:

- **api/**: HTTP layer (handlers, middleware, routes)
- **services/**: Business logic layer
- **repository/**: Data access layer
- **models/**: Data models and structures
- **utils/**: Utility functions and helpers

### Database Organization

- **migrations/**: SQL migration files for database schema changes
- **config/**: Database connection and configuration management

### Testing Structure

- **integration/**: Integration tests for API endpoints
- **unit/**: Unit tests for individual functions and services
- **fixtures/**: Test data and mock objects

## Docker Configuration

### Development Environment

- **docker-compose.yml**: Development setup with hot reloading
- **frontend/Dockerfile**: Development build with volume mounting
- **backend/Dockerfile**: Development build with air for hot reloading

### Production Environment

- **docker-compose.prod.yml**: Production-optimized setup
- **frontend/Dockerfile.prod**: Multi-stage build for production
- **backend/Dockerfile.prod**: Optimized production build

## Configuration Management

### Environment Variables

- **.env.example**: Template for environment variables
- **config/config.go**: Configuration struct and loading logic

### Key Configuration Areas

- Database connection settings
- JWT secret and expiration
- Redis connection
- CORS settings
- Rate limiting configuration
- File upload limits

## Development Workflow

### Local Development

1. Copy `.env.example` to `.env` and configure
2. Run `make dev` to start development environment
3. Frontend runs on http://localhost:3000
4. Backend API runs on http://localhost:8080
5. Database accessible on localhost:5432

### Build Process

1. **Frontend**: Vite build process with TypeScript compilation
2. **Backend**: Go build with dependency optimization
3. **Docker**: Multi-stage builds for production optimization

### Testing Strategy

1. **Frontend**: Jest + React Testing Library for component tests
2. **Backend**: Go's built-in testing package for unit and integration tests
3. **E2E**: Playwright or Cypress for end-to-end testing

## Deployment Structure

### Production Deployment

1. **Nginx**: Reverse proxy and static file serving
2. **Backend**: Go application server
3. **Frontend**: Static files served by Nginx
4. **Database**: PostgreSQL with persistent volumes
5. **Cache**: Redis for performance optimization

### CI/CD Pipeline

1. **GitHub Actions**: Automated testing and building
2. **Docker Registry**: Container image storage
3. **Deployment**: Automated deployment to production environment

## Security Considerations

### Frontend Security

- Input validation and sanitization
- XSS prevention
- Secure token storage
- HTTPS enforcement

### Backend Security

- JWT token validation
- Rate limiting
- CORS configuration
- SQL injection prevention
- Password hashing with bcrypt

### Infrastructure Security

- Environment variable encryption
- Database access control
- Network security rules
- SSL/TLS termination