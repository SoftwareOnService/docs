# gRPC Clients and Servers in Go

You have the proto definition and the generated code. Now you wire them up: a server that implements the service, a client that calls it, and all the production concerns in between -- deadlines, errors, metadata, and connection management.

---

## The Server

Here is a complete gRPC server for a user service:

```go
// server/main.go
package main

import (
    "context"
    "log"
    "net"
    "os"
    "os/signal"
    "syscall"

    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"

    pb "myproject/gen/users/v1"
)

type userServer struct {
    pb.UnimplementedUserServiceServer
    users map[int64]*pb.User
}

func newUserServer() *userServer {
    return &userServer{
        users: map[int64]*pb.User{
            1: {Id: 1, Name: "Alice", Email: "alice@example.com"},
            2: {Id: 2, Name: "Bob", Email: "bob@example.com"},
        },
    }
}

func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    user, ok := s.users[req.GetId()]
    if !ok {
        return nil, status.Errorf(codes.NotFound, "user %d not found", req.GetId())
    }
    return user, nil
}

func (s *userServer) ListUsers(ctx context.Context, req *pb.ListUsersRequest) (*pb.ListUsersResponse, error) {
    var result []*pb.User
    for _, u := range s.users {
        result = append(result, u)
    }
    return &pb.ListUsersResponse{Users: result}, nil
}

func main() {
    // Create a listener
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }

    // Create the gRPC server
    grpcServer := grpc.NewServer()

    // Register the service
    pb.RegisterUserServiceServer(grpcServer, newUserServer())

    // Graceful shutdown
    go func() {
        sigCh := make(chan os.Signal, 1)
        signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
        <-sigCh
        log.Println("shutting down gracefully...")
        grpcServer.GracefulStop()
    }()

    log.Println("server listening on :50051")
    if err := grpcServer.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
    }
}
```

Key details:

1. **`UnimplementedUserServiceServer`** -- embed this in your struct. When you add a new RPC to the proto, the server compiles with a "not implemented" default instead of breaking.

2. **`status.Errorf`** -- return gRPC status errors, not regular errors. The status code is what the client receives. A regular `errors.New` becomes `codes.Unknown`.

3. **`GracefulStop`** -- drain in-flight RPCs before closing. `Stop` kills them immediately.

> 🔑 **Key idea:** Your handler is not your design -- the generated interface is. You embed `UnimplementedUserServiceServer`, implement the RPC methods, and gRPC wires registration, transport, and dispatch around you. Graceful shutdown is the one part you must own: `GracefulStop` drains, `Stop` cuts.

---

## The Client

```go
// client/main.go
package main

import (
    "context"
    "log"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"

    pb "myproject/gen/users/v1"
)

func main() {
    // Connect to the server
    conn, err := grpc.NewClient("localhost:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        log.Fatalf("failed to connect: %v", err)
    }
    defer conn.Close()

    // Create a client
    client := pb.NewUserServiceClient(conn)

    // Make a call with a timeout
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    user, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 1})
    if err != nil {
        // Convert to gRPC status error
        st, ok := status.FromError(err)
        if ok {
            log.Printf("gRPC error: code=%s message=%s", st.Code(), st.Message())
        } else {
            log.Printf("non-gRPC error: %v", err)
        }
        return
    }

    log.Printf("User: %s <%s>", user.GetName(), user.GetEmail())
}
```

### Connection Management

`grpc.NewClient` returns a `*grpc.ClientConn`. This is **not** a single TCP connection -- it is a logical connection that manages a pool of HTTP/2 connections. You should:

- **Reuse connections** -- do not create a new `ClientConn` per request
- **Close when done** -- `defer conn.Close()` when the client shuts down
- **Share across goroutines** -- `ClientConn` is safe for concurrent use

```go
// BAD: new connection per call
func getUserBad(id int64) error {
    conn, _ := grpc.NewClient("localhost:50051", ...)
    defer conn.Close()
    client := pb.NewUserServiceClient(conn)
    _, err := client.GetUser(ctx, &pb.GetUserRequest{Id: id})
    return err
}

// GOOD: reuse connection
var userClient pb.UserServiceClient

func init() {
    conn, _ := grpc.NewClient("localhost:50051", ...)
    userClient = pb.NewUserServiceClient(conn)
}

func getUserGood(id int64) error {
    _, err := userClient.GetUser(ctx, &pb.GetUserRequest{Id: id})
    return err
}
```

