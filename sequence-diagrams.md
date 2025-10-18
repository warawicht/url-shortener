# URL Shortening Service - Sequence Diagrams

## 1. URL Shortening Use Case

### 1.1 Basic URL Shortening (Anonymous User)

```mermaid
sequenceDiagram
    participant User
    participant Frontend as React Frontend
    participant API as Go Backend API
    participant DB as PostgreSQL
    participant Cache as Redis
    
    User->>Frontend: Enters long URL
    Frontend->>Frontend: Validate URL format
    Frontend->>API: POST /api/urls/shorten
    Note over Frontend,API: Request: {original_url: "https://example.com/very/long/path"}
    
    API->>API: Generate short code (6 chars)
    API->>API: Check for collision in cache
    API->>DB: Check if short_code exists
    DB-->>API: Return collision status
    
    alt Collision exists
        API->>API: Generate new short code
        API->>DB: Check again
    end
    
    API->>DB: INSERT into urls table
    Note over API,DB: Fields: id, original_url, short_code, created_at, click_count=0
    DB-->>API: Return URL record with ID
    
    API->>Cache: Cache the URL mapping
    Note over API,Cache: Key: "url:abc123", Value: URL object
    
    API-->>Frontend: Response with short URL
    Note over API,Frontend: Response: {id, short_code, short_url, original_url}
    
    Frontend->>Frontend: Generate QR code
    Frontend->>Frontend: Display results
    Frontend-->>User: Show shortened URL and QR code
```

### 1.2 URL Shortening with Custom Alias (Authenticated User)

```mermaid
sequenceDiagram
    participant User
    participant Frontend as React Frontend
    participant Auth as Auth Middleware
    participant API as Go Backend API
    participant DB as PostgreSQL
    participant Cache as Redis
    
    User->>Frontend: Login with credentials
    Frontend->>API: POST /api/auth/login
    API->>DB: Verify user credentials
    DB-->>API: User record
    API->>API: Generate JWT token
    API-->>Frontend: JWT token + user data
    Frontend->>Frontend: Store token in localStorage
    
    User->>Frontend: Enter URL with custom alias
    User->>Frontend: Set expiration date
    User->>Frontend: Add password protection
    Frontend->>Frontend: Validate all inputs
    
    Frontend->>API: POST /api/urls/shorten
    Note over Frontend,API: Headers: Authorization: Bearer JWT
    Note over Frontend,API: Body: {original_url, custom_alias, expires_at, password}
    
    API->>Auth: Validate JWT token
    Auth-->>API: User ID from token
    
    API->>DB: Check custom_alias availability
    DB-->>API: Alias availability status
    
    alt Alias not available
        API-->>Frontend: Error: Custom alias already taken
        Frontend-->>User: Show error message
    end
    
    API->>API: Hash password with bcrypt
    API->>DB: INSERT with user_id, custom_alias, password_hash
    DB-->>API: URL record with ID
    
    API->>Cache: Cache with custom alias
    API-->>Frontend: Success response with all URL data
    
    Frontend->>Frontend: Generate QR code
    Frontend->>Frontend: Show success message
    Frontend-->>User: Display shortened URL with options
```

## 2. URL Redirection Use Case

### 2.1 Basic URL Redirection

