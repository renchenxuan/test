# CopilotKit + LangGraph 调测智能体实施方案（重构版）

## 1. 文档目的

本方案用于统一当前项目的技术路线与落地策略，解决以下问题：

1. 该不该用 CopilotKit + LangGraph。  
2. 前端到底能看到什么、不能看到什么。  
3. 如何满足你业务中的"监测、追踪、交互、任务切换、人工审批"。  
4. 后端语言（Python / Java）如何选。

---

## 2. 业务需求归纳（统一口径）

调测智能体目标可归纳为 5 项核心能力：

1. **MCP 工具调用**：接入示波器、仿真器、波形分析、特性分析工具。  
2. **非线性诊断闭环**：采集 → 分析 → 判断 → 纠偏重试 → 报告。  
3. **状态持久化与快速切换**：多任务并行，随时暂停与恢复。  
4. **人机协作审批**：关键动作必须人工确认后继续。  
5. **前端可观测与可交互**：看到运行过程，能干预流程。

---

## 3. 总体结论

**结论：CopilotKit + LangGraph 是匹配本场景的组合。**

- **LangGraph** 负责后端执行编排：状态图、循环、分支、`thread_id`、checkpoint、interrupt。  
- **CopilotKit** 负责前端交互与运行时承接：聊天、工具调用呈现、共享状态展示、人机交互组件。  

它们不是替代关系，而是"**编排层 + 交互层**"的分工协作。

### 核心能力映射（5 项业务 → 技术实现）

| 业务能力 | 实现路径 | 关键技术点 |
|---------|---------|-----------|
| MCP 工具调用 | LangGraph + MCP Python SDK | `@modelcontextprotocol/sdk` |
| 非线性诊断闭环 | LangGraph StateGraph | `conditional_edge`、`update_state` |
| 状态持久化/切换 | CheckpointSaver + thread_id | `MemorySaver` / `PostgresSaver` |
| 人机协作审批 | interrupt + 前端 Approval UI | `Command(resume=...)` |
| 前端可观测 | useCoAgent + shared state | `onStateChanged`、`nodeName` |

---

## 4. 关键认知（避免误解）

### 4.1 CopilotKit 是否能看到后端节点信息？

**能看到节点上下文事件（如当前 step/node），但默认不会自动画出完整 DAG 拓扑图。**

换句话说：

- 可以做"当前节点/阶段"的前端展示。  
- 不等于自动获得 LangGraph Studio 那种图形化工作流拓扑。

### 4.2 双向调用机制是否成立？

成立。Runtime 会把前端能力与后端能力都统一为可调用能力：

1. **前端 → Agent**：`useFrontendTool/useCopilotAction` 注册后，Agent 可发起调用。  
2. **Agent → 前端/后端**：Agent 的 tool call 由 Runtime 路由到浏览器或服务端执行，再把结果回传 Agent。

---

## 5. 代码证据（来自整个仓库）

以下证据来自 `C:\Users\t13342\CopilotKit-main` 全仓库，不限单一示例：

1. LangGraph 事件带节点名进入 Runtime：  
   `packages/runtime/src/agents/langgraph/event-source.ts`（`node_name -> nodeName`）

2. React 侧可订阅 step/run 事件得到当前节点：  
   `packages/react-core/src/hooks/use-agent-nodename.ts`

3. CoAgent 返回 `nodeName/threadId`：  
   `packages/react-core/src/hooks/use-coagent.ts`

4. 可按节点名渲染状态卡片：  
   `packages/react-core/src/hooks/use-coagent-state-render.ts`  
   `packages/react-core/src/hooks/use-coagent-state-render-bridge.tsx`

5. 线程切换已在示例中落地（`useThreads + threadId`）：  
   `examples/integrations/langgraph-python-threads/apps/app/src/App.tsx`  
   `examples/integrations/langgraph-python-threads/apps/app/src/components/threads-drawer/threads-drawer.tsx`