> 💡 **Pro tip:** Treat `grpc.ClientConn` like a database connection pool -- create it once at startup, share it everywhere. It multiplexes many concurrent calls over HTTP/2, so a fresh connection per request throws away the whole point.

---

## Error Handling with Status Codes

gRPC defines its own set of status codes. Do not use HTTP status codes -- gRPC maps them internally.

> ⚠️ **Watch out:** Return a plain `errors.New` and the client receives `codes.Unknown` -- the caller loses all semantic meaning. Every handler error should go through `status.Errorf` with the most precise code you can pick.

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

// Server-side: returning errors
func (s *userServer) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.User, error) {
    if req.GetName() == "" {
        return nil, status.Error(codes.InvalidArgument, "name is required")
    }

    if _, exists := s.usersByEmail[req.GetEmail()]; exists {
        return nil, status.Errorf(codes.AlreadyExists, "user with email %s already exists", req.GetEmail())
    }

    user := &pb.User{
        Id:    s.nextID(),
        Name:  req.GetName(),
        Email: req.GetEmail(),
    }
    s.users[user.Id] = user
    return user, nil
}

// Client-side: handling errors
func createUser(client pb.UserServiceClient) {
    _, err := client CreateUser(ctx, &pb.CreateUserRequest{
        Name:  "Alice",
        Email: "alice@example.com",
    })
    if err != nil {
        st, ok := status.FromError(err)
        if !ok {
            // Not a gRPC error -- connection issue, etc.
            log.Fatalf("non-gRPC error: %v", err)
        }

        switch st.Code() {
        case codes.InvalidArgument:
            log.Printf("invalid input: %s", st.Message())
        case codes.AlreadyExists:
            log.Printf("user already exists: %s", st.Message())
        case codes.NotFound:
            log.Printf("not found: %s", st.Message())
        case codes.Unavailable:
            log.Printf("server unavailable (try again): %s", st.Message())
        case codes.DeadlineExceeded:
            log.Printf("request timed out: %s", st.Message())
        default:
            log.Printf("unexpected error: code=%s message=%s", st.Code(), st.Message())
        }
    }
}
```

### Common Status Codes

| Code | Meaning | When to Use |
|------|---------|-------------|
| `OK` | Success | Request succeeded |
| `InvalidArgument` | Bad input | Validation failure |
| `NotFound` | Resource missing | ID does not exist |
| `AlreadyExists` | Duplicate | Unique constraint violation |
| `PermissionDenied` | Authz failure | User lacks permission |
| `Unauthenticated` | Authn failure | Missing or invalid credentials |
| `Internal` | Server bug | Unexpected error |
| `Unavailable` | Server down | Transient, retry-safe |
| `DeadlineExceeded` | Timeout | Request took too long |
| `Canceled` | Client canceled | Context canceled |

### Rich Error Details

Status errors can carry structured details using `status.WithDetails`:

```go
import (
    "google.golang.org/genproto/googleapis/rpc/errdetails"
    "google.golang.org/grpc/status"
)

func (s *userServer) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.User, error) {
    var violations []*errdetails.BadRequest_FieldViolation

    if req.GetName() == "" {
        violations = append(violations, &errdetails.BadRequest_FieldViolation{
            Field:       "name",
            Description: "name is required",
        })
    }
    if req.GetEmail() == "" {
        violations = append(violations, &errdetails.BadRequest_FieldViolation{
            Field:       "email",
            Description: "email is required",
        })
    }

    if len(violations) > 0 {
        st := status.New(codes.InvalidArgument, "validation failed")
        st, _ = st.WithDetails(&errdetails.BadRequest{
            FieldViolations: violations,
        })
        return nil, st.Err()
    }

    // ... create user
}