```mermaid
sequenceDiagram
    participant Visitor
    participant Browser
    participant Nginx as Reverse Proxy
    participant API as Go Backend API
    participant Cache as Redis
    participant DB as PostgreSQL
    participant Analytics as Analytics Service
    
    Visitor->>Browser: Click short URL: https://short.ly/abc123
    Browser->>Nginx: GET /abc123
    Nginx->>API: Forward request to backend
    
    API->>Cache: GET url:abc123
    alt Cache hit
        Cache-->>API: URL record
    else Cache miss
        API->>DB: SELECT * FROM urls WHERE short_code = 'abc123'
        DB-->>API: URL record
        API->>Cache: SET url:abc123 with TTL
    end
    
    alt URL not found
        API-->>Browser: 404 Not Found
        Browser-->>Visitor: Error page
    end
    
    alt URL expired
        API-->>Browser: 410 Gone (URL expired)
        Browser-->>Visitor: Expired URL page
    end
    
    alt Password protected
        API-->>Browser: 302 Redirect to password page
        Browser->>Visitor: Show password input form
        Visitor->>Browser: Enter password
        Browser->>API: POST password
        API->>API: Verify password hash
        alt Password correct
            API->>API: Continue with redirect
        else Password incorrect
            API-->>Browser: 401 Unauthorized
        end
    end
    
    API->>Analytics: Log click asynchronously
    Note over API,Analytics: Async: {url_id, ip, user_agent, referer, timestamp}
    
    API->>DB: UPDATE click_count = click_count + 1
    API-->>Browser: 301 Redirect to original_url
    Browser-->>Visitor: Load original URL
    
    par Analytics Processing
        Analytics->>DB: INSERT into clicks table
        Analytics->>API: Update daily analytics
        Analytics->>Cache: Update analytics cache
    end
```

### 2.2 Click Tracking with GeoIP and Device Detection

```mermaid
sequenceDiagram
    participant Visitor
    participant API as Go Backend API
    participant GeoIP as GeoIP Service
    Device as Device Parser
    participant DB as PostgreSQL
    participant Cache as Redis
    
    Visitor->>API: Click short URL
    API->>API: Extract request metadata
    Note over API: IP address, User-Agent, Referer
    
    API->>GeoIP: Lookup IP address
    GeoIP-->>API: Country, City, Coordinates
    
    API->>Device: Parse User-Agent
    Device-->>API: Device type, Browser, OS
    
    API->>Cache: Check if unique visitor
    Note over API,Cache: Key: "unique:url_id:ip_hash"
    
    alt First visit from this IP
        API->>Cache: Mark as unique visitor
        API->>DB: INSERT click with unique=true
    else Returning visitor
        API->>DB: INSERT click with unique=false
    end
    
    API->>DB: Update url_analytics table
    Note over API,DB: Increment daily_clicks, unique_visitors
    Note over API,DB: Update referral_sources, geographic_data, device_data
    
    API->>Cache: Invalidate analytics cache
    Note over API,Cache: DELETE analytics:url_id:*
    
    API-->>Visitor: Redirect to original URL
```

## 3. User Authentication Use Case

### 3.1 User Registration

```mermaid
sequenceDiagram
    participant User
    participant Frontend as React Frontend
    participant API as Go Backend API
    participant DB as PostgreSQL
    participant Email as Email Service
    
    User->>Frontend: Fill registration form
    Note over User,Frontend: Email, Password, First Name, Last Name
    Frontend->>Frontend: Client-side validation
    Note over Frontend: Email format, password strength
    
    Frontend->>API: POST /api/auth/register
    Note over Frontend,API: {email, password, first_name, last_name}
    
    API->>API: Validate input data
    API->>DB: SELECT * FROM users WHERE email = ?
    DB-->>API: Check if email exists
    
    alt Email already exists
        API-->>Frontend: 409 Conflict - Email already registered
        Frontend-->>User: Show error message
    end
    
    API->>API: Hash password with bcrypt (cost 12)
    API->>DB: INSERT into users table
    Note over API,DB: Fields: id, email, password_hash, first_name, last_name, created_at
    DB-->>API: Return user record with ID
    
    API->>API: Generate email verification token
    API->>Email: Send verification email
    Note over API,Email: Async: Send verification link
    
    API->>API: Generate JWT tokens (access + refresh)
    API-->>Frontend: 201 Created + tokens + user data
    
    Frontend->>Frontend: Store tokens securely
    Frontend->>Frontend: Update UI state (authenticated)
    Frontend-->>User: Show success message + dashboard
    
    par Email Verification
        User->>Email: Click verification link
        Email->>API: GET /api/auth/verify/:token
        API->>API: Validate token
        API->>DB: UPDATE users SET email_verified = true
        API-->>Email: Redirect to success page
    end
```

