# Migration Plan: Pi SDK to Claude Agent SDK

## Executive Summary

OpenClaw currently uses the **Pi SDK** (`@mariozechner/pi-ai`, `@mariozechner/pi-agent-core`, `@mariozechner/pi-coding-agent`) as its LLM abstraction layer. This plan describes migrating the Anthropic/Claude execution path to the **Claude Agent SDK** (`@anthropic-ai/claude-agent-sdk`).

The Pi SDK is a multi-provider abstraction that unifies Anthropic, OpenAI, Google Gemini, AWS Bedrock, GitHub Copilot, and several other providers behind a single `SessionManager` + `streamSimple()` interface. The Claude Agent SDK is Anthropic's official agentic runtime that manages the full tool-execution loop, session persistence, streaming, and subagent orchestration -- but **only for Claude models**.

This means the migration is a **partial replacement**: the Claude Agent SDK replaces Pi SDK for Anthropic-provider agent runs, while Pi SDK (or a lighter alternative) remains for non-Anthropic providers.

---

## Current Architecture

### Core LLM Execution Path

```
User prompt
  -> runEmbeddedPiAgent()           [src/agents/pi-embedded-runner/run.ts]
     -> resolveModel()              [src/agents/pi-embedded-runner/model.ts]
     -> getApiKeyForModel()         [src/agents/model-auth.ts]
     -> runEmbeddedAttempt()        [src/agents/pi-embedded-runner/run/attempt.ts]
        -> SessionManager.open()    [@mariozechner/pi-coding-agent]
        -> createAgentSession()     [@mariozechner/pi-coding-agent]
        -> session.prompt()         [Pi SDK manages the agentic loop]
        -> subscribeEmbeddedPiSession()  [src/agents/pi-embedded-subscribe.ts]
  -> buildEmbeddedRunPayloads()     [src/agents/pi-embedded-runner/run/payloads.ts]
```

### Key Pi SDK Touch Points

| File | Purpose | LOC |
|------|---------|-----|
| `src/agents/pi-embedded-runner/run.ts` | Orchestrates retries, failover, auth resolution | ~900 |
| `src/agents/pi-embedded-runner/run/attempt.ts` | Core session creation + model invocation | ~900 |
| `src/agents/pi-embedded-subscribe.ts` | Streaming response handling | ~600 |
| `src/agents/pi-embedded-block-chunker.ts` | Text/reasoning block chunking | ~300 |
| `src/agents/pi-model-discovery.ts` | Pi SDK model registry wrapper | 14 |
| `src/agents/pi-tools.ts` | Tool creation (bash, read, write, edit, etc.) | ~500 |
| `src/agents/pi-embedded-runner/model.ts` | Model resolution from registry + config | ~120 |
| `src/agents/pi-embedded-runner/google.ts` | Google/Gemini-specific sanitization | ~400 |
| `src/agents/pi-embedded-runner/compact.ts` | Context compaction | ~500 |
| `src/agents/pi-embedded-helpers.ts` | Error classification, validation | ~500 |
| `src/agents/anthropic-payload-log.ts` | Anthropic request/response logging | ~230 |
| `src/agents/cache-trace.ts` | Cache hit/miss tracking | ~250 |
| `src/agents/models-config.providers.ts` | Multi-provider discovery + config | ~400 |
| `src/agents/model-auth.ts` | Auth resolution per provider | ~300 |
| `src/agents/auth-profiles/` | Auth profile management (12+ files) | ~1000 |
| `src/agents/cli-backends.ts` | CLI backend config (claude, codex) | 156 |
| `src/agents/cli-runner.ts` | CLI subprocess runner | ~200 |

### Pi SDK Dependencies (package.json)

```json
"@mariozechner/pi-agent-core": "0.51.1",
"@mariozechner/pi-ai": "0.51.1",
"@mariozechner/pi-coding-agent": "0.51.1",
"@mariozechner/pi-tui": "0.51.1"
```

### Provider Support Matrix (Current)

| Provider | API Type | Pi SDK? | Claude Agent SDK? |
|----------|----------|---------|-------------------|
| Anthropic | anthropic-messages | Yes | Yes (native) |
| OpenAI | openai-completions | Yes | No |
| Google Gemini | google-generative-ai | Yes | No |
| AWS Bedrock | bedrock-converse-stream | Yes | Yes (via env flag) |
| GitHub Copilot | custom | Yes | No |
| MiniMax | openai/anthropic compat | Yes | No |
| Moonshot | openai-compat | Yes | No |
| Ollama | openai-compat | Yes | No |
| Qwen | openai-compat | Yes | No |
| Venice | custom | Yes | No |

