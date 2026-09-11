# Code Generation: protoc, buf, and Generated Code

Writing `.proto` files is the easy part. Turning them into Go code is where most people waste an afternoon. This article shows you two approaches: the traditional `protoc` route (which you need to understand) and `buf` (which you should actually use).

---

## The protoc Route

`protoc` is the protobuf compiler. It parses `.proto` files and hands them to plugins that generate code. For Go, you need two plugins:

1. **protoc-gen-go** -- generates message types (`*.pb.go`)
2. **protoc-gen-go-grpc** -- generates gRPC client/server code (`*_grpc.pb.go`)

### Installation

```bash
# Install protoc (the compiler)
# macOS
brew install protobuf

# Linux
sudo apt install -y protobuf-compiler

# Verify
protoc --version
# libprotoc 25.1

# Install Go plugins
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Make sure $GOPATH/bin is in your PATH
export PATH="$PATH:$(go env GOPATH)/bin"
```

### Directory Structure

```
myproject/
├── proto/
│   └── users/
│       └── v1/
│           └── user.proto
├── gen/
│   └── users/
│       └── v1/
│           ├── user.pb.go
│           └── user_grpc.pb.go
├── go.mod
└── go.sum
```

### The Proto File

```protobuf
// proto/users/v1/user.proto
syntax = "proto3";

package users.v1;

option go_package = "myproject/gen/users/v1;userv1";

message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
}

service UserService {
    rpc GetUser (GetUserRequest) returns (User);
}

message GetUserRequest {
    int64 id = 1;
}
```

The `option go_package` line is **mandatory** for Go code generation. It specifies:
- The import path: `myproject/gen/users/v1`
- The Go package alias: `userv1` (optional, after the semicolon)

### Running protoc

```bash
protoc \
    --proto_path=proto \
    --go_out=gen \
    --go_opt=paths=source_relative \
    --go-grpc_out=gen \
    --go-grpc_opt=paths=source_relative \
    users/v1/user.proto
```

| Flag | Purpose |
|------|---------|
| `--proto_path=proto` | Where to find `.proto` files (the import root) |
| `--go_out=gen` | Output directory for message types |
| `--go_opt=paths=source_relative` | Output path matches input path relative to `--proto_path` |
| `--go-grpc_out=gen` | Output directory for gRPC code |
| `--go-grpc_opt=paths=source_relative` | Same path strategy |

With `paths=source_relative`, `users/v1/user.proto` produces `gen/users/v1/user.pb.go` and `gen/users/v1/user_grpc.pb.go`.

Without this option, the default is `paths=import`, which uses the proto package path -- usually not what you want.

> ⚠️ **Gotcha:** Forget `paths=source_relative` and your generated `.go` files silently land in a nested directory that mirrors the proto *package path*, not your `--proto_path` layout. Import paths break, and debugging "where did my code go?" eats an afternoon.

---

## buf: The Modern Alternative

`buf` is to protobuf what `go` is to Go development. It replaces `protoc` with a simpler CLI that handles dependency management, linting, breaking change detection, and code generation.

### Installation

```bash
# macOS
brew install bufbuild/buf/buf

# Or via Go
go install github.com/bufbuild/buf/cmd/buf@latest

# Verify
buf --version
```

### Project Configuration

Create `buf.yaml` at your repository root:

```yaml
# buf.yaml
version: v2
modules:
  - path: proto
deps:
  - buf.build/googleapis/googleapis
  - buf.build/grpc-ecosystem/grpc-gateway
lint:
  use:
    - STANDARD
  except:
    - UNARY_RPC
    - PACKAGE_VERSION_SUFFIX
breaking:
  use:
    - FILE
```

Create `buf.gen.yaml` for code generation:

```yaml
# buf.gen.yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen
    opt: paths=source_relative
  - remote: buf.build/grpc/go
    out: gen
    opt: paths=source_relative
```

### buf Workflow

```bash
# Lint your proto files
buf lint

# Check for breaking changes against main branch
buf breaking --against "git#branch=main"

# Generate code
buf generate
```

That is it. No `protoc` flags. No plugin installation. No PATH issues. `buf` handles dependency fetching (the `deps` section), plugin management (the `plugins` section), and output paths.

> 🔑 **Key idea:** `buf` is to protobuf what the `go` toolchain is to Go -- one CLI that owns dependency resolution, linting, breaking-change checks, and code generation, so a fresh teammate needs nothing but `buf generate`.

### buf Lint Rules

`buf lint` catches common mistakes:

```bash
$ buf lint
proto/users/v1/user.proto:5:1: Field name "user_id" should be lower_snake_case.
proto/users/v1/user.proto:8:1: Message name "getUserRequest" should be PascalCase.
```

Key lint rules from the STANDARD rule set:

- Package names must be lowercase
- Field names must be lower_snake_case
- Message/enum names must be PascalCase
- Enums must have a 0 value suffixed with `_UNSPECIFIED`
- Comments must exist on public API elements
- Imports must be used

---

## Generated Code Walkthrough

Let us look at what `protoc-gen-go` and `protoc-gen-go-grpc` actually produce from this proto:

```protobuf
syntax = "proto3";
package users.v1;
option go_package = "myproject/gen/users/v1;userv1";

message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
}

service UserService {
    rpc GetUser (GetUserRequest) returns (User);
    rpc ListUsers (ListUsersRequest) returns (ListUsersResponse);
}

message GetUserRequest {
    int64 id = 1;
}

message ListUsersRequest {
    int32 page_size = 1;
    string page_token = 2;
}

message ListUsersResponse {
    repeated User users = 1;
    string next_page_token = 2;
}
```

### Message Types (user.pb.go)

```go
// Code generated by protoc-gen-go. DO NOT EDIT.

type User struct {
    state         protoimpl.MessageState
    sizeCache     protoimpl.SizeCache
    unknownFields protoimpl.UnknownFields

    Id    int64  `protobuf:"varint,1,opt,name=id,proto3" json:"id,omitempty"`
    Name  string `protobuf:"bytes,2,opt,name=name,proto3" json:"name,omitempty"`
    Email string `protobuf:"bytes,3,opt,name=email,proto3" json:"email,omitempty"`
}

func (x *User) GetId() int64 {
    if x != nil {
        return x.Id
    }
    return 0
}

func (x *User) GetName() string {
    if x != nil {
        return x.Name
    }
    return ""
}

func (x *User) GetEmail() string {
    if x != nil {
        return x.Email
    }
    return ""
}

// ProtoReflect returns a protoreflect.Message for this type.
func (x *User) ProtoReflect() protoreflect.Message { ... }

// Reset resets the message.
func (x *User) Reset() { *x = User{} }

// String returns the protobuf representation as a string.
func (x *User) String() string { return protoimpl.X.MessageStringOf(x) }

// ProtoMessage marks this as a protobuf message.
func (x *User) ProtoMessage() {}
```

Key things to notice:

1. **Getter methods** -- `GetId()`, `GetName()`, `GetEmail()` are generated for every field. They return zero values if the message is nil, so you never need nil checks on getters.
2. **Struct tags** -- the `protobuf` tags control binary serialization; the `json` tags provide JSON compatibility.
3. **ProtoReflect** -- the modern protobuf API for reflection.
4. **State/cache fields** -- `state`, `sizeCache`, `unknownFields` are internal bookkeeping.

### gRPC Interfaces (user_grpc.pb.go)

```go
// Code generated by protoc-gen-go-grpc. DO NOT EDIT.

// UserServiceClient is the client API for UserService.
type UserServiceClient interface {
    GetUser(ctx context.Context, in *GetUserRequest, opts ...grpc.CallOption) (*User, error)
    ListUsers(ctx context.Context, in *ListUsersRequest, opts ...grpc.CallOption) (*ListUsersResponse, error)
}

// UserServiceServer is the server API for UserService.
type UserServiceServer interface {
    GetUser(context.Context, *GetUserRequest) (*User, error)
    ListUsers(context.Context, *ListUsersRequest) (*ListUsersResponse, error)
}

// UnimplementedUserServiceServer should be embedded to have forward compatible implementations.
type UnimplementedUserServiceServer struct{}

func (UnimplementedUserServiceServer) GetUser(context.Context, *GetUserRequest) (*User, error) {
    return nil, status.Errorf(codes.Unimplemented, "method GetUser not implemented")
}

func (UnimplementedUserServiceServer) ListUsers(context.Context, *ListUsersRequest) (*ListUsersResponse, error) {
    return nil, status.Errorf(codes.Unimplemented, "method ListUsers not implemented")
}

// UnsafeUserServiceServer may be embedded to opt out of forward compatibility.
type UnsafeUserServiceServer interface {
    mustEmbedUnimplementedUserServiceServer()
}

// RegisterUserServiceServer registers the server.
func RegisterUserServiceServer(s grpc.ServiceRegistrar, srv UserServiceServer) {
    s.RegisterService(&UserService_ServiceDesc, srv)
}

// NewUserServiceClient creates a new client.
func NewUserServiceClient(cc grpc.ClientConnInterface) UserServiceClient {
    return &userServiceClient{cc}
}

type userServiceClient struct {
    cc grpc.ClientConnInterface
}

func (c *userServiceClient) GetUser(ctx context.Context, in *GetUserRequest, opts ...grpc.CallOption) (*User, error) {
    out := new(User)
    err := c.cc.Invoke(ctx, "/users.v1.UserService/GetUser", in, out, opts...)
    if err != nil {
        return nil, err
    }
    return out, nil
}
```

Key observations:

1. **Interface-based** -- both client and server are interfaces. You implement the server interface; the client uses the generated client struct.
2. **UnimplementedUserServiceServer** -- embed this in your server implementation so new methods return "not implemented" by default. This lets you add RPCs without breaking existing server implementations.
3. **Invoke path** -- the full method name is `/{package}.{Service}/{Method}` -- this is the HTTP/2 path gRPC uses.
4. **Client options** -- `opts ...grpc.CallOption` lets you set deadlines, metadata, and more per-call.

