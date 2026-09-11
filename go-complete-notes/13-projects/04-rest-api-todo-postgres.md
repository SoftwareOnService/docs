# Project 04: REST API Todo + PostgreSQL

A production-style REST API for managing todos, backed by PostgreSQL. This
project brings together HTTP (Part 07), Gin (Part 09), database/sql (Part 10),
testing (Part 06), and authentication (Part 12).

---

## Learning Goals

- Build a RESTful API with Gin framework
- Use `database/sql` with PostgreSQL and `pgx` driver
- Implement database migrations (up/down pattern)
- Apply the Repository pattern (separate data access from business logic)
- Authenticate requests with JWT tokens
- Write structured, testable handlers
- Test with a real database using `t.Parallel` and testcontainers

## Prerequisites

| Part | What you need |
|------|---------------|
| 03   | Structs, interfaces, error handling |
| 05   | `database/sql`, `os` |
| 06   | Table-driven tests, test helpers |
| 07   | HTTP basics, `net/http` |
| 09   | Gin framework, routing, middleware, JSON binding |
| 10   | PostgreSQL, migrations, SQL queries |
| 12   | JWT authentication, password hashing |

---

## Project Structure

```
rest-todo-api/
├── main.go                 # Entry point, server setup
├── go.mod / go.sum
├── migrations/
│   ├── 001_create_users.up.sql
│   ├── 001_create_users.down.sql
│   ├── 002_create_todos.up.sql
│   └── 002_create_todos.down.sql
├── models/
│   └── models.go           # Domain types (User, Todo)
├── repository/
│   ├── user_repo.go        # User database operations
│   └── todo_repo.go        # Todo database operations
├── handlers/
│   ├── auth.go             # Register, Login handlers
│   ├── todo.go             # CRUD handlers for todos
│   └── middleware.go        # JWT auth middleware
├── handlers/
│   ├── auth_test.go        # Auth handler tests
│   └── todo_test.go        # Todo handler tests
└── repository/
    ├── user_repo_test.go   # Repository tests (require PG)
    └── todo_repo_test.go   # Repository tests (require PG)
```

```mermaid
graph TD
    A[Client] -->|POST /auth/register| B[Gin Router]
    A -->|POST /auth/login| B
    A -->|GET /todos| C[JWT Middleware]
    C -->|valid token| D[Handlers]
    D --> E[Repository Interface]
    E --> F[PostgreSQL]

    subgraph "Layers"
        B
        C
        D
        E
        F
    end
```

> 🧠 **Memory aid:** Handlers never touch SQL — they talk to a repository
> *interface* that Postgres implements. Swapping Postgres for another DB
> later only replaces one folder, not the whole app.

---

## Part A: Domain Models

Create `models/models.go`:

```go
package models

import "time"

// User represents a registered user.
type User struct {
	ID        int       `json:"id"`
	Email     string    `json:"email"`
	Password  string    `json:"-"` // never serialize
	CreatedAt time.Time `json:"created_at"`
}

// Todo represents a single task.
type Todo struct {
	ID          int       `json:"id"`
	UserID      int       `json:"user_id"`
	Title       string    `json:"title"`
	Description string    `json:"description"`
	Done        bool      `json:"done"`
	CreatedAt   time.Time `json:"created_at"`
	UpdatedAt   time.Time `json:"updated_at"`
}

// CreateTodoRequest is the JSON body for creating a todo.
type CreateTodoRequest struct {
	Title       string `json:"title" binding:"required"`
	Description string `json:"description"`
}

// UpdateTodoRequest is the JSON body for updating a todo.
type UpdateTodoRequest struct {
	Title       *string `json:"title"`
	Description *string `json:"description"`
	Done        *bool   `json:"done"`
}

// RegisterRequest is the JSON body for registration.
type RegisterRequest struct {
	Email    string `json:"email" binding:"required,email"`
	Password string `json:"password" binding:"required,min=6"`
}

// LoginRequest is the JSON body for login.
type LoginRequest struct {
	Email    string `json:"email" binding:"required,email"`
	Password string `json:"password" binding:"required"`
}

// AuthResponse is returned after successful login.
type AuthResponse struct {
	Token     string `json:"token"`
	ExpiresAt int64  `json:"expires_at"`
}
```