// Client side: extracting details
func handleErr(err error) {
    st, _ := status.FromError(err)
    for _, detail := range st.Details() {
        switch d := detail.(type) {
        case *errdetails.BadRequest:
            for _, v := range d.FieldViolations {
                log.Printf("field %s: %s", v.Field, v.Description)
            }
        }
    }
}
```

---

## Deadlines and Timeouts

A deadline is an absolute time by which the RPC must complete. A timeout is a relative duration that becomes a deadline.

```go
// Timeout: 5 seconds from now
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
resp, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 1})

// Deadline: specific point in time
deadline := time.Now().Add(10 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()
resp, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 1})
```

**Every gRPC call should have a deadline.** Without one, the RPC can run indefinitely:

```go
// BAD: no deadline
ctx := context.Background()
resp, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 1})

// GOOD: deadline
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()
resp, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 1})
```

### Propagating Deadlines

Deadlines propagate through gRPC calls. If Service A calls Service B with a 5-second deadline, and Service B calls Service C, the deadline applies to the entire chain.

```go
// Service B: receives request from Service A (5-second deadline)
func (s *orderServer) CreateOrder(ctx context.Context, req *pb.CreateOrderRequest) (*pb.Order, error) {
    // Call to Service C with the same context
    // If Service A's deadline is in 3 seconds, this call has 3 seconds (not the original 5)
    user, err := s.userClient.GetUser(ctx, &pb.GetUserRequest{Id: req.GetUserId()})
    if err != nil {
        return nil, err  // deadline error propagates back to Service A
    }

    // Create the order...
    return order, nil
}
```

This is why `context.Background()` should almost never appear in server handlers -- always use the context from the gRPC handler.

> 🧠 **Think of it as:** the deadline is a shrinking hourglass passed from service to service. Each hop consumes time from the same budget, and the closest deadline wins the race. Swap in `context.Background()` and you smash the hourglass on your own floor.

---

## Metadata

Metadata is the gRPC equivalent of HTTP headers. Use it for auth tokens, request IDs, and tracing information.

### Sending Metadata (Client)

```go
import "google.golang.org/grpc/metadata"

// Attach metadata to a call
md := metadata.Pairs(
    "authorization", "Bearer "+token,
    "x-request-id", "abc-123",
)

// Create a new context with metadata
ctx := metadata.NewOutgoingContext(context.Background(), md)

resp, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 1})
```

### Reading Metadata (Server)

```go
func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Error(codes.Internal, "missing metadata")
    }

    // Get a single value
    authHeader := md.Get("authorization")
    if len(authHeader) == 0 {
        return nil, status.Error(codes.Unauthenticated, "missing authorization header")
    }

    // Get all values for a key (metadata allows multiple values)
    requestIDs := md.Get("x-request-id")

    // ... process request
}
```

### Forwarding Metadata

When you are a middle service that forwards calls, forward the incoming metadata:

```go
func (s *orderServer) CreateOrder(ctx context.Context, req *pb.CreateOrderRequest) (*pb.Order, error) {
    // Forward incoming metadata to the next service
    md, _ := metadata.FromIncomingContext(ctx)
    ctx = metadata.NewOutgoingContext(ctx, md)

    user, err := s.userClient.GetUser(ctx, &pb.GetUserRequest{Id: req.GetUserId()})
    // ...
}
```

### Binary Metadata

Binary metadata keys must end with `-bin` and values are base64 encoded:

```go
// Client: attach binary metadata (protobuf bytes)
traceData, _ := proto.Marshal(traceContext)
md := metadata.Pairs("grpc-trace-bin", string(traceData))

// Server: read binary metadata
vals := md.Get("grpc-trace-bin")
```

gRPC uses `grpc-trace-bin` internally for OpenTelemetry integration.

> 💡 **Note:** Metadata keys follow HTTP header conventions -- `-bin` suffix for binary values (base64 encoded on the wire). Text keys map to lowercase ASCII headers in HTTP/2, so treat them case-insensitively.

---

## Cancellation

When a client cancels a context, the server receives a cancellation signal:

```go
// Client: cancel a long-running call
ctx, cancel := context.WithCancel(context.Background())

