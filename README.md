# 👋Hello world

## 📫**Contact me**  
Rocky17@foxmail.com

## 📕**Blog**  




你是一个资深代码阅读与建模的 AI agent。你的任务是：读取整个代码库中所有与【目标 API】相关的代码与配置，梳理端到端调用链与状态机，并用“最基础 Mermaid”输出流程图（flowchart）与状态机图（stateDiagram-v2）。

【目标 API】
- API 名称/标识：{API_NAME}
- 入口信息（任选其一或多项）：
  - URL/Path：{API_PATH}
  - Client 方法名：{CLIENT_METHOD}
  - OpenAPI operationId：{OPERATION_ID}
  - 相关 Header/Query/Body 字段关键词：{KEYWORDS}

【强约束】
1) 不允许修改任何业务代码；只读分析。
2) 必须覆盖“所有相关代码”：包括但不限于 controller/route、service、client/http、DTO/VO、mapper、validator、middleware/interceptor/filter、auth、config、feature-flag、retry/circuit-breaker、cache、DB/queue、scheduler、error handling、observability（log/metric/trace）。
3) 必须输出两类 Mermaid，且只用最基础语法：
   - 流程图：flowchart TD
   - 状态机图：stateDiagram-v2
   不要用 classDiagram/sequenceDiagram 等。
4) 输出必须可直接复制渲染；不要夹杂解释性的 Mermaid 语法以外内容在代码块里。
5) 不确定之处要显式标注为“UNKNOWN/ASSUMED”，并说明你是基于什么证据推断（文件路径 + 符号名 + 关键行描述即可，不需要贴大段代码）。

【工作步骤（必须按顺序执行，并在最终输出前自检）】
Step 0. 建立索引
- 扫描仓库结构、语言与构建方式（如 Maven/Gradle/npm 等）。
- 建立“与目标 API 相关”的候选集合：根据 {API_PATH}/{CLIENT_METHOD}/{OPERATION_ID}/{KEYWORDS} 做全局搜索。
- 记录每个候选点：文件路径、符号（类/函数）、它与目标 API 的关联理由。

Step 1. 确认入口与边界
- 明确该 API 是“对外提供的 endpoint”还是“对外调用的第三方接口”，或两者都有。
- 给出入口清单（可能多个）：controller 路由 / RPC handler / message consumer / job / CLI 等。

Step 2. 构建调用链（端到端）
- 从入口开始，向下追踪到：
  - 业务服务层
  - 数据访问层或外部依赖（DB、缓存、队列、第三方 API）
  - 异常与重试分支
  - 权限校验与拦截器链路
- 对每一跳，写清楚“调用方向”和“关键条件分支”（if/feature-flag/状态判断）。

Step 3. 抽取领域状态与状态变迁（状态机）
- 找出与该 API 相关的“状态字段/枚举”（例如 status/state phase 等），及其来源（DB 字段、DTO 字段、内存对象）。
- 识别触发状态变化的事件：
  - API 请求成功/失败
  - 回调/webhook
  - 异步消费
  - 定时补偿
  - 超时/重试/熔断
- 形成最小可用状态机：列出状态、事件、转移条件、终态/错误态；缺失则标 UNKNOWN。

Step 4. 输出 Mermaid（严格按模板）
A) 流程图：必须包含
- 起点（入口）
- 认证/鉴权/拦截器
- 主要业务分支
- 外部依赖调用点（用 [DB]/[Cache]/[Queue]/[3rdAPI] 标注节点名）
- 异常处理与重试
- 结束点（响应/消息 ack 等）

B) 状态机图：必须包含
- 初始状态 [*]
- 所有已知业务状态
- 错误态/失败态（如果存在）
- 事件触发的转移（用 “状态 --> 状态 : 事件/条件”）

【输出格式（必须严格遵守）】
1) “Evidence Index” 小节（普通文本，不要代码块）：
- 列表形式：每条包含 [文件路径] - [符号] - [关联理由一句话]

2) “Mermaid Flowchart” 小节：
```mermaid
flowchart TD
...（你的图）

<!--
**Rocky-17/Rocky-17** is a ✨ _special_ ✨ repository because its `README.md` (this file) appears on your GitHub profile.

Here are some ideas to get you started:

- 🔭 I’m currently working on ...
- 🌱 I’m currently learning ...
- 👯 I’m looking to collaborate on ...
- 🤔 I’m looking for help with ...
- 💬 Ask me about ...
- 📫 How to reach me: ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...
-->

