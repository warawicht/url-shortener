# Implementation Guide

## Technical Specifications

### URL Shortening Algorithm

#### Short Code Generation
- **Base62 Encoding**: Use characters [0-9][a-z][A-Z] for URL-friendly short codes
- **Length**: Default 6 characters (provides ~56.8 billion combinations)
- **Collision Handling**: Retry with different random seeds if collision occurs
- **Custom Alias**: Allow user-defined aliases with validation

#### Algorithm Implementation
```go
// Go implementation example
func GenerateShortCode() string {
    const charset = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    const length = 6
    
    seededRand := rand.New(rand.NewSource(time.Now().UnixNano()))
    b := make([]byte, length)
    for i := range b {
        b[i] = charset[seededRand.Intn(len(charset))]
    }
    return string(b)
}
```

### Database Schema Details

#### Users Table
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    is_active BOOLEAN DEFAULT true
);

CREATE INDEX idx_users_email ON users(email);
```

#### URLs Table
```sql
CREATE TABLE urls (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    original_url TEXT NOT NULL,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    custom_alias VARCHAR(50) UNIQUE,
    title VARCHAR(255),
    description TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE,
    password_hash VARCHAR(255),
    click_count INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    UNIQUE(user_id, custom_alias) WHERE custom_alias IS NOT NULL
);

CREATE INDEX idx_urls_short_code ON urls(short_code);
CREATE INDEX idx_urls_user_id ON urls(user_id);
CREATE INDEX idx_urls_custom_alias ON urls(custom_alias) WHERE custom_alias IS NOT NULL;
CREATE INDEX idx_urls_expires_at ON urls(expires_at) WHERE expires_at IS NOT NULL;
```

#### Clicks Table
```sql
CREATE TABLE clicks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    url_id UUID REFERENCES urls(id) ON DELETE CASCADE,
    ip_address INET,
    user_agent TEXT,
    referer TEXT,
    country VARCHAR(2),
    city VARCHAR(100),
    device_type VARCHAR(50),
    browser VARCHAR(100),
    os VARCHAR(100),
    clicked_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_clicks_url_id ON clicks(url_id);
CREATE INDEX idx_clicks_clicked_at ON clicks(clicked_at);
CREATE INDEX idx_clicks_country ON clicks(country);
```

#### Analytics Table
```sql
CREATE TABLE url_analytics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    url_id UUID REFERENCES urls(id) ON DELETE CASCADE,
    analytics_date DATE NOT NULL,
    daily_clicks INTEGER DEFAULT 0,
    unique_visitors INTEGER DEFAULT 0,
    referral_sources JSONB,
    geographic_data JSONB,
    device_data JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(url_id, analytics_date)
);

CREATE INDEX idx_analytics_url_date ON url_analytics(url_id, analytics_date);
```

### API Endpoints Specification

#### Authentication Endpoints

##### POST /api/auth/register
```json
Request:
{
    "email": "user@example.com",
    "password": "securePassword123",
    "first_name": "John",
    "last_name": "Doe"
}

Response (201):
{
    "message": "User registered successfully",
    "user": {
        "id": "uuid",
        "email": "user@example.com",
        "first_name": "John",
        "last_name": "Doe"
    }
}

Error (400):
{
    "error": "Invalid email format"
}
```

##### POST /api/auth/login
```json
Request:
{
    "email": "user@example.com",
    "password": "securePassword123"
}

Response (200):
{
    "token": "jwt_token_here",
    "user": {
        "id": "uuid",
        "email": "user@example.com",
        "first_name": "John",
        "last_name": "Doe"
    }
}

Error (401):
{
    "error": "Invalid credentials"
}
```

#### URL Management Endpoints

##### POST /api/urls/shorten
```json
Request:
{
    "original_url": "https://example.com/very/long/url",
    "custom_alias": "my-link",
    "title": "My Link",
    "description": "Description of my link",
    "expires_at": "2024-12-31T23:59:59Z",
    "password": "optional_password"
}

Response (201):
{
    "id": "uuid",
    "short_code": "abc123",
    "short_url": "https://short.ly/abc123",
    "original_url": "https://example.com/very/long/url",
    "custom_alias": "my-link",
    "title": "My Link",
    "expires_at": "2024-12-31T23:59:59Z",
    "password_protected": true,
    "created_at": "2024-01-01T12:00:00Z"
}

Error (400):
{
    "error": "Invalid URL format"
}
```

##### GET /api/urls/history
```json
Query Parameters:
- page: integer (default: 1)
- limit: integer (default: 10, max: 100)
- search: string (optional)
- sort: string (created_at|click_count, default: created_at)
- order: string (asc|desc, default: desc)

