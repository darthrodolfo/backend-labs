# dotnet-ai-integration-lab — Roadmap & Claude Code Mentoring Guide

## About me

Senior Software Engineer, 15+ years, .NET/C# specialist.
Strong backend, DDD, CQRS, Angular, Flutter, FinTech, enterprise systems.
I use Claude Code and GitHub Copilot daily.
I need to learn AI INTEGRATION patterns in .NET — not ML, not training
models. I need to learn how to consume LLM APIs, build agents, RAG
pipelines, and MCP servers as a backend engineer.

Full technical context: see "Rodolfo Venancio — Contexto Técnico &
Roadmap de AI Integration" document.


---


## Instructions for Claude Code

IMPORTANT — READ THIS BEFORE ANYTHING:

1. ALWAYS respond in English. I need immersion for interview prep.

2. DO NOT write code for me. You are my MENTOR, not my autopilot.
   - Explain WHAT to do and WHY
   - Give me the concept, the pattern, the package name
   - Let ME write the code
   - Review what I wrote and point out improvements
   - If I'm stuck for more than 10 min, give me a HINT, not the answer
   - Only show me code snippets if I explicitly ask "show me"

3. ASK ME technical questions along the way:
   - "Why did you choose this approach over X?"
   - "What happens if the LLM API times out here?"
   - "How would you handle this in production with 1000 concurrent users?"
   - "What's the tradeoff of doing it this way?"
   These simulate real interview questions. I need to practice
   THINKING and EXPLAINING technical decisions in English.

4. CHALLENGE my decisions:
   - If I pick a bad pattern, don't just fix it — ask me to justify it
   - If there's a better way, ask "Have you considered X? Why or why not?"
   - Treat me as a senior — don't baby me, but guide me

5. After each completed feature, give me a MINI REVIEW:
   - What I did well
   - What could be improved
   - One interview question related to what I just built
   - Make me answer it before moving on

6. When explaining AI concepts (embeddings, tokens, MCP protocol, etc.),
   explain like I'm a senior backend engineer who knows APIs, HTTP,
   JSON, databases — but has never touched LLM integration code.
   Don't over-explain basic programming. DO explain AI-specific concepts.


---


## Project structure

One solution. One main API project. One MCP Server project.
No over-engineering. No DDD/CQRS ceremony (I already have 15 years
proving that). Focus 100% on AI integration patterns.

dotnet-ai-integration-lab/
├── src/
│   ├── VehicleCatalog.Api/
│   │   ├── Domain/            ← Entities, Enums
│   │   ├── Data/              ← DbContext, Seed, Migrations
│   │   ├── Endpoints/         ← Minimal API endpoint groups
│   │   ├── Services/          ← AI services, abstractions
│   │   └── Program.cs
│   └── VehicleCatalog.McpServer/
│       ├── Tools/
│       ├── Resources/
│       └── Program.cs
├── docker-compose.yml         ← PostgreSQL + pgvector
├── README.md
└── dotnet-ai-lab.sln

---


## Setup

SDK: .NET 9
IDE: VS Code + C# Dev Kit + Claude Code
DB: Docker — PostgreSQL 16 + pgvector
API Keys: Anthropic (console.anthropic.com) + OpenAI (for embeddings)