---

## Target Architecture

### Design Principle: Provider-Based Dispatch

The new architecture introduces a **provider dispatch layer** that routes agent runs to the appropriate backend:

```
User prompt
  -> runAgent()                           [new unified entry point]
     -> resolveProvider(provider, model)
     |
     +-- If Anthropic/Bedrock:
     |     -> runClaudeAgentSdkSession()  [new: Claude Agent SDK path]
     |        -> query({ prompt, options })
     |        -> iterate SDKMessages
     |        -> map to OpenClaw payloads
     |
     +-- If other provider:
           -> runPiAgentSession()         [existing Pi SDK path, slightly refactored]
              -> (same as current flow)
```

### Claude Agent SDK Path

```
runClaudeAgentSdkSession()
  -> Build options:
     - systemPrompt from buildEmbeddedSystemPrompt()
     - allowedTools (mapped from current tool policy)
     - mcpServers (for custom OpenClaw tools + plugin tools)
     - permissionMode: 'bypassPermissions'
     - model, maxTurns, maxThinkingTokens
     - hooks (mapped from current hook system)
     - abortController
  -> query({ prompt, options })
  -> For each SDKMessage:
     - SDKSystemMessage (init) -> capture session_id
     - SDKAssistantMessage -> map to EmbeddedPiRunResult blocks
     - SDKPartialAssistantMessage -> stream to onBlockReply/onReasoningStream
     - SDKResultMessage -> extract usage, cost, build final payloads
```

---

## Migration Phases

### Phase 0: Preparation (Non-Breaking)

**Goal**: Introduce the abstraction boundary without changing behavior.

#### 0.1 Create Provider Dispatch Interface

Create `src/agents/agent-runner.ts` with a unified interface:

```typescript
export interface AgentRunParams {
  sessionId: string;
  sessionKey?: string;
  sessionFile: string;
  workspaceDir: string;
  config?: OpenClawConfig;
  prompt: string;
  provider: string;
  model: string;
  thinkLevel?: ThinkLevel;
  timeoutMs: number;
  runId: string;
  // ... (subset of current RunEmbeddedPiAgentParams)
}

export interface AgentRunResult {
  // Maps to current EmbeddedPiRunResult
  payloads: ReplyPayload[];
  reasoning?: string;
  usage?: UsageLike;
  sessionId?: string;
  error?: string;
}

export type AgentRunner = (params: AgentRunParams) => Promise<AgentRunResult>;
```

#### 0.2 Extract Provider-Agnostic Logic

Move the following out of `pi-embedded-runner/run.ts` into shared modules:
- Auth profile resolution + failover logic
- Retry/exponential backoff
- Context window guard evaluation
- Payload building (`buildEmbeddedRunPayloads`)
- Error classification and formatting
- Prompt scrubbing (refusal magic string)

These are currently tangled with the Pi SDK session flow but are provider-agnostic.

#### 0.3 Extract Tool Definitions as Data

Currently, tools are created as Pi SDK `AgentTool` objects in `pi-tools.ts`. Extract the **tool metadata** (name, description, schema) into a provider-agnostic format so both Pi SDK and Claude Agent SDK paths can consume them.

For the Claude Agent SDK path, custom OpenClaw tools (channel tools, plugin tools, openclaw-specific tools) will be exposed via in-process MCP servers using `createSdkMcpServer()`.

---

### Phase 1: Add Claude Agent SDK Backend

**Goal**: Implement the Claude Agent SDK runner alongside the existing Pi SDK runner.

#### 1.1 Install the SDK

```bash
pnpm add @anthropic-ai/claude-agent-sdk
```

#### 1.2 Implement `runClaudeAgentSdkSession()`

Create `src/agents/claude-sdk-runner/` with:

| File | Purpose |
|------|---------|
| `run.ts` | Main entry point, builds `query()` options |
| `tools.ts` | Maps OpenClaw custom tools to MCP server definitions |
| `messages.ts` | Maps `SDKMessage` types to `EmbeddedPiRunResult` |
| `streaming.ts` | Handles partial messages + block chunking |
| `hooks.ts` | Maps OpenClaw hook system to SDK hook callbacks |
| `auth.ts` | Resolves ANTHROPIC_API_KEY from auth profiles |

