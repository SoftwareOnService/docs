# Protocol Buffers in Depth

Protocol Buffers are not just "binary JSON." They are a schema-first serialization format with a compact binary encoding, forward and backward compatibility rules, and code generation. This article covers everything beyond the basics: field numbering, wire types, oneofs, maps, imports, and how to evolve your schemas without breaking existing clients.

---

## The Schema: Your Source of Truth

A `.proto` file defines your data structures and services. Here is a complete example:

```protobuf
syntax = "proto3";

package orders;

import "google/protobuf/timestamp.proto";

// An order in the system.
message Order {
    int64 id = 1;
    string customer_id = 2;
    repeated OrderItem items = 3;
    OrderStatus status = 4;
    google.protobuf.Timestamp created_at = 5;
    oneof payment {
        CreditCard credit_card = 6;
        BankTransfer bank_transfer = 7;
    }
    map<string, string> metadata = 8;
}

message OrderItem {
    int64 product_id = 1;
    int32 quantity = 2;
    int64 price_cents = 3;
}

enum OrderStatus {
    ORDER_STATUS_UNSPECIFIED = 0;
    ORDER_STATUS_PENDING = 1;
    ORDER_STATUS_CONFIRMED = 2;
    ORDER_STATUS_SHIPPED = 3;
    ORDER_STATUS_DELIVERED = 4;
    ORDER_STATUS_CANCELLED = 5;
}

message CreditCard {
    string last_four = 1;
    string token = 2;
}

message BankTransfer {
    string routing_number = 1;
    string account_number = 2;
}
```

---

## Field Numbers: The Real Field Names

In JSON, you serialize by field name. In protobuf, you serialize by **field number**. The names are for human readability only.

> 🔑 **Key idea:** Field names are just documentation on the wire. The field number is the real identifier -- it is what gets encoded in every message, so it is what proto clients and servers actually agree on.

```protobuf
message User {
    int32 id = 1;      // field number 1
    string name = 2;    // field number 2
    string email = 3;   // field number 3
}
```

On the wire, a field is encoded as:

```
[field_number << 3 | wire_type] [value]
```

The `<< 3` is a bit shift that leaves room for the wire type in the lowest 3 bits.

### Wire Types

| Wire Type | Meaning | Protobuf Types |
|-----------|---------|----------------|
| 0 | Varint (variable-length integer) | int32, int64, bool, enum |
| 1 | 64-bit fixed | fixed64, double |
| 2 | Length-delimited | string, bytes, embedded messages, packed repeated |
| 5 | 32-bit fixed | fixed32, float |

### Why This Matters

Field numbers are **permanent**. Once you assign `name = 2`, you cannot reuse number 2 even if you remove `name`. This is how protobuf achieves forward and backward compatibility.

```protobuf
message User {
    int32 id = 1;
    // string name = 2;  DO NOT reuse this number
    string email = 3;
    string display_name = 4;  // new field, new number
}
```

Old clients that do not know about `display_name` (field 4) will ignore it. New clients that receive old messages without `display_name` will use the zero value. Both sides keep working.

> ⚠️ **Gotcha:** This forward/backward compatibility works **only if you never reuse a field number**. Once `name = 2` is in the wild, that number is tied to that field forever. Reusing it makes old clients decode the new field as the old one -- silent, nasty data corruption.

---

## Default Values and Zero Values

In protobuf, there is no distinction between "field not present" and "field has zero value":

```protobuf
message User {
    int32 age = 1;
    string name = 2;
    bool active = 3;
}
```

A message `User{age: 0, name: "", active: false}` is identical on the wire to an empty message. You cannot tell whether the sender explicitly set `age = 0` or omitted it.

If you need to distinguish "not set" from "zero," use the `optional` keyword (proto3 optional):

```protobuf
message User {
    optional int32 age = 1;
    optional string name = 2;
}
```

With `optional`, the generated Go code uses `*int32` and `*string` pointers. A nil pointer means "not set." This is the correct approach when zero is a meaningful value.

> 💡 **Pro tip:** The "missing field vs. zero value" decision is really "does `0` mean something real?" If `age: 0` is a meaningful age you must distinguish from "no age provided," use `optional`. Otherwise the extra pointer indirection and nil checks are wasted complexity.

---

## Enums

Enums map names to integer values. The first value **must** be 0 -- this is the default value and represents "unspecified":

```protobuf
enum Priority {
    PRIORITY_UNSPECIFIED = 0;
    PRIORITY_LOW = 1;
    PRIORITY_MEDIUM = 2;
    PRIORITY_HIGH = 3;
}
```

### The Unspecified Rule

