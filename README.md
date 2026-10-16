# Arm AI（usdcAI）

在自己电脑上放一个 USDC 钱包，用它按次付费调用 80 多个大模型（Claude / GPT / Gemini / DeepSeek / Qwen …），在 Cursor、VS Code + Cline、Kiro 里当成 OpenAI 兼容接口用。

上游是 [BlockRun](https://blockrun.ai) 的 x402 接口：每次请求先拿到报价，再用 Base 链上的 USDC 签一张 EIP-3009 授权付款。**私钥只在本机**，中间的转发服务器只能转报价、拿签好的付款，碰不到钱包。

> 本仓库开源的是 Windows 客户端和整套思路。转发服务器没有内置地址，需要自己按下面的协议搭一个（或者改成直接连 BlockRun）。

## 思路

```
Cursor / Cline ──OpenAI 兼容请求 + API Key──▶ 转发服务器 ──▶ BlockRun
                                              │  ◀── 402 报价（x402）
                       WebSocket 签名通道      │
   Arm AI（本机） ◀── {t:"sign", 报价} ─────────┘
        │ 核对：exact + Base USDC + 收款方 = BlockRun + 资源是 blockrun.ai + 不超过单条 / 当天上限
        └── {t:"signed", payment} ──▶ 转发服务器带着付款重发给 BlockRun ──▶ 流式回给编辑器
```

- **为什么要转发服务器**：Cursor 只认「Base URL + API Key」，不会自己做 x402 付款。服务器把 OpenAI 格式请求转给 BlockRun，遇到 402 就把报价推给用户本机的 Arm AI 签名。
- **为什么服务器拿不走钱**：收款地址（BlockRun）、链、币种、上限都由客户端核对，`src/signer.mjs` 里写死；服务器能做的最坏的事是在用户设定的每天上限内替他白用调用，钱到不了攻击者手里。
- **API Key**：客户端本机生成，用钱包 `personal_sign` 签「把这个 Key 的 sha256 绑到这个地址」去服务器登记，服务器只存 hash。
- **本机存储**：私钥和 Key 用 Windows DPAPI 加密存在 `%APPDATA%\ArmAI`（只有这台电脑的这个 Windows 用户能解），账本 / 记录是普通 JSON。
- **服务费（可选）**：客户端按「本机签给 BlockRun 的累计 × 5%」记账，服务器攒够门槛时发 `{t:"fee"}` 来要，客户端只签给写死的收款地址、不超过自己的记账。不需要就把 `FEE_BPS` 设成 0 或删掉这段。
- **界面**：本机 `127.0.0.1` 上的网页，用 Edge / Chrome 的 `--app` 模式开成独立窗口；每次启动一个随机 nonce，所有接口都要带。

## 转发服务器协议

客户端只依赖下面这些接口，自己实现即可（路径可以在 `src/main.mjs` 里改）。

| 接口 | 说明 |
|---|---|
| `POST /api/ai/register` | `{address, keyHash, at, signature, lang}`。`signature` 是对 `Arm AI\nLink API key <keyHash>\nto wallet <address>\nat <at>` 的 `personal_sign`，服务器验签后记下 `keyHash → address` |
| `GET  /api/ai/signer?lang=zh` (WebSocket) | `Authorization: Bearer <Key>`，401 时客户端会重新登记 |
| `POST /api/ai/v1/chat/completions` | OpenAI 兼容，`Authorization: Bearer <Key>`，编辑器直接调这个 |
| `GET  /api/ai/v1/stats?limit=60` | `{calls:[{at, model, served, in_tok, out_tok, cost, ms}], fee:{owed, settleAt}}`，界面「记录」页用 |
| `GET  /api/x402/models` | `[{id, name, owner, desc, context, input, output, free, vision, reasoning}]`，价格是每百万 token 美元 |

签名通道消息（JSON）：

| 方向 | 消息 |
|---|---|
| 服务器 → 客户端 | `{t:"sign", id, model, accepted, resource, bytes}`：`accepted` 是 x402 报价里选中的那项（`scheme / network / asset / payTo / amount / maxTimeoutSeconds / extra`） |
| 客户端 → 服务器 | `{t:"signed", id, payment}`：`payment` 是 base64 的 x402 v2 付款载荷；或 `{t:"refuse", id, reason}` |
| 服务器 → 客户端 | `{t:"fee", id, to, amount}`（可选，服务费结算） |
| 客户端 → 服务器 | `{t:"feeSigned", id, amount, authorization, signature}`：服务器拿去调 USDC 的 `transferWithAuthorization` 上链 |
| 服务器 → 客户端 | `{t:"config", exchanges}`（可选，是否显示交易所充值页） |
| 客户端 → 服务器 | `{t:"lang", lang}`：界面语言变了，服务器写进编辑器的提示跟着换 |

服务器另外要做的：客户端没开 / 拒签 / 超上限时，在编辑器里回一段说明文字而不是付钱；按用户分开记花费；同样的请求重发别重复付款。

## 运行和打包

```powershell
npm install
# 指向你自己的转发服务器（也可以写 %APPDATA%\ArmAI\server.json {"url": "..."}）
$env:ARM_AI_SERVER = "https://your-relay.example"
node src/main.mjs            # --no-open 不开窗口；ARM_AI_DIR 换数据目录
node build.mjs               # → dist/ArmAI.exe（Bun 编译的单文件，自带运行时，约 90 MB）
```

- `i18n/zh.json` 为准，`i18n/check.mjs` 打包前自动检查缺 key / 占位符；加语言 = 加一个 json 并在 `src/main.mjs` 引入。
- `src/editors.mjs`：自动找 Cursor / VS Code / Kiro（运行中的进程 → 卸载注册表 → 默认路径）。
- `src/cline.mjs`：从 Open VSX 下载 Cline（核对 sha256），用编辑器自己的 CLI 安装，并写好 Cline 的「OpenAI Compatible」设置。
- `src/exchanges.mjs`：从官方地址下载币安 / OKX 安装包，Windows 数字签名核对通过才打开。

## 注意

- 只认 **Base 链** USDC，转到其他链不会到账。
- BlockRun 的免费模型（如 `nvidia/nemotron-3-ultra-550b`）不用付钱，适合先试通，但随时可能下线。
- 转售模型调用要看 BlockRun 和各模型厂商的条款，各地对稳定币的规定也不同，部署前自己确认。
- 没有代码签名证书时，Windows SmartScreen 会提示「未知发布者」。

---

**English (short):** a Windows app that keeps a USDC wallet on your PC and pays BlockRun's x402 API per call, so Cursor / Cline / Kiro can use 80+ models through an OpenAI-compatible endpoint. A relay server forwards requests and quotes; the app checks every quote (exact USDC on Base, payee is BlockRun, within your caps) and signs locally, so the relay never holds keys or funds. No relay address is built in: implement the protocol above and set `ARM_AI_SERVER`.

## License

MIT