### 3.2 User Login with JWT

```mermaid
sequenceDiagram
    participant User
    participant Frontend as React Frontend
    participant API as Go Backend API
    participant DB as PostgreSQL
    participant Cache as Redis
    
    User->>Frontend: Enter login credentials
    Frontend->>Frontend: Validate form inputs
    
    Frontend->>API: POST /api/auth/login
    Note over Frontend,API: {email, password}
    
    API->>DB: SELECT * FROM users WHERE email = ?
    DB-->>API: User record (if exists)
    
    alt User not found
        API-->>Frontend: 401 Unauthorized
        Frontend-->>User: Invalid credentials error
    end
    
    API->>API: Compare password hash with bcrypt
    alt Password mismatch
        API-->>Frontend: 401 Unauthorized
        Frontend-->>User: Invalid credentials error
    end
    
    API->>API: Generate JWT access token (15 min expiry)
    API->>API: Generate JWT refresh token (7 days expiry)
    API->>Cache: Store refresh token
    Note over API,Cache: Key: "refresh:user_id", Value: token_hash
    
    API-->>Frontend: 200 OK + tokens + user data
    
    Frontend->>Frontend: Store access token in memory
    Frontend->>Frontend: Store refresh token in httpOnly cookie
    Frontend->>Frontend: Update authentication state
    Frontend-->>User: Redirect to dashboard
    
    par Token Refresh
        Frontend->>API: POST /api/auth/refresh
        Note over Frontend,API: Refresh token from cookie
        API->>Cache: Validate refresh token
        Cache-->>API: Token validity
        API->>API: Generate new access token
        API-->>Frontend: New access token
    end
```

## 4. Analytics Dashboard Use Case

### 4.1 Loading Analytics Dashboard

```mermaid
sequenceDiagram
    participant User
    participant Frontend as React Frontend
    participant API as Go Backend API
    participant Cache as Redis
    participant DB as PostgreSQL
    participant WS as WebSocket
    
    User->>Frontend: Navigate to analytics dashboard
    Frontend->>Frontend: Check authentication status
    
    Frontend->>API: GET /api/analytics/dashboard
    Note over Frontend,API: Headers: Authorization: Bearer JWT
    
    API->>API: Validate JWT token
    API->>Cache: GET dashboard:user_id
    alt Cache hit
        Cache-->>API: Cached dashboard data
    else Cache miss
        API->>DB: Query aggregated analytics
        Note over API,DB: Total clicks, unique visitors, top URLs
        DB-->>API: Raw analytics data
        API->>API: Process and aggregate data
        API->>Cache: SET dashboard:user_id with TTL 5min
    end
    
    API-->>Frontend: Dashboard data response
    Note over API,Frontend: {total_urls, total_clicks, top_urls, recent_activity}
    
    Frontend->>Frontend: Render dashboard components
    Frontend->>Frontend: Initialize charts with data
    
    Frontend->>WS: Connect to WebSocket
    Note over Frontend,WS: ws://localhost:8080/ws/analytics/user_id
    WS-->>Frontend: Connection established
    
    Frontend-->>User: Display analytics dashboard
    
    par Real-time Updates
        WS->>Frontend: New click event
        Note over WS,Frontend: {url_id, timestamp, country}
        Frontend->>Frontend: Update charts in real-time
        Frontend-->>User: Show live analytics updates
    end
```

### 4.2 Detailed URL Analytics

