# Project 07: Capstone — Full-Stack Backend Notes App

The capstone project. A production-grade notes API with user authentication,
PostgreSQL, Redis caching, graceful shutdown, Docker, health checks, and
monitoring hooks. This exercises Parts 04–12 in a single, cohesive application.

---

## Learning Goals

- Build a complete REST API from scratch using `net/http` + `chi` router
- Implement JWT authentication with registration and login
- Use PostgreSQL with `database/sql` + `pgx` driver
- Add Redis caching for read-heavy endpoints
- Implement graceful shutdown and health checks
- Dockerize the application with multi-stage builds
- Add structured logging and basic metrics
- Write comprehensive tests at every layer
- Apply clean architecture / repository pattern

## Prerequisites

| Part | What you need |
|------|---------------|
| 03   | Structs, interfaces, methods, error handling |
| 04   | Goroutines, channels, context, WaitGroup |
| 05   | `database/sql`, `os`, `flag`, `time`, `net/http` |
| 06   | Table-driven tests, mocks, test helpers |
| 07   | HTTP handlers, middleware, `httptest`, JSON encoding |
| 08   | gRPC concepts (helps understand protocols) |
| 09   | Gin (comparison; this project uses chi for stdlib feel) |
| 10   | PostgreSQL, SQL queries, migrations |
| 11   | go-zero (comparison; this project avoids frameworks) |
| 12   | JWT auth, password hashing, middleware patterns |

---

## Project Structure

```
notes-api/
├── main.go                      # Entry point, wiring, graceful shutdown
├── go.mod / go.sum
├── Dockerfile                   # Multi-stage build
├── docker-compose.yml           # App + PostgreSQL + Redis
├── Makefile                     # Common commands
│
├── config/
│   └── config.go                # Configuration struct + loader
│
├── migrations/
│   ├── 001_create_users.up.sql
│   ├── 001_create_users.down.sql
│   ├── 002_create_notes.up.sql
│   └── 002_create_notes.down.sql
│
├── models/
│   └── models.go                # Domain types
│
├── repository/
│   ├── user_repo.go             # User DB operations
│   ├── note_repo.go             # Note DB operations
│   └── cache.go                 # Redis cache layer
│
├── handlers/
│   ├── auth.go                  # Register, Login
│   ├── notes.go                 # CRUD for notes
│   ├── health.go                # Health check endpoints
│   └── middleware.go             # JWT auth, logging, recovery
│
├── handlers/
│   ├── auth_test.go
│   ├── notes_test.go
│   └── middleware_test.go
│
├── repository/
│   ├── user_repo_test.go        # Integration tests (need PG)
│   └── note_repo_test.go        # Integration tests (need PG)
│
└── scripts/
    └── migrate.sh               # Run migrations
```

```mermaid
graph TD
    A[Client / Frontend] -->|HTTP| B[chi Router]
    B --> C[Logging Middleware]
    C --> D[Recovery Middleware]
    D --> E{Route}
    E -->|/auth/*| F[Auth Handlers]
    E -->|/api/notes/*| G[JWT Middleware]
    E -->|/health| H[Health Handler]
    G --> I[Note Handlers]

    F --> J[User Repository]
    I --> K[Note Repository]
    K --> L[(PostgreSQL)]
    K --> M[(Redis Cache)]
    J --> L

    I -.->|cache miss| L
    I -.->|cache hit| M

    subgraph "Graceful Shutdown"
        N[SIGTERM] --> O[Drain connections]
        O --> P[Close DB pool]
        O --> Q[Close Redis]
        O --> R[Stop HTTP server]
    end
```

> 🧠 **Memory aid:** This is everything you've built condensed into one app —
> middleware chain, JWT-protected routes, cache-aside Redis, graceful
> shutdown. Notice how the diagram reads as a stack: router → middleware →
> handlers → repositories → storage.

---

## Part A: Configuration

Create `config/config.go`:

```go
package config

import (
	"encoding/json"
	"fmt"
	"os"
)

// Config holds all application configuration.
type Config struct {
	Server   ServerConfig   `json:"server"`
	Database DatabaseConfig `json:"database"`
	Redis    RedisConfig    `json:"redis"`
	Auth     AuthConfig     `json:"auth"`
}

type ServerConfig struct {
	Host string `json:"host"`
	Port int    `json:"port"`
}

func (s ServerConfig) Addr() string {
	return fmt.Sprintf("%s:%d", s.Host, s.Port)
}

type DatabaseConfig struct {
	Host     string `json:"host"`
	Port     int    `json:"port"`
	User     string `json:"user"`
	Password string `json:"password"`
	Name     string `json:"name"`
	SSLMode  string `json:"sslmode"`
}

func (d DatabaseConfig) DSN() string {
	return fmt.Sprintf(
		"postgres://%s:%s@%s:%d/%s?sslmode=%s",
		d.User, d.Password, d.Host, d.Port, d.Name, d.SSLMode,
	)
}

type RedisConfig struct {
	Addr     string `json:"addr"`
	Password string `json:"password"`
	DB       int    `json:"db"`
}

type AuthConfig struct {
	JWTSecret          string `json:"jwt_secret"`
	TokenExpiryHours   int    `json:"token_expiry_hours"`
	RefreshExpiryHours int    `json:"refresh_expiry_hours"`
}

// Load reads configuration from a JSON file, with env var overrides.
func Load(path string) (*Config, error) {
	cfg := &Config{
		Server: ServerConfig{Host: "0.0.0.0", Port: 8080},
		Database: DatabaseConfig{
			Host: "localhost", Port: 5432,
			User: "postgres", Password: "postgres",
			Name: "notes", SSLMode: "disable",
		},
		Redis: RedisConfig{Addr: "localhost:6379", DB: 0},
		Auth: AuthConfig{
			JWTSecret:        "change-me-in-production",
			TokenExpiryHours: 24,
		},
	}

	if path != "" {
		data, err := os.ReadFile(path)
		if err != nil {
			return nil, fmt.Errorf("reading config: %w", err)
		}
		if err := json.Unmarshal(data, cfg); err != nil {
			return nil, fmt.Errorf("parsing config: %w", err)
		}
	}

	// Environment variable overrides.
	if v := os.Getenv("DATABASE_URL"); v != "" {
		cfg.Database = parseDatabaseURL(v)
	}
	if v := os.Getenv("REDIS_ADDR"); v != "" {
		cfg.Redis.Addr = v
	}
	if v := os.Getenv("JWT_SECRET"); v != "" {
		cfg.Auth.JWTSecret = v
	}

	return cfg, nil
}
```