Response (200):
{
    "urls": [
        {
            "id": "uuid",
            "short_code": "abc123",
            "short_url": "https://short.ly/abc123",
            "original_url": "https://example.com/very/long/url",
            "title": "My Link",
            "click_count": 42,
            "created_at": "2024-01-01T12:00:00Z",
            "expires_at": null
        }
    ],
    "pagination": {
        "page": 1,
        "limit": 10,
        "total": 25,
        "total_pages": 3
    }
}
```

#### Analytics Endpoints

##### GET /api/analytics/:id
```json
Response (200):
{
    "url": {
        "id": "uuid",
        "short_code": "abc123",
        "original_url": "https://example.com/very/long/url",
        "total_clicks": 150,
        "unique_visitors": 120,
        "created_at": "2024-01-01T12:00:00Z"
    },
    "daily_stats": [
        {
            "date": "2024-01-01",
            "clicks": 25,
            "unique_visitors": 20
        }
    ],
    "top_countries": [
        {
            "country": "US",
            "count": 50
        }
    ],
    "top_referrers": [
        {
            "referrer": "google.com",
            "count": 30
        }
    ],
    "devices": [
        {
            "device": "desktop",
            "count": 80
        }
    ]
}
```

### Frontend Component Specifications

#### UrlShortenerForm Component
```typescript
interface UrlShortenerFormProps {
    onUrlCreated: (url: UrlData) => void;
    user?: User;
}

interface UrlData {
    id: string;
    short_code: string;
    short_url: string;
    original_url: string;
    custom_alias?: string;
    title?: string;
    expires_at?: string;
    password_protected: boolean;
}

// Form validation schema
const urlSchema = z.object({
    original_url: z.string().url("Invalid URL format"),
    custom_alias: z.string().optional(),
    title: z.string().max(255).optional(),
    description: z.string().max(500).optional(),
    expires_at: z.string().datetime().optional(),
    password: z.string().min(6).optional()
});
```

#### Analytics Dashboard Components
```typescript
interface DashboardProps {
    userId?: string;
}

interface ClickData {
    date: string;
    clicks: number;
    unique_visitors: number;
}

interface CountryData {
    country: string;
    count: number;
}

interface DeviceData {
    device: string;
    count: number;
}

// Chart.js configuration
const chartOptions = {
    responsive: true,
    plugins: {
        legend: {
            position: 'top' as const,
        },
        title: {
            display: true,
            text: 'Click Analytics'
        }
    },
    scales: {
        y: {
            beginAtZero: true
        }
    }
};
```

### Security Implementation

#### JWT Token Configuration
```go
type JWTConfig struct {
    SecretKey     string        `env:"JWT_SECRET" env-required:"true"`
    ExpiryTime    time.Duration `env:"JWT_EXPIRY" env-default:"24h"`
    RefreshTime   time.Duration `env:"JWT_REFRESH" env-default:"168h"` // 7 days
}

type Claims struct {
    UserID string `json:"user_id"`
    Email  string `json:"email"`
    jwt.RegisteredClaims
}
```

#### Rate Limiting Implementation
```go
type RateLimiter struct {
    limiter *rate.Limiter
}

func NewRateLimiter(rps int) *RateLimiter {
    return &RateLimiter{
        limiter: rate.NewLimiter(rate.Limit(rps), rps),
    }
}

func (rl *RateLimiter) Middleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        if !rl.limiter.Allow() {
            c.JSON(http.StatusTooManyRequests, gin.H{
                "error": "Rate limit exceeded",
            })
            c.Abort()
            return
        }
        c.Next()
    }
}
```

#### Input Validation
```go
type URLRequest struct {
    OriginalURL string    `json:"original_url" binding:"required,url"`
    CustomAlias *string   `json:"custom_alias" binding:"omitempty,min=3,max=50,alphanum"`
    Title       *string   `json:"title" binding:"omitempty,max=255"`
    Description *string   `json:"description" binding:"omitempty,max=500"`
    ExpiresAt   *time.Time `json:"expires_at" binding:"omitempty,after"`
    Password    *string   `json:"password" binding:"omitempty,min=6,max=100"`
}

func ValidateURL(url string) error {
    if !strings.HasPrefix(url, "http://") && !strings.HasPrefix(url, "https://") {
        return errors.New("URL must start with http:// or https://")
    }
    
    parsedURL, err := url.Parse(url)
    if err != nil {
        return errors.New("Invalid URL format")
    }
    
    if parsedURL.Host == "" {
        return errors.New("URL must have a valid host")
    }
    
    return nil
}
```

### QR Code Generation

#### Backend Implementation
```go
import (
    "github.com/boombuler/barcode"
    "github.com/boombuler/barcode/qr"
)