**Key implementation decisions:**

1. **System Prompt**: Use `{ type: 'preset', preset: 'claude_code', append: openclawSystemPrompt }` to get Claude Code's built-in capabilities plus OpenClaw-specific instructions. Alternatively, use a fully custom system prompt if the preset doesn't fit.

2. **Tools**:
   - Built-in tools (Read, Write, Edit, Bash, Glob, Grep) are provided by the SDK natively -- no need to redefine them.
   - Custom OpenClaw tools (channel message tools, schedule tools, memory tools, plugin tools) are exposed via `createSdkMcpServer()`.
   - Tool policy filtering still applies -- use `allowedTools` + `disallowedTools` options.

3. **Permissions**: Use `permissionMode: 'bypassPermissions'` since OpenClaw has its own permission/approval system (`exec-approvals.ts`, hook-based gating).

4. **Streaming**: Enable `includePartialMessages: true` and map `SDKPartialAssistantMessage` to the existing `onBlockReply`/`onReasoningStream` callbacks.

5. **Sessions**: Map the SDK's session persistence to OpenClaw's session file system. The SDK manages its own JSONL session files; OpenClaw needs to either:
   - Use the SDK's session IDs and `resume` option directly, storing a mapping from OpenClaw session keys to SDK session IDs.
   - Or wrap the SDK in single-turn mode and manage history externally (less ideal).

6. **Subagents**: The SDK has built-in subagent support via the `Task` tool and `agents` option. Map OpenClaw's current subagent patterns to this.

#### 1.3 Implement Provider Dispatch

In `src/agents/agent-runner.ts`:

```typescript
export function resolveAgentRunner(provider: string): AgentRunner {
  if (isAnthropicProvider(provider) || isBedrockProvider(provider)) {
    return runClaudeAgentSdkSession;
  }
  return runPiAgentSession; // existing Pi SDK path
}
```

#### 1.4 Wire into Existing Call Sites

The main call site is `src/commands/agent/invoke.ts` which calls `runEmbeddedPiAgent()`. Update this to use the new dispatch layer:

```typescript
const runner = resolveAgentRunner(provider);
const result = await runner(params);
```

---

### Phase 2: Feature Parity for Anthropic Path

**Goal**: Ensure the Claude Agent SDK path supports all current Anthropic-specific features.

#### 2.1 Auth Profile Failover

Current system rotates through auth profiles on rate limits/auth errors. The Claude Agent SDK doesn't have built-in multi-key failover. Implement this as a wrapper:

```typescript
async function runWithFailover(params: AgentRunParams): Promise<AgentRunResult> {
  const profiles = resolveAuthProfileOrder(params.provider, params.config);
  for (const profile of profiles) {
    if (isProfileInCooldown(profile)) continue;
    try {
      const result = await runClaudeAgentSdkSession({
        ...params,
        env: { ANTHROPIC_API_KEY: profile.apiKey },
      });
      markAuthProfileGood(profile);
      return result;
    } catch (err) {
      if (isRateLimitError(err) || isAuthError(err)) {
        markAuthProfileFailure(profile);
        continue;
      }
      throw err;
    }
  }
  throw new FailoverError("All auth profiles exhausted");
}
```

#### 2.2 Context Window Management

The current system has:
- Context window guards (`context-window-guard.ts`)
- Compaction (`compact.ts`)
- Cache TTL tracking (`cache-ttl.ts`)

The Claude Agent SDK handles context management internally (compaction is built-in, emits `SDKCompactBoundaryMessage`). However, OpenClaw may need to:
- Monitor context usage for display/logging
- Apply custom compaction strategies

Map `SDKCompactBoundaryMessage` events to the existing compaction tracking.

#### 2.3 Image/Vision Support

Current system detects vision-capable models and injects images into prompts. The Claude Agent SDK handles images natively through its message format. Map OpenClaw's image loading pipeline to SDK input messages.

#### 2.4 Payload Logging

Replace `anthropic-payload-log.ts` wrapping of `streamFn` with SDK hooks:
- `PostToolUse` hook for tool call logging
- Extract usage from `SDKResultMessage` for cost tracking

#### 2.5 Cache Trace