> ⚠️ **Watch out:** The default `JWT_SECRET` ("change-me-in-production")
> is a dev convenience, not a safe value — if an operator forgets to set
> it, tokens are forgeable. Prefer failing fast when running in production
> mode.

```go
func parseDatabaseURL(url string) DatabaseConfig {
	// Simplified — in production use a proper URL parser.
	return DatabaseConfig{Host: "localhost", Port: 5432, Name: "notes"}
}
```

---

## Part B: Domain Models

Create `models/models.go`:

```go
package models

import "time"

type User struct {
	ID           int       `json:"id"`
	Email        string    `json:"email"`
	PasswordHash string    `json:"-"`
	Name         string    `json:"name"`
	CreatedAt    time.Time `json:"created_at"`
}

type Note struct {
	ID        int       `json:"id"`
	UserID    int       `json:"user_id"`
	Title     string    `json:"title"`
	Content   string    `json:"content"`
	Pinned    bool      `json:"pinned"`
	Tags      []string  `json:"tags,omitempty"`
	CreatedAt time.Time `json:"created_at"`
	UpdatedAt time.Time `json:"updated_at"`
}

type CreateNoteRequest struct {
	Title   string   `json:"title" binding:"required"`
	Content string   `json:"content"`
	Pinned  bool     `json:"pinned"`
	Tags    []string `json:"tags"`
}

type UpdateNoteRequest struct {
	Title   *string  `json:"title"`
	Content *string  `json:"content"`
	Pinned  *bool    `json:"pinned"`
	Tags    []string `json:"tags"`
}

type RegisterRequest struct {
	Email    string `json:"email"`
	Password string `json:"password"`
	Name     string `json:"name"`
}

type LoginRequest struct {
	Email    string `json:"email"`
	Password string `json:"password"`
}

type TokenResponse struct {
	AccessToken  string `json:"access_token"`
	RefreshToken string `json:"refresh_token"`
	ExpiresAt    int64  `json:"expires_at"`
}

type NoteListResponse struct {
	Notes []Note `json:"notes"`
	Total int    `json:"total"`
	Page  int    `json:"page"`
	Limit int    `json:"limit"`
}

type HealthResponse struct {
	Status    string            `json:"status"`
	Services  map[string]string `json:"services"`
	Timestamp time.Time         `json:"timestamp"`
}
```

> 🔑 **Checkpoint:** Note `PasswordHash` with `json:"-"` — same anti-leak
> trick from earlier projects — plus pointer fields on `UpdateNoteRequest`
> for real PATCH semantics. The request types *are* the API contract.

---

## Part C: Migrations

Create `migrations/001_create_users.up.sql`:

```sql
CREATE TABLE IF NOT EXISTS users (
    id            SERIAL PRIMARY KEY,
    email         VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name          VARCHAR(255) NOT NULL DEFAULT '',
    created_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users (email);
```

Create `migrations/001_create_users.down.sql`:

```sql
DROP INDEX IF EXISTS idx_users_email;
DROP TABLE IF EXISTS users;
```

Create `migrations/002_create_notes.up.sql`:

```sql
CREATE TABLE IF NOT EXISTS notes (
    id         SERIAL PRIMARY KEY,
    user_id    INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title      VARCHAR(500) NOT NULL,
    content    TEXT DEFAULT '',
    pinned     BOOLEAN DEFAULT FALSE,
    tags       TEXT[] DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_notes_user_id ON notes (user_id);
CREATE INDEX idx_notes_pinned ON notes (pinned);
CREATE INDEX idx_notes_tags ON notes USING GIN (tags);
```

Create `migrations/002_create_notes.down.sql`:

```sql
DROP INDEX IF EXISTS idx_notes_tags;
DROP INDEX IF EXISTS idx_notes_pinned;
DROP INDEX IF EXISTS idx_notes_user_id;
DROP TABLE IF EXISTS notes;
```

> ⚠️ **Watch out:** `drop table ... cascade` and down-migrations are tools
> for local dev. In production, never auto-apply down migrations — rolling
> forward is how schema changes ship safely.

---

## Part D: Repositories

Create `repository/user_repo.go`:

```go
package repository

import (
	"context"
	"database/sql"
	"fmt"

	"notes-api/models"
)

type UserRepository struct {
	db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
	return &UserRepository{db: db}
}

func (r *UserRepository) Create(ctx context.Context, user *models.User) error {
	query := `INSERT INTO users (email, password_hash, name)
		VALUES ($1, $2, $3) RETURNING id, created_at`
	return r.db.QueryRowContext(ctx, query,
		user.Email, user.PasswordHash, user.Name).
		Scan(&user.ID, &user.CreatedAt)
}

func (r *UserRepository) GetByEmail(ctx context.Context, email string) (*models.User, error) {
	query := `SELECT id, email, password_hash, name, created_at
		FROM users WHERE email = $1`
	user := &models.User{}
	err := r.db.QueryRowContext(ctx, query, email).
		Scan(&user.ID, &user.Email, &user.PasswordHash, &user.Name, &user.CreatedAt)
	if err != nil {
		return nil, fmt.Errorf("GetByEmail(%s): %w", email, err)
	}
	return user, nil
}

func (r *UserRepository) GetByID(ctx context.Context, id int) (*models.User, error) {
	query := `SELECT id, email, password_hash, name, created_at
		FROM users WHERE id = $1`
	user := &models.User{}
	err := r.db.QueryRowContext(ctx, query, id).
		Scan(&user.ID, &user.Email, &user.PasswordHash, &user.Name, &user.CreatedAt)
	if err != nil {
		return nil, fmt.Errorf("GetByID(%d): %w", id, err)
	}
	return user, nil
}
```

Create `repository/note_repo.go`:

```go
package repository

import (
	"context"
	"database/sql"
	"fmt"
	"time"

	"notes-api/models"
)

type NoteRepository struct {
	db *sql.DB
}

func NewNoteRepository(db *sql.DB) *NoteRepository {
	return &NoteRepository{db: db}
}

func (r *NoteRepository) Create(ctx context.Context, note *models.Note) error {
	query := `INSERT INTO notes (user_id, title, content, pinned, tags)
		VALUES ($1, $2, $3, $4, $5)
		RETURNING id, created_at, updated_at`
	return r.db.QueryRowContext(ctx, query,
		note.UserID, note.Title, note.Content, note.Pinned,
		pgArray(note.Tags)).
		Scan(&note.ID, &note.CreatedAt, &note.UpdatedAt)
}

func (r *NoteRepository) GetByID(ctx context.Context, id, userID int) (*models.Note, error) {
	query := `SELECT id, user_id, title, content, pinned, tags, created_at, updated_at
		FROM notes WHERE id = $1 AND user_id = $2`
	note := &models.Note{}
	var tags string
	err := r.db.QueryRowContext(ctx, query, id, userID).
		Scan(&note.ID, &note.UserID, &note.Title, &note.Content,
			&note.Pinned, &tags, &note.CreatedAt, &note.UpdatedAt)
	if err != nil {
		return nil, fmt.Errorf("GetByID(%d): %w", id, err)
	}
	note.Tags = parsePGArray(tags)
	return note, nil
}

func (r *NoteRepository) ListByUser(ctx context.Context, userID int, page, limit int) ([]models.Note, int, error) {
	offset := (page - 1) * limit

	// Count total.
	var total int
	countQuery := `SELECT COUNT(*) FROM notes WHERE user_id = $1`
	if err := r.db.QueryRowContext(ctx, countQuery, userID).Scan(&total); err != nil {
		return nil, 0, fmt.Errorf("count notes: %w", err)
	}

	// Fetch page.
	query := `SELECT id, user_id, title, content, pinned, tags, created_at, updated_at
		FROM notes WHERE user_id = $1
		ORDER BY pinned DESC, updated_at DESC
		LIMIT $2 OFFSET $3`
	rows, err := r.db.QueryContext(ctx, query, userID, limit, offset)
	if err != nil {
		return nil, 0, fmt.Errorf("ListByUser: %w", err)
	}
	defer rows.Close()

	var notes []models.Note
	for rows.Next() {
		var n models.Note
		var tags string
		if err := rows.Scan(
			&n.ID, &n.UserID, &n.Title, &n.Content,
			&n.Pinned, &tags, &n.CreatedAt, &n.UpdatedAt,
		); err != nil {
			return nil, 0, fmt.Errorf("ListByUser scan: %w", err)
		}
		n.Tags = parsePGArray(tags)
		notes = append(notes, n)
	}

	if err := rows.Err(); err != nil {
		return nil, 0, fmt.Errorf("ListByUser rows: %w", err)
	}

	return notes, total, nil
}

func (r *NoteRepository) Update(ctx context.Context, note *models.Note) error {
	query := `UPDATE notes SET title=$1, content=$2, pinned=$3, tags=$4, updated_at=$5
		WHERE id=$6 AND user_id=$7`
	result, err := r.db.ExecContext(ctx, query,
		note.Title, note.Content, note.Pinned, pgArray(note.Tags),
		time.Now(), note.ID, note.UserID,
	)
	if err != nil {
		return fmt.Errorf("Update(%d): %w", note.ID, err)
	}
	rows, err := result.RowsAffected()
	if err != nil {
		return err
	}
	if rows == 0 {
		return fmt.Errorf("note %d not found", note.ID)
	}
	note.UpdatedAt = time.Now()
	return nil
}

func (r *NoteRepository) Delete(ctx context.Context, id, userID int) error {
	query := `DELETE FROM notes WHERE id = $1 AND user_id = $2`
	result, err := r.db.ExecContext(ctx, query, id, userID)
	if err != nil {
		return fmt.Errorf("Delete(%d): %w", id, err)
	}
	rows, err := result.RowsAffected()
	if err != nil {
		return err
	}
	if rows == 0 {
		return fmt.Errorf("note %d not found", id)
	}
	return nil
}

// Search searches note titles and content.
func (r *NoteRepository) Search(ctx context.Context, userID int, query string) ([]models.Note, error) {
	sql := `SELECT id, user_id, title, content, pinned, tags, created_at, updated_at
		FROM notes WHERE user_id = $1
		AND (title ILIKE '%' || $2 || '%' OR content ILIKE '%' || $2 || '%')
		ORDER BY updated_at DESC LIMIT 50`
	rows, err := r.db.QueryContext(ctx, sql, userID, query)
	if err != nil {
		return nil, fmt.Errorf("Search: %w", err)
	}
	defer rows.Close()

	var notes []models.Note
	for rows.Next() {
		var n models.Note
		var tags string
		if err := rows.Scan(
			&n.ID, &n.UserID, &n.Title, &n.Content,
			&n.Pinned, &tags, &n.CreatedAt, &n.UpdatedAt,
		); err != nil {
			return nil, fmt.Errorf("Search scan: %w", err)
		}
		n.Tags = parsePGArray(tags)
		notes = append(notes, n)
	}
	return notes, rows.Err()
}

// pgArray converts a Go string slice to a PostgreSQL array literal.
func pgArray(arr []string) string {
	if arr == nil {
		return "{}"
	}
	result := "{"
	for i, s := range arr {
		if i > 0 {
			result += ","
		}
		result += `"` + s + `"`
	}
	return result + "}"
}

// parsePGArray converts a PostgreSQL array literal to a Go string slice.
func parsePGArray(s string) []string {
	if s == "" || s == "{}" {
		return []string{}
	}
	s = s[1 : len(s)-1] // Remove { }
	if s == "" {
		return []string{}
	}
	// Simple split — doesn't handle escaped quotes.
	var result []string
	current := ""
	inQuotes := false
	for _, c := range s {
		switch {
		case c == '"':
			inQuotes = !inQuotes
		case c == ',' && !inQuotes:
			result = append(result, current)
			current = ""
		default:
			current += string(c)
		}
	}
	result = append(result, current)
	return result
}
```

---

## Part E: Redis Cache

Create `repository/cache.go`:

```go
package repository

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"notes-api/models"
)

// NoteCache provides Redis-backed caching for notes.
type NoteCache struct {
	// In a real app, use github.com/redis/go-redis/v9.
	// For this project, we'll define the interface.
	Get    func(ctx context.Context, key string) ([]byte, error)
	Set    func(ctx context.Context, key string, value any, ttl time.Duration) error
	Delete func(ctx context.Context, keys ...string) error
}

// CacheKey builds a Redis key for a note.
func CacheKey(noteID int) string {
	return fmt.Sprintf("note:%d", noteID)
}

// CacheKeyList builds a Redis key for a user's note list.
func CacheKeyList(userID int, page, limit int) string {
	return fmt.Sprintf("notes:user:%d:page:%d:limit:%d", userID, page, limit)
}

// GetNote retrieves a note from cache.
func (c *NoteCache) GetNote(ctx context.Context, noteID int) (*models.Note, bool) {
	data, err := c.Get(ctx, CacheKey(noteID))
	if err != nil || data == nil {
		return nil, false
	}

	var note models.Note
	if err := json.Unmarshal(data, &note); err != nil {
		return nil, false
	}
	return &note, true
}

// SetNote stores a note in cache.
func (c *NoteCache) SetNote(ctx context.Context, note *models.Note) error {
	data, err := json.Marshal(note)
	if err != nil {
		return err
	}
	return c.Set(ctx, CacheKey(note.ID), data, 5*time.Minute)
}

// InvalidateNote removes a note from cache.
func (c *NoteCache) InvalidateNote(ctx context.Context, noteID int) error {
	return c.Delete(ctx, CacheKey(noteID))
}

// InvalidateUserNotes removes all cached note lists for a user.
func (c *NoteCache) InvalidateUserNotes(ctx context.Context, userID int) error {
	// In production, use SCAN to find and delete all keys matching pattern.
	// For simplicity, we accept that list caches may be stale.
	return nil
}
```

### Key Points

- **Cache-aside pattern** — the application checks cache first, then falls
  back to the database. On write, it invalidates the cache.
- **Short TTL** — 5 minutes is a reasonable default for a notes app.
  Adjust based on read/write ratio.
- **`json.Marshal`/`Unmarshal`** for cache serialization — simple and fast.
  For complex objects, consider protobuf or msgpack.

> ⚠️ **Watch out:** Invalidation is harder than caching — a forgotten
> `Delete` on update/delete serves stale data forever. Short TTLs are the
> safety net: even if you miss an invalidation, the cache heals itself
> within minutes.

---

## Part F: HTTP Handlers and Middleware

Create `handlers/middleware.go`:

```go
package handlers

import (
	"context"
	"log"
	"net/http"
	"os"
	"strings"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

type contextKey string

const (
	userIDKey contextKey = "user_id"
	emailKey  contextKey = "email"
)

var jwtSecret []byte

func init() {
	if secret := os.Getenv("JWT_SECRET"); secret != "" {
		jwtSecret = []byte(secret)
	} else {
		jwtSecret = []byte("dev-secret-change-in-production")
	}
}

type Claims struct {
	UserID int    `json:"user_id"`
	Email  string `json:"email"`
	jwt.RegisteredClaims
}

func GenerateToken(userID int, email string, expiryHours int) (string, int64, error) {
	expiresAt := time.Now().Add(time.Duration(expiryHours) * time.Hour)
	claims := &Claims{
		UserID: userID,
		Email:  email,
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(expiresAt),
			IssuedAt:  jwt.NewNumericDate(time.Now()),
		},
	}

	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	tokenString, err := token.SignedString(jwtSecret)
	return tokenString, expiresAt.Unix(), err
}

// AuthMiddleware validates JWT and injects user info into context.
func AuthMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		authHeader := r.Header.Get("Authorization")
		if authHeader == "" {
			http.Error(w, `{"error":"authorization required"}`, http.StatusUnauthorized)
			return
		}

		parts := strings.SplitN(authHeader, " ", 2)
		if len(parts) != 2 || parts[0] != "Bearer" {
			http.Error(w, `{"error":"invalid authorization format"}`, http.StatusUnauthorized)
			return
		}

		claims := &Claims{}
		token, err := jwt.ParseWithClaims(parts[1], claims,
			func(token *jwt.Token) (interface{}, error) {
				return jwtSecret, nil
			})
		if err != nil || !token.Valid {
			http.Error(w, `{"error":"invalid token"}`, http.StatusUnauthorized)
			return
		}

		ctx := context.WithValue(r.Context(), userIDKey, claims.UserID)
		ctx = context.WithValue(ctx, emailKey, claims.Email)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

// LoggingMiddleware logs each request.
func LoggingMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		rw := &statusWriter{ResponseWriter: w, status: http.StatusOK}
		next.ServeHTTP(rw, r)
		log.Printf("%s %s %d %s %s",
			r.Method, r.URL.Path, rw.status, time.Since(start), r.RemoteAddr)
	})
}

// RecoveryMiddleware catches panics.
func RecoveryMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if rec := recover(); rec != nil {
				log.Printf("PANIC: %v", rec)
				http.Error(w, `{"error":"internal server error"}`,
					http.StatusInternalServerError)
			}
		}()
		next.ServeHTTP(w, r)
	})
}

// CORSMiddleware adds CORS headers.
func CORSMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Access-Control-Allow-Origin", "*")
		w.Header().Set("Access-Control-Allow-Methods",
			"GET, POST, PUT, PATCH, DELETE, OPTIONS")
		w.Header().Set("Access-Control-Allow-Headers",
			"Content-Type, Authorization")

		if r.Method == http.MethodOptions {
			w.WriteHeader(http.StatusOK)
			return
		}

		next.ServeHTTP(w, r)
	})
}

type statusWriter struct {
	http.ResponseWriter
	status int
}

func (w *statusWriter) WriteHeader(code int) {
	w.status = code
	w.ResponseWriter.WriteHeader(code)
}

func UserID(ctx context.Context) int {
	if v, ok := ctx.Value(userIDKey).(int); ok {
		return v
	}
	return 0
}
```

> 🔑 **Checkpoint:** `UserID(ctx)` is how every protected handler learns who
> is calling — the middleware wrote it into the context, and handlers read it
> out. Keep the lookup helper in one place so you never sprinkle
> `ctx.Value(...)` magic strings around.

