# Agent Note: session-telemetry-otel 移除内置 endpoint 缺省值

Status: implemented

[English](2026-08-22-telemetry-endpoint-requires-explicit-url.md) | 中文

## 问题

一次针对出厂默认配置的安全审查（[2026-07-31 默认挂载 Note](../feature/2026-07-31-web-telemetry-default-mount.md)）发现，`DSH_TELEMETRY_MODE` 与 `DSH_TELEMETRY_OTLP_URL` 并非两个相互独立的授权项：共享基础组合包中的 `exporter.url` 在未设置覆盖值时，会缺省指向一个固定的、由 DeepSeek 运营的 collector（`https://harness-telemetry.deepseeksvc.com/v1/logs`）。该 Note 自身的“后果”一节已经披露，一旦启用导出即为原始捕获副本——用户/助手消息全文、工具参数与工具结果、系统提示词、`session.cwd` 本地路径，这是其“未挂载脱敏规则”的裁决所致。因此仅设置 `DSH_TELEMETRY_MODE=FULL` 一项，就足以把这份原始副本发送到一个部署方从未单独选择过的目的地。`DSH_TELEMETRY_MODE` 本身是受信任的、由操作方设置的环境变量——这里不存在攻击者可触达的路径——但把“授权上传”与“默认发到这里”耦合在一起，对于一个本应只在目的地被刻意选定后才让真实会话内容上线的路径而言，是不必要的风险。

## 决策

共享基础组合包（`packages/bundle/base/cordis.patch.yml`）不再为 `exporter.url` 提供缺省值；它现在原样读取 `DSH_TELEMETRY_OTLP_URL`，未设置时解析为 `undefined`。`@deepseek-ai/dsh-session-telemetry-otel` 自身既有的加载期校验（`session-telemetry-otel/src/index.ts` 中 `DISABLED` 模式提前返回之后的分支）在解析出的模式为 `FULL` 或 `FEEDBACK_ONLY` 且缺少 URL 时，本就会抛出 `exporter.url is required (the full OTLP logs endpoint)`——本 Note 移除的是让该校验从未触发过的组合层缺省值。插件代码未作任何改动；修复完全落在共享组合包的配置取值上。

`DISABLED`（出厂默认，依据[默认关闭 Note](../feature/2026-08-10-telemetry-default-off.md)）不受影响：构造函数在该分支下会在读取 URL 之前直接返回，因此全新 profile 依然不发出任何遥测网络请求,也无需配置任何 URL。启用上传现在要求在同一启动环境中同时设置 `DSH_TELEMETRY_MODE` 与 `DSH_TELEMETRY_OTLP_URL`——两个各自独立、都需显式启用的决策，而不是一个开关顺带决定另一个。

本 Note 仅部分取代[默认挂载 Note](../feature/2026-07-31-web-telemetry-default-mount.md)中的 endpoint 一项裁决；该 Note 中关于模式默认关闭、硬性退出变量、上报节奏、退出 drain 上界、压缩、CI 隔离的裁决均未改变，仍是这些事实的现行依据，并已就地更新为指向本 Note 的交叉引用。

## 考虑过的替代方案

**在 URL 缺失时只发出警告，而非直接失败。** 不采用：在 headless 或批处理部署中，一条日志很容易被忽略,而这里的核心诉求正是——真实会话内容不应在部署方未单独选定目的地的情况下离开本机；警告仍然放行了这次上传。

**不改动 endpoint 缺省值，转而增加脱敏规则。** 不采用，这是另一个更大的问题：脱敏能缩小一个仍然隐式存在的目的地所收到的内容，但无法解决“目的地本身从未是操作方在只设置模式时做出的决策”这一点。[默认挂载 Note](../feature/2026-07-31-web-telemetry-default-mount.md)已经把缺失的脱敏规则记录为独立的、仍待完成的部署工作。

**为 URL 增加白名单校验，而非仅要求其非空。** 不采用，属于冗余：现有加载期校验已经要求操作方提供自行填写的 http(s) URL（schemastery `Config`，在构造函数中校验）；白名单只会为一个操作方本就亲手输入的值增加第二套信任配置面。

## 后果

- `DSH_TELEMETRY_MODE=FULL` 或 `FEEDBACK_ONLY` 且未设置 `DSH_TELEMETRY_OTLP_URL` 时，现在会在插件构造期直接 fail-loud，而不再默默上传到 DeepSeek 运营的 collector。任何此前依赖该隐式缺省值的部署——包括依据默认挂载 Note 中“内部 collector”表述、DeepSeek 自身的部署——现在必须在设置 `DSH_TELEMETRY_MODE` 的同时显式设置 `DSH_TELEMETRY_OTLP_URL`。这是仅针对该隐式缺省值场景的破坏性变更。
- 对出厂默认值（`DISABLED`）或此前已同时设置两个变量的部署,行为均无变化。
- 基础组合包测试（`packages/bundle/base/tests/base.spec.ts`）只固定了 `mode` 表达式，未涉及 `exporter.url`，因此无需改动；仓库中没有任何测试针对真实基础组合包设置过 `DSH_TELEMETRY_MODE=FULL`，这印证了没有测试依赖被移除的缺省值。
