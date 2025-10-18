# URL Shortening Service Architecture

## System Overview

This URL shortening service consists of a React frontend, Go backend API, and PostgreSQL database, all containerized with Docker for easy deployment.

## Architecture Diagram

```mermaid
graph TB
    subgraph "Client Layer"
        A[React Frontend]
    end
    
    subgraph "API Gateway"
        B[Nginx Reverse Proxy]
    end
    
    subgraph "Application Layer"
        C[Go Backend API]
        D[JWT Authentication]
        E[URL Shortening Service]
        F[Analytics Service]
    end
    
    subgraph "Data Layer"
        G[PostgreSQL Database]
        H[Redis Cache]
    end
    
    subgraph "External Services"
        I[QR Code Generation API]
    end
    
    A --> B
    B --> C
    C --> D
    C --> E
    C --> F
    E --> G
    F --> G
    C --> H
    C --> I
    G --> G
```

## Database Schema

```mermaid
erDiagram
    users {
        uuid id PK
        string email UK
        string password_hash
        timestamp created_at
        timestamp updated_at
    }
    
    urls {
        uuid id PK
        uuid user_id FK
        string original_url
        string short_code UK
        string custom_alias UK
        timestamp created_at
        timestamp expires_at
        boolean password_protected
        string password_hash
        integer click_count
        boolean active
    }
    
    clicks {
        uuid id PK
        uuid url_id FK
        string ip_address
        string user_agent
        string referer
        string country
        string city
        timestamp clicked_at
    }
    
    url_analytics {
        uuid id PK
        uuid url_id FK
        date analytics_date
        integer daily_clicks
        integer unique_visitors
        json referral_sources
        json geographic_data
        timestamp updated_at
    }
    
    users ||--o{ urls : creates
    urls ||--o{ clicks : tracks
    urls ||--o{ url_analytics : generates
```

## Technology Stack

### Frontend
- React 18 with TypeScript
- Tailwind CSS for styling
- Axios for API calls
- React Router for navigation
- React Query for data fetching
- Chart.js for analytics visualization
- QRCode.js for QR code generation

### Backend
- Go 1.21+ with Gin framework
- GORM for database ORM
- PostgreSQL as primary database
- Redis for caching
- JWT for authentication
- bcrypt for password hashing
- Swagger for API documentation

### DevOps & Deployment
- Docker & Docker Compose
- Nginx as reverse proxy
- Multi-stage builds for optimization
- Environment-based configuration

## API Endpoints

### Authentication (Optional)
- POST /api/auth/register - User registration
- POST /api/auth/login - User login
- POST /api/auth/logout - User logout
- GET /api/auth/profile - Get user profile

### URL Management
- POST /api/urls/shorten - Create shortened URL
- GET /api/urls/:id - Get URL details
- PUT /api/urls/:id - Update URL settings
- DELETE /api/urls/:id - Delete URL
- GET /api/urls/history - Get user's URL history

### Analytics
- GET /api/analytics/:id - Get URL analytics
- GET /api/analytics/:id/clicks - Get click details
- GET /api/analytics/dashboard - Get dashboard data

### Redirection
- GET /:shortCode - Redirect to original URL
- GET /:shortCode/qr - Get QR code for URL

## Security Features

1. **Input Validation**: All URLs are validated and sanitized
2. **Rate Limiting**: API endpoints protected from abuse
3. **CORS Configuration**: Proper cross-origin resource sharing
4. **Security Headers**: HSTS, CSP, and other security headers
5. **Password Protection**: Optional password for sensitive URLs
6. **JWT Authentication**: Secure token-based authentication
7. **SQL Injection Prevention**: Parameterized queries via ORM

## Performance Optimizations

1. **Database Indexing**: Optimized queries for frequent lookups
2. **Redis Caching**: Cache frequently accessed URLs
3. **Connection Pooling**: Efficient database connections
4. **CDN Ready**: Static assets can be served via CDN
5. **Lazy Loading**: Frontend components loaded on demand
6. **Pagination**: Large datasets paginated for better performance

## Deployment Architecture

```mermaid
graph LR
    subgraph "Docker Containers"
        A[React Frontend Container]
        B[Go Backend Container]
        C[PostgreSQL Container]
        D[Redis Container]
        E[Nginx Container]
    end
    
    subgraph "External"
        F[Domain Name]
        G[SSL Certificate]
    end
    
    F --> E
    E --> A
    E --> B
    B --> C
    B --> D
    G --> E
```

## Development Workflow

1. **Local Development**: Docker Compose for local environment
2. **Version Control**: Git with feature branches
3. **Testing**: Unit tests, integration tests, and E2E tests
4. **CI/CD**: Automated testing and deployment
5. **Monitoring**: Application logs and metrics
6. **Backup**: Regular database backups

## Scalability Considerations

1. **Horizontal Scaling**: Multiple backend instances
2. **Database Sharding**: Partition data if needed
3. **Load Balancing**: Nginx as load balancer
4. **Microservices**: Potential to split services later
5. **Caching Strategy**: Multi-level caching approach