func GenerateQRCode(text string, size int) ([]byte, error) {
    qrCode, err := qr.Encode(text, qr.M, qr.Auto)
    if err != nil {
        return nil, err
    }
    
    qrCode, err = barcode.Scale(qrCode, size, size)
    if err != nil {
        return nil, err
    }
    
    buf := new(bytes.Buffer)
    err = png.Encode(buf, qrCode)
    if err != nil {
        return nil, err
    }
    
    return buf.Bytes(), nil
}
```

#### Frontend Implementation
```typescript
import QRCode from 'qrcode';

interface QRCodeDisplayProps {
    url: string;
    size?: number;
    download?: boolean;
}

const QRCodeDisplay: React.FC<QRCodeDisplayProps> = ({ 
    url, 
    size = 200, 
    download = false 
}) => {
    const [qrCodeUrl, setQrCodeUrl] = useState<string>('');
    
    useEffect(() => {
        const generateQR = async () => {
            try {
                const qrDataUrl = await QRCode.toDataURL(url, {
                    width: size,
                    margin: 2,
                    color: {
                        dark: '#000000',
                        light: '#FFFFFF'
                    }
                });
                setQrCodeUrl(qrDataUrl);
            } catch (err) {
                console.error('Error generating QR code:', err);
            }
        };
        
        generateQR();
    }, [url, size]);
    
    const downloadQR = () => {
        const link = document.createElement('a');
        link.download = 'qrcode.png';
        link.href = qrCodeUrl;
        link.click();
    };
    
    return (
        <div className="qr-code-container">
            <img src={qrCodeUrl} alt="QR Code" />
            {download && (
                <button onClick={downloadQR} className="download-btn">
                    Download QR Code
                </button>
            )}
        </div>
    );
};
```

### Real-time Analytics

#### WebSocket Implementation
```go
type WebSocketHub struct {
    clients    map[*Client]bool
    broadcast  chan []byte
    register   chan *Client
    unregister chan *Client
}

type Client struct {
    hub  *WebSocketHub
    conn *websocket.Conn
    send chan []byte
}

func (h *WebSocketHub) Run() {
    for {
        select {
        case client := <-h.register:
            h.clients[client] = true
            
        case client := <-h.unregister:
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
            }
            
        case message := <-h.broadcast:
            for client := range h.clients {
                select {
                case client.send <- message:
                default:
                    close(client.send)
                    delete(h.clients, client)
                }
            }
        }
    }
}
```

#### Frontend WebSocket Client
```typescript
class AnalyticsWebSocket {
    private ws: WebSocket | null = null;
    private urlId: string;
    private onMessage: (data: any) => void;
    
    constructor(urlId: string, onMessage: (data: any) => void) {
        this.urlId = urlId;
        this.onMessage = onMessage;
        this.connect();
    }
    
    private connect() {
        this.ws = new WebSocket(`ws://localhost:8080/ws/analytics/${this.urlId}`);
        
        this.ws.onmessage = (event) => {
            const data = JSON.parse(event.data);
            this.onMessage(data);
        };
        
        this.ws.onclose = () => {
            // Reconnect after 5 seconds
            setTimeout(() => this.connect(), 5000);
        };
    }
    
    disconnect() {
        if (this.ws) {
            this.ws.close();
        }
    }
}
```

### Performance Optimizations

#### Database Query Optimization
```go
// Efficient pagination with cursor-based pagination
func (r *URLRepository) GetURLsByUser(userID string, cursor string, limit int) ([]URL, string, error) {
    query := r.db.Where("user_id = ?", userID)
    
    if cursor != "" {
        query = query.Where("created_at < ?", cursor)
    }
    
    var urls []URL
    err := query.Order("created_at DESC").
        Limit(limit + 1).
        Find(&urls).Error
    
    if err != nil {
        return nil, "", err
    }
    
    nextCursor := ""
    if len(urls) > limit {
        nextCursor = urls[limit-1].CreatedAt.Format(time.RFC3339)
        urls = urls[:limit]
    }
    
    return urls, nextCursor, nil
}
```

#### Caching Strategy
```go
type CacheService struct {
    redis *redis.Client
}

func (c *CacheService) GetURL(shortCode string) (*URL, error) {
    // Try cache first
    cached, err := c.redis.Get(context.Background(), "url:"+shortCode).Result()
    if err == nil {
        var url URL
        json.Unmarshal([]byte(cached), &url)
        return &url, nil
    }
    
    // Fallback to database
    url, err := c.repository.GetByShortCode(shortCode)
    if err != nil {
        return nil, err
    }
    
    // Cache the result
    data, _ := json.Marshal(url)
    c.redis.Set(context.Background(), "url:"+shortCode, data, time.Hour)
    
    return url, nil
}
```

This implementation guide provides detailed technical specifications for building the URL shortening service with all the requested features. The guide covers database design, API specifications, frontend components, security measures, and performance optimizations.