Map the existing cache trace system to SDK hooks or the `SDKResultMessage` usage data.

#### 2.6 Heartbeat / Keep-Alive

Current system has heartbeat prompts for long-running sessions. This needs to be handled outside the SDK (send periodic prompts to keep the session alive).

---

### Phase 3: Custom Tool Migration

**Goal**: Migrate all OpenClaw-specific tools to MCP server format.

#### 3.1 Inventory of Custom Tools

| Tool | Source | Migration Strategy |
|------|--------|-------------------|
| Exec (bash) | `bash-tools.ts` | Use SDK's built-in `Bash` tool |
| Read/Write/Edit | `pi-tools.ts` (via pi-coding-agent) | Use SDK's built-in tools |
| Apply Patch | `apply-patch.ts` | MCP tool |
| Channel Message tools | `channel-tools.ts` | MCP tool |
| OpenClaw tools (schedule, memory, etc.) | `openclaw-tools.ts` | MCP tool |
| Plugin tools | `plugins/tools.ts` | MCP tool (per-plugin server) |
| Image tool | `tools/image-tool.ts` | MCP tool |
| Sandbox tools | `sandbox.ts` | Configure via SDK's `sandbox` option |

#### 3.2 Create MCP Server Factories

```typescript
// src/agents/claude-sdk-runner/tools.ts
export function createOpenClawMcpServer(params: {
  config: OpenClawConfig;
  sessionKey?: string;
  channel?: string;
}): McpSdkServerConfigWithInstance {
  const tools = [
    ...buildChannelTools(params),
    ...buildOpenClawTools(params),
    ...buildPluginTools(params),
  ].map(convertToSdkTool);

  return createSdkMcpServer({
    name: "openclaw-tools",
    tools,
  });
}
```

#### 3.3 Tool Policy Mapping

Map the existing tool policy system (`pi-tools.policy.ts`) to the SDK's `allowedTools` / `disallowedTools` / `canUseTool` options.

---

### Phase 4: Streaming and Response Mapping

**Goal**: Ensure the response format matches what downstream consumers expect.

#### 4.1 SDKMessage to OpenClaw Payload Mapping

| SDK Message Type | OpenClaw Equivalent |
|-----------------|---------------------|
| `SDKSystemMessage` (init) | Session metadata capture |
| `SDKAssistantMessage` (text block) | `ReplyPayload` with text content |
| `SDKAssistantMessage` (tool_use block) | Tool call metadata for `toolMetas` |
| `SDKPartialAssistantMessage` | `onBlockReply` / `onReasoningStream` callbacks |
| `SDKResultMessage` | Final `EmbeddedPiRunResult` with usage/cost |
| `SDKCompactBoundaryMessage` | Compaction event logging |

#### 4.2 Block Chunking

The existing `EmbeddedBlockChunker` handles splitting long text into channel-appropriate chunks. This still applies to SDK output -- the chunker processes text from `SDKAssistantMessage` blocks.

#### 4.3 Reasoning Tag Handling

Current system strips `<thinking>` tags and extracts reasoning. The Claude Agent SDK emits reasoning as separate content blocks in `SDKAssistantMessage`. Map these to the existing reasoning display pipeline.

---

### Phase 5: Testing and Validation

#### 5.1 Unit Tests

- Test the provider dispatch logic
- Test SDKMessage -> OpenClaw payload mapping
- Test tool policy mapping
- Test MCP server creation for custom tools
- Test auth profile failover wrapper

#### 5.2 Integration Tests

- Run against Anthropic API with real keys
- Verify session resume/continue works
- Verify streaming output matches current format
- Verify custom tool invocation via MCP
- Verify image/vision support

#### 5.3 Regression Testing

- Run full test suite (`pnpm test`)
- Run live tests (`pnpm test:live`)
- Run Docker integration tests
- Verify all messaging channels still receive correctly formatted responses

---

### Phase 6: Cleanup and Optimization

#### 6.1 Remove Pi SDK for Anthropic Path

Once the Claude Agent SDK path is validated:
- Remove Anthropic-specific code from Pi SDK path (Anthropic payload logging, Anthropic turn validation, Anthropic refusal scrubbing)
- The Pi SDK path becomes exclusively for non-Anthropic providers

#### 6.2 Evaluate Pi SDK Retention

