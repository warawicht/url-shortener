# URL Shortening Service - Technical Deep Dive

## 🔍 **Database Design Deep Dive**

### **Schema Optimization Strategy**

#### **Primary Tables with Indexing Strategy**

```sql
-- URLs table with optimized indexing
CREATE TABLE urls (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    original_url TEXT NOT NULL,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    custom_alias VARCHAR(50) UNIQUE,
    title VARCHAR(255),
    description TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE,
    password_hash VARCHAR(255),
    click_count INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    last_clicked_at TIMESTAMP WITH TIME ZONE,
    
    -- Performance indexes
    CONSTRAINT urls_short_code_check CHECK (length(short_code) >= 4 AND length(short_code) <= 10),
    CONSTRAINT urls_original_url_check CHECK (length(original_url) <= 2048)
);

-- Critical indexes for performance
CREATE INDEX idx_urls_short_code ON urls(short_code); -- Primary lookup index
CREATE INDEX idx_urls_user_id_created_at ON urls(user_id, created_at DESC); -- User history
CREATE INDEX idx_urls_custom_alias ON urls(custom_alias) WHERE custom_alias IS NOT NULL; -- Custom alias lookup
CREATE INDEX idx_urls_expires_at ON urls(expires_at) WHERE expires_at IS NOT NULL; -- Expiration cleanup
CREATE INDEX idx_urls_click_count_desc ON urls(click_count DESC, created_at DESC); -- Popular URLs
CREATE INDEX idx_urls_last_clicked ON urls(last_clicked_at DESC) WHERE last_clicked_at IS NOT NULL; -- Recent activity

-- Partial index for active URLs only
CREATE INDEX idx_urls_active ON urls(short_code, is_active) WHERE is_active = true;

-- Composite index for user's active URLs
CREATE INDEX idx_urls_user_active ON urls(user_id, is_active, created_at DESC) WHERE is_active = true;
```

#### **Analytics Tables with Time-Series Optimization**

```sql
-- Clicks table optimized for time-series queries
CREATE TABLE clicks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    url_id UUID NOT NULL REFERENCES urls(id) ON DELETE CASCADE,
    ip_address INET,
    user_agent TEXT,
    referer TEXT,
    country VARCHAR(2),
    city VARCHAR(100),
    device_type VARCHAR(50),
    browser VARCHAR(100),
    os VARCHAR(100),
    clicked_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Partitioning key for time-series data
    partition_date DATE GENERATED ALWAYS AS (date(clicked_at)) STORED
) PARTITION BY RANGE (partition_date);

-- Create monthly partitions for better performance
CREATE TABLE clicks_y2024m01 PARTITION OF clicks
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Indexes for analytics queries
CREATE INDEX idx_clicks_url_id_clicked_at ON clicks(url_id, clicked_at DESC);
CREATE INDEX idx_clicks_partition_date ON clicks(partition_date);
CREATE INDEX idx_clicks_country ON clicks(country) WHERE country IS NOT NULL;
CREATE INDEX idx_clicks_device_type ON clicks(device_type) WHERE device_type IS NOT NULL;

-- Aggregated analytics table for fast dashboard queries
CREATE TABLE url_analytics_daily (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    url_id UUID NOT NULL REFERENCES urls(id) ON DELETE CASCADE,
    analytics_date DATE NOT NULL,
    total_clicks INTEGER DEFAULT 0,
    unique_visitors INTEGER DEFAULT 0,
    top_countries JSONB,
    top_referrers JSONB,
    device_distribution JSONB,
    hourly_distribution JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE(url_id, analytics_date)
);

CREATE INDEX idx_analytics_url_date ON url_analytics_daily(url_id, analytics_date DESC);
CREATE INDEX idx_analytics_date ON url_analytics_daily(analytics_date DESC);
```

### **Database Connection Pooling Configuration**

