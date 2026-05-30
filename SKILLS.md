# One-Person Economy — Operator Skill

Design and build autonomous, agent-driven systems that let a single operator run what previously required a team. Use this skill when building AI agent workflows, persistent memory systems, autonomous pipelines, or any "one-person stack" architecture.

## Core mental model

The operator does not perform tasks — they orchestrate loops. Each loop is:

```
Define goal → Delegate to agent → Review edge cases → Adjust → Ship
```

You are not replacing every specialist. You are reaching the threshold where customers don't leave. Complete beats perfect.

---

## Agents

An agent is a task loop with a model at the center. Design agents to be:

- **Scoped** — one responsibility, clear input/output contract
- **Resumable** — persists state so a crash doesn't lose work
- **Observable** — emits structured logs the operator can skim in seconds

### Minimal agent pattern (TypeScript + Anthropic SDK)

```ts
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

async function runAgent(task: string, context: string): Promise<string> {
  const response = await client.messages.create({
    model: "claude-opus-4-8",
    max_tokens: 8096,
    system: `You are a specialized operator agent. Complete the task and return structured output only.`,
    messages: [
      {
        role: "user",
        content: `Context:\n${context}\n\nTask:\n${task}`,
      },
    ],
  });

  const block = response.content[0];
  if (block.type !== "text") throw new Error("Unexpected response type");
  return block.text;
}
```

### Multi-agent pipeline

Route tasks to specialized agents rather than one general agent:

```ts
type AgentRole = "researcher" | "writer" | "reviewer" | "deployer";

const agents: Record<AgentRole, (input: string) => Promise<string>> = {
  researcher: (q) => runAgent("Research this topic deeply", q),
  writer: (brief) => runAgent("Write production-ready content", brief),
  reviewer: (draft) => runAgent("Review for quality and accuracy", draft),
  deployer: (content) => runAgent("Prepare and format for deployment", content),
};

async function pipeline(topic: string): Promise<string> {
  const research = await agents.researcher(topic);
  const draft = await agents.writer(research);
  const reviewed = await agents.reviewer(draft);
  return agents.deployer(reviewed);
}
```

### Tool use — giving agents hands

Agents become operators themselves when you give them tools:

```ts
const tools: Anthropic.Tool[] = [
  {
    name: "search_web",
    description: "Search the web for current information",
    input_schema: {
      type: "object" as const,
      properties: {
        query: { type: "string", description: "Search query" },
      },
      required: ["query"],
    },
  },
  {
    name: "write_file",
    description: "Write content to a file",
    input_schema: {
      type: "object" as const,
      properties: {
        path: { type: "string" },
        content: { type: "string" },
      },
      required: ["path", "content"],
    },
  },
];

async function agentWithTools(task: string): Promise<string> {
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: task },
  ];

  while (true) {
    const response = await client.messages.create({
      model: "claude-opus-4-8",
      max_tokens: 8096,
      tools,
      messages,
    });

    if (response.stop_reason === "end_turn") {
      const text = response.content.find((b) => b.type === "text");
      return text?.text ?? "";
    }

    if (response.stop_reason !== "tool_use") break;

    const toolUses = response.content.filter((b) => b.type === "tool_use");
    const toolResults: Anthropic.ToolResultBlockParam[] = [];

    for (const toolUse of toolUses) {
      if (toolUse.type !== "tool_use") continue;
      const result = await dispatchTool(toolUse.name, toolUse.input);
      toolResults.push({
        type: "tool_result",
        tool_use_id: toolUse.id,
        content: result,
      });
    }

    messages.push({ role: "assistant", content: response.content });
    messages.push({ role: "user", content: toolResults });
  }

  return "";
}

async function dispatchTool(name: string, input: unknown): Promise<string> {
  // wire to real implementations
  return JSON.stringify({ tool: name, input, result: "dispatched" });
}
```

---

## Memory

Memory is what turns a stateless agent into a persistent operator. Three layers:

| Layer | What it holds | When to use |
|---|---|---|
| **In-context** | Current conversation + task | Single session, short tasks |
| **External (vector)** | Semantic long-term knowledge | Cross-session recall, large knowledge bases |
| **Structured (DB)** | Entities, relations, history | Facts that must be exact, queryable |

### In-context memory with prompt caching

Expensive context (docs, long system prompts) should be cached:

```ts
async function agentWithCache(
  task: string,
  knowledgeBase: string
): Promise<string> {
  const response = await client.messages.create({
    model: "claude-opus-4-8",
    max_tokens: 4096,
    system: [
      {
        type: "text",
        text: knowledgeBase,
        cache_control: { type: "ephemeral" }, // cache the expensive part
      },
      {
        type: "text",
        text: "You are an expert operator. Use the knowledge above to complete tasks precisely.",
      },
    ],
    messages: [{ role: "user", content: task }],
  });

  const block = response.content[0];
  return block.type === "text" ? block.text : "";
}
```