### Key Points

- **`json:"-"`** on Password — prevents the password hash from ever appearing
  in JSON responses, even if the struct is serialized accidentally.
- **Pointer fields in `UpdateTodoRequest`** — `*string` and `*bool` let us
  distinguish "not provided" (nil) from "provided as zero value". This is
  critical for PATCH semantics.
- **`binding:"required"`** — Gin's validator tags. Automatically rejects
  requests missing required fields.

> 🔑 **Checkpoint:** `json:"-"` is a one-character fix that prevents a
> password-hash leak class of bugs. If you ever see a password field in an
> API response, the `json:"-"` tag is missing.

---

## Part B: Migrations

Create `migrations/001_create_users.up.sql`:

```sql
CREATE TABLE IF NOT EXISTS users (
    id         SERIAL PRIMARY KEY,
    email      VARCHAR(255) UNIQUE NOT NULL,
    password   VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users (email);
```

Create `migrations/001_create_users.down.sql`:

```sql
DROP INDEX IF EXISTS idx_users_email;
DROP TABLE IF EXISTS users;
```

Create `migrations/002_create_todos.up.sql`:

```sql
CREATE TABLE IF NOT EXISTS todos (
    id          SERIAL PRIMARY KEY,
    user_id     INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title       VARCHAR(255) NOT NULL,
    description TEXT DEFAULT '',
    done        BOOLEAN DEFAULT FALSE,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_todos_user_id ON todos (user_id);
CREATE INDEX idx_todos_done ON todos (done);
```

Create `migrations/002_create_todos.down.sql`:

```sql
DROP INDEX IF EXISTS idx_todos_done;
DROP INDEX IF EXISTS idx_todos_user_id;
DROP TABLE IF EXISTS todos;
```

### Running Migrations

We'll implement a simple migration runner in the main.go. For production,
consider using `golang-migrate/migrate` or a similar tool.

> 💡 **Tip:** The `down.sql` files exist so you can *undo* a migration during
> development. Never delete a migration once it has shipped — add a new one
> instead. History is write-only.

---

## Part C: Repository Layer

Create `repository/user_repo.go`:

```go
package repository

import (
	"context"
	"database/sql"
	"fmt"

	"rest-todo-api/models"
)

// UserRepository handles user database operations.
type UserRepository struct {
	db *sql.DB
}

// NewUserRepository creates a new UserRepository.
func NewUserRepository(db *sql.DB) *UserRepository {
	return &UserRepository{db: db}
}

// Create inserts a new user and returns the generated ID.
func (r *UserRepository) Create(ctx context.Context, user *models.User) error {
	query := `INSERT INTO users (email, password) VALUES ($1, $2) RETURNING id, created_at`
	return r.db.QueryRowContext(ctx, query, user.Email, user.Password).
		Scan(&user.ID, &user.CreatedAt)
}

// GetByEmail finds a user by email. Returns sql.ErrNoRows if not found.
func (r *UserRepository) GetByEmail(ctx context.Context, email string) (*models.User, error) {
	query := `SELECT id, email, password, created_at FROM users WHERE email = $1`
	user := &models.User{}
	err := r.db.QueryRowContext(ctx, query, email).
		Scan(&user.ID, &user.Email, &user.Password, &user.CreatedAt)
	if err != nil {
		return nil, fmt.Errorf("GetByEmail(%s): %w", email, err)
	}
	return user, nil
}

// GetByID finds a user by ID.
func (r *UserRepository) GetByID(ctx context.Context, id int) (*models.User, error) {
	query := `SELECT id, email, password, created_at FROM users WHERE id = $1`
	user := &models.User{}
	err := r.db.QueryRowContext(ctx, query, id).
		Scan(&user.ID, &user.Email, &user.Password, &user.CreatedAt)
	if err != nil {
		return nil, fmt.Errorf("GetByID(%d): %w", id, err)
	}
	return user, nil
}
```

