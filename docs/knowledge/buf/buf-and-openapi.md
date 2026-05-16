# Buf 与 OpenAPI 生成说明

本文说明本项目中 Buf 的用途、常用命令、配置文件职责，以及如何一次性生成 Protobuf Go 代码、gRPC 代码、grpc-gateway 代码和 OpenAPI 文档。

## 1. Buf 在项目里的作用

Buf 是 Protobuf 的工程化工具，主要负责三类事情：

1. 管理 proto 模块和外部依赖
2. 检查 proto 文件规范
3. 调用插件生成代码和接口文档

本项目的 proto 源文件位于：

```text
api/proto/whocall
```

生成后的 Go 代码位于：

```text
api/gen/go/whocall
```

生成后的 OpenAPI 文档位于：

```text
api/openapi/who-call.swagger.json
```

## 2. 关键配置文件

### buf.yaml

`buf.yaml` 定义 Buf 模块、依赖、lint 规则和 breaking change 检查规则。

当前项目配置含义：

```yaml
version: v2

modules:
  - path: api/proto

deps:
  - buf.build/googleapis/googleapis

lint:
  use:
    - STANDARD

breaking:
  use:
    - FILE
```

字段说明：

| 字段 | 作用 | 当前值说明 |
| --- | --- | --- |
| `version` | Buf 配置文件版本 | `v2` 表示使用 Buf v2 配置格式 |
| `modules` | 声明当前仓库里的 proto 模块列表 | 本项目只有一个模块 |
| `modules.path` | proto 模块根目录 | `api/proto`，所以 import 路径从这个目录开始计算 |
| `deps` | 声明远程 proto 依赖 | `buf.build/googleapis/googleapis` 提供 `google/api/annotations.proto` 等 Google API 定义 |
| `lint` | 配置 proto lint 规则 | 控制 `buf lint` 检查哪些规范 |
| `lint.use` | 启用的 lint 规则集 | `STANDARD` 是 Buf 推荐的标准规则集，会检查 package、目录、命名等 |
| `breaking` | 配置破坏性变更检查 | 控制 `buf breaking` 如何判断兼容性 |
| `breaking.use` | 启用的 breaking 规则集 | `FILE` 按文件维度检查字段删除、编号复用、类型变化等破坏性变更 |

当前目录结构和 package 的关系：

```text
api/proto/whocall/auth/v1/auth.proto
package whocall.auth.v1;
```

Buf `STANDARD` 规则要求 package 和目录对应，所以 `whocall.auth.v1` 应放在 `whocall/auth/v1` 目录下。

### buf.gen.yaml

`buf.gen.yaml` 定义线上 Buf remote plugins 的生成方式，作为本地插件缺失时的回退方案。

完整配置：

```yaml
version: v2

clean: true

plugins:
  - remote: buf.build/protocolbuffers/go
    out: api/gen/go
    opt:
      - paths=source_relative

  - remote: buf.build/grpc/go
    out: api/gen/go
    opt:
      - paths=source_relative
      - require_unimplemented_servers=false

  - remote: buf.build/grpc-ecosystem/gateway
    out: api/gen/go
    opt:
      - paths=source_relative
      - generate_unbound_methods=true

  - remote: buf.build/grpc-ecosystem/openapiv2
    out: api/openapi
    strategy: all
    opt:
      - allow_merge=true
      - merge_file_name=who-call
      - json_names_for_fields=true
```

通用字段说明：

| 字段 | 作用 | 当前值说明 |
| --- | --- | --- |
| `version` | Buf 生成配置版本 | `v2` 表示使用 Buf v2 生成配置格式 |
| `clean` | 生成前是否清理插件输出目录中旧文件 | `true` 可以避免 proto 删除或移动后残留旧生成文件 |
| `plugins` | 生成插件列表 | 每个插件负责一种产物 |
| `remote` | 使用 Buf Registry 上的远程插件 | 需要访问 `buf.build`，适合作为本地插件缺失时的回退 |
| `local` | 使用本机 `PATH` 中的插件命令 | 只在 `buf.local.gen.yaml` 中使用，不依赖远程插件服务 |
| `out` | 插件输出目录 | Go 代码输出到 `api/gen/go`，OpenAPI 输出到 `api/openapi` |
| `opt` | 传给具体 protoc 插件的参数 | 不同插件支持的参数不同 |
| `strategy` | Buf 调用插件的策略 | OpenAPI 使用 `all`，让插件一次性看到全部 proto 文件，方便合并 swagger |

