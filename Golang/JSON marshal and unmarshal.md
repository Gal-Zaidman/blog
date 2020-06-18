# JSON Marsheling and Unmarshaling

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

## Faster JSON marshal and unmarshal:

The JSON marshaling and unmarshaling provided by the encoding/json package uses reflection to figure out values and types each time. Reflection, provided by the reflect package, takes time to figure out types and values each time a message is acted on (in Go is fairly fast).
If you’re repeatedly acting on the same structures, quite a bit of time will be spent reflecting.
Additionally, reflection allocates memory that needs to be garbage-collected, and there’s a small computational cost to that as well.

The solution is to use optimized generated code

### Codec

The package [github.com/ugorji/go/codec](https://pkg.go.dev/github.com/ugorji/go/codec?tab=doc), provides a High Performance codec/encoding library for binc, msgpack, cbor, json formats.

To use it we need to define our stract with the codec tag (instead of JSON):

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

go generate will see the first comment line of the file, which is specially formatted for it, and execute codecgen. The output file is named user_generated.go. In the generated file, you’ll notice that two public methods have been added to the User type:

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

Note you have to install:
go get -u github.com/ugorji/go/codec/codecgen

### Easyjson

TODO

## Sources:

- Go in Practice, Published by Manning Publications, 2016.
- 