```go
// Optimized database connection pool
type DatabaseConfig struct {
    Host         string `env:"DB_HOST" env-default:"localhost"`
    Port         int    `env:"DB_PORT" env-default:"5432"`
    User         string `env:"DB_USER" env-default:"urlshortener"`
    Password     string `env:"DB_PASSWORD" env-required:"true"`
    DBName       string `env:"DB_NAME" env-default:"urlshortener"`
    SSLMode      string `env:"DB_SSLMODE" env-default:"require"`
    MaxOpenConns int    `env:"DB_MAX_OPEN_CONNS" env-default:"25"`
    MaxIdleConns int    `env:"DB_MAX_IDLE_CONNS" env-default:"5"`
    MaxLifetime int    `env:"DB_MAX_LIFETIME" env-default:"300"` // seconds
}

func NewDatabaseConnection(config DatabaseConfig) (*sql.DB, error) {
    dsn := fmt.Sprintf("host=%s port=%d user=%s password=%s dbname=%s sslmode=%s",
        config.Host, config.Port, config.User, config.Password, config.DBName, config.SSLMode)
    
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, err
    }
    
    // Configure connection pool
    db.SetMaxOpenConns(config.MaxOpenConns)
    db.SetMaxIdleConns(config.MaxIdleConns)
    db.SetConnMaxLifetime(time.Duration(config.MaxLifetime) * time.Second)
    
    // Test connection
    if err := db.Ping(); err != nil {
        return nil, err
    }
    
    return db, nil
}
```

## 🚀 **Backend Performance Deep Dive**

### **URL Shortening Algorithm Optimization**

```go
// Optimized short code generation with collision handling
type ShortCodeGenerator struct {
    charset    []rune
    length     int
    maxAttempts int
    cache      *redis.Client
    db         *sql.DB
}

func NewShortCodeGenerator(cache *redis.Client, db *sql.DB) *ShortCodeGenerator {
    return &ShortCodeGenerator{
        charset:     []rune("0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"),
        length:      6,
        maxAttempts: 10,
        cache:       cache,
        db:          db,
    }
}

func (g *ShortCodeGenerator) GenerateShortCode(ctx context.Context) (string, error) {
    for attempt := 0; attempt < g.maxAttempts; attempt++ {
        code := g.generateRandomCode()
        
        // Check cache first for performance
        exists, err := g.cache.Exists(ctx, "url:"+code).Result()
        if err != nil {
            log.Printf("Cache error: %v", err)
            // Fallback to database check
            exists, err = g.checkDatabaseExists(ctx, code)
            if err != nil {
                return "", err
            }
        }
        
        if exists == 0 {
            return code, nil
        }
    }
    
    return "", errors.New("failed to generate unique short code after maximum attempts")
}

func (g *ShortCodeGenerator) generateRandomCode() string {
    seededRand := rand.New(rand.NewSource(time.Now().UnixNano()))
    b := make([]rune, g.length)
    for i := range b {
        b[i] = g.charset[seededRand.Intn(len(g.charset))]
    }
    return string(b)
}

func (g *ShortCodeGenerator) checkDatabaseExists(ctx context.Context, code string) (int64, error) {
    var exists int64
    err := g.db.QueryRowContext(ctx, 
        "SELECT 1 FROM urls WHERE short_code = $1 LIMIT 1", code).Scan(&exists)
    return exists, err
}
```

### **Advanced Caching Strategy**