插件说明：

| 插件 | 生成内容 | 输出 |
| --- | --- | --- |
| `buf.build/protocolbuffers/go` | message、enum 等 Go 类型，也就是 `*.pb.go` | `api/gen/go` |
| `buf.build/grpc/go` | gRPC server/client 接口，也就是 `*_grpc.pb.go` | `api/gen/go` |
| `buf.build/grpc-ecosystem/gateway` | grpc-gateway HTTP 反向代理代码，也就是 `*.pb.gw.go` | `api/gen/go` |
| `buf.build/grpc-ecosystem/openapiv2` | OpenAPI v2 swagger JSON | `api/openapi` |

插件参数说明：

| 参数 | 所属插件 | 作用 |
| --- | --- | --- |
| `paths=source_relative` | `go`、`go-grpc`、`grpc-gateway` | 生成文件路径跟 proto 源文件相对路径一致。例如 `whocall/auth/v1/auth.proto` 生成到 `whocall/auth/v1/auth.pb.go` |
| `require_unimplemented_servers=false` | `go-grpc` | 生成 gRPC server 接口时，不强制嵌入 `UnimplementedXXXServer` |
| `generate_unbound_methods=true` | `grpc-gateway` | 没有写 `google.api.http` 注解的 rpc 也生成默认映射，便于调试或补齐网关代码 |
| `allow_merge=true` | `openapiv2` | 允许多个 proto 文件的 OpenAPI 输出合并成一个 swagger 文件 |
| `merge_file_name=who-call` | `openapiv2` | 合并后的文件名为 `who-call.swagger.json` |
| `json_names_for_fields=true` | `openapiv2` | OpenAPI 字段名使用 proto 的 JSON 名称，例如 `avatar_url` 对应 `avatarUrl` |
| `strategy: all` | `openapiv2` | Buf 一次性把所有 proto 文件传给插件，避免合并 swagger 时出现重复文件名 warning |

### buf.local.gen.yaml

`buf.local.gen.yaml` 定义本地 protoc 插件的生成方式。当前项目优先使用本地插件，避免每次生成都依赖 `buf.build`。

本地生成依赖以下命令存在于 `PATH`：

```text
protoc-gen-go
protoc-gen-go-grpc
protoc-gen-grpc-gateway
protoc-gen-openapiv2
```

它和 `buf.gen.yaml` 的主要区别：

| 配置 | 插件来源 | 使用场景 |
| --- | --- | --- |
| `buf.local.gen.yaml` | 本机 `PATH` 中的 `protoc-gen-*` 命令 | 本地开发优先使用，速度快，不依赖远程插件服务 |
| `buf.gen.yaml` | `buf.build` 上的 remote plugins | 本地插件缺失时回退使用，或 CI 想统一远程插件版本时使用 |

无论使用本地插件还是线上插件，都会同时生成：

- Go message 结构体
- gRPC service stubs
- grpc-gateway HTTP 反向代理代码
- OpenAPI v2 swagger 文档

线上回退配置示例：

```yaml
plugins:
  - remote: buf.build/protocolbuffers/go
    out: api/gen/go

  - remote: buf.build/grpc/go
    out: api/gen/go

  - remote: buf.build/grpc-ecosystem/gateway
    out: api/gen/go

  - remote: buf.build/grpc-ecosystem/openapiv2
    out: api/openapi
```

### buf.lock

`buf.lock` 是 Buf 的依赖锁文件，作用类似 `go.sum`。

它记录 `buf.yaml` 中依赖的精确版本，保证不同开发环境和 CI 使用同一份 proto 依赖。

一般规则：

- 需要提交到 Git
- 不要手动修改
- 更新依赖时使用 `buf dep update`

## 3. 常用命令

### 查看可用 Make 命令

```bash
make help
```

### 一次性生成 proto 代码和 OpenAPI 文档

推荐使用：