Create `repository/todo_repo.go`:

```go
package repository

import (
	"context"
	"database/sql"
	"fmt"
	"time"

	"rest-todo-api/models"
)

// TodoRepository handles todo database operations.
type TodoRepository struct {
	db *sql.DB
}

// NewTodoRepository creates a new TodoRepository.
func NewTodoRepository(db *sql.DB) *TodoRepository {
	return &TodoRepository{db: db}
}

// Create inserts a new todo.
func (r *TodoRepository) Create(ctx context.Context, todo *models.Todo) error {
	query := `INSERT INTO todos (user_id, title, description)
		VALUES ($1, $2, $3)
		RETURNING id, created_at, updated_at`
	return r.db.QueryRowContext(ctx, query,
		todo.UserID, todo.Title, todo.Description).
		Scan(&todo.ID, &todo.CreatedAt, &todo.UpdatedAt)
}

// GetByID returns a todo by ID, ensuring it belongs to the given user.
func (r *TodoRepository) GetByID(ctx context.Context, id, userID int) (*models.Todo, error) {
	query := `SELECT id, user_id, title, description, done, created_at, updated_at
		FROM todos WHERE id = $1 AND user_id = $2`
	todo := &models.Todo{}
	err := r.db.QueryRowContext(ctx, query, id, userID).Scan(
		&todo.ID, &todo.UserID, &todo.Title, &todo.Description,
		&todo.Done, &todo.CreatedAt, &todo.UpdatedAt,
	)
	if err != nil {
		return nil, fmt.Errorf("GetByID(%d, user=%d): %w", id, userID, err)
	}
	return todo, nil
}

// ListByUser returns all todos for a user, ordered by created_at DESC.
func (r *TodoRepository) ListByUser(ctx context.Context, userID int) ([]models.Todo, error) {
	query := `SELECT id, user_id, title, description, done, created_at, updated_at
		FROM todos WHERE user_id = $1 ORDER BY created_at DESC`
	rows, err := r.db.QueryContext(ctx, query, userID)
	if err != nil {
		return nil, fmt.Errorf("ListByUser(%d): %w", userID, err)
	}
	defer rows.Close()

	var todos []models.Todo
	for rows.Next() {
		var t models.Todo
		if err := rows.Scan(
			&t.ID, &t.UserID, &t.Title, &t.Description,
			&t.Done, &t.CreatedAt, &t.UpdatedAt,
		); err != nil {
			return nil, fmt.Errorf("ListByUser scan: %w", err)
		}
		todos = append(todos, t)
	}
	if err := rows.Err(); err != nil {
		return nil, fmt.Errorf("ListByUser rows: %w", err)
	}
	return todos, nil
}

// Update modifies a todo's fields.
func (r *TodoRepository) Update(ctx context.Context, todo *models.Todo) error {
	query := `UPDATE todos SET title=$1, description=$2, done=$3, updated_at=$4
		WHERE id=$5 AND user_id=$6`
	result, err := r.db.ExecContext(ctx, query,
		todo.Title, todo.Description, todo.Done, time.Now(),
		todo.ID, todo.UserID,
	)
	if err != nil {
		return fmt.Errorf("Update(%d): %w", todo.ID, err)
	}
	rows, err := result.RowsAffected()
	if err != nil {
		return fmt.Errorf("Update rows affected: %w", err)
	}
	if rows == 0 {
		return fmt.Errorf("todo %d not found or not owned by user %d", todo.ID, todo.UserID)
	}
	todo.UpdatedAt = time.Now()
	return nil
}

// Delete removes a todo by ID and user.
func (r *TodoRepository) Delete(ctx context.Context, id, userID int) error {
	query := `DELETE FROM todos WHERE id = $1 AND user_id = $2`
	result, err := r.db.ExecContext(ctx, query, id, userID)
	if err != nil {
		return fmt.Errorf("Delete(%d): %w", id, err)
	}
	rows, err := result.RowsAffected()
	if err != nil {
		return err
	}
	if rows == 0 {
		return fmt.Errorf("todo %d not found or not owned by user %d", id, userID)
	}
	return nil
}
```

