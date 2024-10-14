---
title: 'Golang Module 机制探究'
tags:
- golang
date: 2022-06-01
draft: true
---

[[TODO]] 通读文档 https://go.dev/ref/mod
精简这个笔记，删除不必要的，细节交给官方文档

# 摘要
在 modules 包管理机制下，执行 `go mod tidy` 后就会自动下载 module 中导入的所有外部包，那么 go 工具是如何通过 import 语句找到并下载外部包的，下载的源代码又放在哪里？

自己写的 module 准备托管在 github 上又该怎么命名？
应该直接命名为 github 仓库名

# repo, module, package 的区别与联系
依赖的下载是以 repo 为单位，依赖在 go.mod 中的组织（require）是以 module 为单位，依赖在代码中的使用（import）是以 package 为单位

大多数时候 module 的地址就是一个 repo 的根目录地址

之所以有些 import 的是个 module，是因为它们既是 module 又是 package，比如 influxdb
就像有些既是 repo 又是 module，所以构成了上面的等号

总结就是：repo >= module >= package

# Module paths
module path = 仓库根路径 + module 子目录  + 主版本后缀（只有主版本号高于 2 时才可以）
- **仓库根路径 (repository root path)**：仓库根路径对应 module 的版本控制仓库的根目录。大多数 module 都是在仓库根目录定义的。
- **module 子目录 (module subdirectory)**：若 module 不是直接定义在仓库根路径，则需要 module 子目录来表明 module 所在的子目录。
- **主版本后缀 (major version suffix)**：若 module 主版本号大于等于 2，则 module path 必须以 `/v2` 这样的主版本后缀结尾，这个后缀可以是也可以不是 module 子目录名。

[[TODO]] 像下表一样抓典型以便将以下元素联系起来理解
| 1 module path | 1-1 repository root path / 2-1 root-path | 1-2 module subdirectory | 1-3 major version suffix | 2 "?go-get=1" query | 2-2 vcs | 2-3 repo-url |
| - | - | - | - | - | - | - |
| golang.org/x/net | golang.org/x/net | - | - | https://golang.org/x/net?go-get=1 | git | https://go.googlesource.com/net |
| golang.org/x/tools/gopls | golang.org/x/tools | gopls | - | https://golang.org/x/tools/gopls?go-get=1 | git | https://go.googlesource.com/tools |
| github.com/influxdata/influxdb-client-go/v2 | github.com/influxdata/influxdb-client-go | - | /v2 | https://github.com/influxdata/influxdb-client-go?go-get=1 | git | https://github.com/influxdata/influxdb-client-go.git |
| gopkg.in/yaml.v2 | gopkg.in/yaml.v2 | - | .v2 | https://gopkg.in/yaml.v2?go-get=1 | git | https://gopkg.in/yaml.v2 |

[[path tree]]

在这里我还发现 module 内部还可以有 module，如 golang.org/x/tools 和 gopls 都是 module

go get golang.org/x/tools 会下载包括 gopls 的 tools？
go get golang.org/x/tools/gopls 会下载 tools 还是 gopls？
moudle path -> root_path -> repo-url + module subdirectory ?
其实是不固定的，会进行多次尝试确定，见下文

# TOTALLY MESS 以下可取翻译成果，但逻辑要重新梳理
我目前发现应该是以 GOPROXY 为中心，分为同级的两种方式：
1. proxy https://go.dev/ref/mod#module-proxy：很好理解，逻辑已经理顺了，但是还没写到这里
2. direct https://go.dev/ref/mod#vcs：文档直接就用 module path，却没说怎么从 package path 获取到它的，应该测试一下

# Versions
版本号遵循语义化版本规范 [[semantic_versioning]]

# 找到 package 的 module
https://go.dev/ref/mod#resolve-pkg-mod

import path/package path 分为两部分
```go
import "<module-path>/<package-path-in-module>"
```
go 命令需要区分 import path 中的这两部分，即识别出其中的 module path，剩下的即为 package 在 module 内的相对路径

1. go 命令首先在 build list 中搜索有没有 module 是 import path 的前缀。

如 `import example.com/a/b` 这个 package时，若 build list 中有一个 module `example.com/a`，则 go 命令会检查 module path `example.com/a` 下的目录 `b` 中是否含有这个 package。

- 若恰好找到一个 module 提供了这个 package，那就使用这个 module；
- 若没有一个或有多个 module 提供这个 package，go 命令会报错。如果加了 `-mod=mod` flag，go 命令就会去找提供了这个 package 的新 modules 并更新 go.mod 和 go.sum，`go get` 和 `go mod tidy` 命令都会自动这样做。

