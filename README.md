# distUsageProbe
**声明：本项目完全由AI完成，包括宣传视频，介意的可以无视**

# 用量面板 · Usage Panel

> 请求级真实用量分析：KPI、活跃热力图、模型分时分布、请求大小分布与三级下钻。  
> 数据全部来自你自己机器上的本地记录，**一个字节都不上传**。

给 WorkBuddy 加一个侧边栏面板，把「AI 到底花了多少钱、花在哪」这件一直说不清的事，变成一本**能翻的明细账**。

![九块视图](https://img.shields.io/badge/视图-9_块-6630F8)

![三级下钻](https://img.shields.io/badge/下钻-3_级-6630F8)

![隐私](https://img.shields.io/badge/数据-仅本机-8FF740)



---

## 它能看到什么

| 视图           | 内容                                                       |
| ------------ | -------------------------------------------------------- |
| **KPI 七指标**  | 合计消耗、日均、会话数、轮次数、输入输出比、缓存命中率、预估金额（按 1000 credits ≈ 1 元折算） |
| **当前会话**     | 上下文 token 占用与「还能聊多久」                                     |
| **账户额度**     | 直连官方计费接口拉取，剩余 / 已用 / 冻结一屏看清                              |
| **活跃热力图**    | 颜色越亮 = 那个时间桶消耗越大，粒度 1 分钟 ~ 20 分钟可切                       |
| **0–24 时分布** | 山脊图，每条曲线是一个模型，悬停读出该小时谁花了多少                               |
| **每日堆叠柱**    | 按天 × 模型拆开，点模型卡片可只看它                                      |
| **请求大小分布**   | 单次请求有多大，P50 / P90 各在哪                                    |
| **三级下钻**     | 总览 → 工作空间 → 会话 → 单次请求，每行带聊天预览                            |
| **最近请求**     | 任意一列可排序                                                  |

**读的是每一次请求的真实用量，不是估算。**

---

## 环境要求

- **WorkBuddy**（桌面版，需支持第三方扩展的版本）
- ~~Node.js~~ **不需要装**。服务端跑在 WorkBuddy 自带的运行时上，且只用到 `http` / `fs` / `path` / `os` 等 Node 内置模块，**零外部依赖**。
- ~~API Key~~ **不需要**。账户额度走 WorkBuddy 自己的登录态直连官方计费接口。

---

## 安装

### 1. 克隆

```bash
git clone https://github.com/tmmovo/distUsageProbe.git
```

### 2. 放进扩展目录

WorkBuddy 启动时会扫描扩展目录，**任何含有 `extension.json` 的子目录都会被识别为一个扩展**。

把克隆下来的文件夹（确认里面直接就是 `extension.json`，不是多套了一层同名目录）复制到：

| 系统            | 路径                                                 |
| ------------- | -------------------------------------------------- |
| Windows       | `%USERPROFILE%\.workbuddy\extensions\usage-probe\` |
| macOS / Linux | `~/.workbuddy/extensions/usage-probe/`             |

命令行示例（Windows PowerShell）：

```powershell
git clone https://github.com/tmmovo/distUsageProbe.git "$env:USERPROFILE\.workbuddy\extensions\usage-probe"
```

> **注意目录名**：文件夹名必须叫 `usage-probe`，且**不能**多包一层。  
> 正确：`.workbuddy/extensions/usage-probe/extension.json`  
> 错误：`.workbuddy/extensions/distUsageProbe/distUsageProbe/extension.json`
>
> 实在不想动命令行，也可以直接下载 ZIP 解压后手动拖过去 —— 效果一样。

### 3. 重启 WorkBuddy

**必须重启。** 扩展扫描发生在启动阶段，装完不重启是看不到的。

重启后左侧边栏应出现「用量面板」。

---

## 确认装成功了

打开面板，右上角应显示 **「服务端在线」**。

如果显示「服务端离线」或者面板一片空白，按下面顺序排查：

<details>

<summary><b>排查清单（点开）</b></summary>

**① 目录结构**

```bash
# 应该能打印出内容
cat ~/.workbuddy/extensions/usage-probe/extension.json
```

打印不出 → 文件夹套错层了，回到第 2 步。

**② 扩展有没有被注册**

在 WorkBuddy 日志（`%USERPROFILE%\.workbuddy\logs\daemon.log`，macOS/Linux 在  
`~/.workbuddy/logs/daemon.log`）里搜 `ext-perm`，应该能看到这一行：

```
[ext-perm] registered usage-probe kind=builtin perms=[*]
```

看到这行就说明**扩展已经被识别并注册**。搜 `ext-scan` 能看到本机所有被扫到的扩展清单。

**③ 服务端有没有起来**

```bash
curl http://127.0.0.1:39119/api/health
```

正常会返回：

```json
{"ok":true,"extensionId":"usage-probe","version":"0.6.4","rssMb":35,
 "counts":{"requests":425,"quota":1028},"auth":"token-required"}
```

默认端口是 `39119`；被占用时会自动换端口并写进  
`~/.workbuddy/extensions/usage-probe/server/.port`。

**④ 看服务端自己的日志**

面板连不上时，服务端会把每个请求记到：

```
~/.workbuddy/usage-panel-data/access.log
```

这一行告诉你请求有没有到、Origin 是什么、有没有带令牌：

```
2026-… denied /api/analytics tok=NOTOK origin=http://127.0.0.1:39991 …
2026-… ok      /api/analytics tok=36c71b  origin=wb-extension://usage-probe …
```

`denied` 说明令牌没读到或已过期；`ok` 说明服务端认了这个调用。

</details>

---

## 数据与隐私

面板采集的数据全部写在**你自己的机器上**：

```
~/.workbuddy/usage-panel-data/
├── requests.jsonl    每次请求一行
├── quota.jsonl       额度快照
├── credits.jsonl     credits 账单
├── context.jsonl     上下文占用
├── diag.jsonl        额度接口诊断
└── access.log        服务端访问日志
```

- **不联网**：服务端只监听 `127.0.0.1`，所有接口都要带访问令牌（启动时随机生成，存在  
  `server/.token`），未授权一律 401。
- **不上传**：没有出站请求。唯一的对外访问是 WorkBuddy 自己去官方计费接口查你的额度。
- **可随时删除**：删掉整个 `usage-panel-data/` 目录即可，WorkBuddy 会重新建。

> ⚠️ 这些文件里含有**你机器上的对话预览和路径**。别把它们传到任何地方。

---

## 更新

```bash
cd ~/.workbuddy/extensions/usage-probe
git pull
```

然后重启 WorkBuddy。

---

## 卸载

```bash
rm -rf ~/.workbuddy/extensions/usage-probe          # 删扩展
rm -rf ~/.workbuddy/usage-panel-data                 # 顺手删数据（可选）
```

重启 WorkBuddy 后侧边栏不再显示。

---

## 工作原理

```
┌─ WorkBuddy 宿主 ────────────────────────────┐
│                                              │
│  侧边栏 UI (ui/assets/remoteEntry.js)       │
│      │  Module Federation                    │
│      │  读 wb-extension:// 协议下的资源       │
│      ▼                                       │
│  HTTP  127.0.0.1:39119  (?t=<token>)         │
│      ▲                                       │
│      │  stdio 生命周期事件                    │
│  服务端 (server/index.cjs)                   │
│      └─ 逐行流式扫描 usage-panel-data/*.jsonl │
│         聚合成 KPI / 热力图 / 分布 / 下钻       │
└──────────────────────────────────────────────┘
```

**几个设计上的取舍：**

- **逐行流式扫描**，不把整个文件读进内存 —— 数据量再大内存也平稳（实测常驻约 78 MB）。
- **索引文件**加速常用查询，避免每次打开都重扫。
- **30fps → 60fps**：UI 素材是 30fps，渲染时用慢放铺满镜头而不是插帧，保持画面干净。
- **零硬切**：所有元素出场都走统一封装 `<Enter>`（淡入 + 位移 + 可选缩放），  
  禁止裸写 `opacity: 0→1`。
- **不建不透明黑底**：镜头组件不画黑底，保留幕底的星点与雾底。

---

## 开发者：改完怎么生效

改 `ui/assets/*.js` 或 `server/index.cjs` 之后：

1. **UI 改动**：切走再切回侧边栏即可（资源是运行时从磁盘读的）
2. **服务端改动**：需要重启 WorkBuddy（子进程是 fork 出来的，改代码不会热更新）
3. 想看效果而不动正式目录，可以装到开发目录，优先级更高、不会覆盖正式版：

```bash
git clone https://github.com/tmmovo/distUsageProbe.git ~/.workbuddy/extensions-dev/usage-probe
```

扩展扫描优先级：`extensions-dev` > `extensions` > 内置。

---

## 许可

MIT
