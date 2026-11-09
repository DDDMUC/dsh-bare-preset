# dsh-bare-preset

「裸模模式」agent 预设，做成一个**插件**（自己向 `agentPresets` 注册），顺手把宿主强加给 agent 的运行时开销一并剪掉。

## 安装

```
npm:  dsh-bare-preset@0.1.1
repo: https://github.com/DDDMUC/dsh-bare-preset
```

装法二选一：

1. **插件管理器**里搜 `dsh-bare-preset`，装上启用（这会往 profile 的 `dependencies` + `dsh.profile.bundles` 里登记）。
2. 手工改 `~/.dsh/profiles/<profile>/package.json`：`dependencies` 加
   `"dsh-bare-preset": "0.1.1"`（或 `"github:DDDMUC/dsh-bare-preset"`），
   `dsh.profile.bundles` 数组里加 `"dsh-bare-preset"`，然后重启。

重启后预设选择器里会出现 **「裸模模式」**（id `bare`），跟它开一个新会话即可。

## 为什么不用预设目录 + bundle patch

那些坑都踩过：bundle patch 里的行要注册进 registry，而 registry 用它**自己的** baseUrl 解析相对路径（`./persona-file.mjs` 直接 import 失败，预设从选择器里消失）；`id` 必须是字符串；声明归属还要和「预设工作台」的运行时声明抢所有权。

现在这个包自己做主：激活时 `presets.register()`，卸载时注销。组合里的提示词行用包内绝对 file URL，没有任何路径歧义。

## 剪掉的三层

| 层 | 谁加的 | 手段 | 省 |
|---|---|---|---|
| 系统提示词 | — | 人设行 `complete: true` + 空文本，`suppressRuntimeContext()` | 全部 |
| 全局工具 | 宿主各插件 | `tools.restrict({ allow: [] })`（persona 行） | 全部 |
| agent 作用域工具（Agent Teams 9 个） | `tool-agent-team` 用 `agent.ctx` 注册 | 把注册对象的 `when` 换成恒假，`resolve()` 的 admit 阶段剔除 | ~1.5K token |
| `team:identity`、`working-directory:current` | 同上 / `working-directory` | `required` 翻 false（全局那条在本层放 `required:false` 的影子） | ~90 token |

最后一项全部滤掉后，`joinContextSections([])` 是空串，`project()` 返回 `undefined`，那条 runtime-context user 消息整条不出现。注意**不能**在 `agent/pre-step` 里删它 —— agent-loop 在 append 之前会 `refreshContext` + `project` 重新算一遍，删的和追加的不是同一份。

## 开关

`lib/index.js` 顶部：

- `PRUNED_TOOL_NAMES` —— 要剪掉的 agent 作用域工具（默认是 Agent Teams 那 9 个，清空数组即可保留）
- `SUPPRESSED_CONTEXTS` —— 要滤掉的 runtime context 名（清空即保留 cwd / 团队身份）
- `noTools`（组合里 persona 行的 config）—— 关掉就不做全局工具 restrict

## 预期效果

同一 profile 下 `standard` 会话首步约 19K token；本预设约 **15–20 token**（你的那句话 + 服务端角色标记）。
## 备注

- `tools.restrict` / `when` / `required` 三处都是改注册对象本身（registry 返回的是真身，不是副本），不落盘；插件重载后由 `agent/created` 与 `turn/start` 重新施加。DSH 升级若改了内部结构，表现是"工具/上下文又冒出来"，不会崩。
- 预设 id 是 `bare`。