```mermaid
sequenceDiagram
    participant User
    participant Frontend as React Frontend
    participant API as Go Backend API
    participant Cache as Redis
    participant DB as PostgreSQL
    
    User->>Frontend: Click on URL in dashboard
    Frontend->>API: GET /api/analytics/:urlId
    Note over Frontend,API: Query params: date_range, group_by
    
    API->>Cache: GET analytics:url_id:date_range
    alt Cache hit
        Cache-->>API: Cached analytics data
    else Cache miss
        API->>DB: Query detailed analytics
        Note over API,DB: Daily clicks, countries, referrers, devices
        DB-->>API: Raw analytics data
        API->>API: Process data for charts
        API->>Cache: SET analytics:url_id:date_range with TTL
    end
    
    API-->>Frontend: Detailed analytics response
    Note over API,Frontend: {daily_stats, countries, referrers, devices}
    
    Frontend->>Frontend: Render multiple chart types
    Note over Frontend: Line chart for trends, Pie charts for distribution
    
    User->>Frontend: Change date range
    Frontend->>API: GET /api/analytics/:urlId?date_range=new_range
    API->>Cache: Check new cache key
    API-->>Frontend: Updated analytics data
    Frontend->>Frontend: Update all charts with new data
    
    User->>Frontend: Export analytics
    Frontend->>API: GET /api/analytics/:urlId/export?format=csv
    API->>DB: Query export data
    API-->>Frontend: CSV file download
    Frontend->>Frontend: Trigger file download
    Frontend-->>User: Download analytics CSV
```

## 5. URL Management Use Case

### 5.1 URL History with Search and Filtering

```mermaid
sequenceDiagram
    participant User
    participant Frontend as React Frontend
    participant API as Go Backend API
    participant DB as PostgreSQL
    participant Cache as Redis
    
    User->>Frontend: Navigate to URL history page
    Frontend->>API: GET /api/urls/history
    Note over Frontend,API: Query: page=1, limit=10, sort=created_at, order=desc
    
    API->>Cache: GET urls:user_id:page1:limit10
    alt Cache hit
        Cache-->>API: Cached URL list
    else Cache miss
        API->>DB: SELECT urls WHERE user_id = ?
        Note over API,DB: ORDER BY created_at DESC LIMIT 10 OFFSET 0
        DB-->>API: URL records with pagination
        API->>Cache: SET urls:user_id:page1:limit10 with TTL
    end
    
    API-->>Frontend: Paginated URL list
    Note over API,Frontend: {urls: [], pagination: {page, total, pages}}
    
    Frontend->>Frontend: Render URL list with pagination
    
    User->>Frontend: Enter search query
    Frontend->>API: GET /api/urls/history?search=query
    API->>DB: SELECT urls WHERE user_id = ? AND (original_url ILIKE ? OR title ILIKE ?)
    DB-->>API: Filtered results
    API-->>Frontend: Search results
    
    User->>Frontend: Apply filters (date range, status)
    Frontend->>API: GET /api/urls/history?filters=...
    API->>DB: Apply additional WHERE clauses
    DB-->>API: Filtered results
    API-->>Frontend: Filtered URL list
    
    User->>Frontend: Change sort order
    Frontend->>API: GET /api/urls/history?sort=click_count&order=desc
    API->>DB: Change ORDER BY clause
    DB-->>API: Sorted results
    API-->>Frontend: Sorted URL list
    Frontend->>Frontend: Update UI with new sort
```

### 5.2 URL Editing and Deletion

