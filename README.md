# WarpScout-Chain

极简单文件 Windows GUI 客户端，硬编码 **Warp-on-Warp（双层 WARP 嵌套链式代理）**，解决中国大陆网络环境下直连 WARP 被封锁以及出口 IP 位于国内/香港导致 AI 平台被拒的问题。

## 架构原理
- **外层隧道 (Outer Tunnel)**: AmneziaWG (AWG) 或 MASQUE (H2/H3 CONNECT-UDP)，直连大陆低延迟 Cloudflare Anycast 边缘 IP，突破 GFW 阻断。
- **内层隧道 (Inner Tunnel)**: Standard WireGuard，在外层加密安全管道中向 Cloudflare 发起二次握手，分配海外纯净出口 IP（美/日/新等），解锁 OpenAI / Claude / Gemini。

## 目录结构
```
warpscout-chain/
├── .github/workflows/build.yml   # GitHub Actions 自动化 Windows 单文件编译
├── frontend/
│   └── index.html                # 现代暗黑响应式 GUI 界面 (无额外 npm 编译依赖)
├── main.go                       # Wails v2 应用程序入口
├── app.go                        # 核心控制器：扫描测速、双账号注册、链式组装与本地订阅
├── go.mod                        # Go 模块定义
├── wails.json                    # Wails 项目配置文件
└── README.md                     # 说明文档
```

## 云端零环境打包 (GitHub Actions)
1. 将本项目所有代码上传至 GitHub 仓库。
2. 进入仓库页面的 **Actions** 标签页。
3. 选择 **Build Windows Single Exe** 工作流，点击 **Run workflow** (工作流仅手动触发, push 不构建)。
4. 编译完成后，在 **Artifacts** 区域即可直接下载打包好的单文件 exe。

## 新版特性 (2026-09)
- **机场式扁平分组**: sing-box 配置只有两个组 — 「节点选择」(手动 selector, 平铺 AI 专线 + 全部 WARP 优选 + direct, 选哪个走哪个) 与「测速分组」(urltest 自动选最快, 带 tolerance 防抖)。一份配置同时适配官方 sing-box GUI (SFW)、Karing、Hiddify、NekoBox。
- **AI 专线数量修复 (09-13)**: 原版 AI 解锁检测让内层账号直连外层优选端点 — 端点分钟级漂移失活时 WireGuard 握手永远完不成, 被误报为 "ChatGPT 封禁" 导致 4 条 AI 只剩 1 条。现在检测与最终配置同构 (内层 detour 外层的真实链路), 且检测未通过的内层账号不再丢弃 — 全部保留为 AI 专线节点, 生成后可在分组内实测切换。
- **YouTube 视频流修复 (09-13)**: 视频流域名 googlevideo.com / youtubei.googleapis.com / ggpht.com / gvt1.com 此前缺失于 DNS 分流与路由规则 — 表现为 "有速度看不到画面" (视频 CDN IP 被污染)。现已补进 sing-box DNS (dns-remote) 与路由 (节点选择) 及 Clash 显式规则。
- **Clash DNS 根治 (09-13)**: nameserver 从明文 UDP 53 (1.1.1.1/8.8.8.8, 国内被投毒/拦截 → yt3.ggpht.com 解析失败) 改为加密 DoH (https://1.1.1.1/dns-query); 移除使 fake-ip 失效的 fake-ip-filter 通配 "*"; WARP 自动优选 url-test 加 tolerance 150 + lazy false 防断流抖动。
- **手机端免手动 (09-13)**: /sub 订阅按 User-Agent 自动识别手机客户端 (SFA/SFI/Android/iPhone/okhttp) 下发 TUN 版配置 — 手机上没有 "系统代理" 概念, TUN 虚拟网卡自动接管全局流量, 订阅即用。TUN MTU 从 9000 修正为 1280 (双层封装下大 MTU 触发分片卡顿)。注意: 手机 WiFi 里的「手动代理」必须关闭 — SFA 是 VPN 型客户端, 系统代理指向不存在的端口会导致 "有上传下载但打不开"。
- **V2rayN 一键复制**: 导出页一键复制 wireguard:// 链接 (publickey 无下划线 + reserved 十进制逗号, 按官方 WireguardFmt 解析器定制), 外层节点与 AI 内层节点 (WARP-AI) 都包含; V2rayN 内核切 sing-box 即可使用。
- **MASQUE 自愈**: 内层注册撞上失活端口时自动重新探测 engage 池并在新端口复活外层隧道; engage 三轮全灭时轮换探测账号 (不同账号路由哈希落到不同 CF 机房)。
- **端点真实性验证**: 12-18 候选全部走真实 WireGuard 握手 + 数据面测速, 并发闸 8 防假死; MASQUE 网关 (162.159.198.2) 生成前 TLS 预检, CF 阻断期自动回退 WireGuard 外层。
- **临时文件根治**: 探测配置写入系统 Temp, 启动时自动清扫遗留垃圾。
- **Clash (mihomo) 同步**: 扁平分组 + WARP 自动优选 url-test + GLOBAL 兜底, 与 sing-box 侧行为对齐。
- **warp-in-warp 整链测速 (09-13)**: 单层端点测速只代表外层直连性能, 套娃链路双重 WG 封装开销必须以链路实测为准 — 每条 AI 专线起同构临时链式隧道 (内层 detour 外层) 跑 CF 测速流, 实测速度直接标注进节点名 (sing-box + Clash 双侧同步), 如 "AI-WARP专线-02 (13.7Mbps链路)"。测速失败不丢节点。
- **一键系统代理后端 (09-13)**: 后端已内置 SetSystemProxy/IsSystemProxyOn 方法 (HKCU 注册表 ProxyEnable/ProxyServer, 127.0.0.1:2080), 界面按钮暂未启用 — 桌面端走 SFW 概述页「系统 HTTP 代理」开关或 Windows 设置手动配置即可。
- **套娃失效自愈说明**: AI 专线内层 detour「测速分组」(urltest) — 外层节点失效时自动切换到活端点; 内层账号失效不自动换 (AI 分组必须手动选, 防止自动测速切到被 OpenAI 风控的出口)。
## 客户端导入速查
| 客户端 | 用法 |
|--------|------|
| 官方 SFW GUI | 下载 JSON 改名 config.json 放 SFW 目录, SFW.bat 启动; 或 sub.txt 填订阅链接。概述页打开「系统 HTTP 代理」开关 (SFW 无 TUN, 必须走系统代理 127.0.0.1:2080) |
| 手机 SFA/SFI/Karing | 直接添加订阅链接 — 自动识别手机 UA 下发 TUN 版配置, 全局接管免任何手动设置 |
| Karing / Hiddify | 订阅链接添加 (Sing-box 类型), 或导入 JSON |
| Clash Verge / mihomo | 导入 warp-clash-multi.yaml |
| V2rayN | 一键复制链接 → 「服务器→从剪贴板导入批量URL」, 内核切 sing-box |
| WireGuard 手机版 | 解压 zip 内 .conf 逐个导入 |