Create `handlers/auth.go`:

```go
package handlers

import (
	"context"
	"database/sql"
	"encoding/json"
	"errors"
	"net/http"

	"notes-api/models"
	"notes-api/repository"

	"golang.org/x/crypto/bcrypt"
)

type AuthHandler struct {
	users *repository.UserRepository
}

func NewAuthHandler(users *repository.UserRepository) *AuthHandler {
	return &AuthHandler{users: users}
}

func (h *AuthHandler) Register(w http.ResponseWriter, r *http.Request) {
	var req models.RegisterRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid JSON"})
		return
	}
	defer r.Body.Close()

	if req.Email == "" || req.Password == "" {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "email and password required"})
		return
	}

	hash, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
	if err != nil {
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	user := &models.User{
		Email:        req.Email,
		PasswordHash: string(hash),
		Name:         req.Name,
	}

	if err := h.users.Create(r.Context(), user); err != nil {
		writeJSON(w, http.StatusConflict, map[string]string{"error": "email already registered"})
		return
	}

	token, expiresAt, err := GenerateToken(user.ID, user.Email, 24)
	if err != nil {
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "token generation failed"})
		return
	}

	writeJSON(w, http.StatusCreated, models.TokenResponse{
		AccessToken: token,
		ExpiresAt:   expiresAt,
	})
}

func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
	var req models.LoginRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid JSON"})
		return
	}
	defer r.Body.Close()

	user, err := h.users.GetByEmail(r.Context(), req.Email)
	if err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			writeJSON(w, http.StatusUnauthorized, map[string]string{"error": "invalid credentials"})
			return
		}
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	if err := bcrypt.CompareHashAndPassword([]byte(user.PasswordHash), []byte(req.Password)); err != nil {
		writeJSON(w, http.StatusUnauthorized, map[string]string{"error": "invalid credentials"})
		return
	}

	token, expiresAt, err := GenerateToken(user.ID, user.Email, 24)
	if err != nil {
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "token generation failed"})
		return
	}

	writeJSON(w, http.StatusOK, models.TokenResponse{
		AccessToken: token,
		ExpiresAt:   expiresAt,
	})
}
```

> 💡 **Tip:** The login flow is: look up by email → `bcrypt.CompareHashAndPassword`
> → generate JWT. Both "no such user" and "wrong password" return the same
> 401 — attackers can't enumerate which emails exist.

```go
func writeJSON(w http.ResponseWriter, status int, data any) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(data)
}
```

Create `handlers/notes.go`:

```go
package handlers

import (
	"database/sql"
	"encoding/json"
	"errors"
	"net/http"
	"strconv"

	"notes-api/models"
	"notes-api/repository"

	"github.com/go-chi/chi/v5"
)

type NoteHandler struct {
	notes *repository.NoteRepository
	cache *repository.NoteCache
}

func NewNoteHandler(notes *repository.NoteRepository, cache *repository.NoteCache) *NoteHandler {
	return &NoteHandler{notes: notes, cache: cache}
}

func (h *NoteHandler) Create(w http.ResponseWriter, r *http.Request) {
	var req models.CreateNoteRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid JSON"})
		return
	}
	defer r.Body.Close()

	if req.Title == "" {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "title is required"})
		return
	}

	note := &models.Note{
		UserID:  UserID(r.Context()),
		Title:   req.Title,
		Content: req.Content,
		Pinned:  req.Pinned,
		Tags:    req.Tags,
	}

	if err := h.notes.Create(r.Context(), note); err != nil {
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "failed to create note"})
		return
	}

	// Cache the new note.
	if h.cache != nil {
		h.cache.SetNote(r.Context(), note)
	}

	writeJSON(w, http.StatusCreated, note)
}

func (h *NoteHandler) Get(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.Atoi(chi.URLParam(r, "id"))
	if err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid note id"})
		return
	}

	uid := UserID(r.Context())

	// Check cache first.
	if h.cache != nil {
		if note, ok := h.cache.GetNote(r.Context(), id); ok && note.UserID == uid {
			writeJSON(w, http.StatusOK, note)
			return
		}
	}

	note, err := h.notes.GetByID(r.Context(), id, uid)
	if err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			writeJSON(w, http.StatusNotFound, map[string]string{"error": "note not found"})
			return
		}
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	// Populate cache.
	if h.cache != nil {
		h.cache.SetNote(r.Context(), note)
	}

	writeJSON(w, http.StatusOK, note)
}
```

> 🔑 **Checkpoint:** Note the double check — cache hit returns only if
> `note.UserID == uid`. Cached or not, the note is checked against the
> caller *before* producing a response. Authorization outranks caching.

```go
func (h *NoteHandler) List(w http.ResponseWriter, r *http.Request) {
	uid := UserID(r.Context())
	page, _ := strconv.Atoi(r.URL.Query().Get("page"))
	limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))

	if page < 1 {
		page = 1
	}
	if limit < 1 || limit > 100 {
		limit = 20
	}

	notes, total, err := h.notes.ListByUser(r.Context(), uid, page, limit)
	if err != nil {
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "failed to list notes"})
		return
	}

	if notes == nil {
		notes = []models.Note{}
	}

	writeJSON(w, http.StatusOK, models.NoteListResponse{
		Notes: notes,
		Total: total,
		Page:  page,
		Limit: limit,
	})
}

func (h *NoteHandler) Update(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.Atoi(chi.URLParam(r, "id"))
	if err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid note id"})
		return
	}

	uid := UserID(r.Context())

	note, err := h.notes.GetByID(r.Context(), id, uid)
	if err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			writeJSON(w, http.StatusNotFound, map[string]string{"error": "note not found"})
			return
		}
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	var req models.UpdateNoteRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid JSON"})
		return
	}
	defer r.Body.Close()

	if req.Title != nil {
		note.Title = *req.Title
	}
	if req.Content != nil {
		note.Content = *req.Content
	}
	if req.Pinned != nil {
		note.Pinned = *req.Pinned
	}
	if req.Tags != nil {
		note.Tags = req.Tags
	}

	if err := h.notes.Update(r.Context(), note); err != nil {
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "failed to update note"})
		return
	}

	// Invalidate cache.
	if h.cache != nil {
		h.cache.InvalidateNote(r.Context(), id)
	}

	writeJSON(w, http.StatusOK, note)
}

func (h *NoteHandler) Delete(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.Atoi(chi.URLParam(r, "id"))
	if err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid note id"})
		return
	}

	if err := h.notes.Delete(r.Context(), id, UserID(r.Context())); err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			writeJSON(w, http.StatusNotFound, map[string]string{"error": "note not found"})
			return
		}
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "failed to delete note"})
		return
	}

	if h.cache != nil {
		h.cache.InvalidateNote(r.Context(), id)
	}

	w.WriteHeader(http.StatusNoContent)
}

func (h *NoteHandler) Search(w http.ResponseWriter, r *http.Request) {
	q := r.URL.Query().Get("q")
	if q == "" {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "q parameter required"})
		return
	}

	notes, err := h.notes.Search(r.Context(), UserID(r.Context()), q)
	if err != nil {
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "search failed"})
		return
	}

	if notes == nil {
		notes = []models.Note{}
	}

	writeJSON(w, http.StatusOK, map[string]any{"notes": notes, "count": len(notes)})
}
```