```mermaid
sequenceDiagram
    participant User
    participant Frontend as React Frontend
    participant API as Go Backend API
    participant DB as PostgreSQL
    participant Cache as Redis
    
    User->>Frontend: Click edit on URL
    Frontend->>Frontend: Show edit modal with current data
    
    User->>Frontend: Modify URL settings
    Note over User,Frontend: Change title, expiration, password
    Frontend->>Frontend: Validate form inputs
    
    Frontend->>API: PUT /api/urls/:id
    Note over Frontend,API: {title, expires_at, password}
    
    API->>API: Validate user ownership
    API->>DB: SELECT * FROM urls WHERE id = ? AND user_id = ?
    DB-->>API: URL record
    
    alt New password provided
        API->>API: Hash new password with bcrypt
    end
    
    API->>DB: UPDATE urls SET title=?, expires_at=?, password_hash=?
    DB-->>API: Updated URL record
    
    API->>Cache: Invalidate URL cache
    Note over API,Cache: DELETE url:short_code
    API->>Cache: Invalidate user URL list cache
    Note over API,Cache: DELETE urls:user_id:*
    
    API-->>Frontend: Updated URL data
    Frontend->>Frontend: Update UI with new data
    Frontend-->>User: Show success message
    
    User->>Frontend: Click delete on URL
    Frontend->>Frontend: Show confirmation dialog
    User->>Frontend: Confirm deletion
    
    Frontend->>API: DELETE /api/urls/:id
    API->>DB: DELETE FROM urls WHERE id = ? AND user_id = ?
    DB-->>API: Deletion confirmation
    
    API->>DB: DELETE FROM clicks WHERE url_id = ?
    API->>DB: DELETE FROM url_analytics WHERE url_id = ?
    
    API->>Cache: Clear all related caches
    API-->>Frontend: 204 No Content
    
    Frontend->>Frontend: Remove URL from list
    Frontend-->>User: Show deletion confirmation
```

## 6. QR Code Generation Use Case

### 6.1 QR Code Generation and Download

```mermaid
sequenceDiagram
    participant User
    participant Frontend as React Frontend
    participant API as Go Backend API
    participant QR as QR Service
    participant Cache as Redis
    
    User->>Frontend: Click "Generate QR Code" on URL
    Frontend->>API: GET /api/urls/:id/qr
    Note over Frontend,API: Query: size=200, format=png
    
    API->>Cache: GET qr:url_id:size200
    alt Cache hit
        Cache-->>API: Cached QR code image
    else Cache miss
        API->>DB: SELECT short_url FROM urls WHERE id = ?
        DB-->>API: Short URL
        
        API->>QR: Generate QR code
        Note over API,QR: Input: short_url, size, format
        QR-->>API: QR code image bytes
        
        API->>Cache: SET qr:url_id:size200 with TTL 24h
    end
    
    API-->>Frontend: QR code image (base64 or binary)
    Frontend->>Frontend: Display QR code in modal
    
    User->>Frontend: Click "Download QR Code"
    Frontend->>Frontend: Create download link
    Note over Frontend: Convert base64 to blob, create download URL
    
    Frontend-->>User: Trigger file download
    
    User->>Frontend: Request different QR format
    Frontend->>API: GET /api/urls/:id/qr?format=svg&size=300
    API->>QR: Generate SVG QR code
    QR-->>API: SVG QR code data
    API-->>Frontend: SVG QR code
    Frontend->>Frontend: Display SVG QR code
```

## 7. Real-time Updates Use Case

### 7.1 Live Analytics Updates

```mermaid
sequenceDiagram
    participant Visitor
    participant API as Go Backend API
    participant WS as WebSocket Hub
    participant Frontend as React Frontend
    participant User as Dashboard User
    
    Visitor->>API: Click short URL
    API->>API: Process click (as in redirection flow)
    
    API->>WS: Broadcast click event
    Note over API,WS: Message: {type: "click", url_id, timestamp, country}
    
    WS->>WS: Filter by subscribed users
    WS->>Frontend: Send to subscribed dashboard users
    Note over WS,Frontend: WebSocket message with click data
    
    Frontend->>Frontend: Parse WebSocket message
    Frontend->>Frontend: Update relevant charts
    Note over Frontend: Increment counters, update live map
    
    Frontend-->>User: Show real-time click notification
    Note over Frontend,User: "New click on your URL from United States"
    
    par Multiple concurrent clicks
        Visitor2->>API: Click another URL
        API->>WS: Broadcast another click event
        WS->>Frontend: Send update
        Frontend->>Frontend: Update multiple charts
    end
    
    User->>Frontend: Close dashboard tab
    Frontend->>WS: Disconnect WebSocket
    WS->>WS: Remove client from subscribers
```

These sequence diagrams provide detailed technical specifications for all major use cases in the URL shortening service, showing the exact flow of data between components, error handling paths, and performance optimizations like caching strategies.