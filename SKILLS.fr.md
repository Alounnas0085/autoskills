# Économie de l'Opérateur Solo — Skill

Concevoir et construire des systèmes autonomes pilotés par des agents, permettant à un seul opérateur de faire tourner ce qui nécessitait autrefois toute une équipe. Utiliser ce skill lors de la construction de workflows d'agents IA, de systèmes de mémoire persistante, de pipelines autonomes, ou de toute architecture "stack d'une seule personne".

## Modèle mental de base

L'opérateur n'exécute pas les tâches — il orchestre des boucles. Chaque boucle est :

```
Définir l'objectif → Déléguer à l'agent → Examiner les cas limites → Ajuster → Livrer
```

Il ne s'agit pas de remplacer chaque spécialiste. Il s'agit d'atteindre le seuil à partir duquel les clients ne partent pas. Complet bat parfait.

---

## Agents

Un agent est une boucle de tâche avec un modèle au centre. Concevoir des agents pour qu'ils soient :

- **Délimités** — une seule responsabilité, contrat d'entrée/sortie clair
- **Reprenables** — conserve l'état pour qu'un crash ne perde pas le travail
- **Observables** — émet des logs structurés que l'opérateur peut parcourir en quelques secondes

### Pattern d'agent minimal (TypeScript + Anthropic SDK)

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

### Pipeline multi-agents

Acheminer les tâches vers des agents spécialisés plutôt qu'un agent généraliste :

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

### Utilisation d'outils — donner des mains aux agents

Les agents deviennent eux-mêmes des opérateurs quand on leur donne des outils :

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
  // brancher sur les implémentations réelles
  return JSON.stringify({ tool: name, input, result: "dispatched" });
}
```

---

## Mémoire

La mémoire est ce qui transforme un agent sans état en opérateur persistant. Trois couches :

| Couche | Ce qu'elle contient | Quand l'utiliser |
|---|---|---|
| **En contexte** | Conversation + tâche en cours | Session unique, tâches courtes |
| **Externe (vectorielle)** | Connaissance sémantique long terme | Rappel inter-sessions, grandes bases de connaissance |
| **Structurée (BDD)** | Entités, relations, historique | Faits devant être exacts et interrogeables |

### Mémoire en contexte avec mise en cache des prompts

Le contexte coûteux (docs, longs prompts système) doit être mis en cache :

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
        cache_control: { type: "ephemeral" }, // mettre en cache la partie coûteuse
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

### Mémoire externe — recherche sémantique

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
    // brancher sur votre fournisseur d'embeddings (OpenAI, Cohere, modèle local)
    return Array.from({ length: 1536 }, () => Math.random());
  }
}
```

### Mémoire structurée — état de l'opérateur

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

## Patterns systèmes solo

### La boucle nocturne

L'opérateur définit le travail avant de dormir. Les agents tournent. L'opérateur examine les cas limites le matin.

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
  console.log("\n=== Rapport du matin ===\n" + summary);
}
```

### Levier d'exécution — prompt → livré

Remplacer la réflexion synchrone par des pipelines asynchrones :

| Ancien modèle | Nouveau modèle |
|---|---|
| Rédiger → relire → publier | Agent rédige → opérateur valide → publication auto |
| Rechercher → synthétiser → écrire | Agent recherche → agent synthèse → agent rédaction |
| Bug signalé → trié → corrigé | Hook d'erreur → agent diagnostic → agent patch → PR |
| Ticket support → humain lit → répond | Agent triage → lookup contexte → brouillon de réponse → envoi 1-clic opérateur |

### Vitesse d'itération

L'avantage de l'opérateur est le **temps de cycle**, pas la qualité par cycle.

```
Équipe :    idée → réunion → spec → build → review → livraison   (semaines)
Opérateur : idée → prompt → agent → examen des limites → livraison (heures)
```

Concevoir chaque workflow pour minimiser la distance entre l'intention et le résultat déployé.

---

## Checklist

Avant de livrer un système solo :

- [ ] Chaque agent a une responsabilité unique et testable
- [ ] Les prompts coûteux utilisent `cache_control: { type: "ephemeral" }`
- [ ] Les sorties des agents sont loggées avec suffisamment de contexte pour déboguer sans relancer
- [ ] La couche mémoire sépare le rappel sémantique de la recherche de faits exacts
- [ ] La boucle nocturne produit un résumé matinal qui ne remonte que ce qui nécessite une décision humaine
- [ ] Les chemins d'échec sont silencieux (retry + log) et non bloquants (crash + réveil de l'opérateur)
- [ ] La surface de revue de l'opérateur est < 5 minutes pour une journée complète de sorties agents