> 🧠 **Think of it as:** `UnimplementedUserServiceServer` is a contract safety net. Embed it and adding a new RPC to the proto degrades gracefully to a "not implemented" status instead of failing your whole server build or panic-ing at runtime.

---

## Package Layout in Go

### The `option go_package` Deep Dive

```
option go_package = "myproject/gen/users/v1;userv1";
```

This tells `protoc-gen-go`:
1. Put the generated code in package `userv1`
2. The full import path is `myproject/gen/users/v1`

In your Go code:
```go
import (
    pb "myproject/gen/users/v1;userv1"
)
```

### Multiple Proto Packages

When your project has multiple proto packages, keep them in separate Go packages:

```
proto/
├── users/v1/user.proto      -> gen/users/v1/user.pb.go      (package userv1)
├── orders/v1/order.proto    -> gen/orders/v1/order.pb.go     (package orderv1)
└── shared/v1/errors.proto   -> gen/shared/v1/errors.pb.go    (package sharedv1)
```

Each proto package maps to exactly one Go package. This keeps generated code isolated and avoids naming conflicts.

> 💡 **Pro tip:** Keep the mapping 1 proto package -> 1 Go package and mirror your repo layout in both import paths. It makes "which .proto does this code come from" answerable at a glance -- and keeps `buf`'s import graph consistent with your dependency tree.

### The `internal` Pattern

Put generated code behind `internal/` to prevent external consumers from depending on it:

```go
// In buf.gen.yaml
plugins:
  - remote: buf.build/protocolbuffers/go
    out: internal/gen
    opt: paths=source_relative
```

Then re-export types you want public:
```go
// pkg/users/types.go
package users

import gen "myproject/internal/gen/users/v1"

type User = gen.User  // type alias
```

---

## Dependencies Between Proto Files

When one proto file imports another, the generated Go code uses the corresponding Go package:

```protobuf
// proto/orders/v1/order.proto
import "users/v1/user.proto";

message Order {
    int64 id = 1;
    users.v1.User customer = 2;  // cross-package reference
}
```

The generated Go:
```go
type Order struct {
    // ...
    Customer *userv1.User `protobuf:"bytes,2,opt,name=customer,proto3" json:"customer,omitempty"`
}
```

With `buf`, the `deps` section in `buf.yaml` handles fetching external dependencies. With `protoc`, you must manually specify import paths and download dependencies.

> ⚠️ **Watch out:** With raw `protoc`, cross-file imports are your job -- you hand-craft `--proto_path` flags for every dependency and pray they stay in sync. This is precisely the version-mismatch / missing-import pain `buf` was built to remove.

---

## Modern Practices

- **Use `buf` instead of raw `protoc`**. It eliminates plugin installation, handles dependency management, and provides linting and breaking change detection in one tool.
- **Commit generated code**. Consumers should be able to `go get` your package without installing `protoc` or `buf`. Regenerate only when `.proto` files change.
- **Use the `internal/` pattern** for generated code. Re-export only the types that should be public. This prevents external packages from coupling to generated internals.
- **Version your proto packages** (`v1`, `v2`) from day one. It is much easier to add a version later than to migrate all consumers.
- **Use `buf.gen.yaml` with remote plugins** (`buf.build/protocolbuffers/go`, `buf.build/grpc/go`) to avoid managing plugin binaries locally.
- **Set up CI to run `buf lint` and `buf breaking`** on every pull request that touches `.proto` files.

---

## Common Mistakes

| Mistake | Why It Hurts | Fix |
|---|---|---|
| Omitting `option go_package` | `protoc-gen-go` errors out | Always include it |
| Not committing generated code | Consumers must install `protoc` | Commit `gen/` directory |
| Using `protoc` when `buf` is available | Manual plugin management, no linting | Switch to `buf` |
| Regenerating on every build | Merge conflicts, unnecessary complexity | Regenerate only on proto changes |
| Not using `internal/` for generated code | External packages couple to generated types | Re-export via type aliases |
| Ignoring lint rules | Inconsistent naming across teams | Run `buf lint` in CI |

---

## Exercises

1. Install `buf` and run `buf lint` on a new proto file with intentional style violations (wrong casing, missing comments, missing unspecified enum value). Fix each lint error.

2. Create a proto file that imports `google/protobuf/timestamp.proto` and uses `google.protobuf.Timestamp` in a message. Generate Go code and verify that the timestamp field has the correct Go type (`*timestamppb.Timestamp`).

3. Create two proto files where one imports the other. Generate code and write a small Go program that constructs a message from one package and embeds it in a message from the other package.

---

## Key Takeaways

- `protoc` with `protoc-gen-go` and `protoc-gen-go-grpc` is the traditional approach
- `buf` replaces protoc with simpler configuration, linting, and dependency management
- Generated code includes message types with getters, and gRPC client/server interfaces
- Always embed `UnimplementedXxxServer` in your server implementations
- Commit generated code -- consumers should not need `protoc` installed
- The `option go_package` is mandatory for Go code generation

---

## Next

[gRPC Clients and Servers in Go](04-grpc-servers-clients.md) -- server setup, client setup, status codes, deadlines, metadata, and cancellation.