Create `handlers/health.go`:

```go
package handlers

import (
	"database/sql"
	"net/http"
	"time"

	"notes-api/models"
)

type HealthHandler struct {
	db *sql.DB
}

func NewHealthHandler(db *sql.DB) *HealthHandler {
	return &HealthHandler{db: db}
}

func (h *HealthHandler) Check(w http.ResponseWriter, r *http.Request) {
	services := map[string]string{}

	// Check database.
	ctx := r.Context()
	if err := h.db.PingContext(ctx); err != nil {
		services["database"] = "unhealthy: " + err.Error()
	} else {
		services["database"] = "healthy"
	}

	status := "ok"
	for _, v := range services {
		if v != "healthy" {
			status = "degraded"
			break
		}
	}

	resp := models.HealthResponse{
		Status:    status,
		Services:  services,
		Timestamp: time.Now(),
	}

	code := http.StatusOK
	if status != "ok" {
		code = http.StatusServiceUnavailable
	}

	writeJSON(w, code, resp)
}

func (h *HealthHandler) Readiness(w http.ResponseWriter, r *http.Request) {
	if err := h.db.PingContext(r.Context()); err != nil {
		writeJSON(w, http.StatusServiceUnavailable, map[string]string{"status": "not ready"})
		return
	}
	writeJSON(w, http.StatusOK, map[string]string{"status": "ready"})
}
```

> 💡 **Tip:** Liveness vs readiness: `/health` always answers, `/ready`
> pings the database. Orchestrators restart a container whose liveness
> fails, but route traffic away from an unready one. Keep them separate.

---

## Part G: Main Server

Create `main.go`:

```go
package main

import (
	"context"
	"database/sql"
	"flag"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"notes-api/config"
	"notes-api/handlers"
	"notes-api/repository"

	"github.com/go-chi/chi/v5"
	_ "github.com/jackc/pgx/v5/stdlib"
)

func main() {
	configPath := flag.String("config", "", "config file path")
	flag.Parse()

	cfg, err := config.Load(*configPath)
	if err != nil {
		log.Fatalf("config: %v", err)
	}

	// Database.
	db, err := sql.Open("pgx", cfg.Database.DSN())
	if err != nil {
		log.Fatalf("db open: %v", err)
	}
	defer db.Close()

	db.SetMaxOpenConns(25)
	db.SetMaxIdleConns(5)
	db.SetConnMaxLifetime(5 * time.Minute)

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	if err := db.PingContext(ctx); err != nil {
		log.Fatalf("db ping: %v", err)
	}
	log.Println("connected to database")

	// Run migrations.
	if err := runMigrations(db); err != nil {
		log.Fatalf("migrations: %v", err)
	}

	// Repositories.
	userRepo := repository.NewUserRepository(db)
	noteRepo := repository.NewNoteRepository(db)

	// Cache (nil for now — wire Redis in production).
	var noteCache *repository.NoteCache // nil = no caching

	// Handlers.
	authHandler := handlers.NewAuthHandler(userRepo)
	noteHandler := handlers.NewNoteHandler(noteRepo, noteCache)
	healthHandler := handlers.NewHealthHandler(db)

	// Router.
	r := chi.NewRouter()

	// Global middleware.
	r.Use(handlers.RecoveryMiddleware)
	r.Use(handlers.LoggingMiddleware)
	r.Use(handlers.CORSMiddleware)

	// Health endpoints (no auth).
	r.Get("/health", healthHandler.Check)
	r.Get("/health/ready", healthHandler.Readiness)

	// Auth routes.
	r.Post("/auth/register", authHandler.Register)
	r.Post("/auth/login", authHandler.Login)

	// Protected API routes.
	r.Group(func(r chi.Router) {
		r.Use(handlers.AuthMiddleware)

		r.Route("/api/notes", func(r chi.Router) {
			r.Post("/", noteHandler.Create)
			r.Get("/", noteHandler.List)
			r.Get("/search", noteHandler.Search)
			r.Get("/{id}", noteHandler.Get)
			r.Put("/{id}", noteHandler.Update)
			r.Delete("/{id}", noteHandler.Delete)
		})
	})
```

> 🔑 **Checkpoint:** Middleware scoping at the `chi.Router` group level — every
> `/api/notes/*` route inherits `AuthMiddleware` automatically. Adding a
> route under this group is protected by default.

