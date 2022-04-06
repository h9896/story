# 利用 Protobuf 建立 RESTful API

## 建立一個 Protobuf

- 這裡利用 Protobuf 建立了一個 EchoService

```protobuf
syntax = "proto3";
package services.echo.v1;
option go_package = "services/echo/v1;v1";

import "google/api/annotations.proto";

service EchoService {
   rpc Echo(EchoRequest) returns (EchoResponse) {
    option (google.api.http) = {
        post: "/api/v1/echo"
        body: "*"
    };
   }
}

message EchoRequest {
       string msg = 1;
}

message EchoResponse {
    string echo_msg = 1;
}
```

在上面會發現有`import "google/api/annotations.proto"`, 這是[googleapis](https://github.com/googleapis/googleapis)中的檔案, 所以在 local build 的過程中可能會遇到 error, 要解決這個問題 [buf](https://github.com/bufbuild/buf) 是個不錯的選擇

---

## 使用 Buf

Step

1. Run `buf mod init`

   - 建立 buf.yaml

2. Add dependency

    - 修改 buf.yaml 檔案, 並新增 `buf.build/googleapis/googleapis`

3. Run `buf mod update`

    - 產生 `buf.lock`, 裡面會有相依性的檔案

4. `touch buf.gen.yaml` and modify it

    - 這邊使用的 `plugins` 均為 `remote`, 也可以改成`name`使用自行安裝的packages

---

```yaml
# buf.yaml
version: v1
breaking:
  use:
    - FILE
lint:
  use:
    - DEFAULT
deps:
  - buf.build/googleapis/googleapis
```

```yaml
# buf.gen.yaml
version: v1
plugins:
  - remote: buf.build/protocolbuffers/plugins/go
    out: gen/go
    opt:
      - paths=source_relative
  - remote: buf.build/grpc/plugins/go
    out: gen/go
    opt: 
      - paths=source_relative
      - require_unimplemented_servers=false
  - remote: buf.build/grpc-ecosystem/plugins/grpc-gateway
    out: gen/go
    opt: paths=source_relative
  - remote: buf.build/grpc-ecosystem/plugins/openapiv2
    out: gen/swagger
```

---

## 建立 Service

可以利用 `./gen/go` 底下的檔案來建立 service

```go
package main

import (
  "context"
  echopb "echo/gen/go/services/echo/v1"
  "fmt"
  "net"
  "net/http"

  "github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
  "google.golang.org/grpc"
  "google.golang.org/grpc/reflection"
)

type echoService struct {
}

func NewService() echopb.EchoServiceServer {
  return &echoService{}
}

func (s *echoService) Echo(ctx context.Context, req *echopb.EchoRequest) (*echopb.EchoResponse, error) {
  return &echopb.EchoResponse{EchoMsg: fmt.Sprintf("Here is an echo, %s", req.Msg)}, nil
}

func httpServer() error {
  ctxr := context.Background()
  ctx, cancel := context.WithCancel(ctxr)
  defer cancel()
  mux := runtime.NewServeMux()
  opts := []grpc.DialOption{grpc.WithInsecure()}
  err := echopb.RegisterEchoServiceHandlerFromEndpoint(ctx, mux, ":8080", opts)
  if err != nil {
    return err
  }
  return http.ListenAndServe(":8081", mux)
}

func grpcServer() {

  apiListener, err := net.Listen("tcp", ":8080")
  if err != nil {
    panic(err)
  }

  grpc := grpc.NewServer()

  echopb.RegisterEchoServiceServer(grpc, NewService())

  reflection.Register(grpc)

  if err = grpc.Serve(apiListener); err != nil {
    panic(err)
  }
}
func main() {
  go httpServer()
  grpcServer()
}
```

---

## 測試

Run `curl -X POST 127.0.0.1:8081/api/v1/echo --data '{"msg":"hello"}'`

Result `{"echoMsg":"Here is an echo, hello"}`

也可以查看 `./gen/swagger` 底下的資料來做測試

---

## 心得

用這個方式來產生RESTful API, 之後維護時, 只需要更動 Protobuf, 便能自動產生相對應的 interface, 這樣一來也不會發生忘了更動相應的服務