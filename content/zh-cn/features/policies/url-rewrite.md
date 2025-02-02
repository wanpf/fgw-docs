---
title: "URL 重写"
description: "URL 重写功能为系统提供了更大的灵活性来适应后端服务的不断变化。本文档将介绍如何使用 FGW 的 URL 重写功能。"
weight: 10
---

## 介绍

URL 重写功能为 FGW 用户提供了一种在流量进入目标服务之前修改请求 URL 的方法。这不仅提供了更大的灵活性来适应后端服务的变化，而且确保了应用的流畅迁移和 URL 的规范化。

URL 重写常被用在以下场景：

- 版本迁移：当应用或网站进行版本迁移时，可以使用前缀匹配来将旧版本的路径前缀替换为新版本的路径前缀。
- 目录结构更改：若应用或网站的目录结构发生变化，但仍需保持旧的 URL 可访问，可利用全路径匹配重写功能。
- URL 规范化：保证 URL 符合特定规范或风格。

## 前置条件

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- 后端服务
- 了解 [HTTP 负载均衡的配置](/features/http-load-balancer/)

后端服务可以使用 Pipy 来模拟一个返回正在请求的路径的服务。

```shell
pipy -e "pipy().listen(8081).serveHTTP(msg => new Message('You are requesting ' + msg.head.headers.host + msg.head.path))"
```

它会返回我们正在请求的路径。

```shell
curl http://localhost:8081/api/hi
You are requesting localhost:8081/api/hi
curl http://localhost:8081/user
You are requesting localhost:8081/user
```

## 配置说明