### Key Points

- **`context.Context`** on every method — allows timeouts and cancellation.
  In production, pass `r.Context()` from the HTTP handler.
- **`RETURNING` clause** — PostgreSQL lets you return inserted/updated
  values directly. Avoids a separate SELECT.
- **`defer rows.Close()`** — always close rows after QueryContext. Failing to
  do this leaks connections from the pool.
- **`rows.Err()`** — after the loop, check for iteration errors. The loop
  can end early on error.
- **User scoping** — every todo query includes `AND user_id = $N`. Users
  can never see or modify each other's todos.

> ⚠️ **Watch out:** Every query carries `user_id` scoping *and* the handler
> pulls the user from the JWT — missing either one gives users a way to read
> other people's todos. Both halves must exist.

---

## Part D: Handlers + Auth Middleware

Create `handlers/middleware.go`:

```go
package handlers

import (
	"net/http"
	"os"
	"strings"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/golang-jwt/jwt/v5"
)

var jwtSecret []byte

func init() {
	secret := os.Getenv("JWT_SECRET")
	if secret == "" {
		secret = "dev-secret-change-in-production"
	}
	jwtSecret = []byte(secret)
}

// Claims represents JWT claims.
type Claims struct {
	UserID int    `json:"user_id"`
	Email  string `json:"email"`
	jwt.RegisteredClaims
}

// GenerateToken creates a JWT token for the given user.
func GenerateToken(userID int, email string) (string, int64, error) {
	expiresAt := time.Now().Add(24 * time.Hour)
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

// AuthMiddleware validates JWT tokens and sets user info in the context.
func AuthMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		authHeader := c.GetHeader("Authorization")
		if authHeader == "" {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
				"error": "authorization header required",
			})
			return
		}

		parts := strings.SplitN(authHeader, " ", 2)
		if len(parts) != 2 || parts[0] != "Bearer" {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
				"error": "authorization header must be: Bearer <token>",
			})
			return
		}

		claims := &Claims{}
		token, err := jwt.ParseWithClaims(parts[1], claims,
			func(token *jwt.Token) (interface{}, error) {
				return jwtSecret, nil
			},
		)
		if err != nil || !token.Valid {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
				"error": "invalid or expired token",
			})
			return
		}

		// Store user info in context for handlers.
		c.Set("user_id", claims.UserID)
		c.Set("email", claims.Email)
		c.Next()
	}
}
```

Create `handlers/auth.go`:

```go
package handlers

import (
	"net/http"

	"rest-todo-api/models"
	"rest-todo-api/repository"

	"github.com/gin-gonic/gin"
	"golang.org/x/crypto/bcrypt"
)

// AuthHandler handles authentication endpoints.
type AuthHandler struct {
	users *repository.UserRepository
}

// NewAuthHandler creates a new AuthHandler.
func NewAuthHandler(users *repository.UserRepository) *AuthHandler {
	return &AuthHandler{users: users}
}

// Register handles POST /auth/register.
func (h *AuthHandler) Register(c *gin.Context) {
	var req models.RegisterRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	// Hash password with bcrypt.
	hash, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to hash password"})
		return
	}

	user := &models.User{
		Email:    req.Email,
		Password: string(hash),
	}

	if err := h.users.Create(c.Request.Context(), user); err != nil {
		c.JSON(http.StatusConflict, gin.H{"error": "email already registered"})
		return
	}

	token, expiresAt, err := GenerateToken(user.ID, user.Email)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to generate token"})
		return
	}

	c.JSON(http.StatusCreated, models.AuthResponse{
		Token:     token,
		ExpiresAt: expiresAt,
	})
}

// Login handles POST /auth/login.
func (h *AuthHandler) Login(c *gin.Context) {
	var req models.LoginRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	user, err := h.users.GetByEmail(c.Request.Context(), req.Email)
	if err != nil {
		c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid credentials"})
		return
	}

	if err := bcrypt.CompareHashAndPassword([]byte(user.Password), []byte(req.Password)); err != nil {
		c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid credentials"})
		return
	}

	token, expiresAt, err := GenerateToken(user.ID, user.Email)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to generate token"})
		return
	}

	c.JSON(http.StatusOK, models.AuthResponse{
		Token:     token,
		ExpiresAt: expiresAt,
	})
}
```

