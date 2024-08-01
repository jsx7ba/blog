---
title: Marshalling Small Go Types with Protocol Buffers
date: 2024-07-30T18:04:43-04:00
tags:
    - "Protocol Buffers"
    - "Go"
    - "types"
ShowCodeCopyButtons: true
draft: true
---

## Introduction

Go supports a range of integer sizes from 8 bit to 64 bit (signed and unsinged).  Protocol Buffers support only 32 and 64 bits.  This article discusses how to let the API caller determine which type they need.  

## Small Go Types

Protocol Buffers support types for int32, uint32, int64, and uint64. With some clever dynamic typing smaller types can be made to work.
In the example below, the unmarshal function has a signiture of `func unmarshal(anyVal *anypb.Any)(interface{}, error)`.

## From typical int32 or int64 types

### Handling anypb.Any

```go
type ConvertableType interface {
    constraints.Ordered | []string | string |
        ~[]int | ~[]int64 | ~[]int32 | ~[]int16 | ~[]int8 |
        ~[]float32 | ~[]float64
}

func UnmarshalType[T ConvertableType](anyVal *anypb.Any) (T, error) {
    d := unmarshal(anyVal)
    var tempVal T
    if d.err != nil {
        return tempVal, d.err
    }

    var dynamicVal interface{} = tempVal
    switch dynamicVal.(type) {
    case int:
        var x = d.v.(int64)
        dynamicVal = int(x)
    case int16:
        var x = d.v.(int32)
        dynamicVal = int16(x)
    case int8:
        var x = d.v.(int32)
        dynamicVal = int8(x)
    case uint:
        var x = d.v.(uint64)
        dynamicVal = uint(x)
    case uint16:
        var x = d.v.(uint32)
        dynamicVal = uint16(x)
    case uint8:
        var x = d.v.(uint32)
        dynamicVal = uint8(x)
    default:
        var ok bool
        dynamicVal, ok = d.v.(T)
        if !ok {
            return tempVal, errors.New("cannot cast to message value to " + reflect.TypeOf(d.v).Name())
        }
    }

    return dynamicVal.(T), nil
}
```

## Explanation

```go 
func TestConvertUint8(t *testing.T) {
	expected := uint8(42)
	anyVal, err := Marshal(expected)
	if err != nil {
		t.Fatal(err)
	}

	actual, err := UnmarshalType[uint8](anyVal)
	if err != nil {
		t.Fatal(err)
	}
	if expected != actual {
		t.Fatalf("expected [%d] got [%d]", expected, actual)
	}
}
```