If all non-Anthropic providers can be served by simpler alternatives (e.g., direct API calls via `@anthropic-ai/sdk` for Bedrock, or `openai` package for OpenAI-compatible providers), consider removing Pi SDK entirely. This is a separate decision that depends on the value Pi SDK provides for non-Anthropic providers.

#### 6.3 Dependency Cleanup

Remove `@mariozechner/pi-tui` if only used for the Anthropic path. Keep `@mariozechner/pi-agent-core` and `@mariozechner/pi-coding-agent` if needed for non-Anthropic providers.

---

## Risk Assessment

### High Risk

1. **Session format incompatibility**: Pi SDK uses JSONL session files with a specific format. The Claude Agent SDK has its own session format. Migrating existing sessions or maintaining dual formats is complex.

2. **Tool execution behavior differences**: Pi SDK tools are implemented locally. SDK tools are executed by the Claude Code runtime. Behavior differences (e.g., Bash sandboxing, file access patterns) could cause regressions.

3. **Streaming timing differences**: The SDK manages its own streaming. The current chunking/batching system in `EmbeddedBlockChunker` may need adjustment for different streaming characteristics.

### Medium Risk

4. **Custom tool latency**: Custom tools going through MCP adds IPC overhead compared to current in-process tool execution.

5. **Auth profile failover**: The SDK doesn't natively support multi-key failover. The wrapper approach works but adds complexity.

6. **Context compaction control**: Losing fine-grained control over when/how compaction happens. The SDK manages this internally.

### Low Risk

7. **Non-Anthropic providers unaffected**: The Pi SDK path for OpenAI, Google, etc. remains unchanged.

8. **Dependency addition**: Adding `@anthropic-ai/claude-agent-sdk` is straightforward with no known conflicts.

---

## Key Files to Create

| File | Purpose |
|------|---------|
| `src/agents/agent-runner.ts` | Unified dispatch interface |
| `src/agents/claude-sdk-runner/run.ts` | Claude Agent SDK session runner |
| `src/agents/claude-sdk-runner/tools.ts` | Custom tool -> MCP server mapping |
| `src/agents/claude-sdk-runner/messages.ts` | SDKMessage -> OpenClaw payload mapping |
| `src/agents/claude-sdk-runner/streaming.ts` | Streaming + chunking adapter |
| `src/agents/claude-sdk-runner/hooks.ts` | OpenClaw hooks -> SDK hooks mapping |
| `src/agents/claude-sdk-runner/auth.ts` | Auth profile resolution for SDK |
| `src/agents/claude-sdk-runner/sessions.ts` | Session ID mapping + resume logic |

## Key Files to Modify

| File | Change |
|------|--------|
| `src/commands/agent/invoke.ts` | Use dispatch layer instead of direct `runEmbeddedPiAgent` |
| `src/agents/pi-embedded-runner/run.ts` | Extract shared logic, keep as Pi-only path |
| `package.json` | Add `@anthropic-ai/claude-agent-sdk` dependency |
| `src/agents/defaults.ts` | No change (provider/model defaults stay) |
| `src/config/types.ts` | Add SDK-specific config options if needed |

## Files That Can Eventually Be Simplified/Removed (Anthropic Path Only)

| File | Reason |
|------|--------|
| `src/agents/anthropic-payload-log.ts` | Replaced by SDK hooks |
| `src/agents/pi-embedded-runner/google.ts` | Only used for Pi SDK Gemini path |
| `src/agents/pi-embedded-runner/compact.ts` | SDK handles compaction |
| `src/agents/cache-trace.ts` | SDK manages caching |
| `src/agents/pi-embedded-subscribe.ts` | Replaced by SDK message iteration |
| `src/agents/pi-embedded-block-chunker.ts` | Simplified -- SDK provides cleaner boundaries |

---

## Estimated Scope

- **Phase 0** (Preparation): ~15 files touched, extract + refactor
- **Phase 1** (SDK Backend): ~8 new files, ~5 modified files
- **Phase 2** (Feature Parity): ~10 files touched
- **Phase 3** (Tool Migration): ~5 new/modified files
- **Phase 4** (Streaming): ~3 new files
- **Phase 5** (Testing): ~10 test files
- **Phase 6** (Cleanup): ~15 files simplified

Total: ~50-70 files across all phases. The migration is substantial but can be delivered incrementally since the dispatch layer allows both paths to coexist.
