---
title: Using Protocol Buffer `Any` Types in Go
date: 2024-07-30
tags:
    - "Protocol Buffers"
    - "Go"
ShowCodeCopyButtons: true
---

## Introduction

This article utilizes [Protocol Buffers](https://protobuf.dev/) and [GRPC](https://grpc.io/) in the context of creating a [key value store](https://en.wikipedia.org/wiki/Key%E2%80%93value_database) in Go.  Protocol Buffers are a great way to serialize message data for transmission and GRPC builds on Protocol Buffers to describe services and how the messages passed between them.  A simplistic put request for a key value store could be described with the following protocol buffer code:

```protobuf {lineNos=table}
message PutRequest {
    string key = 1;
    string value = 2;
}
```

The `PutRequest` has two fields, `key` and a `value`, which are assigned numbers for encoding the Protocol Buffer [wire format](https://protobuf.dev/programming-guides/encoding/).  Generating Go code will create a struct looking like this:

```go {lineNos=table}
type PutRequest struct {
    Key   string 
    Value string
}
```

The client code can easily access the `Value` field.  However, This PutRequest isn't ideal in the context of the key value store because it only stores strings.  Procol Buffers provide a field type called `Any` to allow a field to represent multiple types.  

## Using protobuf.Any

Using the `protobuf.Any` type requires changing the protocol buffer file:
* Setting the syntax to `proto3`
* Including `any.proto`
* Changing `value` field type to `google.protobuf.Any`

So now the protocol bufffer file looks like this:

```protobuf {lineNos=table}
syntax = "proto3";
import "google/protobuf/any.proto";

message PutRequest {
    string key = 1;
    google.protobuf.Any value = 2;
}
```

That was easy.  However, getting the value from the generated code is now more complicated.  The Value field now expects a pointer to a `anypb.Any` type.
```go {lineNos=table,hl_lines=[3]}
type PutRequest struct {
   Key   string
   Value *anypb.Any
}
```                                           

And the `anypb.Any` struct is defined like this:

```go {lineNos=table}
type struct Any {
    TypeUrl string 
    Value   []byte 
}
```

The `anypb.Any` struct contains two fields.  The `TypeUrl` field is a string containing the message type in the form of a url: `type.googleapis.com/google.protobuf.StringValue`.  The `Value` field is a slice of bytes. So the `TypeUrl` field indicates how to interpret the slice of bytes in `Value`. 

Working with the `anypb.Any` type requires two Go packages to be installed: [anypb](https://pkg.go.dev/google.golang.org/protobuf/types/known/anypb) and [wrapperspb](https://pkg.go.dev/google.golang.org/protobuf/types/known/wrapperspb).  The `anypb` package defines the `anypb.Any` struct and the methods for marshalling values through it.  The `wrapperspb` package provides structs for wrapping simple scalar types.

### Marshalling Values

Marshalling an `interface{}` value through an `anypb.Any` field requires 3 steps:
1. A type switch to determine which wrapper to use (line 3)
2. Creating the wrapper message (line 5)
3. Calling [`anypb.New()`](https://pkg.go.dev/google.golang.org/protobuf/types/known/anypb#New) on the wrapper message (line 12)

```go {lineNos=table,hl_lines=[3,5,12]}
func Marshal(val interface{}) (*anypb.Any, error) {
    var m proto.Message
    switch v := val.(type) {
    case string:
        m = wrapperspb.String(v)  // wrap the value 
    case float32:
        m = wrapperspb.Float(v)
    // other cases here

    return anypb.New(m), nil     // create an Any field
}
```

### Unmarshalling Values

Unmarshalling is also accomplished in 3 steps:
1. Calling [UnmarshalNew()](https://pkg.go.dev/google.golang.org/protobuf/types/known/anypb#UnmarshalNew) method on the value field (line 2)
2. Determine the type of the field in a type switch (line 8)
3. Calling [UnmarshalTo()](https://pkg.go.dev/google.golang.org/protobuf/types/known/anypb#UnmarshalTo) method (line 9)

```go {lineNos=table,hl_lines=[2,7,9]}
func (s *KVServer) Put(ctx context.Context, r *gen.PutRequest) (*gen.Response, error) {
    m, err := r.value.UnmarshalNew()
    // handle error

    switch m := m.(type) {
    case *wrapperspb.StringValue:
        err = r.value.UnmarshalTo(m)
        // handle error
        handleStringValues(m.GetValue())

    case *wrapperspb.Int64Value:
        err = r.value.UnmarshalTo(m)
        // handle error
        handleInt64Values(m.GetValue())

    // other cases here
    }
}
```

### Handling Array Types

The `wrapperspb` package doesn't provide any way to wrap complex or array types, but it's strait forward to support.  First, define a message with a [repeated](https://protobuf.dev/programming-guides/proto3/#field-labels) type:

```protobuf
message StringArray {
    repeated string value = 1;
}
```

Next, add a case to the type switch:

```go {lineNos=table,hl_lines=[7-10]}
switch m := m.(type) {
case *wrapperspb.StringValue:
    err = r.value.UnmarshalTo(m)
    // handle error
    handleStringValues(m.GetValue())

case *gen.StringArray:
    err = r.value.UnmarshalTo(m)
    // handle error
    handleStringArrayValues(m.GetValue())

// other cases here
}
```

## Conclusion

Using `anypb.Any` field type in Protocol Buffers is an easy way allow a Protocol Buffer message field to have multiple types.

* Marshalling and unmarshalling requires a wrapper messages. 
    * Primitive types can use messages from [wrapperspb](https://pkg.go.dev/google.golang.org/protobuf/types/known/wrapperspb)
    * Complex or array types need wrapped in custom messages

The complete protocol buffer file can be found here, and the Go code that marshals the types can be found here.

