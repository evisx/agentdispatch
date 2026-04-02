# agentdispatch

本项目提供本地 HTTP 入口，把应用请求按 `poll` 或 `hash` 策略分发到 `agentproxy` 池中的某个节点，并通过内部 HTTPS relay 把请求透明转发出去。

## 本地入口语义

- `/<site>/<upstream-path...>?query` -> `http://<site>/<upstream-path...>?query`
- `/ssl/<site>/<upstream-path...>?query` -> `https://<site>/<upstream-path...>?query`

示例：

```text
GET /example.com/search?q=test
```

会被改写成：

```text
https://<selected-agentproxy>/relay/<DISPATCH_SECRET>/proxy/example.com/search?q=test
```

示例：

```text
POST /ssl/api.openai.com/v1/responses?model=gpt-4.1
```

会被改写成：

```text
https://<selected-agentproxy>/relay/<DISPATCH_SECRET>/proxyssl/api.openai.com/v1/responses?model=gpt-4.1
```

## 运行时配置

- `AGENTPROXY_POOL`
  逗号分隔的 `https://` agentproxy 节点列表，例如 `https://proxy-a.internal,https://proxy-b.internal`
- `DISPATCH_SECRET`
  与所有 `agentproxy` 节点共享的 relay secret
- `DISPATCH_STRATEGY`
  支持 `poll` 或 `hash`
- `RELAY_CONNECT_TIMEOUT_MS`
  内部 relay 连接超时，单位毫秒
- `RELAY_RESPONSE_TIMEOUT_MS`
  内部 relay 响应流超时，单位毫秒

## 分发策略

### `poll`

- 按 `AGENTPROXY_POOL` 配置顺序轮询
- 游标只保存在当前本地 `agentdispatch` 实例内
- 轮到池尾后回绕到第一个节点

### `hash`

- 按 `target site + Authorization` 计算稳定索引
- 缺失 `Authorization` 时按空字符串参与哈希
- 池长度或顺序变化会导致映射重排；这不是一致性哈希

## 透明转发边界

`agentdispatch` 会尽量保留：

- 原始 HTTP method
- 上游 path 与 query string
- `Authorization`、`Cookie`、`User-Agent` 等端到端请求头
- 请求 body 流
- 响应状态码
- `Set-Cookie`
- 流式响应 body

为了保持标准代理语义，hop-by-hop 头部例如 `Connection`、`Transfer-Encoding`、`Host`、`Content-Length` 不会继续转发。

## 失败语义

- 选中的 `agentproxy` 节点失败、连接超时或内部 relay 请求异常时，当前请求直接失败
- 第一版不会自动切换到池中的下一个节点
- 响应流在 relay 阶段超时后会直接中断给客户端

## 快速开始

```bash
npm install
npm test
```

常用命令：

- `npm run dev`
- `npm run typecheck`
- `npm test`
- `npm run build`

## 迁移注意事项

1. 先在所有 `agentproxy` 节点配置相同的 `DISPATCH_SECRET`，确认 `/relay/<secret>/proxyssl/...` 可用。
2. 部署 `agentdispatch`，配置 `AGENTPROXY_POOL`、`DISPATCH_STRATEGY` 和超时参数。
3. 将应用入口从“直连 `agentproxy`”切换到“访问本地 `agentdispatch` 的 `/<site>/...` 或 `/ssl/<site>/...`”。
4. 切换完成后，直接访问 `agentproxy` 的旧 `/proxy`、`/proxyssl` 入口会持续返回 `404`。