```go
// Multi-layer caching implementation
type CacheService struct {
    redis    *redis.Client
    localCache *ristretto.Cache
    metrics  *prometheus.CounterVec
}

type CacheKey struct {
    Prefix string
    ID     string
    Params map[string]string
}

func (c *CacheService) GetURL(ctx context.Context, shortCode string) (*models.URL, error) {
    key := &CacheKey{
        Prefix: "url",
        ID:     shortCode,
    }
    
    // Try local cache first (fastest)
    if url, found := c.localCache.Get(key.String()); found {
        c.metrics.WithLabelValues("local", "hit").Inc()
        return url.(*models.URL), nil
    }
    c.metrics.WithLabelValues("local", "miss").Inc()
    
    // Try Redis cache
    cached, err := c.redis.Get(ctx, key.String()).Result()
    if err == nil {
        var url models.URL
        if err := json.Unmarshal([]byte(cached), &url); err == nil {
            // Store in local cache
            c.localCache.SetWithTTL(key.String(), &url, 5*time.Minute)
            c.metrics.WithLabelValues("redis", "hit").Inc()
            return &url, nil
        }
    }
    c.metrics.WithLabelValues("redis", "miss").Inc()
    
    return nil, cache.ErrNotFound
}

func (c *CacheService) SetURL(ctx context.Context, url *models.URL, ttl time.Duration) error {
    key := &CacheKey{
        Prefix: "url",
        ID:     url.ShortCode,
    }
    
    // Serialize URL
    data, err := json.Marshal(url)
    if err != nil {
        return err
    }
    
    // Set in Redis
    if err := c.redis.Set(ctx, key.String(), data, ttl).Err(); err != nil {
        return err
    }
    
    // Set in local cache
    c.localCache.SetWithTTL(key.String(), url, ttl)
    
    c.metrics.WithLabelValues("set", "success").Inc()
    return nil
}

// Cache invalidation strategy
func (c *CacheService) InvalidateURL(ctx context.Context, shortCode string) error {
    key := &CacheKey{
        Prefix: "url",
        ID:     shortCode,
    }
    
    // Delete from Redis
    if err := c.redis.Del(ctx, key.String()).Err(); err != nil {
        return err
    }
    
    // Delete from local cache
    c.localCache.Del(key.String())
    
    // Invalidate related caches
    c.invalidateRelatedCaches(ctx, shortCode)
    
    return nil
}

func (c *CacheService) invalidateRelatedCaches(ctx context.Context, shortCode string) {
    // Invalidate user URL list cache
    pattern := "urls:user:*"
    keys, _ := c.redis.Keys(ctx, pattern).Result()
    if len(keys) > 0 {
        c.redis.Del(ctx, keys...)
    }
    
    // Invalidate analytics cache
    analyticsPattern := "analytics:" + shortCode + ":*"
    analyticsKeys, _ := c.redis.Keys(ctx, analyticsPattern).Result()
    if len(analyticsKeys) > 0 {
        c.redis.Del(ctx, analyticsKeys...)
    }
}
```

### **Background Job Processing System**

```go
// Job queue system for bulk operations
type JobQueue struct {
    redis   *redis.Client
    workers int
    jobs    chan Job
    ctx     context.Context
    cancel  context.CancelFunc
}

type Job struct {
    ID        string                 `json:"id"`
    Type      string                 `json:"type"`
    UserID    string                 `json:"user_id"`
    Data      map[string]interface{} `json:"data"`
    Status    string                 `json:"status"`
    CreatedAt time.Time              `json:"created_at"`
    StartedAt *time.Time             `json:"started_at,omitempty"`
    CompletedAt *time.Time           `json:"completed_at,omitempty"`
    Error     string                 `json:"error,omitempty"`
    Progress  int                    `json:"progress"`
}

type JobHandler interface {
    Handle(ctx context.Context, job *Job) error
}

func NewJobQueue(redis *redis.Client, workers int) *JobQueue {
    ctx, cancel := context.WithCancel(context.Background())
    return &JobQueue{
        redis:   redis,
        workers: workers,
        jobs:    make(chan Job, 1000),
        ctx:     ctx,
        cancel:  cancel,
    }
}

func (q *JobQueue) Start() {
    // Start workers
    for i := 0; i < q.workers; i++ {
        go q.worker(i)
    }
    
    // Process jobs from Redis queue
    go q.processQueue()
}

func (q *JobQueue) worker(id int) {
    for job := range q.jobs {
        log.Printf("Worker %d processing job %s", id, job.ID)
        
        // Update job status
        job.Status = "processing"
        now := time.Now()
        job.StartedAt = &now
        q.updateJobStatus(&job)
        
        // Handle job based on type
        var handler JobHandler
        switch job.Type {
        case "bulk_import":
            handler = &BulkImportHandler{db: q.db, storage: q.storage}
        case "bulk_export":
            handler = &BulkExportHandler{db: q.db, storage: q.storage}
        case "bulk_update":
            handler = &BulkUpdateHandler{db: q.db}
        default:
            handler = &DefaultHandler{}
        }
        
        // Execute job
        if err := handler.Handle(q.ctx, &job); err != nil {
            job.Status = "failed"
            job.Error = err.Error()
            log.Printf("Job %s failed: %v", job.ID, err)
        } else {
            job.Status = "completed"
            job.Progress = 100
        }
        
        // Update final status
        completedAt := time.Now()
        job.CompletedAt = &completedAt
        q.updateJobStatus(&job)
        
        log.Printf("Worker %d completed job %s", id, job.ID)
    }
}

func (q *JobQueue) processQueue() {
    for {
        select {
        case <-q.ctx.Done():
            return
        default:
            // Get job from Redis queue
            result, err := q.redis.BLPop(q.ctx, 5*time.Second, "jobs:queue").Result()
            if err != nil {
                continue
            }
            
            if len(result) < 2 {
                continue
            }
            
            var job Job
            if err := json.Unmarshal([]byte(result[1]), &job); err != nil {
                log.Printf("Failed to unmarshal job: %v", err)
                continue
            }
            
            // Send job to workers
            select {
            case q.jobs <- job:
            case <-q.ctx.Done():
                return
            }
        }
    }
}
```