The `UNSPECIFIED = 0` convention is not optional -- it is a safety requirement. If a client sends a message without setting this field (or uses an old version of your proto), the value will be 0. Without an `_UNSPECIFIED` case, you would silently treat unknown input as a valid enum value.

> 🧠 **Think of it as:** enum value 0 is the "silent default" -- it is what arrives when nothing was sent. The `_UNSPECIFIED` case is your safety net naming that silence explicitly, so unknown data never masquerades as a real state.

### Enum Restrictions

Protobuf enums are **open** by default. A client can send an integer that does not map to any defined name. Your server must handle unknown enum values:

```go
// The generated enum type includes an "unrecognized" case
switch order.Status {
case pb.OrderStatus_ORDER_STATUS_PENDING:
    // handle pending
case pb.OrderStatus_ORDER_STATUS_CONFIRMED:
    // handle confirmed
default:
    // includes unknown values -- do NOT ignore
}
```

---

## Oneofs: Sum Types

A `oneof` field means exactly one of the listed fields can be set. This is protobuf's version of a tagged union:

```protobuf
message Payment {
    oneof method {
        CreditCard credit_card = 1;
        BankTransfer bank_transfer = 2;
        string promo_code = 3;
    }
}
```

In Go, this generates a struct with a method interface and wrapper types:

```go
// Access the oneof
switch p := payment.Method.(type) {
case *pb.Payment_CreditCard:
    fmt.Println("Credit card:", p.CreditCard.LastFour)
case *pb.Payment_BankTransfer:
    fmt.Println("Bank transfer:", p.BankTransfer.RoutingNumber)
case *pb.Payment_PromoCode:
    fmt.Println("Promo code:", p.PromoCode)
}
```

**Rule**: you cannot combine `oneof` with `repeated` or `map`. If you need a list of arbitrary payment methods, model it differently.

---

## Maps

Maps are syntactic sugar for repeated key-value pairs:

```protobuf
message Config {
    map<string, string> settings = 1;
    map<int32, string> labels = 2;
}
```

On the wire, a map is encoded as a repeated message:

```protobuf
message MapEntry {
    Key key = 1;
    Value value = 2;
}
repeated MapEntry field = N;
```

**Important limitations**:
- Keys must be integral types or strings (not floats, bytes, or messages)
- Maps cannot use `repeated` keys
- Map ordering is not guaranteed
- Maps are not the same as repeated messages -- they have different semantics and wire format

---

## Repeated Fields

Use `repeated` for ordered lists or "In Protocol Buffers, the `repeated` keyword is simply the Protobuf equivalent of an **array** or a **list**.":

```protobuf
message ShoppingCart {
    repeated CartItem items = 1;
}
```

By default, scalar types in repeated fields use **packed encoding** (all values in a single length-delimited block). This is more efficient than encoding each value separately.

---

## Nested Messages

Messages can contain other messages, defined inline or as separate types:

```protobuf
message Address {
    message GeoCoordinates {
        double latitude = 1;
        double longitude = 2;
    }

    string street = 1;
    string city = 2;
    GeoCoordinates coordinates = 3;
}
```

In Go, nested messages become types in the same package: `pb.Address_GeoCoordinates`.

---

## Imports and Package Organization

Proto files can import other proto files:

```protobuf
syntax = "proto3";

import "google/protobuf/timestamp.proto";
import "google/protobuf/field_mask.proto";
import "orders/order.proto";

message Shipment {
    int64 id = 1;
    orders.Order order = 2;
    google.protobuf.Timestamp shipped_at = 3;
    google.protobuf.FieldMask update_mask = 4;
}
```

### Well-Known Types

Protobuf includes well-known types in `google/protobuf/`:

| Type | Purpose |
|------|---------|
| `Timestamp` | Points in time (seconds + nanos) |
| `Duration` | Lengths of time |
| `FieldMask` | Partial updates (which fields to update) |
| `Any` | Wrapper for arbitrary proto messages |
| `Struct` | Dynamic JSON-like structures |
| `Wrappers` | Optional wrappers (DoubleValue, StringValue, etc.) |

These are available by importing them and are generated into your Go package as `timestamppb`, `durationpb`, `fieldmaskpb`, etc.

### Best Practice: Import Paths

Use import paths that match your repository structure:

```
# File: proto/orders/v1/order.proto
# Import: proto/users/v1/user.proto
import "users/v1/user.proto";
```

This makes the dependency graph explicit and avoids circular imports.

---

## The FieldMask Pattern

When updating resources, you often need to specify which fields the client wants to change. `FieldMask` solves this:

```protobuf
import "google/protobuf/field_mask.proto";

service UserService {
    rpc UpdateUser (UpdateUserRequest) returns (User);
}

message UpdateUserRequest {
    User user = 1;
    google.protobuf.FieldMask update_mask = 2;
}
```