### External memory — semantic search

```ts
interface MemoryEntry {
  id: string;
  content: string;
  embedding: number[];
  metadata: Record<string, unknown>;
  createdAt: Date;
}

class SemanticMemory {
  private entries: MemoryEntry[] = [];

  async store(content: string, metadata: Record<string, unknown> = {}): Promise<string> {
    const embedding = await this.embed(content);
    const entry: MemoryEntry = {
      id: crypto.randomUUID(),
      content,
      embedding,
      metadata,
      createdAt: new Date(),
    };
    this.entries.push(entry);
    return entry.id;
  }

  async recall(query: string, topK = 5): Promise<MemoryEntry[]> {
    const queryEmbedding = await this.embed(query);
    return this.entries
      .map((e) => ({ entry: e, score: this.cosine(queryEmbedding, e.embedding) }))
      .sort((a, b) => b.score - a.score)
      .slice(0, topK)
      .map((r) => r.entry);
  }

  private cosine(a: number[], b: number[]): number {
    const dot = a.reduce((sum, v, i) => sum + v * b[i], 0);
    const magA = Math.sqrt(a.reduce((sum, v) => sum + v * v, 0));
    const magB = Math.sqrt(b.reduce((sum, v) => sum + v * v, 0));
    return dot / (magA * magB);
  }

  private async embed(_text: string): Promise<number[]> {
    // wire to your embedding provider (OpenAI, Cohere, local model)
    return Array.from({ length: 1536 }, () => Math.random());
  }
}
```

### Structured memory — operator state

```ts
interface OperatorState {
  activeTasks: Task[];
  completedToday: number;
  knowledgeUpdatedAt: Date;
  agentOutputs: Map<string, string>;
}

interface Task {
  id: string;
  description: string;
  status: "pending" | "running" | "done" | "failed";
  assignedAgent: string;
  output?: string;
}

class OperatorMemory {
  private state: OperatorState = {
    activeTasks: [],
    completedToday: 0,
    knowledgeUpdatedAt: new Date(),
    agentOutputs: new Map(),
  };

  enqueue(description: string, agent: string): Task {
    const task: Task = {
      id: crypto.randomUUID(),
      description,
      status: "pending",
      assignedAgent: agent,
    };
    this.state.activeTasks.push(task);
    return task;
  }

  complete(id: string, output: string): void {
    const task = this.state.activeTasks.find((t) => t.id === id);
    if (!task) return;
    task.status = "done";
    task.output = output;
    this.state.agentOutputs.set(id, output);
    this.state.completedToday++;
  }

  getPending(): Task[] {
    return this.state.activeTasks.filter((t) => t.status === "pending");
  }
}
```

---

## One-person system patterns

### The overnight loop

The operator defines work before sleeping. Agents run. Operator reviews edges in the morning.

```ts
async function overnightLoop(tasks: string[]): Promise<void> {
  const memory = new OperatorMemory();
  const results: string[] = [];

  for (const task of tasks) {
    const job = memory.enqueue(task, "general");
    try {
      const output = await runAgent(task, "");
      memory.complete(job.id, output);
      results.push(`✓ ${task.slice(0, 60)}`);
    } catch (err) {
      results.push(`✗ ${task.slice(0, 60)} — ${(err as Error).message}`);
    }
  }

  const summary = results.join("\n");
  console.log("\n=== Morning Report ===\n" + summary);
}
```

### Execution leverage — prompt → shipped

Replace synchronous thinking with async pipelines:

| Old model | New model |
|---|---|
| Write copy → review → publish | Agent drafts → operator approves → auto-publish |
| Research topic → synthesize → write | Research agent → synthesis agent → writer agent |
| Bug reported → triaged → fixed | Error hook → diagnosis agent → patch agent → PR |
| Support ticket → human reads → replies | Triage agent → context lookup → draft reply → operator 1-click send |

### Iteration speed

The operator advantage is **cycle time**, not quality per cycle.

```
Team:     idea → meeting → spec → build → review → ship   (weeks)
Operator: idea → prompt → agent → edge review → ship       (hours)
```

Design every workflow to minimize the distance from intent to deployed output.

---

## Checklist

Before shipping a one-person system:

- [ ] Each agent has a single, testable responsibility
- [ ] Expensive prompts use `cache_control: { type: "ephemeral" }`
- [ ] Agent outputs are logged with enough context to debug without re-running
- [ ] Memory layer separates semantic recall from exact fact lookup
- [ ] The overnight loop has a morning summary that surfaces only what needs a human decision
- [ ] Failure paths are silent (retry + log) not blocking (crash + wake operator)
- [ ] The operator's review surface is < 5 minutes for a full day's agent output