## 🎨 **Frontend Performance Deep Dive**

### **Component Optimization Strategy**

```typescript
// Optimized URL shortening form with React Query
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { debounce } from 'lodash';

const urlSchema = z.object({
    original_url: z.string().url("Please enter a valid URL"),
    custom_alias: z.string().min(3).max(50).regex(/^[a-zA-Z0-9-]+$/, 
        "Alias can only contain letters, numbers, and hyphens").optional(),
    title: z.string().max(255).optional(),
    description: z.string().max(500).optional(),
    expires_at: z.string().datetime().optional(),
    password: z.string().min(6).optional(),
});

type URLFormData = z.infer<typeof urlSchema>;

const UrlShortenerForm: React.FC = () => {
    const queryClient = useQueryClient();
    
    const {
        control,
        handleSubmit,
        watch,
        setError,
        formState: { errors, isSubmitting }
    } = useForm<URLFormData>({
        resolver: zodResolver(urlSchema),
        defaultValues: {
            original_url: '',
            custom_alias: '',
            title: '',
            description: '',
            expires_at: '',
            password: '',
        }
    });

    // Debounced alias availability check
    const customAlias = watch('custom_alias');
    const debouncedAliasCheck = useMemo(
        () => debounce(async (alias: string) => {
            if (alias && alias.length >= 3) {
                try {
                    const response = await api.checkAliasAvailability(alias);
                    if (!response.available) {
                        setError('custom_alias', {
                            message: 'This alias is already taken'
                        });
                    }
                } catch (error) {
                    console.error('Error checking alias availability:', error);
                }
            }
        }, 500),
        [setError]
    );

    useEffect(() => {
        if (customAlias) {
            debouncedAliasCheck(customAlias);
        }
    }, [customAlias, debouncedAliasCheck]);

    const createUrlMutation = useMutation({
        mutationFn: (data: URLFormData) => api.createUrl(data),
        onSuccess: (data) => {
            queryClient.invalidateQueries({ queryKey: ['urls'] });
            toast.success('URL shortened successfully!');
            // Generate QR code
            generateQRCode(data.short_code);
        },
        onError: (error: any) => {
            toast.error(error.message || 'Failed to shorten URL');
        },
    });

    const onSubmit = (data: URLFormData) => {
        createUrlMutation.mutate(data);
    };

    return (
        <form onSubmit={handleSubmit(onSubmit)} className="space-y-6">
            <Controller
                name="original_url"
                control={control}
                render={({ field }) => (
                    <div>
                        <label htmlFor="original_url" className="block text-sm font-medium text-gray-700">
                            Long URL
                        </label>
                        <input
                            {...field}
                            type="url"
                            id="original_url"
                            className="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
                            placeholder="https://example.com/very/long/url"
                        />
                        {errors.original_url && (
                            <p className="mt-1 text-sm text-red-600">
                                {errors.original_url.message}
                            </p>
                        )}
                    </div>
                )}
            />

            <Controller
                name="custom_alias"
                control={control}
                render={({ field }) => (
                    <div>
                        <label htmlFor="custom_alias" className="block text-sm font-medium text-gray-700">
                            Custom Alias (Optional)
                        </label>
                        <div className="mt-1 relative rounded-md shadow-sm">
                            <div className="absolute inset-y-0 left-0 pl-3 flex items-center pointer-events-none">
                                <span className="text-gray-500 sm:text-sm">short.ly/</span>
                            </div>
                            <input
                                {...field}
                                type="text"
                                id="custom_alias"
                                className="pl-20 block w-full rounded-md border-gray-300 focus:border-blue-500 focus:ring-blue-500"
                                placeholder="my-custom-link"
                            />
                        </div>
                        {errors.custom_alias && (
                            <p className="mt-1 text-sm text-red-600">
                                {errors.custom_alias.message}
                            </p>
                        )}
                    </div>
                )}
            />

            {/* Additional form fields... */}

            <button
                type="submit"
                disabled={isSubmitting || createUrlMutation.isLoading}
                className="w-full flex justify-center py-2 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white bg-blue-600 hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-blue-500 disabled:opacity-50"
            >
                {isSubmitting || createUrlMutation.isLoading ? (
                    <LoadingSpinner className="h-4 w-4" />
                ) : (
                    'Shorten URL'
                )}
            </button>
        </form>
    );
};
```