6. Python LangGraph 与 CopilotKit 桥接示例：  
   `examples/integrations/langgraph-python/src/app/api/copilotkit/[[...slug]]/route.ts`

### 代码证据验证结果

| 文档说法 | 验证结果 | 代码位置 |
|---------|---------|---------|
| "前端能看到节点上下文事件" | ✅ 正确 | `use-agent-nodename.ts` 订阅 `onStepStartedEvent` 返回 `stepName` |
| "useCoAgent 返回 nodeName/threadId" | ✅ 正确 | `use-coagent.ts:156-160` 明确返回这两个字段 |
| "支持 interrupt 暂停与恢复" | ✅ 正确 | `event-source.ts:166-179` 捕获 `OnInterrupt` / `OnCopilotKitInterrupt` |
| "thread 切换已落地" | ✅ 正确 | `examples/integrations/langgraph-python-threads` 提供完整示例 |
| "Python 路径示例已存在" | ✅ 正确 | `examples/integrations/langgraph-python/` 提供桥接示例 |

### ⚠️ 需注意的能力边界

1. **前端不自动绘制 DAG 拓扑图**  
   CopilotKit 只暴露 `nodeName`（当前节点），完整 DAG 拓扑需要：
   - LangGraph Studio（云端可视化）
   - LangSmith（链路追踪）
   - 自行实现状态可视化组件

2. **HITL 审批流程需自行实现 UI**  
   CopilotKit 提供 interrupt 事件，需要自行实现审批组件：
   ```tsx
   // 示例：监听 interrupt 事件并渲染审批卡片
   const { state } = useCoAgent({ name: "diagnose-agent" });
   if (state.pendingApproval) {
     return <ApprovalCard
       reason={state.approvalReason}
       impact={state.approvalImpact}
       onApprove={() => agent.runAgent({ resume: { approved: true } })}
       onReject={() => agent.runAgent({ resume: { approved: false } })}
     />;
   }
   ```

3. **MCP 工具需通过 LangGraph 封装**  
   CopilotKit 不直接管理 MCP 工具，链路为：
   ```
   LangGraph Agent → tool_call → CopilotKit Runtime → MCP Servers
   ```

---

## 6. 统一架构设计

```text
┌─────────────────────────────────────────────────────────────────┐
│  [React 前端层]                                                  │
│  ├── CopilotKit UI（聊天框/消息流/工具调用卡片）                    │
│  ├── 自定义状态卡片（按 nodeName 渲染）                            │
│  ├── 审批组件（interrupt 触发时呈现）                              │
│  └── 线程切换器（多任务并行切换）                                   │
└─────────────────────────────┬───────────────────────────────────┘
                              │ HTTP POST + SSE
┌─────────────────────────────┴───────────────────────────────────┐
│  [BFF 层 - Node.js Runtime]                                      │
│  ├── CopilotRuntime                                              │
│  ├── 线程管理 API（创建/切换/销毁 thread）                          │
│  ├── 前端工具注册（useFrontendTool）                              │
│  └── 状态转发（shared state ↔ LangGraph state）                   │
└─────────────────────────────┬───────────────────────────────────┘
                              │ LangGraph SDK / REST
┌─────────────────────────────┴───────────────────────────────────┐
│  [Agent 层 - Python LangGraph]                                    │
│  ├── StateGraph（诊断逻辑编排）                                    │
│  │   └── nodes: capture / analyze / diagnose / report            │
│  ├── MCP Client（工具调用）                                       │
│  ├── CheckpointSaver（状态持久化）                                 │
│  └── interrupt（人机审批点）                                       │
└─────────────────────────────┬───────────────────────────────────┘
                              │ MCP Protocol
┌─────────────────────────────┴───────────────────────────────────┐
│  [工具层]                                                         │
│  ├── MCP Servers（示波器/仿真器/波形分析）                          │
│  └── PostgreSQL（checkpoint 存储）+ 对象存储（波形文件）            │
└─────────────────────────────────────────────────────────────────┘
```