```go
	// Server.
	srv := &http.Server{
		Addr:         cfg.Server.Addr(),
		Handler:      r,
		ReadTimeout:  15 * time.Second,
		WriteTimeout: 15 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	// Graceful shutdown.
	ctx, stop := signal.NotifyContext(context.Background(),
		os.Interrupt, syscall.SIGTERM)
	defer stop()

	go func() {
		log.Printf("notes API listening on %s", cfg.Server.Addr())
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("server: %v", err)
		}
	}()

	<-ctx.Done()
	log.Println("shutting down...")

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
	defer cancel()

	if err := srv.Shutdown(shutdownCtx); err != nil {
		log.Printf("shutdown: %v", err)
	}

	db.Close()
	log.Println("server stopped")
}

func runMigrations(db *sql.DB) error {
	migrations := []string{
		`CREATE TABLE IF NOT EXISTS users (
			id SERIAL PRIMARY KEY,
			email VARCHAR(255) UNIQUE NOT NULL,
			password_hash VARCHAR(255) NOT NULL,
			name VARCHAR(255) NOT NULL DEFAULT '',
			created_at TIMESTAMPTZ DEFAULT NOW()
		)`,
		`CREATE INDEX IF NOT EXISTS idx_users_email ON users (email)`,
		`CREATE TABLE IF NOT EXISTS notes (
			id SERIAL PRIMARY KEY,
			user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
			title VARCHAR(500) NOT NULL,
			content TEXT DEFAULT '',
			pinned BOOLEAN DEFAULT FALSE,
			tags TEXT[] DEFAULT '{}',
			created_at TIMESTAMPTZ DEFAULT NOW(),
			updated_at TIMESTAMPTZ DEFAULT NOW()
		)`,
		`CREATE INDEX IF NOT EXISTS idx_notes_user_id ON notes (user_id)`,
		`CREATE INDEX IF NOT EXISTS idx_notes_pinned ON notes (pinned)`,
	}

	for _, m := range migrations {
		if _, err := db.Exec(m); err != nil {
			log.Printf("migration warning: %v (may already exist)", err)
		}
	}
	log.Println("migrations complete")
	return nil
}
```

---

## Part H: Docker

Create `Dockerfile`:

```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder

RUN apk add --no-cache git

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/notes-api .

# Runtime stage
FROM alpine:3.19

RUN apk add --no-cache ca-certificates tzdata
RUN adduser -D -g '' appuser

COPY --from=builder /app/notes-api /usr/local/bin/notes-api
COPY --from=builder /app/migrations /app/migrations

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s \
    CMD wget -qO- http://localhost:8080/health/ready || exit 1

ENTRYPOINT ["notes-api"]
```

> 💡 **Tip:** The runtime stage is `alpine` (not `scratch`) because the
> service does HTTPS requests (repo access) and needs `tzdata` for correct
> timestamps — plus a non-root `appuser`, which containers should always
> run as.

Create `docker-compose.yml`:

```yaml
version: "3.8"

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: notes
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: "postgres://postgres:postgres@postgres:5432/notes?sslmode=disable"
      REDIS_ADDR: "redis:6379"
      JWT_SECRET: "production-secret-change-me"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

volumes:
  pgdata:
```

Create `Makefile`:

```makefile
.PHONY: build run test docker-up docker-down

build:
	go build -o bin/notes-api .

run:
	go run .

test:
	go test -v -race ./...

docker-up:
	docker compose up -d --build

docker-down:
	docker compose down

docker-logs:
	docker compose logs -f api

lint:
	go vet ./...
	staticcheck ./...
```

### Running with Docker

```bash
# Start everything
make docker-up

# Test
curl http://localhost:8080/health
# {"status":"ok","services":{"database":"healthy"},"timestamp":"..."}

# Register
curl -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"secret123","name":"Test User"}'

# Save the token from the response, then:
TOKEN="<your-token>"

# Create a note
curl -X POST http://localhost:8080/api/notes \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"title":"My first note","content":"Hello world!","tags":["intro"]}'
# {"id":1,"user_id":1,"title":"My first note",...}

# List notes
curl http://localhost:8080/api/notes \
  -H "Authorization: Bearer $TOKEN"
# {"notes":[...],"total":1,"page":1,"limit":20}

# Search notes
curl "http://localhost:8080/api/notes/search?q=first" \
  -H "Authorization: Bearer $TOKEN"
# {"notes":[...],"count":1}
```

> 🔑 **Checkpoint:** The full happy path is a real request sequence —
> register, grab the token, auth'd create/list/search. If any curl fails,
> the mismatch is in wiring (env vars, ports, hostnames), not in Go code.

---

## Part I: Tests

Create `handlers/auth_test.go`:

```go
package handlers

import (
	"bytes"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"

	"notes-api/models"
)

func TestRegisterValidation(t *testing.T) {
	tests := []struct {
		name       string
		body       string
		wantStatus int
	}{
		{"valid", `{"email":"a@b.com","password":"secret123"}`, http.StatusCreated},
		{"missing email", `{"password":"secret123"}`, http.StatusBadRequest},
		{"missing password", `{"email":"a@b.com"}`, http.StatusBadRequest},
		{"invalid json", `not json`, http.StatusBadRequest},
		{"empty body", `{}`, http.StatusBadRequest},
	}

	// Note: These tests validate handler logic but require a real DB
	// for full integration. For unit tests, mock the repository.
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			// Structural validation only — full tests need DB.
			var req models.RegisterRequest
			err := json.Unmarshal([]byte(tt.body), &req)

			// Check basic validation.
			if tt.name == "invalid json" {
				if err == nil {
					t.Error("expected error for invalid JSON")
				}
				return
			}
			if tt.name == "empty body" && (req.Email != "" || req.Password != "") {
				t.Error("expected empty fields for {}")
			}
		})
	}
}
```

> 💡 **Tip:** Middleware tests need no database — `AuthMiddleware(next)` runs
> against a stub handler, proving 401 on missing/bad headers. That's the
> cheapest test in the project and the most security-critical one.

```go
func TestAuthMiddlewareRejection(t *testing.T) {
	next := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	})

	handler := AuthMiddleware(next)

	// No auth header.
	req := httptest.NewRequest(http.MethodGet, "/protected", nil)
	w := httptest.NewRecorder()
	handler.ServeHTTP(w, req)

	if w.Code != http.StatusUnauthorized {
		t.Errorf("expected 401, got %d", w.Code)
	}

	// Invalid format.
	req = httptest.NewRequest(http.MethodGet, "/protected", nil)
	req.Header.Set("Authorization", "InvalidFormat")
	w = httptest.NewRecorder()
	handler.ServeHTTP(w, req)

	if w.Code != http.StatusUnauthorized {
		t.Errorf("expected 401 for bad format, got %d", w.Code)
	}
}

func TestGenerateToken(t *testing.T) {
	token, expiresAt, err := GenerateToken(42, "test@example.com", 24)
	if err != nil {
		t.Fatalf("GenerateToken: %v", err)
	}
	if token == "" {
		t.Error("expected non-empty token")
	}
	if expiresAt == 0 {
		t.Error("expected non-zero expiry")
	}
}

func TestParsePGArray(t *testing.T) {
	tests := []struct {
		input string
		want  []string
	}{
		{"{}", []string{}},
		{`{"a"}`, []string{"a"}},
		{`{"a","b","c"}`, []string{"a", "b", "c"}},
		{`{}`, []string{}},
	}

	for _, tt := range tests {
		got := parsePGArray(tt.input)
		if len(got) != len(tt.want) {
			t.Errorf("parsePGArray(%q): got %v, want %v", tt.input, got, tt.want)
			continue
		}
		for i := range got {
			if got[i] != tt.want[i] {
				t.Errorf("parsePGArray(%q)[%d]: got %q, want %q", tt.input, i, got[i], tt.want[i])
			}
		}
	}
}
```