### **Real-time Analytics with WebSocket**

```typescript
// WebSocket hook for real-time analytics
import { useEffect, useRef, useState } from 'react';
import { useQueryClient } from '@tanstack/react-query';

interface AnalyticsEvent {
    type: 'click' | 'update';
    url_id: string;
    timestamp: string;
    data: any;
}

export const useRealTimeAnalytics = (urlId: string) => {
    const [isConnected, setIsConnected] = useState(false);
    const [lastEvent, setLastEvent] = useState<AnalyticsEvent | null>(null);
    const ws = useRef<WebSocket | null>(null);
    const queryClient = useQueryClient();
    const reconnectTimeout = useRef<NodeJS.Timeout>();

    const connect = () => {
        const wsUrl = `${process.env.REACT_APP_WS_URL}/ws/analytics/${urlId}`;
        ws.current = new WebSocket(wsUrl);

        ws.current.onopen = () => {
            setIsConnected(true);
            console.log('WebSocket connected');
        };

        ws.current.onmessage = (event) => {
            try {
                const data: AnalyticsEvent = JSON.parse(event.data);
                setLastEvent(data);

                // Update React Query cache
                if (data.type === 'click') {
                    queryClient.setQueryData(
                        ['analytics', urlId],
                        (oldData: any) => {
                            if (!oldData) return oldData;
                            return {
                                ...oldData,
                                total_clicks: oldData.total_clicks + 1,
                                recent_activity: [data, ...oldData.recent_activity.slice(0, 9)]
                            };
                        }
                    );

                    // Invalidate related queries
                    queryClient.invalidateQueries({ queryKey: ['dashboard'] });
                }
            } catch (error) {
                console.error('Error parsing WebSocket message:', error);
            }
        };

        ws.current.onclose = () => {
            setIsConnected(false);
            console.log('WebSocket disconnected');
            
            // Attempt to reconnect after 5 seconds
            reconnectTimeout.current = setTimeout(() => {
                connect();
            }, 5000);
        };

        ws.current.onerror = (error) => {
            console.error('WebSocket error:', error);
            ws.current?.close();
        };
    };

    useEffect(() => {
        connect();

        return () => {
            if (reconnectTimeout.current) {
                clearTimeout(reconnectTimeout.current);
            }
            ws.current?.close();
        };
    }, [urlId]);

    const sendMessage = (message: any) => {
        if (ws.current && ws.current.readyState === WebSocket.OPEN) {
            ws.current.send(JSON.stringify(message));
        }
    };

    return {
        isConnected,
        lastEvent,
        sendMessage
    };
};

// Analytics dashboard component with real-time updates
const AnalyticsDashboard: React.FC<{ urlId: string }> = ({ urlId }) => {
    const { data: analytics, isLoading } = useQuery({
        queryKey: ['analytics', urlId],
        queryFn: () => api.getUrlAnalytics(urlId),
        refetchInterval: 30000, // Refetch every 30 seconds as fallback
    });

    const { isConnected, lastEvent } = useRealTimeAnalytics(urlId);

    // Update chart data when new events arrive
    const [chartData, setChartData] = useState<any[]>([]);

    useEffect(() => {
        if (analytics) {
            setChartData(analytics.daily_stats);
        }
    }, [analytics]);

    useEffect(() => {
        if (lastEvent && lastEvent.type === 'click') {
            setChartData(prev => {
                const today = new Date().toISOString().split('T')[0];
                const existingIndex = prev.findIndex(item => item.date === today);
                
                if (existingIndex >= 0) {
                    const updated = [...prev];
                    updated[existingIndex] = {
                        ...updated[existingIndex],
                        clicks: updated[existingIndex].clicks + 1
                    };
                    return updated;
                } else {
                    return [...prev, {
                        date: today,
                        clicks: 1,
                        unique_visitors: 1
                    }];
                }
            });
        }
    }, [lastEvent]);

    if (isLoading) {
        return <LoadingSpinner />;
    }

    return (
        <div className="space-y-6">
            <div className="flex items-center justify-between">
                <h2 className="text-2xl font-bold text-gray-900">Analytics Dashboard</h2>
                <div className="flex items-center space-x-2">
                    <div className={`w-3 h-3 rounded-full ${isConnected ? 'bg-green-500' : 'bg-red-500'}`} />
                    <span className="text-sm text-gray-600">
                        {isConnected ? 'Connected' : 'Disconnected'}
                    </span>
                </div>
            </div>

            {/* Stats Cards */}
            <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
                <StatsCard
                    title="Total Clicks"
                    value={analytics?.total_clicks || 0}
                    change={lastEvent ? '+1' : '0'}
                    changeType="increase"
                />
                <StatsCard
                    title="Unique Visitors"
                    value={analytics?.unique_visitors || 0}
                    change="0"
                    changeType="neutral"
                />
                <StatsCard
                    title="Conversion Rate"
                    value={`${((analytics?.conversion_rate || 0) * 100).toFixed(2)}%`}
                    change="0"
                    changeType="neutral"
                />
            </div>

            {/* Click Chart */}
            <div className="bg-white p-6 rounded-lg shadow">
                <h3 className="text-lg font-medium text-gray-900 mb-4">Click Trends</h3>
                <LineChart
                    data={chartData}
                    xAxis="date"
                    yAxis="clicks"
                    height={300}
                />
            </div>

            {/* Additional analytics components... */}
        </div>
    );
};
```

