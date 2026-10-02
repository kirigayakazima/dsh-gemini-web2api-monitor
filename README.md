# @dsh-external/gemini-web2api-monitor

监控 [gemini-web2api](https://github.com/Sophomoresty/gemini-web2api) 反代服务的**模型降级**状态：
读上游权威字段 `slot39`（而非可能造假的模型标签），定时检测，并在 DSH 侧边栏给出状态面板与告警。

## 为什么需要它

Gemini 网页版的会话靠 `__Secure-1PSIDTS` 这类**轮换 cookie**维持。它一过期，反代会**静默降级**——
你请求 `gemini-3.8-flash`，上游实际给你 `Flash-Lite`，但 HTTP 依然是 200、对话依然"正常"。
只看响应根本发现不了，而输出质量已经掉了。

本插件定期向 gemini.google.com 发一个 `StreamGenerate` 诊断请求，解析响应里的：

| 字段 | 含义 |
|---|---|
| `slot39` | 上游**实际服务**的模型内部 ID（权威，防标签造假） |
| `slot42` | 上游模型标签（可能造假，仅供参考） |

判断规则：
- `slot39 === <期望ID>`（如 `56fdd199312815e2` = 3.8 Flash）→ ✅ 正常
- `slot39 === cf41b0e0dd7d53e5`（上游"拒绝模型"标记）→ ❌ 已降级，通常是 cookie 过期

## 安装（DSH）

作为第三方 bundle 挂进某个 profile。以 `desktop` profile 为例，`package.json`：

```json
{
  "dependencies": {
    "@dsh-external/gemini-web2api-monitor": "file:D:/CodePackage/DSPlug/gemini-web2api-monitor"
  },
  "dsh": {
    "profile": {
      "bundles": [
        "@deepseek-ai/dsh-base",
        "@deepseek-ai/dsh-web-app",
        "@dsh-external/gemini-web2api-monitor"
      ]
    }
  }
}
```

本仓库的 `cordis.patch.yml` 提供挂载点：

```yaml
- insert:
    - id: gemini-web2api-monitor
      name: "@dsh-external/gemini-web2api-monitor"
```

也可以把目录 junction 到 profile 的 `node_modules/@dsh-external/` 下（Windows）：

```powershell
cmd /c mklink /J "<profile>\node_modules\@dsh-external\gemini-web2api-monitor" "<本仓库路径>"
```

## HTTP 端点

插件通过 `webServer` 注册两个只读端点：

| 端点 | 说明 |
|---|---|
| `GET /api/gemini-web2api-monitor/status` | 返回最近一次检测结果与历史 |
| `GET /api/gemini-web2api-monitor/probe` | 立即触发一次检测后返回状态 |

`status` 响应示例：

```json
{
  "ok": true,
  "checkedAt": "2026-10-01T09:02:28Z",
  "modelLabel": "3.8 Flash",
  "slot39": "56fdd199312815e2",
  "expectedId": "56fdd199312815e2",
  "degradedSince": null,
  "degradedCount": 0,
  "cookieHasTs": true,
  "lastReason": "timer#1"
}
```

## 配置

| 字段 | 默认值 | 说明 |
|---|---|---|
| `intervalMs` | `600000` | 检测间隔（10 分钟；太频繁会触发上游限流） |
| `geminiDir` | `D:/CodePackage/DSP/gemini-web2api` | 反代项目目录（读它的 `config.json` 与认证文件） |
| `modelId` | `56fdd199312815e2` | 期望的上游模型内部 ID（3.8 Flash） |
| `label` | `3.8 Flash` | 期望模型的展示名 |
| `stateFile` | `$DSH_HOME/super-injector/gemini-web2api-monitor-state.json` | 状态落盘位置 |
| `logFile` | `$DSH_HOME/super-injector/gemini-web2api-monitor.log` | 日志位置 |
| `proxy` | `http://127.0.0.1:7890` | 访问上游用的 HTTP 代理 |

> 探针请求委派给同目录的 `probe.py`（Node 的 TLS 栈过不了 Clash 一类的代理）。

## ⚠️ 一个必须避开的坑：不要导出 `Config = null`

历史版本曾把 `Config` 手改为 `null` 以规避 peerDep `schemastery`：

```js
// 千万不要这样写
const Config = null
export { name, inject, Config, apply }
```

DSH 的 `SettingsForms.schema()` 会执行：

```js
return schema !== undefined && 'toJSON' in schema ? schema : undefined;
```

可选链**只对 `undefined` 短路**，`null` 会真的执行 `'toJSON' in null`，抛
`TypeError: Cannot use 'in' operator to search for 'toJSON' in null`。
后果是 `/api/settings/describe` 整条 RPC 失败 → 桌面欢迎窗口报
`desktop welcome: Web RPC failed` → 触发 `DesktopFatalRecovery`
「禁用全部第三方插件，备份 profile patch，重启」，用户的 profile 配置会被重置。

**正确做法**：不导出 `Config`（见 `lib/index.js` 的 `export { name, inject, apply }`）。
运行时也不需要该 schema —— `apply()` 内部用 `cfg` 兜底默认值。

## 构建

```bash
DSH_CHECKOUT=<dsh 源码检出> bash scripts/build.sh
```

`lib/` 是实际运行产物（宿主 `index.js` + 浏览器端 `client.js`），**已纳入版本控制**，
因此 clone 下来即可直接挂载运行，不需要先构建。

## 相关仓库

- [`dsh-gemini-web2api-toolkit`](https://github.com/kirigayakazima/dsh-gemini-web2api-toolkit)
  —— 反代补丁 + 认证预检工具 `verify_auth.py` + 服务管理脚本

## 许可

MIT