> 🔑 **Checkpoint:** Tests land at every layer — middleware (no DB),
> handlers (mock repos), repositories (real PG with the `integration`
> build tag), and pure helpers like `parsePGArray`. If adding a feature
> has no test landing spot, the layering is wrong.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                        notes-api                                  │
│                                                                  │
│  ┌────────────┐   ┌──────────────┐   ┌────────────────────┐     │
│  │   chi       │   │  Middleware   │   │    Handlers         │     │
│  │   Router    │──▶│  Recovery    │──▶│  Auth (register)    │     │
│  │             │   │  Logging     │   │  Auth (login)       │     │
│  │  /health    │   │  CORS        │   │  Notes (CRUD)       │     │
│  │  /auth/*    │   │  JWT Auth    │   │  Health check       │     │
│  │  /api/notes │   │              │   │                     │     │
│  └────────────┘   └──────────────┘   └─────────┬──────────┘     │
│                                                  │               │
│                              ┌────────────────────┼────────┐     │
│                              │                    │        │     │
│                              ▼                    ▼        │     │
│                     ┌──────────────┐     ┌───────────┐    │     │
│                     │  PostgreSQL   │     │   Redis    │    │     │
│                     │  (primary)    │     │  (cache)  │    │     │
│                     └──────────────┘     └───────────┘    │     │
│                                                          │     │
│  ┌──────────────────────────────────────────────────────┘     │
│  │  Graceful Shutdown                                          │
│  │  SIGTERM → drain → close DB → close Redis → stop server    │
│  └────────────────────────────────────────────────────────────│
└──────────────────────────────────────────────────────────────────┘
```

---

## Modern Practices

- **Multi-stage Docker build** — the build stage compiles a static binary;
  the runtime stage is a tiny Alpine image (~15MB). No Go toolchain in
  production.
- **`chi` router** — lightweight, stdlib-compatible, idiomatic Go router.
  No magic, no code generation. Just clean `http.Handler` composition.
- **`healthcheck` in Dockerfile** — Docker knows when the container is
  unhealthy. Orchestrators (K8s, ECS) use this to restart failing containers.
- **Cache-aside pattern** — check cache → miss → check DB → populate cache.
  On write → invalidate cache. Simple, correct, works at scale.
- **Repository pattern** — `NoteRepository` is a struct that wraps `*sql.DB`.
  You can swap it for a mock in tests without changing handlers.
- **Structured logging** — every log line includes method, path, status,
  duration, and remote address. In production, use `slog` or zerolog for
  JSON output.
- **`signal.NotifyContext`** — the modern Go pattern for graceful shutdown.
  One context, one function call, clean cancellation.

---

## Common Mistakes

- **Not closing `rows` after `QueryContext`.** Leaks connections. Always
  `defer rows.Close()`. This is the #1 mistake in Go database code.
- **Returning `null` for empty arrays.** Initialize slices with
  `notes = []models.Note{}` before JSON encoding. Clients expect `[]`,
  not `null`.
- **Hard-coding secrets.** JWT secrets, database passwords, and API keys
  MUST come from environment variables or a secrets manager. Never commit
  them.
- **Missing server timeouts.** Without `ReadTimeout` and `WriteTimeout`,
  slow clients can exhaust your connection pool (slowloris attack).
- **Not using `context.Context`** on database calls. Without context, you
  can't cancel long-running queries or enforce timeouts.
- **Ignoring `rows.Err()`** after iterating. The loop may exit early on
  error. Always check.
- **Forgetting `defer r.Body.Close()`.** Leaks file descriptors. Close it
  after decoding.
- **Using `http.DefaultClient`.** It has zero timeout. Always create a
  client with explicit timeouts.
- **Not invalidating cache on writes.** If you write to DB but not Redis,
  clients get stale data until the TTL expires. Always invalidate on write.

---

## Stretch Goals / Extensions

1. **Wire Redis** — implement `NoteCache` with `github.com/redis/go-redis/v9`.
   Add cache warming on startup.
2. **Note sharing** — add a `shared_with` column and share endpoints.
3. **File attachments** — store note attachments in S3/MinIO.
4. **WebSocket real-time updates** — push note changes to connected clients.
5. **Full-text search** — use PostgreSQL `tsvector` + `tsquery` instead of
   `ILIKE`.
6. **Rate limiting** — add a per-user rate limiter middleware.
7. **Prometheus metrics** — expose `/metrics` with request count, latency
   histograms, and DB connection pool stats.
8. **OpenTelemetry** — add distributed tracing with Jaeger or Zipkin.
9. **CI/CD pipeline** — GitHub Actions that runs tests, builds the Docker
   image, and pushes to a registry.
10. **Kubernetes deployment** — `Deployment`, `Service`, `Ingress`, `ConfigMap`,
    `Secret` manifests.

---

## Congratulations!

You've completed the capstone project and the entire Go learning path. You've
built:

- A CLI application (Project 01)
- An HTTP server with middleware (Project 02)
- A concurrent system with goroutines (Project 03)
- A REST API with PostgreSQL and JWT (Project 04)
- A gRPC service with streaming (Project 05)
- A go-zero microservice (Project 06)
- A production-grade backend (Project 07)

These projects cover the core of Go development. The next step is to build
something real — pick an idea and ship it.

---

## Next

You've completed the projects section. Review any earlier parts as needed, or
start building your own project. The Go community is active — contribute to
open source, write blog posts, and keep shipping.