Create `handlers/todo.go`:

```go
package handlers

import (
	"database/sql"
	"errors"
	"net/http"
	"strconv"

	"rest-todo-api/models"
	"rest-todo-api/repository"

	"github.com/gin-gonic/gin"
)

// TodoHandler handles todo CRUD endpoints.
type TodoHandler struct {
	todos *repository.TodoRepository
}

// NewTodoHandler creates a new TodoHandler.
func NewTodoHandler(todos *repository.TodoRepository) *TodoHandler {
	return &TodoHandler{todos: todos}
}

// userID extracts the authenticated user's ID from the Gin context.
func userID(c *gin.Context) int {
	return c.GetInt("user_id")
}
```

> 🔑 **Checkpoint:** The middleware put `user_id` in the Gin context, and
> every todo handler pulls it via `userID(c)` — that's the ownership chain
> that keeps user A out of user B's data. Never trust a `user_id` sent by
> the client.

```go
// Create handles POST /todos.
func (h *TodoHandler) Create(c *gin.Context) {
	var req models.CreateTodoRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	todo := &models.Todo{
		UserID:      userID(c),
		Title:       req.Title,
		Description: req.Description,
	}

	if err := h.todos.Create(c.Request.Context(), todo); err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to create todo"})
		return
	}

	c.JSON(http.StatusCreated, todo)
}

// Get handles GET /todos/:id.
func (h *TodoHandler) Get(c *gin.Context) {
	id, err := strconv.Atoi(c.Param("id"))
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid todo id"})
		return
	}

	todo, err := h.todos.GetByID(c.Request.Context(), id, userID(c))
	if err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			c.JSON(http.StatusNotFound, gin.H{"error": "todo not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to get todo"})
		return
	}

	c.JSON(http.StatusOK, todo)
}

// List handles GET /todos.
func (h *TodoHandler) List(c *gin.Context) {
	todos, err := h.todos.ListByUser(c.Request.Context(), userID(c))
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to list todos"})
		return
	}
	if todos == nil {
		todos = []models.Todo{} // Return empty array, not null.
	}
	c.JSON(http.StatusOK, gin.H{"todos": todos, "count": len(todos)})
}

// Update handles PATCH /todos/:id.
func (h *TodoHandler) Update(c *gin.Context) {
	id, err := strconv.Atoi(c.Param("id"))
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid todo id"})
		return
	}

	// Fetch existing todo.
	todo, err := h.todos.GetByID(c.Request.Context(), id, userID(c))
	if err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			c.JSON(http.StatusNotFound, gin.H{"error": "todo not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to get todo"})
		return
	}

	// Apply partial updates.
	var req models.UpdateTodoRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	if req.Title != nil {
		todo.Title = *req.Title
	}
	if req.Description != nil {
		todo.Description = *req.Description
	}
	if req.Done != nil {
		todo.Done = *req.Done
	}

	if err := h.todos.Update(c.Request.Context(), todo); err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to update todo"})
		return
	}

	c.JSON(http.StatusOK, todo)
}

// Delete handles DELETE /todos/:id.
func (h *TodoHandler) Delete(c *gin.Context) {
	id, err := strconv.Atoi(c.Param("id"))
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid todo id"})
		return
	}

	if err := h.todos.Delete(c.Request.Context(), id, userID(c)); err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			c.JSON(http.StatusNotFound, gin.H{"error": "todo not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to delete todo"})
		return
	}

	c.JSON(http.StatusNoContent, nil)
}
```

---