### 分层职责说明

| 层级 | 职责边界 | 需实现的代码 |
|------|---------|--------------|
| **前端层** | 展示、交互、订阅状态 | 审批 UI、状态卡片、线程切换组件 |
| **BFF 层** | 协议转换、路由、工具注册 | CopilotRuntime 配置、前端工具注册 |
| **Agent 层** | 业务逻辑、状态机、工具编排 | StateGraph、MCP 封装、interrupt 逻辑 |
| **工具层** | 硬件/软件工具执行 | MCP Server 实现（如已有可跳过）|

---

## 7. 监测 / 追踪 / 交互能力边界（最终版）

| 维度 | CopilotKit | LangGraph |
| :--- | :--- | :--- |
| 监测（运行态） | 消息流、工具调用、共享状态、节点上下文事件 | 节点执行事件源 |
| 追踪（会话） | thread 侧前端管理与切换入口 | `thread_id` + checkpoint + 恢复 |
| 追踪（拓扑） | 不默认绘制 DAG | Studio/LangSmith 级图与链路 |
| 交互（HITL） | 前端审批 UI / 工具 UI | `interrupt` 暂停与恢复 |
| 交互（工具） | 前端工具注册并执行 | 后端工具/MCP 编排执行 |

---

## 8. 贴合业务的端到端示例（波形调测）

### 场景

- 站点 A：执行 `capture_waveform -> run_fft -> diagnose -> report`。  
- 在 `diagnose` 阶段发现高风险，触发 `interrupt` 等待人工审批。  
- 同时站点 B 报警，工程师切换到 B 处理。  
- B 结束后切回 A，从断点继续。

### 前端可观测内容

1. 工具调用过程（采样、FFT、比对标准）。  
2. 共享状态（当前任务摘要、异常标签、建议动作）。  
3. 当前节点上下文（例如正在 `diagnose`）。  
4. 线程列表（A/B），点击即切换对应会话。

### 后端依赖组件

1. LangGraph 会话隔离：`thread_id=station_A / station_B`。  
2. Checkpointer：保存各任务中间状态。  
3. `interrupt`：审批后继续、拒绝则走替代路径。

---

## 9. 后端语言选型结论（Python / Java）

**两者都可行，但成熟度不同：**

1. **Python（推荐）**  
   - LangGraph 生态最成熟，示例与文档最全。  
   - 当前仓库已有 Python 路径与样例，改造成本最低。

2. **Java（可做）**  
   - 需要自行补齐与 Runtime 的协议衔接和工程封装。  
   - 若团队已有强 Java 中台，可作为长期演进方向。

**结论：短中期先 Python 落地，Java 作为企业化扩展选项。**

---

## 10. 实施策略（按阶段）

### 阶段 A：MVP（先打通）

1. 一条完整诊断链路。  
2. 一个 interrupt 审批点。  
3. 两个 thread 的切换与恢复。

### 阶段 B：工程化增强

1. 多 MCP 服务接入（波形/特性/知识库）。  
2. 结构化输出约束（Pydantic 等）。  
3. 指标与日志（耗时、失败率、重试、审批轨迹）。

### 阶段 C：生产化

1. 权限与审批策略分级。  
2. 审计留痕（谁在何时批准何操作）。  
3. 大对象状态分层存储（状态存摘要，原始波形存对象存储）。

### MVP 代码骨架建议

#### 1. LangGraph Agent（Python 侧）