go func() {
    // If the user clicks "cancel" or navigates away
    time.Sleep(2 * time.Second)
    cancel()
}()

resp, err := client.RunReport(ctx, &pb.RunReportRequest{Id: 1})
// err will be status.Error(codes.Canceled, "context canceled")
```

```go
// Server: respect cancellation
func (s *reportServer) RunReport(ctx context.Context, req *pb.RunReportRequest) (*pb.Report, error) {
    for i := 0; i < 100; i++ {
        // Check for cancellation between expensive operations
        select {
        case <-ctx.Done():
            return nil, status.Error(codes.Canceled, "report canceled")
        default:
        }

        // Do expensive work
        s.processChunk(i)
    }
    return report, nil
}
```

### Stream Cancellation

Cancellation is especially important for streaming RPCs. If the client disconnects, the server must stop producing data:

```go
func (s *streamServer) WatchEvents(req *pb.WatchRequest, stream pb.EventService_WatchEventsServer) error {
    for event := range s.eventChan {
        select {
        case <-stream.Context().Done():
            return nil  // client disconnected, stop sending
        default:
        }

        if err := stream.Send(event); err != nil {
            return err
        }
    }
    return nil
}
```

> 🔑 **Remember:** Check `stream.Context().Done()` (or `ctx.Done()`) wherever your handler loops. "Client disconnected" is not an error -- it is a normal end-of-life event, and respecting it promptly is the difference between a clean drain and a goroutine leak.

---

## Modern Practices

- **Always use `context.WithTimeout`** on every client call. Without deadlines, a stuck server can hold resources forever.
- **Never use `context.Background()` in server handlers**. Always derive from the incoming handler context to respect client deadlines and cancellation.
- **Reuse `grpc.ClientConn`** -- it is designed for concurrent use and manages a connection pool internally. Creating one per request defeats HTTP/2 multiplexing.
- **Use `status.Errorf` for all server errors**. A plain `errors.New` becomes `codes.Unknown` on the client, losing error semantics.
- **Use rich error details** (`status.WithDetails`) for validation errors. They provide structured, machine-readable error information.
- **Forward metadata** in service-to-service chains. Auth tokens and trace context must propagate across hops.

---

## Common Mistakes

| Mistake | Why It Hurts | Fix |
|---|---|---|
| No deadline on client calls | Stuck servers hold resources forever | Always use `context.WithTimeout` |
| `context.Background()` in handlers | Ignores client deadlines and cancellation | Use the handler's `ctx` parameter |
| New `ClientConn` per request | Prevents connection reuse and HTTP/2 multiplexing | Share and reuse connections |
| Returning `errors.New` from handlers | Client gets `codes.Unknown` | Use `status.Errorf` with appropriate codes |
| Ignoring errors from client calls | Nil pointer panics | Always check `err != nil` |
| Not forwarding metadata | Auth tokens and trace context lost across services | Use `metadata.NewOutgoingContext` with incoming md |

---

## Exercises

1. Build a complete gRPC server and client for a calculator service. Implement `Add(a, b) -> Result` and `Divide(a, b) -> Result`. Return `InvalidArgument` when dividing by zero. The client should handle both success and error cases.

2. Add metadata support to your calculator server. The client sends an `x-operator-id` metadata value, and the server logs it with every request. Verify it works with `grpcurl` or a test client.

3. Implement deadline propagation: Calculator server calls an internal "audit" service. Set a 2-second deadline on the client side. Verify that if the audit service takes 3 seconds, the entire chain returns `DeadlineExceeded`.

---

## Key Takeaways

- `grpc.NewClient` returns a reusable connection -- do not create one per request
- Always return `status.Errorf` with appropriate codes from server handlers
- Every gRPC call needs a deadline -- use `context.WithTimeout`
- Deadlines propagate automatically through service chains
- Metadata carries auth tokens, request IDs, and tracing data
- Check `ctx.Done()` in long-running server operations
- Embed `UnimplementedXxxServer` so new RPCs do not break your server

---

## Next

[gRPC Streaming](05-grpc-streaming.md) -- server, client, and bidirectional streaming, flow control, and stream errors.
