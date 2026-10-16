# 免密车道的新线缆（2026-10-14 实测）

`/zen/v1` 上的免费车道在 2026-10-13/14 之间对插件全线返回
`403 {"type":"error","error":{"type":"FreeTierError", …}}`，而官方客户端仍然可用。
本文件记录实测到的五处变化、判定依据，以及 `src/upstream.js` / `src/http.js` 里对应的修正。

## 结论：现在的闸门

| 项目 | 实测结果 |
| --- | --- |
| 聊天端点 | **`https://opencode.ai/inference/v1/chat/completions`**。`/zen/v1/chat/completions` 即使其余全对也返回 403 FreeTierError；`/inference/v1/models` 与 `/zen/v1/models` 都仍返回 85 个模型 id |
| 会话头 | **`x-opencode-session-id` 必须存在**（值与 `x-opencode-session` 相同）。去掉它 → 403 |
| 会话 id 形状 | `ses_` 后 12 位 hex 必须是**铸造**形状 `~(ms·0x1000+seq)`（低 12 位 ≈ `0xFFE`）。sha256 派生或随机 hex → 403 |
| 流式 | **`body.stream` 必须为 `true`**。`false` 或缺省 → 403 FreeTierError，且与其余条件无关 —— 头、端点、会话、工具指纹全对也照样 403 |
| 工具指纹 | **`bash` 与 `read` 必须同时声明**，且各自的参数 schema 至少有一个 property。只声明其一 → 403；`properties: {}` → 403；两者齐备 + 任意 property → 200，且可附带你自己的工具 |
| User-Agent | 含 `deepseek-harness`（大小写不敏感、位置无关）→ 403；`dsh/…`、`opencode/1.18.22` 均 200 |
| 其余 | `authorization: Bearer public` 必须保留；`x-opencode-request` / `x-opencode-project` / `x-opencode-client` 可省（仍照发）；HTTP/1.1 与 HTTP/2、Node 与 Bun 的 TLS 栈**无差别** |

## 判定方法

1. 在本机跑官方客户端 `opencode-ai@1.19.0`（linux-arm64-musl，Alpine 的 musl loader），确认
   `opencode run -m opencode/cat-jam-free` 可用 —— 说明闸门是客户端指纹，不是账号或配额。
2. 用自签 CA 的 TLS 中间人（`NODE_EXTRA_CA_CERTS`）把客户端指到本地，读到它真正发出的请求：
   路径是 **`/inference/v1/chat/completions`**，并携带 `x-opencode-session-id`。
3. 用**逐字节重放**做单变量对照（普通 Node `https`，不涉及任何 TLS 技巧）：
   从真实请求出发，每次只删/改一项，记录 200/403。上表即该对照的结果。
4. 工具维度另做矩阵：`bash` 单独 → 403、`read` 单独 → 403、`bash`+`read` 空 schema → 403、
   `bash`+`read` 各带一个 property → 200、`Read`/`BASH` 大小写变体 → 403。
5. 2026-10-16 又做了一次**单变量对照**（`mimo-v2.6-flash-free`，从已知 200 的请求出发，
   每次只改一项），补测流式这一项，并复核「可省」那几个头：

   | 变量 | HTTP |
   | --- | --- |
   | 基线（`stream:true` + 全部头） | 200 |
   | **`stream:false`** | **403 FreeTierError** |
   | 去掉 `x-opencode-request` | 200 |
   | 去掉 `x-opencode-project` | 200 |
   | 两者都去掉 | 200 |
   | 非铸造 session（对照组，证明闸门确实是活的） | 403 |

   对照组必须保留：它证明这一轮的 200 不是因为闸门整条关掉了。

   这一项此前**没有写进本文件**，只在 PR #352 的实测表里。后果是：手工用
   `stream:false` 探测会拿到一个与真实故障无法区分的 403，很容易被误判成
   「IP/机房被封」或「车道整条挂了」—— 本次排查就这么绕了一大圈。

## 修正

- `endpointFor()` / `wireFor()`：三条路径移到 `/inference/v1`；清单地址同步移动，
  测试用的 stand-in 网关（`scripts/lib/fake-kernel.mjs` 等）跟着改。
- `gatewayHeaders()`：补 `x-opencode-session-id`，`x-opencode-client` 报 `cli`。
- `sessionForConversation()`：改用 `mintSessionId()` 铸造并按会话缓存（保留
  「一个会话一个 session」的配额语义，同时满足形状校验）。
- `applyFingerprint()`：诱饵与调用者自带的 `bash`/`read` 都补齐至少一个 property；
  只补 schema，不改名字、不改描述、不复制工具。
- `http.js` 的 `userAgentWith()`：归属 UA 里的 `deepseek-harness` 改写为 `dsh`，
  并去掉重复该 token 的括号 URL；其余归属信息保留。
- 流式：此前只有调用方各自保证 —— `adapter.js` 构建 body 时写死 `stream: true`，
  `aggregate.js` 的 `postAggregateStreamed()` 用 `{ ...body, stream: true }` 重建，
  `http.js` 的 `gatewayHeaders()` 固定发 `accept: text/event-stream`。三者都对，但
  契约是隐式的：任何一条新路径忘了这个 flag 就会收到一个看起来像「车道挂了」的 403。

  本次把它下沉到出口 —— `postStreamed()` 现在同样重建 body 并强制 `stream: true`
  （写法与 `postAggregateStreamed()` 一致），并补了一条回归断言。

  仍然透传调用方 `stream` 的只剩 `forward.js`（独立服务的 OpenAI / Anthropic 兼容入口，
  `wantsStream = body.stream === true`），它不走 `postStreamed`。那里若收到
  `stream:false`，转发到本车道必然 403 —— 属于独立服务那一条链路的已知边界，
  要修得在那一层把流式结果聚合成非流式响应，不在本次改动范围内。

回归测试：`scripts/fingerprint-test.mjs`（工具指纹）与新增的
`scripts/free-lane-wire-test.mjs`（路径、会话头、会话形状、UA 净化，含一次走 stand-in 网关的
端到端断言）。两者都在 `npm run test:contributor` 内。
