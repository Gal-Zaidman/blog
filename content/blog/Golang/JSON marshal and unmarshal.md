---
title: "JSON Marsheling and Unmarshaling"
subTitle: "Learn Golang By Doing"
date: 2020-09-26T00:10:00+03:00
draft: false

# post thumb
image: "images/Golang/golang-marsheling-unmarshaling.png"

# meta description
author: "Gal Zaidman"
description: "We will learn how to Marshele and Unmarshale JSON Objects in Golang"

# taxonomies
categories:
  - "Golang"

tags:
  - "Golang"
  - "JSON"

# post type
type: "post"
---

Most of the communication that happens over REST involves passing data as JSON.
JSON is a language-independent data format, but when we use it in our programs it is basically a string.We need to make sure that our JSON string is always valid JSON object this can be hard when we have a complex Struct that contains many fields. Marsheling and Unmarshaling objects helps us achieve that.
Marsheling is the operation of converting Go objects into JSON strings.
Unmarshalling is the operation of converting JSON strings into Go objects.

The first step is to add support for json to our struct:

```go
type User struct {
	Fname  string `json:"fname"`
	Lname string `json:"lname"`
}
```

We need to tell GO what is the name of the field in the JSON format and to which filed it correlates.
To convert it to a JSON we use the json.Marshal method from the encoding/json package:

```go
func (u User) toJSON() string {
    byteArray, err := json.Marshal(book)
    if err != nil {
        // HANDLE
    }
    return string(byteArray)
}
```

When we want to Unmarshal the object we use the json.Unmarshal method which receives a byte array containing the JSON and a pointer to the var object to populate:

```go
func Unmarshal(data []byte, v interface{}) error
```

For example:

```go
func jsonToUser(j string) *User {
    var p User
    err := json.Unmarshal([]byte(jsonString), &p)
    if err != nil {
            // HANDLE
        }
    return &p
}
```

## Faster JSON Marshal and Unmarshal:

The JSON marshaling and unmarshaling provided by the encoding/json package uses reflection to figure out values and types each time. Reflection, provided by the reflect package, takes time to figure out types and values each time a message is acted on (in Go is fairly fast).
If you’re repeatedly acting on the same structures, quite a bit of time will be spent reflecting.
Additionally, reflection allocates memory that needs to be garbage-collected, and there’s a small computational cost to that as well.

There are a number of packages outside the standard library that aim to solve this problem.
In this blog post we will cover the Codec Package which is very fast and popular but there are other good alternatives that will not be covered here such as:

- [easyjson](https://pkg.go.dev/github.com/mailru/easyjson)
- [ffjson](https://pkg.go.dev/github.com/pquerna/ffjson/ffjson)

The reason I choose to talk about Codec is that it is just considered more stable (currently as time goes by things will change).

### Codec

The package [github.com/ugorji/go/codec](https://pkg.go.dev/github.com/ugorji/go/codec?tab=doc), provides a High Performance codec/encoding library for binc, msgpack, cbor and json formats.

It is important to note that with all its benefits it is not always safe to use codec, and you should look at the documentation for warnings.
At the time of writing Codec is NOT safe for concurrent use, and state that the usage model is basically:

- Create and initialize the Handle before any use.
  Once created, DO NOT modify it.
- Multiple Encoders or Decoders can now use the Handle concurrently.
  They only read information off the Handle (never write).
- However, each Encoder or Decoder MUST not be used concurrently
- To re-use an Encoder/Decoder, call Reset(...) on it first.
  This allows you use state maintained on the Encoder/Decoder.

Look at [Usage](https://pkg.go.dev/github.com/ugorji/go/codec#hdr-Usage) for updates.

The first thing we need to do is to install the package with go get:

```bash
go get github.com/ugorji/go/codec/codecgen
```

To use it we first need to define our struct with the codec tag (instead of JSON):

```go
// go:generate codecgen -o user_generated.go user.go

type User struct {
	Fname  string `codec:"fname"`
	Lname string `codec:"lname"`
}
```

When we run:

```bash
go generate ./
```

go generate will see the first comment line of the file, which is specially formatted for it, and execute codecgen. The output file is named user_generated.go.
In the generated file, you’ll notice that two public methods have been added to the User type:

- CodecEncodeSelf
- CodecDecodeSelf

When these are present, the codec package uses them to encode or decode the type. When they’re absent, the codec package falls back to doing these at runtime.

```go
func (u User) toJSON() string {
    jh := new(codec.JsonHandle)
    var out []byte
    err := codec.NewEncoderBytes(&out, jh).Encode()
    if err != nil {
        // HANDLE
    }
    return string(byteArray)
}
```

```go
func jsonToPerson(j string) *Person {
    jh := new(codec.JsonHandle)
    var p User
    err := json.NewDecoderBytes(j, jh).Decode(&p)
    if err != nil {
        // HANDLE
    }
    return &p
}
```