下面参考 [URL重写文档](/reference/configuration/#4134-httpurlrewritefilter) 对配置字段进行说明。

* `Hostname`：重写后请求的目标域名或主机名。此字段可用于改变请求的目标地址，将流量引导至一个完全不同的域名或子域名。值为任何有效的域名，如 `example.com` 或 `sub.example.com`。可选字段，不配置时会继续使用原请求中 `host`。
* `Path`：对 path 路径的重写规则。

  - `Type`：URL 重写匹配规则，此字段是必须的，它定义了路径重写时所采用的匹配方式。参考值有：`ReplacePrefixMatch` 表示前缀匹配，和 `ReplaceFullPath` 表示全路径匹配。选择合适的匹配规则是重写功能的基础。
  - `ReplacePrefixMatch`：前缀匹配时的 path，当 `Type` 字段设置为 `ReplacePrefixMatch` 时，此字段必须配置。它定义了需要被替换的 URL 前缀。例如，如果设置为 `/old-prefix`，那么所有以 `/old-prefix` 开始的路径都会被替换。
  - `ReplacePrefix`：前缀匹配时替换成这个 path，当 `Type` 字段设置为 `ReplacePrefixMatch` 时，此字段可以配置，但不是必须的。它定义了用来替换的新前缀。默认值为 `/`。例如，如果 `ReplacePrefixMatch` 设置为 `/old-prefix` 并且 `ReplacePrefix` 设置为 `/new-prefix`，那么 `/old-prefix/example` 会被重写为 `/new-prefix/example`。
  - `ReplaceFullPath`：全路径匹配时的 path，当 `Type` 字段设置为 `ReplaceFullPath` 时，此字段必须配置。它定义了需要完全匹配并替换的路径。例如，如果设置为 `/old-path/example`, 那么只有完全匹配到这个路径的请求会被重写。

### 示例

```json
{
  "Filters": [
    {
      "Type": "HTTPURLRewriteFilter",
      "UrlRewrite": {
        "Hostname": "",
        "Path": {
          "Type": "ReplacePrefixMatch",
          "ReplacePrefixMatch": "/path-prefix"
        }
      }
    },
    {
      "Type": "HTTPURLRewriteFilter",
      "UrlRewrite": {
        "Hostname": "",
        "Path": {
          "Type": "ReplacePrefixMatch",
          "ReplacePrefixMatch": "/path-prefix",
          "ReplacePrefix": "/new-path-prefix"
        }
      }
    },
    {
      "Type": "HTTPURLRewriteFilter",
      "UrlRewrite": {
        "Hostname": "",
        "Path": {
          "Type": "ReplaceFullPath",
          "ReplaceFullPath": "/path-prefix"
        }
      }
    }
  ]
}

```

## 配置

### 负载均衡配置

我们借助在文档 [HTTP 负载均衡](/features/http-load-balancer/) 中的负载均衡器配置，修改服务的断点列表为我们的后端服务地址 `127.0.0.1:8081`；同时将该后端服务的路由路径设置为 `/sample`；最后根据 [HTTP 插件链配置](/reference/plugin/#http-路由)，将 `filter/url-rewrite.js` 插件配置在相应的配置。

```json
{
  "Listeners": [
    {
      "Protocol": "HTTP",
      "Port": 8080
    }
  ],
  "RouteRules": {
    "8080": {
      "*": {
        "RouteType": "HTTP",
        "Matches": [
          {
            "Path": {
              "Type": "Prefix",
              "Path": "/sample"
            },
            "BackendService": {
              "backendService1": {
                "Weight": 100
              }
            }
          }
        ]
      }
    }
  },
  "Services": {
    "backendService1": {
      "Endpoints": {
        "127.0.0.1:8081": {
          "Weight": 100
        }
      }
    }
  },
  "Chains": {
    "HTTPRoute": [
      "common/consumer.js",
      "http/codec.js",
      "http/route.js",
      "http/service.js",
      "filter/url-rewrite.js",
      "http/forward.js",
      "http/default.js"
    ]
  }
}
```

测试服务的访问，`/sample` 的前缀会被带到后端服务。

```shell
curl http://localhost:8080/sample/api/hi
You are requesting localhost:8080/sample/api/hi
```

### 前缀重写配置

假如前缀 `/sample` 并不是后端服务的合法路径，负载均衡器在转发请求的时候需要将该前缀去掉。我们使用 `ReplacePrefixMatch` 类型的 URL 重写规则，将 `/sample` 作为前缀匹配请求，然后将该前缀替换为 `/release`。

```json
{
  "Matches": [
    {
      "Path": {},
      "BackendService": {},
      "Filters": [
        {
          "Type": "URLRewrite",
          "UrlRewrite": {
            "Path": {
              "Type": "ReplacePrefixMatch",
              "ReplacePrefixMatch": "/release"
            }
          }
        }
      ]
    }
  ]
}
```

还是使用之前的地址发送请求，这次后端服务接收到的请求路径为 `/release/api/hi`，前缀被替换。

```shell
curl http://localhost:8080/sample/api/hi
You are requesting localhost:8080/release/api/hi
```

如果想将其重写为 `/api/hi`，也就是去掉前缀场景，可以将 `ReplacePrefixMatch` 设置为 `""`（空字符串）。

### 全路径重写配置

有些情况下如果服务路径已经不再使用，但出于安全和合规的原因，不希望外部用户访问它，可以使用全路径重写将其重定向到一个通用的 404 页面或其他安全页面。

添加 `HTTPURLRewriteFilter` 全路径重写规则，将访问服务的所有请求路径都修改为目标路径 `/404`。

```json
{
  "Matches": [
    {
      "Path": {},
      "BackendService": {},
      "Filters": [
        {
          "Type": "URLRewrite",
          "UrlRewrite": {
            "Path": {
              "Type": "ReplaceFullPath",
              "ReplaceFullPath": "/404"
            }
          }
        }
      ]
    }
  ]
}
```

此时再请求，就会去到 /404 地址。

```shell
curl http://localhost:8080/sample/api/hi
You are requesting localhost:8080/404
```

### 重写主机名

在上面的例子中，后端服务打印的主机名都是 `localhost:8080` 也就是代理的地址。有时候，我们希望对上游服务隐藏代理的地址，这时候我们就需要重写主机名。

这里就不需要重写路径了，而是需要 `Hostname` 字段。在下面的代码片段中，我们将主机名改写为 `example.com`。

```json
{
  "Matches": [
    {
      "Path": {},
      "BackendService": {},
      "Filters": [
        {
          "Type": "URLRewrite",
          "UrlRewrite": {
            "Hostname": "example.com"
          }
        }
      ]
    }
  ]
}
```

应用配置后，我们还是发送原来的请求，可以看到后端服务拿到的主机名为 `example.com`。

```shell
curl http://localhost:8080/sample/api/hi
You are requesting example.com/sample/api/hi
```