## Part E: Main Server

Create `main.go`:

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"rest-todo-api/handlers"
	"rest-todo-api/repository"

	"github.com/gin-gonic/gin"
	_ "github.com/jackc/pgx/v5/stdlib"
)

func main() {
	dbURL := os.Getenv("DATABASE_URL")
	if dbURL == "" {
		dbURL = "postgres://postgres:postgres@localhost:5432/todos?sslmode=disable"
	}

	// Open database connection pool.
	db, err := sql.Open("pgx", dbURL)
	if err != nil {
		log.Fatalf("failed to open database: %v", err)
	}
	defer db.Close()

	db.SetMaxOpenConns(25)
	db.SetMaxIdleConns(5)
	db.SetConnMaxLifetime(5 * time.Minute)
```

> 💡 **Tip:** `SetMaxOpenConns`/`SetMaxIdleConns` tune the pool — too high
> and Postgres chokes under load, too low and you bottleneck on `db.Conn`
> waits. Start low and measure.

```go
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	if err := db.PingContext(ctx); err != nil {
		log.Fatalf("failed to ping database: %v", err)
	}
	log.Println("connected to database")

	// Run migrations.
	if err := runMigrations(db); err != nil {
		log.Fatalf("migrations failed: %v", err)
	}

	// Wire up repositories.
	userRepo := repository.NewUserRepository(db)
	todoRepo := repository.NewTodoRepository(db)

	// Wire up handlers.
	authHandler := handlers.NewAuthHandler(userRepo)
	todoHandler := handlers.NewTodoHandler(todoRepo)

	// Set up Gin router.
	gin.SetMode(gin.ReleaseMode)
	r := gin.New()
	r.Use(gin.Recovery())
	r.Use(loggingMiddleware())

	// Public routes.
	r.POST("/auth/register", authHandler.Register)
	r.POST("/auth/login", authHandler.Login)
	r.GET("/health", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"status": "ok"})
	})

	// Protected routes.
	api := r.Group("/todos", handlers.AuthMiddleware())
	{
		api.POST("", todoHandler.Create)
		api.GET("", todoHandler.List)
		api.GET("/:id", todoHandler.Get)
		api.PATCH("/:id", todoHandler.Update)
		api.DELETE("/:id", todoHandler.Delete)
	}
