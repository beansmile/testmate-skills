# 更新 widget（`/testmate update`）/ Update Widget

按顺序执行；任何命令失败时停止并报告失败的步骤。

## 硬性规则 / Hard Rules

- 不要读取或打印 `~/.makethisbetter/config.json`（含 bearer token）。用 `makethisbetter info` 查看脱敏状态。
- 只询问项目**尚未配置**的新选项；已配置的保持不变，除非用户明确要求修改。
- **例外：** `apiUrl` 缺失或不是 `https://testmate.beansmile.ai/api/v1` 时，这是错误而非可选项——报告并修正（否则反馈会发往 makethisbetter.dev）。
- Signing Secret 不得出现在源码、浏览器代码、聊天、总结或日志中。
- 不要提交测试反馈。

## 1. 前置检查

```bash
command -v makethisbetter && makethisbetter project list --json >/dev/null
```

任一失败 → 提示先运行 `/testmate setup` 并停止。

## 2. 定位现有安装

```bash
grep -rn "MakeThisBetter.init(" --include="*.js" --include="*.ts" --include="*.erb" --include="*.html" --include="*.vue" --include="*.svelte" --include="*.tsx" --include="*.jsx" --include="*.liquid" --include="*.php" .
```

依次查看：`app/javascript/`、`app/assets/`、`app/views/layouts/`、`src/` / `pages/` / `components/`、`*.html` / `*.liquid` / `*.php` / `*.jsp`。Turbo 安装中配置可能在 `meta[name="makethisbetter-config"]` 的 `data-*` 属性里。未找到 `init(` → 提示运行 `/testmate setup` 并停止。记录文件路径和完整配置对象。

## 3. 解析当前配置

| 选项 | 类型 | 默认值 |
|---|---|---|
| `projectKey` | string | *(必填)* |
| `apiUrl` | string | SDK 默认指向 makethisbetter.dev；**TestMate 必填** `https://testmate.beansmile.ai/api/v1` |
| `entryMode` | `'button'` \| `'api'` | `'button'` |
| `locale` | string | `document.documentElement.lang ?? 'en'` |
| `position` | `'left'` \| `'right'` | `'right'` |
| `theme` | `'light'` \| `'dark'` \| `'auto'` | `'auto'` |
| `frustrationDetection` | boolean | `true` |
| `tabText` | string | SDK 本地化默认 |
| `brandColors` | `{ primary, hover, active, onPrimary }` | SDK 默认 |
| `userToken` | string | `undefined` |
| `userTokenFn` | `() => Promise<string>` | `undefined` |
| `user` | `{ id?, email?, name? }` | `undefined` |

得到 **configured**（已有键）与 **available**（未配置、可新增的键，不含 `projectKey` / `apiUrl`）。先检查 `apiUrl`；**available** 为空则报告配置已是最新，跳到第 5 步。

## 4. 推断并确认新选项

用与 `setup.md` 第 5.2 步相同的信号推断：

| 选项 | 信号 | 建议 |
|---|---|---|
| `locale` | `config/locales/`、i18n 库、`<html lang>` | 单语言设该语言；多语言插值当前 locale |
| `theme` | 主题切换、`dark:`、`prefers-color-scheme` | 支持暗色保持 `auto`；仅亮色设 `light` |
| `position` | Intercom/Crisp/Chatwoot/Drift 等右下角控件 | 冲突设 `left` |
| `tabText` | 产品既有用语 | 默认保持本地化文案 |
| `brandColors` | 完整的四个语义色 | 四个都存在才推荐，列出值与来源；不生成/推算缺失色 |
| `frustrationDetection` | 无需信号 | 保持默认 `true`，说明其检测内容 |
| `entryMode` | 宿主已有自定义触发按钮 | 仅此时推荐 `'api'` |
| `userToken` / `userTokenFn` | Devise、NextAuth、Clerk、session | 推荐 `/testmate setup-auth` |
| `user` | 客户端已暴露未签名用户信息 | 仅此时推荐 |

用**一个**确认问题列出新选项：名称、SDK 默认值、推荐值及依据。默认值已合适的标为「保持 SDK 默认」，不写入 `init()`。用户可逐项接受、拒绝或给出明确值。无效的 `brandColors` 组会回退为 SDK 默认。

## 5. 检查 widget 版本

Import Map / vendored：

```bash
head -5 vendor/javascript/makethisbetter.js 2>/dev/null
npm view makethisbetter version
```

npm 安装：

```bash
node -p "require('makethisbetter/package.json').version" 2>/dev/null || \
  jq -r '.dependencies.makethisbetter // .devDependencies.makethisbetter' package.json
npm view makethisbetter version
```

CDN：确认使用主版本固定的 `https://unpkg.com/makethisbetter@1`（自动获得最新 minor，无需更新）。

落后时报告并建议更新。Import Map：

```bash
curl -sL "https://unpkg.com/makethisbetter@<VERSION>/dist/makethisbetter.standalone.js" -o vendor/javascript/makethisbetter.js
head -1 vendor/javascript/makethisbetter.js | grep -qE '^const |^var |^import |^/[/*]' || { echo "Downloaded content is not JS, CDN may not have synced"; exit 1; }
```

npm：`npm update makethisbetter`。

## 6. 应用修改

就地编辑 `init()`，保留 `projectKey`、已有选项、代码风格与周边代码。只写非默认值（`apiUrl` 必须存在）：

```diff
 MakeThisBetter.init({
   projectKey: "mtb_proj_xxx",
+  apiUrl: "https://testmate.beansmile.ai/api/v1",
   locale: "zh-CN",
+  position: "left",
 })
```

## 7. 验证

1. `init()` 语法有效，无重复键，`apiUrl` 正确。
2. 有开发服务器与浏览器工具时，加载页面检查 widget 控制台错误，并确认请求发往 `https://testmate.beansmile.ai/api/v1`。

收尾：修改的文件、新增选项、保持默认（未写入）的选项、版本状态、`apiUrl` 状态，以及其他建议（如 `/testmate setup-auth`）。

---

### English summary

Locate the existing `MakeThisBetter.init(` call, make sure `apiUrl` is
`https://testmate.beansmile.ai/api/v1` (fix it if missing — that is a bug, not
an option), infer recommendations only for options not yet configured, confirm
them in one question, check the installed `makethisbetter` version against npm
latest (`@1` CDN needs nothing), apply the edits in place, and verify. Never read
the CLI config file and never submit test feedback here.
