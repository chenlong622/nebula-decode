# 修复记录：CF 自家站点（ChatGPT 等）打不开

## 现象

- 节点本身能连上、延迟正常，Google / Baidu / ping0.cc 等都能打开；
- 但 **chatgpt.com、www.cloudflare.com 等托管在 Cloudflare 的站点打不开**（TLS 无响应，0 字节）。

## 原因（两层）

1. **平台限制**：Cloudflare Workers/Pages 的出站 `connect()` **禁止连接 Cloudflare 自家 IP 段**
   （官方文档：*Outbound TCP sockets to Cloudflare IP ranges are blocked*）。
   chatgpt.com 等站点解析出来就在 CF 网段内，所以直连必然失败。
2. **代码缺陷（本次修复）**：`forwardTCP()` 的第一跳直连没有 try/catch。
   CF 对被禁地址是**立即抛错**而不是挂起，异常直接炸掉整个握手 ——
   下面的 ProxyIP 回落逻辑永远没有机会执行（配置了也白配）。

另外，当时面板里的 `proxyIP` 填的是 `bestcf.top` —— 它本身解析到 CF IP，
即使回落逻辑正常，Worker 也连不上它，等于没配。

## 修复

- `forwardTCP()` 第一跳加 try/catch：直连被拒时立即回落 ProxyIP 重连；
- 无 ProxyIP 时给出明确错误信息；
- 面板配置的 proxyIP 已改为 `ProxyIP.US.CMLiussss.net`（非 CF 段的 SNI 中继池）。
- 本地单元测试 122/122 通过；已提交 GitHub（commit 530ffeb）。

## ProxyIP 的要求（面板里换值时注意）

必须是「**非 Cloudflare IP** 的、按 SNI 中继裸 TCP」的服务器：
Worker 会把客户端发来的原始 TLS 握手（含目标站 SNI）转发给它，由它去连目标站。
优选域名（bestcf.top、cf.090227.xyz 之类）解析到 CF IP，**不能**当 ProxyIP 用。

若 `ProxyIP.US.CMLiussss.net` 失效，可在面板换其他社区维护的反代值后实测。

## 待办：部署

GitHub 上的 `smzxtv/nebula-decode` 已是修复版；但线上 `cs-9r7` 项目
属于另一个 Cloudflare 账号，本机 wrangler 凭据无法直接部署。
部署方式（任选其一）：

1. 控制台上传：`桌面\nebula-decode-修复版.zip` → Cloudflare Dashboard →
   Workers 和 Pages → cs-9r7 → 创建部署（上传 zip）；
2. 或用部署目标账号登录 wrangler 后执行 `npx wrangler pages deploy public --project-name cs-9r7`。

部署后无需改客户端，订阅/节点配置不变，立即生效。