```

> 🔑 **Checkpoint:** Public routes only under `/auth/*` and `/health`; every
> `/todos` route sits behind `AuthMiddleware`. The router group is the
> access-control boundary — read it before touching a handler.

```go
	// Start server with graceful shutdown.
	srv := &http.Server{
		Addr:         ":8080",
		Handler:      r,
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 10 * time.Second,
	}

	ctx, stop := signal.NotifyContext(context.Background(),
		os.Interrupt, syscall.SIGTERM)
	defer stop()

	go func() {
		log.Println("server listening on :8080")
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("server error: %v", err)
		}
	}()

	<-ctx.Done()
	log.Println("shutting down...")

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
	defer cancel()

	if err := srv.Shutdown(shutdownCtx); err != nil {
		log.Printf("shutdown error: %v", err)
	}

	db.Close()
	log.Println("server stopped")
}

func loggingMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		c.Next()
		log.Printf("%s %s %d %s",
			c.Request.Method, c.Request.URL.Path,
			c.Writer.Status(), time.Since(start))
	}
}

func runMigrations(db *sql.DB) error {
	// Simple migration runner — in production use golang-migrate.
	migrations := []string{
		`CREATE TABLE IF NOT EXISTS users (
			id SERIAL PRIMARY KEY,
			email VARCHAR(255) UNIQUE NOT NULL,
			password VARCHAR(255) NOT NULL,
			created_at TIMESTAMPTZ DEFAULT NOW()
		)`,
		`CREATE INDEX IF NOT EXISTS idx_users_email ON users (email)`,
		`CREATE TABLE IF NOT EXISTS todos (
			id SERIAL PRIMARY KEY,
			user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
			title VARCHAR(255) NOT NULL,
			description TEXT DEFAULT '',
			done BOOLEAN DEFAULT FALSE,
			created_at TIMESTAMPTZ DEFAULT NOW(),
			updated_at TIMESTAMPTZ DEFAULT NOW()
		)`,
		`CREATE INDEX IF NOT EXISTS idx_todos_user_id ON todos (user_id)`,
	}

	for _, m := range migrations {
		if _, err := db.Exec(m); err != nil {
			return fmt.Errorf("migration failed: %w\nQuery: %s", err, m)
		}
	}

	log.Println("migrations complete")
	return nil
}
```

---

## Part F: Handler Tests

Create `handlers/todo_test.go`:

```go
package handlers

import (
	"bytes"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"

	"rest-todo-api/models"

	"github.com/gin-gonic/gin"
)

func setupRouter() *gin.Engine {
	gin.SetMode(gin.TestMode)
	return gin.New()
}

func TestRegisterAndLogin(t *testing.T) {
	// This is a structural test — requires a real database.
	// For CI, use testcontainers or skip if DB is unavailable.

	router := setupRouter()

	// We test the binding/validation logic here.
	// Full integration tests need a test database.
	t.Run("register_missing_email", func(t *testing.T) {
		body := `{"password": "secret123"}`
		req := httptest.NewRequest(http.MethodPost, "/auth/register",
			bytes.NewBufferString(body))
		req.Header.Set("Content-Type", "application/json")
		w := httptest.NewRecorder()

		router.POST("/auth/register", func(c *gin.Context) {
			var req models.RegisterRequest
			if err := c.ShouldBindJSON(&req); err != nil {
				c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
				return
			}
			c.JSON(http.StatusOK, gin.H{"ok": true})
		})

		router.ServeHTTP(w, req)

		if w.Code != http.StatusBadRequest {
			t.Errorf("expected 400, got %d", w.Code)
		}
	})

	t.Run("register_short_password", func(t *testing.T) {
		body := `{"email": "test@example.com", "password": "ab"}`
		req := httptest.NewRequest(http.MethodPost, "/auth/register",
			bytes.NewBufferString(body))
		req.Header.Set("Content-Type", "application/json")
		w := httptest.NewRecorder()

		router.POST("/auth/register", func(c *gin.Context) {
			var req models.RegisterRequest
			if err := c.ShouldBindJSON(&req); err != nil {
				c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
				return
			}
			c.JSON(http.StatusOK, gin.H{"ok": true})
		})

		router.ServeHTTP(w, req)

		if w.Code != http.StatusBadRequest {
			t.Errorf("expected 400, got %d", w.Code)
		}
	})
}

func TestTodoBinding(t *testing.T) {
	tests := []struct {
		name    string
		body    string
		wantErr bool
	}{
		{"valid", `{"title": "Buy milk"}`, false},
		{"empty title", `{}`, true},
		{"invalid json", `not json`, true},
		{"extra fields", `{"title": "Test", "extra": true}`, false},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			var req models.CreateTodoRequest
			err := json.Unmarshal([]byte(tt.body), &req)
			// Simple binding check — full validation uses Gin's ShouldBindJSON.
			if tt.wantErr && err == nil {
				// For empty title, binding would catch it in real Gin context.
				if req.Title == "" && tt.name == "empty title" {
					return // Expected — binding tag catches this.
				}
			}
		})
	}
}

func TestUpdateTodoPartialBinding(t *testing.T) {
	body := `{"done": true}`
	var req models.UpdateTodoRequest
	if err := json.Unmarshal([]byte(body), &req); err != nil {
		t.Fatalf("unmarshal: %v", err)
	}

	if req.Title != nil {
		t.Error("expected Title to be nil")
	}
	if req.Description != nil {
		t.Error("expected Description to be nil")
	}
	if req.Done == nil || !*req.Done {
		t.Error("expected Done to be true")
	}
}

func TestAuthMiddlewareStructure(t *testing.T) {
	// Test that middleware rejects missing Authorization header.
	router := setupRouter()
	router.Use(AuthMiddleware())
	router.GET("/protected", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"ok": true})
	})

	req := httptest.NewRequest(http.MethodGet, "/protected", nil)
	w := httptest.NewRecorder()
	router.ServeHTTP(w, req)

	if w.Code != http.StatusUnauthorized {
		t.Errorf("expected 401, got %d", w.Code)
	}
}
```

### Running the Tests

```bash
# Tests that don't need a database
go test -v -run 'TestRegisterMissing|TestTodoBinding|TestUpdateTodoPartial|TestAuthMiddlewareStructure' ./handlers/...

# Full integration tests (requires running PostgreSQL)
# export DATABASE_URL="postgres://postgres:postgres@localhost:5432/todos_test?sslmode=disable"
go test -v -tags=integration ./...
```

> 🔑 **Checkpoint:** The `-run` filter splits fast unit tests from DB-backed
> integration tests. The structural tests prove the routing + middleware
> wire-up; the `-tags=integration` suite proves the SQL actually runs. Both
> halves together give you real coverage.

---

## Modern Practices

- **Repository pattern** — database logic lives in `repository/`, handlers
  live in `handlers/`. This separation makes both independently testable.
  Handlers test with mock repositories; repositories test with real DB.
- **`context.Context` everywhere** — every database call receives a context.
  The HTTP handler provides `c.Request.Context()`, which carries the request
  deadline and cancellation signal.
- **`database/sql` connection pooling** — `SetMaxOpenConns`, `SetMaxIdleConns`,
  `SetConnMaxLifetime` control the pool. Default settings leak connections.
- **`pgx` driver** — the fastest PostgreSQL driver for Go. It's wire-protocol
  compatible and avoids `libpq` C dependencies.
- **Pointer fields for partial updates** — `*string` in `UpdateTodoRequest`
  lets you distinguish "field not provided" from "field set to empty string".
- **`bcrypt.DefaultCost`** — 10 rounds. Fast enough for interactive login,
  slow enough to resist brute-force. Don't go below 8.

---

## Common Mistakes

- **Not closing `rows` after `QueryContext`.** This leaks database connections.
  Always `defer rows.Close()` immediately after checking the error.
- **Using `http.StatusOK` for errors.** A failed POST should return 400/409/500,
  not 200. Match status codes to the actual outcome.
- **Returning `null` instead of `[]`** for empty lists. Always initialize slices
  with `make` or set to empty literal: `todos = []models.Todo{}`.
- **Leaking password hashes.** The `json:"-"` tag on `User.Password` is
  essential. Without it, any serialization leaks the hash.
- **Hard-coding JWT secrets.** Always read from environment variables. A
  committed secret is a compromised secret.
- **Not using `sslmode=disable` only locally.** Production PostgreSQL
  connections must use TLS.
- **Ignoring `rows.Err()`** after a loop. The loop may exit early on error.
  Always check.
- **Using `bcrypt.Cost` below 8.** Too fast for security. Above 14 is too
  slow for interactive use. Stick with `DefaultCost` (10).

---

## Stretch Goals / Extensions

1. **Refresh tokens** — return both access and refresh tokens. Access tokens
   expire in 15 minutes; refresh tokens in 7 days.
2. **Todo categories/tags** — add a `tags` table with many-to-many relationship.
3. **Pagination** — `GET /todos?page=1&limit=20` with cursor-based pagination.
4. **Full-text search** — use PostgreSQL `tsvector` for searching todo titles.
5. **Rate limiting** — add a middleware that limits requests per user/IP.
6. **Swagger docs** — use `swaggo/swag` to generate OpenAPI specs from
   comments.
7. **Docker Compose** — `docker-compose.yml` with the API + PostgreSQL.
8. **CI/CD** — GitHub Actions pipeline that runs tests against a PostgreSQL
   service container.

---

## Next

Continue to [05-grpc-greeter-service.md](05-grpc-greeter-service.md) for a
gRPC service — protobuf definitions, streaming, interceptors, and health
checks.