```bash
make gen
```

`make gen` 会先检查本地插件是否齐全。

如果本地插件齐全，执行：

```bash
buf generate --template buf.local.gen.yaml
```

如果本地插件不齐全，回退执行：

```bash
buf generate
```

也可以显式指定生成方式：

```bash
make gen-local
make gen-remote
```

它会同时生成：

```text
api/gen/go/**/*.pb.go
api/gen/go/**/*.pb.gw.go
api/gen/go/**/*_grpc.pb.go
api/openapi/who-call.swagger.json
```

### 检查 proto 规范

```bash
buf lint
```

项目里的统一 lint 命令是：

```bash
make lint
```

它会执行：

```bash
golangci-lint run ./...
buf lint
```

### 更新 Buf 依赖锁

```bash
buf dep update
```

这个命令会根据 `buf.yaml` 更新 `buf.lock`。

## 4. OpenAPI 是如何生成的

OpenAPI 文档来自 proto 文件里的 HTTP 注解。

示例：

```proto
import "google/api/annotations.proto";

service AuthService {
  rpc Register(RegisterRequest) returns (RegisterResponse) {
    option (google.api.http) = {
      post: "/api/v1/auth/register"
      body: "*"
    };
  }
}
```

生成链路：

```text
api/proto/whocall/**/*.proto
        |
        | buf generate
        v
grpc-gateway openapiv2 插件
        |
        v
api/openapi/who-call.swagger.json
```

当前 `buf.gen.yaml` 里 OpenAPI 插件配置为：

```yaml
- remote: buf.build/grpc-ecosystem/openapiv2
  out: api/openapi
  opt:
    - allow_merge=true
    - merge_file_name=who-call
    - json_names_for_fields=true
```

说明：

- `allow_merge=true` 把多个 proto service 合并成一个 swagger 文件
- `merge_file_name=who-call` 生成 `who-call.swagger.json`
- `json_names_for_fields=true` 使用字段的 JSON 命名

## 5. 新增接口时的推荐流程

1. 在 `api/proto/whocall/<domain>/v1/*.proto` 中新增 message 和 service
2. 如果需要 HTTP API，给 rpc 添加 `google.api.http` 注解
3. 执行 `buf lint` 检查 proto 是否规范
4. 执行 `make gen` 生成 Go 代码和 OpenAPI 文档
5. 执行 `go test ./...` 确认生成代码可编译
6. 提交 proto、生成代码和 OpenAPI 文档

## 6. 常见问题

### make openapi 报 buf.openapi.gen.yaml 不存在

错误示例：

```text
Failure: open buf.openapi.gen.yaml: no such file or directory
```

原因是命令引用了不存在的模板文件。

当前项目已经把生成入口统一为：

```bash
make gen
```

不需要单独维护 `buf.openapi.gen.yaml`。

### cannot find RequestMeta in this scope

如果当前 proto package 是 `whocall.auth.v1`，但类型定义在 `whocall.common.v1`，跨 package 引用时必须写完整包名。

错误写法：

```proto
RequestMeta meta = 1;
```

正确写法：

```proto
whocall.common.v1.RequestMeta meta = 1;
```

### make gen 什么时候需要联网

默认 `make gen` 本地插件优先。

如果以下本地插件都存在，则不需要访问线上 Buf 插件：

```text
protoc-gen-go
protoc-gen-go-grpc
protoc-gen-grpc-gateway
protoc-gen-openapiv2
```

如果插件不齐全，`make gen` 会回退到 `buf.gen.yaml` 的 remote plugin：

```yaml
remote: buf.build/protocolbuffers/go
```

此时可能需要访问 `buf.build`。如果 CI 环境不能联网，应先安装本地插件，或直接使用 `make gen-local` 并保证插件完整。

## 7. 推荐约定

- proto 源文件和生成代码一起提交
- `buf.lock` 一起提交
- 修改 proto 后必须重新执行 `make gen`
- 修改 HTTP 注解后必须检查 `api/openapi/who-call.swagger.json`
- 不手动编辑 `api/gen/go` 和 `api/openapi/who-call.swagger.json`
- OpenAPI 路由统一通过 `google.api.http` 注解声明