## 🔐 **Security Implementation Deep Dive**

### **Advanced Rate Limiting Implementation**

```go
// Sophisticated rate limiting with multiple strategies
type RateLimiter struct {
    redis       *redis.Client
    configs     map[string]RateLimitConfig
    metrics     *prometheus.CounterVec
    ipDetector  *IPDetector
}

type RateLimitConfig struct {
    Requests   int           `json:"requests"`
    Window     time.Duration `json:"window"`
    Strategy   string        `json:"strategy"` // "fixed", "sliding", "token_bucket"
    Burst      int           `json:"burst"`
    SkipSuccessful bool      `json:"skip_successful"`
}

type RateLimitResult struct {
    Allowed     bool          `json:"allowed"`
    Remaining   int           `json:"remaining"`
    ResetTime   time.Time     `json:"reset_time"`
    RetryAfter  time.Duration `json:"retry_after"`
    LimitType   string        `json:"limit_type"`
}

func (rl *RateLimiter) CheckLimit(ctx context.Context, identifier string, limitType string, metadata map[string]string) (*RateLimitResult, error) {
    config, exists := rl.configs[limitType]
    if !exists {
        return nil, fmt.Errorf("rate limit config not found for type: %s", limitType)
    }

    var result *RateLimitResult
    var err error

    switch config.Strategy {
    case "sliding":
        result, err = rl.checkSlidingWindow(ctx, identifier, config, metadata)
    case "token_bucket":
        result, err = rl.checkTokenBucket(ctx, identifier, config, metadata)
    default: // "fixed"
        result, err = rl.checkFixedWindow(ctx, identifier, config, metadata)
    }

    if err != nil {
        return nil, err
    }

    // Log rate limit events for monitoring
    if !result.Allowed {
        rl.metrics.WithLabelValues(limitType, "blocked").Inc()
        rl.logRateLimitEvent(identifier, limitType, metadata)
    } else {
        rl.metrics.WithLabelValues(limitType, "allowed").Inc()
    }

    return result, nil
}

func (rl *RateLimiter) checkSlidingWindow(ctx context.Context, identifier string, config RateLimitConfig, metadata map[string]string) (*RateLimitResult, error) {
    now := time.Now()
    windowStart := now.Add(-config.Window)
    
    key := fmt.Sprintf("rate_limit:sliding:%s:%s", identifier, config.Window)
    
    // Use Redis sorted set for sliding window
    pipe := rl.redis.Pipeline()
    
    // Remove old entries outside the window
    pipe.ZRemRangeByScore(ctx, key, "0", fmt.Sprintf("%d", windowStart.Unix()))
    
    // Count current entries in window
    countCmd := pipe.ZCard(ctx, key)
    
    // Add current request
    pipe.ZAdd(ctx, key, redis.Z{Score: float64(now.Unix()), Member: fmt.Sprintf("%d", now.UnixNano())})
    
    // Set expiration
    pipe.Expire(ctx, key, config.Window)
    
    _, err := pipe.Exec(ctx)
    if err != nil {
        return nil, err
    }
    
    count, err := countCmd.Result()
    if err != nil {
        return nil, err
    }
    
    allowed := count < int64(config.Requests)
    remaining := config.Requests - int(count)
    
    if !allowed {
        // Calculate when the oldest request will expire
        oldest, err := rl.redis.ZRange(ctx, key, 0, 0).Result()
        if err == nil && len(oldest) > 0 {
            oldestTime, _ := strconv.ParseInt(oldest[0], 10, 64)
            resetTime := time.Unix(oldestTime, 0).Add(config.Window)
            retryAfter := time.Until(resetTime)
            
            return &RateLimitResult{
                Allowed:    false,
                Remaining:  0,
                ResetTime:  resetTime,
                RetryAfter: retryAfter,
                LimitType:  "sliding_window",
            }, nil
        }
    }
    
    return &RateLimitResult{
        Allowed:    allowed,
        Remaining:  remaining,
        ResetTime:  now.Add(config.Window),
        RetryAfter: 0,
        LimitType:  "sliding_window",
    }, nil
}

func (rl *RateLimiter) checkTokenBucket(ctx context.Context, identifier string, config RateLimitConfig, metadata map[string]string) (*RateLimitResult, error) {
    key := fmt.Sprintf("rate_limit:token_bucket:%s", identifier)
    
    now := time.Now()
    refillRate := float64(config.Requests) / config.Window.Seconds()
    
    // Get current bucket state
    result, err := rl.redis.Get(ctx, key).Result()
    if err == redis.Nil {
        // Initialize bucket
        result = fmt.Sprintf("%.2f:%d", float64(config.Burst), now.Unix())
    } else if err != nil {
        return nil, err
    }
    
    parts := strings.Split(result, ":")
    if len(parts) != 2 {
        // Reset bucket if corrupted
        result = fmt.Sprintf("%.2f:%d", float64(config.Burst), now.Unix())
        parts = strings.Split(result, ":")
    }
    
    tokens, _ := strconv.ParseFloat(parts[0], 64)
    lastRefill, _ := strconv.ParseInt(parts[1], 10, 64)
    lastRefillTime := time.Unix(lastRefill, 0)
    
    // Refill tokens based on time elapsed
    timePassed := now.Sub(lastRefillTime)
    tokensToAdd := timePassed.Seconds() * refillRate
    tokens = math.Min(float64(config.Burst), tokens+tokensToAdd)
    
    // Check if request can be processed
    allowed := tokens >= 1
    if allowed {
        tokens -= 1
    }
    
    // Update bucket state
    newResult := fmt.Sprintf("%.2f:%d", tokens, now.Unix())
    rl.redis.Set(ctx, key, newResult, config.Window)
    
    var remaining int
    if tokens >= 1 {
        remaining = int(tokens)
    } else {
        // Calculate time until next token
        remaining = 0
    }
    
    return &RateLimitResult{
        Allowed:    allowed,
        Remaining:  remaining,
        ResetTime:  now.Add(time.Duration(float64(config.Window)/float64(config.Requests)) * time.Second),
        RetryAfter: 0,
        LimitType:  "token_bucket",
    }, nil
}

// IP-based rate limiting with detection of proxies and VPNs
func (rl *RateLimiter) getIdentifier(r *http.Request) string {
    // Check for real IP behind proxies
    forwardedFor := r.Header.Get("X-Forwarded-For")
    if forwardedFor != "" {
        ips := strings.Split(forwardedFor, ",")
        if len(ips) > 0 {
            // Take the first IP (original client)
            return strings.TrimSpace(ips[0])
        }
    }
    
    realIP := r.Header.Get("X-Real-IP")
    if realIP != "" {
        return realIP
    }
    
    // Fallback to remote address
    ip, _, err := net.SplitHostPort(r.RemoteAddr)
    if err != nil {
        return r.RemoteAddr
    }
    
    return ip
}

func (rl *RateLimiter) logRateLimitEvent(identifier string, limitType string, metadata map[string]string) {
    event := map[string]interface{}{
        "timestamp":   time.Now(),
        "identifier":  identifier,
        "limit_type":  limitType,
        "user_agent":  metadata["user_agent"],
        "endpoint":    metadata["endpoint"],
        "method":      metadata["method"],
        "ip_country":  rl.ipDetector.GetCountry(identifier),
        "is_proxy":    rl.ipDetector.IsProxy(identifier),
    }
    
    // Log to monitoring system
    log.Printf("Rate limit exceeded: %+v", event)
    
    // Send to alerting system if suspicious
    if rl.isSuspiciousActivity(identifier, metadata) {
        rl.sendAlert(event)
    }
}

func (rl *RateLimiter) isSuspiciousActivity(identifier string, metadata map[string]string) bool {
    // Check for multiple IPs from same user agent
    userAgent := metadata["user_agent"]
    if userAgent != "" {
        key := fmt.Sprintf("suspicious:user_agent:%s", hashUserAgent(userAgent))
        count, _ := rl.redis.Incr(context.Background(), key).Result()
        rl.redis.Expire(context.Background(), key, time.Hour)
        
        if count > 10 { // More than 10 different IPs for same user agent in an hour
            return true
        }
    }
    
    // Check for requests from known proxy/VPN ranges
    if rl.ipDetector.IsProxy(identifier) {
        return true
    }
    
    return false
}
```

This technical deep dive provides detailed implementation strategies for the most critical aspects of the URL shortening service, focusing on performance optimization, security implementation, and advanced feature development.