The client sends:
```go
req := &pb.UpdateUserRequest{
    User: &pb.User{
        Id:    42,
        Name:  "Alice",
        Email: "alice@new.com",
    },
    UpdateMask: &fieldmaskpb.FieldMask{
        Paths: []string{"email"},  // only update email
    },
}
```

The server uses the mask to apply only the specified fields. Without `FieldMask`, you either update everything (risky) or build custom logic to track changes.

---

## Schema Evolution Rules

These rules keep old and new versions of your proto files compatible:

1. **Never change field numbers** -- the binary format depends on them
2. **Never reuse field numbers** -- even if you remove the field
3. **New fields must have new numbers** -- adding fields is safe
4. **Removed fields should use `reserved`** -- prevents accidental reuse
5. **Renaming fields is safe** -- names are not in the binary format
6. **Changing a field's type is only safe in specific cases** -- see the compatibility matrix

### The Reserved Pattern

When you remove a field, reserve its number and name:

```protobuf
message User {
    reserved 2, 5, 8;  // field numbers
    reserved "name", "old_email", "legacy_id";  // field names

    int32 id = 1;
    string email = 3;
    string display_name = 4;
}
```

If someone accidentally reuses number 2 or the name "name", `protoc` will catch it.

> 💡 **Note:** Consider `reserved` the protobuf version of "do not touch" tape. Reserve both the old field numbers and the old field names -- one prevents wire-level collisions, the other keeps JSON/tooling from silently matching a placeholder.

### Type Compatibility

Some type changes are safe:

| From | To | Safe? |
|------|-----|-------|
| int32 | int64 | Yes (varint encoding) |
| int32 | uint32 | Yes |
| bool | int32 | Yes |
| string | bytes | Yes |
| float | double | Yes |
| repeated | repeated (packed) | Yes |

Changes that are **not safe**: changing between varint and fixed, changing between string and bytes (for non-UTF-8), changing the meaning of a field.

> 🧠 **Memory aid:** When in doubt about a type change, ask "does the wire encoding stay the same?" Two types share a wire representation -> safe to swap. Different wire types on the old and new sides? Old parsers will garble the bytes.

---

## Modern Practices

- **Prefer `optional` over map tricks** when you need to distinguish "not set" from zero value. Proto3 optional generates pointer types in Go that are nil when absent.
- **Always suffix enums with `_UNSPECIFIED = 0`**. The `buf lint` STANDARD rule enforces this -- adopt it as a team convention.
- **Use `FieldMask` for all update RPCs**. It is the standard way to implement partial updates and avoids the "send everything or nothing" dilemma.
- **Version your proto packages** (`v1`, `v2`) when making breaking changes. Run `buf breaking --against "git#branch=main"` to catch regressions.
- **Keep proto files in a dedicated `proto/` directory** with import paths matching your repository structure for clean dependency management.

---

## Common Mistakes

| Mistake | Why It Hurts | Fix |
|---|---|---|
| Reusing field numbers after removal | Old clients decode garbage | Always use `reserved` for removed fields |
| Starting enums at 1 instead of 0 | Unknown input silently becomes a valid value | Always `_UNSPECIFIED = 0` |
| Making every field `optional` | 4 pointer fields + nil checks everywhere | Only use when zero is meaningful |
| Using field names for compatibility | Misleading -- names are cosmetic | Rely on field numbers |
| Ignoring wire type changes | Int32-to-int64 is safe; int32-to-fixed32 is not | Check the compatibility matrix |

---

## Exercises

1. Define a `BlogPost` proto message with title, body, author, tags (repeated string), comments (repeated message with author + text + timestamp), and a status enum (draft, published, archived). Include at least one `oneof` for either a URL or file attachment.

2. Take an existing proto message and evolve it: add two new fields, remove one old field (using `reserved`), and change one field from `int32` to `int64`. Verify that `protoc` does not flag any breaking changes.

3. Define a `SearchRequest` with a `map<string, string>` for filters and a `repeated string` for sort fields. Write the proto and generate Go code (see [Code Generation](03-code-generation.md)). Verify the generated types in Go.

---

## Key Takeaways

- Field numbers, not names, are the permanent identifiers in protobuf
- Enums must start with `UNSPECIFIED = 0` for safety
- `oneof` gives you tagged unions -- exactly one field set at a time
- Maps encode as repeated key-value messages
- `FieldMask` is essential for partial updates
- Schema evolution follows strict rules: add new fields, reserve removed ones, never reuse numbers

---

## Next

[Code Generation: protoc, buf, and Generated Code](03-code-generation.md) -- turning `.proto` files into Go code, `buf` workflow, and generated code anatomy.