```python
from langgraph.graph import StateGraph, END
from langgraph.types import Command, interrupt
from pydantic import BaseModel
from typing import Optional

class DiagnosisState(BaseModel):
    station_id: str
    waveform_summary: Optional[dict] = None
    risk_level: Optional[str] = None
    pending_approval: bool = False
    diagnosis_result: Optional[str] = None

def diagnose_node(state: DiagnosisState) -> Command:
    # 诊断逻辑...
    risk = analyze_risk(state.waveform_summary)
    
    if risk == "HIGH":
        # 触发人工审批
        approval = interrupt({
            "reason": f"检测到高风险波形",
            "impact": "可能影响设备安全",
            "suggestion": "建议停机检查"
        })
        return Command(
            goto=END,
            update={"pending_approval": True}
        )
    
    return {"diagnosis_result": "正常", "risk_level": risk}

graph = StateGraph(DiagnosisState)
graph.add_node("diagnose", diagnose_node)
# ... 其他节点
```

#### 2. CopilotRuntime 桥接（Node.js 侧）

```typescript
// app/api/copilotkit/[[...slug]]/route.ts
import { CopilotRuntime, copilotRuntimeNodeHTTPEndpoint } from "@copilotkit/runtime";
import { LangGraph讵配 } from "@copilotkit/sdk-js"; // 如有)

const runtime = new CopilotRuntime({
  debug: true,
});

export async function POST(req: Request) {
  const { langGraphCloudEndpoint, headers } = await req.json();
  
  return copilotRuntimeNodeHTTPEndpoint({
    runtime,
    ...langGraphCloudEndpoint && {
      // LangGraph Cloud 连接配置
    }
  })(req);
}
```

#### 3. React 前端（审批组件）

```tsx
// components/ApprovalCard.tsx
import { useCoAgent } from "@copilotkit/react-core";

export function ApprovalCard() {
  const { state, run } = useCoAgent<DiagnosisState>({
    name: "diagnose-agent",
    config: { configurable: { threadId: currentThreadId } }
  });

  if (!state.pending_approval) return null;

  return (
    <div className="approval-card">
      <h3>⚠️ 需要人工审批</h3>
      <p>原因：{state.approval_reason}</p>
      <p>影响：{state.approval_impact}</p>
      <div className="actions">
        <button onClick={() => run({ resume: { approved: true } })}>
          ✅ 批准继续
        </button>
        <button onClick={() => run({ resume: { approved: false } })}>
          ❌ 拒绝并停止
        </button>
      </div>
    </div>
  );
}
```

---

## 11. 风险与控制

1. **工具稳定性**：统一超时、重试、错误码。  
2. **状态膨胀**：只在 state 存引用与摘要。  
3. **多任务混淆**：前端始终显式展示当前 thread。  
4. **人工审批体验差**：审批卡片必须包含原因、影响、可逆性。  
5. **密钥风险**：禁止明文写 key，全部走环境变量/密钥管理。

---

## 下一步行动（立即可做）

### Step 1：跑通基础 Demo

```bash
# 克隆仓库后进入 Python LangGraph 示例
cd examples/integrations/langgraph-python

# 安装依赖
pip install -r requirements.txt

# 启动前端
cd apps/app && npm install && npm run dev

# 配置 .env（参考 .env.example）
OPENAI_API_KEY=sk-...
LANGGRAPH_CLOUD_ENDPOINT=http://localhost:8000  # 本地 LangGraph 服务
```

### Step 2：实现带 interrupt 的状态机

参考上方「MVP 代码骨架建议」中的 Python 示例，在 `diagnose` 节点加入 `interrupt()` 调用。

### Step 3：实现前端审批组件

参考上方「MVP 代码骨架建议」中的 React 示例，订阅 `pending_approval` 状态并渲染审批卡片。

### Step 4：验证线程切换

使用 `examples/integrations/langgraph-python-threads` 中的 `useThreads` + `threadId` 机制，验证多任务并行与恢复。

---

## 12. 最终方案一句话

**以 LangGraph（Python）做后端状态编排与任务恢复，以 CopilotKit 做前端运行时可观测与人机交互，以 MCP 做工具标准接入，以 thread/checkpoint 支撑多任务调测现场的快速切换。**