2. 当 go 命令去找提供了这个 package 的新 modules 时，它会遍历所有 module path 可能值，向 GOPROXY 中的每一个代理服务器请求 module 的最新版本。

对于每个成功请求的 module path 可能值，go 命令会下载它的最新版本并检查这个 module 是否包含请求的 package。

- 若一个或多个 modules 包含这个 package，则选择具有最长 module path 的 module
- 若成功请求了一个或多个 modules 但都不包含这个 package，则报错。
- 若一个 module 都没有成功请求到，go 命令会尝试 GOPROXY 中的下一个代理，直到没有下一个代理了，就报错

例如，go 命令正在找一个提供了 `golang.org/x/net/html` 这个 package 的 module，且 `GOPROXY="https://corp.example.com,https://proxy.golang.org"`，则会发出以下请求
- 给 `https://corp.example.com/` 发送的（并行）
    - `golang.org/x/net/html` 的最新版本
    - `golang.org/x/net` 的最新版本
    - `golang.org/x` 的最新版本
    - `golang.org` 的最新版本
- 若以上请求都因 404 或 410 失败了，再给 `https://proxy.golang.org/` 发送一遍以上的请求

最终只有 `golang.org/x/net` 存在且提供了这个 package
当 go 命令请求一个 module 的最新版本时，他会先发送 `$module/@v/list` 请求，

找到了一个合适的 module 后，go 命令会在 go.mod 文件中添加它的 module path 和版本。这样能够保证之后使用相同的 package 时，会使用和这个相同版本的相同 module。若这个 package 不是由主 module 导入的，则会有一个 `//indirect` 注释

# 找到并下载 module path 的 repo
https://go.dev/ref/mod#vcs-find

- 如果 module path 中含有 VCS 后缀（`.bzr, .fossil, .git, .hg, .svn`），则从开头到该 VCS 后缀的字符串即为 repo 的 URL。

如对于 `example.com/foo.git/bar` module，go 命令会用 git 下载 `example.com/foo.git` 这个 repo，期望能够在它的 `bar` 子目录下找到这个 module。

- 如果 module path 中没有 VCS 后缀，go 命令则会发送一个 HTTP GET 请求 module path 加上 `?go-get=1` 这个 URL

例如，对于 `golang.org/x/mod` module，go 命令可能发送以下请求
```
https://golang.org/x/mod?go-get=1 (preferred)
http://golang.org/x/mod?go-get=1  (fallback, only with GOINSECURE)
```
go 命令会忽略响应报文的错误代码

服务器必须返回一个 HTML，在它的 head tag 中找一个格式如下的 meta tag
```html
<meta name="go-import" content="root-path vcs repo-url">
``` 
`root-path` 是仓库根路径，它必须是 module path 的前缀或严格匹配，若不是严格匹配，则会发出另一个请求并检查它返回的是否是同一个 meta tag
`vcs` 必须是关键字 `mod` 或下表中的工具
`repo-url` 是 repo 的 URL。
如果这个 URL 没有 scheme （因为 module path 中含有 VCS 后缀或因为 meta tag 中没有 scheme），go 命令会尝试每个版本控制系统支持的协议。

| Name | Command | GOVCS default | Secure schemes |
| - | - | - | - |
| Bazaar | bzr | Private only | https, bzr+ssh |
| Fossil | fossil | Private only | https |
| Git | git | Public and private | https, git+ssh, ssh |
| Mercurial | hg | Public and private | https, ssh |
| Subversion | svn | Private only | https, svn+ssh |

GitHub 以及其他受欢迎的托管服务的所有 repo 都会响应 `?go-get=1` 查询。

找到 repo URL 后，go 命令会下载 repo 到  module cache

# Module cache
https://go.dev/ref/mod#module-cache

Module cache 是 go 命令存储已下载的 module 文件的目录。
默认的 module cache 位置是 `$GOPATH/pkg/mod`，可以设置 GOMODCACHE 环境变量来改变该位置。

module cache 没有大小上限，且 go 命令不会自动移除它的内容。

同一机器上的所有项目可以共享 module cache

module cache 内的源文件和目录都是 read-only 的，这可以防止它们被修改，但会导致很难使用 `rm -rf` 这样的命令删除 cache，而可以使用 `go clean -modcache` 来删除。使用 `-modcacherw` flag 将创建有 read-write 权限的新目录，这会增加 module cache 内的文件被修改的风险。`go mod verify` 命令可以检查依赖的哈希值是否和 go.sum 中一致，从而检测主 module 的依赖是否被修改。

module cache 下的目录结构及含义见 [Module cache](https://go.dev/ref/mod#module-cache)