docker-compose.yml:
```yaml
services:
  postgres:
    image: pgvector/pgvector:pg16
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: vehiclecatalog
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev123
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

NuGet packages (main API):
- Microsoft.EntityFrameworkCore
- Npgsql.EntityFrameworkCore.PostgreSQL
- Pgvector.EntityFrameworkCore
- Anthropic.SDK (or raw HttpClient)
- OpenAI (for embeddings)
- Serilog.AspNetCore

NuGet packages (MCP Server):
- ModelContextProtocol


---


## PHASE 1 — Foundation: CRUD + Modern C#

### Goal

Minimal API with a rich Vehicle entity. This is the DATABASE that
all AI features will query. No AI yet — just a solid backend.

### Vehicle entity

Use modern C# features: record, required, init-only, enums,
IReadOnlyList, FrozenDictionary for lookups.

Fields:
- Id (Guid)
- Brand (string)
- Model (string)
- Year (int)
- Type (enum: Sedan, SUV, Hatchback, Truck, Coupe, Van)
- Fuel (enum: Gasoline, Diesel, Electric, Hybrid, Flex)
- Price (decimal)
- Horsepower (int)
- Mileage (int)
- Description (string?, rich text for RAG later)
- Features (IReadOnlyList<string>)
- Status (enum: Available, Sold, Reserved)
- CreatedAt (DateTimeOffset)

### Endpoints

POST   /api/vehicles
GET    /api/vehicles?page=1&size=10&type=SUV&fuel=Electric
GET    /api/vehicles/{id}
DELETE /api/vehicles/{id}

### Seed data

Create 20+ vehicles with REALISTIC descriptions in English.
These descriptions will be used for RAG later, so make them
2-3 paragraphs each with specs, features, pros/cons.

### Claude Code mentoring focus

- Ask me why I chose record vs class for the entity
- Ask me about nullable reference types strategy
- Ask me how I'd handle validation
- Ask me about the pagination approach


---


## PHASE 2 — Chat Completions API

### Goal

Add a chat endpoint that talks to Claude API. User asks a question,
backend sends it to LLM, returns the response.

### What to build

1. Interface IAiProvider with method for chat completion
2. AnthropicProvider implementation using HttpClient
3. Simple chat endpoint (non-streaming first)
4. Then: streaming endpoint using SSE (Server-Sent Events)
5. System prompt: "You are a vehicle catalog assistant for a
   dealership. Be helpful, concise, and knowledgeable about cars."
6. Conversation history: in-memory Dictionary<string, List<Message>>

### Endpoints

POST /api/ai/chat
{
  "message": "What's a good family car?",
  "conversationId": "abc-123"
}

POST /api/ai/chat/stream    ← Same but returns SSE stream

### Concepts to understand

- How the Messages API works (roles: system, user, assistant)
- What tokens are and why they matter (cost, context window limit)
- Streaming with SSE: why it matters for UX
- IAsyncEnumerable in .NET for streaming responses
- Why we abstract the provider (swap Anthropic for OpenAI or Bedrock
  without changing business logic)

### Claude Code mentoring focus

- Ask me about error handling: what if the API returns 429?
- Ask me about the provider abstraction: is it over-engineering?
- Ask me how I'd test this without burning API credits
- Ask me what happens when conversation history exceeds context window


---


## PHASE 3 — Function Calling / Tool Use

### Goal

The chat assistant can now QUERY THE REAL DATABASE. The LLM doesn't
have the vehicle data in its training — it calls tools that your
backend exposes, which execute real SQL queries.

### What to build

1. Define 2 tools for the LLM:
   - search_vehicles(type?, fuel?, minPrice?, maxPrice?, brand?)
   - get_vehicle_details(id)
2. Implement the full Tool Use cycle:
   - User asks question
   - LLM responds with tool_use (wants to call a function)
   - Backend executes the function against PostgreSQL
   - Backend sends tool_result back to LLM
   - LLM formulates final answer with real data
3. Handle parallel tool calls (LLM calls 2 tools at once)

### Same endpoint, smarter behavior

POST /api/ai/chat
{ "message": "Compare your cheapest electric car with your cheapest hybrid" }

The LLM will:
1. Call search_vehicles(fuel=Electric, sort=price_asc, limit=1)
2. Call search_vehicles(fuel=Hybrid, sort=price_asc, limit=1)
3. Receive both results
4. Write a comparison using REAL data from YOUR database

### Concepts to understand

- Tool definition with JSON Schema (name, description, input_schema)
- The request/response cycle of tool use (it's multi-turn)
- Why the LLM doesn't "execute" anything — it just says WHAT to call
- How to validate tool parameters from the LLM (they can hallucinate)
- Security: sanitizing LLM-generated parameters before querying DB

### Claude Code mentoring focus

- Ask me to explain the tool use cycle step by step
- Ask me what happens if the LLM hallucinates a tool name
- Ask me how I'd add a new tool without modifying existing code
- Ask me about SQL injection risk from LLM-generated parameters


---


## PHASE 4 — RAG (Document Search with Embeddings)

### Goal

Add documents (vehicle specs/manuals) to the catalog and enable
semantic search. User asks "What's the towing capacity of the F-150?"
and the system finds the relevant chunk from the docs and answers.

### What to build

1. New table: document_chunks (id, vehicle_id, content, embedding)
   - embedding column: vector(1536) using pgvector
2. Ingest endpoint: receives text, splits into chunks (by paragraph),
   generates embeddings via OpenAI API, saves to DB
3. Search endpoint: query → generate embedding → cosine similarity
   search in pgvector → top 5 chunks → send as context to LLM →
   LLM answers based on the documents
4. Seed: on startup, ingest 5-10 realistic vehicle spec texts
   so there's data to search from the beginning

### Endpoints

POST /api/vehicles/{id}/documents
{ "content": "The 2024 F-150 Lightning has a towing capacity of..." }

POST /api/ai/search
{ "query": "Which vehicles can tow over 5000 lbs?" }

### Concepts to understand

- What embeddings are (text → dense vector of numbers)
- Why similar meanings produce similar vectors
- Cosine similarity: how to measure "closeness" of two vectors
- Chunking: why you can't embed an entire document at once
- The RAG pipeline: retrieve relevant chunks → augment the prompt
  with those chunks → generate answer
- pgvector: PostgreSQL extension that stores and indexes vectors
- Why embedding model ≠ chat model (different purpose, different API)

### Claude Code mentoring focus

- Ask me what happens if chunks are too big or too small
- Ask me why we use a separate embedding model, not Claude
- Ask me how I'd handle a document with 500 pages
- Ask me about the tradeoff of top K: too few = miss info,
  too many = noise + token waste


---


## PHASE 5 — AI Agent

### Goal

Build an autonomous agent that receives a complex task, decides
which tools to call, executes them, evaluates results, and keeps
going until it has enough info to answer.

### What to build

1. Agent loop:
   - Receive task from user
   - Send to LLM with available tools
   - LLM picks a tool → execute → send result back
   - LLM picks another tool OR responds with final answer
   - Repeat until done or max iterations reached
2. Available tools: search_vehicles, get_vehicle_details,
   search_documents, compare_vehicles
3. Guardrails: max 8 iterations, token budget
4. Stream the agent's reasoning in real-time (SSE)

### Endpoint

POST /api/ai/advisor
{
  "task": "I have a family of 5, budget of $45k, need good fuel
           economy and safety. Find me the best 3 options and
           explain your reasoning step by step."
}

Response (streamed):
  → "I'll search for family-friendly vehicles under $45k..."
  → [calls search_vehicles(types=[SUV,Van], maxPrice=45000)]
  → "Found 8 options. Let me check fuel economy details..."
  → [calls get_vehicle_details(id=...)]
  → [calls get_vehicle_details(id=...)]
  → "Now searching for safety information in the docs..."
  → [calls search_documents(query="safety features")]
  → "Here are my top 3 recommendations: ..."

### Concepts to understand

- Agent = Tool Use in a LOOP with autonomous decision-making
- ReAct pattern: Reason → Act → Observe → Repeat
- Why guardrails are critical (infinite loops, cost explosion)
- Difference between agent and simple tool use (autonomy of planning)
- How to make the agent's reasoning transparent (streaming thoughts)
- When to use an agent vs simple tool use (complexity threshold)

### Claude Code mentoring focus

- Ask me to trace through the agent loop for a specific query
- Ask me what happens if the agent gets stuck in a loop
- Ask me how I'd add human-in-the-loop confirmation for actions
- Ask me about cost: how many tokens does a typical agent run burn?


---


## PHASE 6 — MCP Server

### Goal

Expose the vehicle catalog as an MCP Server. Any MCP-compatible
client (Claude Desktop, Claude Code, Cursor) can connect to your
server and query your catalog through natural language.

### What to build

New project: VehicleCatalog.McpServer

Tools (functions the LLM can call):
- search_vehicles(query, filters)
- get_vehicle(id)
- search_documents(query)

Resources (data the LLM can read):
- vehicles://catalog → summary of all vehicles
- vehicles://stats → current catalog statistics

### How to test

1. Build and run the MCP Server
2. Register it in Claude Desktop config:
   ~/Library/Application Support/Claude/claude_desktop_config.json
   (Mac) or equivalent
3. Open Claude Desktop
4. Ask "What SUVs are available in the catalog?"
5. Claude Desktop calls YOUR MCP Server, which queries YOUR database
6. Claude responds with real data from your PostgreSQL

RECORD A GIF/VIDEO OF THIS. Put it in the README.

### Concepts to understand

- MCP architecture: Host ↔ Client ↔ Server
- Transport: stdio (local process) vs HTTP+SSE (remote)
- JSON-RPC 2.0 protocol
- Tools vs Resources vs Prompts in MCP
- How MCP differs from plain Function Calling (standardized protocol,
  discoverable, any client can connect)
- Why MCP matters: "USB-C for AI" — build once, connect to any LLM app

### Claude Code mentoring focus

- Ask me to explain MCP vs REST API: what's the actual difference?
- Ask me why MCP uses JSON-RPC instead of REST
- Ask me how I'd secure an MCP Server in production
- Ask me about the discovery problem: how do clients find servers?


---


## PHASE 7 — Polish & GitHub

### README.md

Write a professional README with:
- One-line description of what this is
- Architecture overview (text or mermaid diagram)
- Tech stack list
- How to run (docker-compose up + dotnet run, that's it)
- Example requests and responses for each endpoint
- GIF/screenshot of MCP Server working in Claude Desktop
- List of AI patterns implemented

Tone: "Proof-of-concept implementations exploring AI integration
patterns in enterprise .NET backends."

NEVER say: exercise, study, tutorial, learning project, hello world.

### Code cleanup

- Consistent naming
- XML comments on public AI service interfaces
- Remove dead code
- Make sure it builds clean with no warnings

### GitHub

- Push to github.com/darthrodolfo/dotnet-ai-integration-lab
- Pin the repo on your profile
- Update pinned repos: remove old tutorial repos, pin this one


---


## What was intentionally cut

- DDD/CQRS ceremony → 15 years of career proves this. No need for
  MediatR in a 3-day proof of concept. If asked in interview, explain
  you'd apply it in production.

- AWS Bedrock → mention in README that IAiProvider abstraction allows
  plugging Bedrock. Don't need to implement it.

- Angular frontend → you already have Angular on your resume. This
  repo is backend + AI focused.

- PDF upload → plain text is enough to demonstrate RAG. Same concept.

- Playwright tests → if there's time, add 2-3 integration tests
  with WebApplicationFactory. Otherwise, cut it.


---


## What to NEVER cut

1. Chat Completions with streaming (table stakes)
2. Function Calling / Tool Use (LLM interacting with your backend)
3. RAG with pgvector (Document Search is explicitly in the job post)
4. AI Agent (demonstrates orchestration capability)
5. MCP Server (competitive differentiator — very few candidates will have this)
6. Professional README (first thing hiring manager sees)