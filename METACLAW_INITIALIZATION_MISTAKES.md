# MetaClaw ChatOpenAI Initialization: Mistakes to Avoid

## Summary of Findings

Based on the history.md in `chatbot_mcp_client` and code review of `langChain-application` and `mcp-gen`, several repositories are making the **same ChatOpenAI initialization mistake** that was already fixed in chatbot_mcp_client.

---

## The Critical Mistake

### **WRONG: Using `model_kwargs` to pass MetaClaw-specific headers**

```python
# ❌ INCORRECT - Found in langChain-application and mcp-gen
llm = ChatOpenAI(
    api_key=...,
    base_url=metaclaw_url,
    model_kwargs={
        "session_done": True  # This goes in the model's API call body, NOT as HTTP headers
    }
)
```

```typescript
// ❌ INCORRECT - Found in mcp-gen/genai.ts
llm = new ChatOpenAI({
    apiKey: ...,
    baseURL: metaclaw_url,
    modelKwargs: {
        session_done: sessionDone  // Wrong! This is for model parameters, not custom headers
    }
})
```

### **CORRECT: Using `default_headers` for custom headers**

```python
# ✅ CORRECT - Fixed in chatbot_mcp_client/metaclaw_client.py
headers = {"X-Session-Done": "true"} if session_done else None

return ChatOpenAI(
    model=model_name,
    temperature=temperature,
    api_key=api_key,
    base_url=base_url,
    max_retries=2,
    default_headers=headers,  # ✅ Custom headers go here
    top_p=config.metaclaw_top_p,
    max_tokens=config.metaclaw_max_tokens,
)
```

---

## Why This Matters

### MetaClaw Requires Custom HTTP Headers

The MetaClaw proxy expects specific custom headers like `X-Session-Done` to trigger memory ingestion:

```
POST /v1/chat/completions HTTP/1.1
Host: metaclaw-server
Content-Type: application/json
X-Session-Done: true  ← Custom header for MetaClaw memory system

{
  "model": "gemini-2.5-flash",
  "messages": [...],
  "temperature": 0.0
  // "session_done" does NOT belong here
}
```

### What `model_kwargs` Does

`model_kwargs` in LangChain adds parameters to the **request body** of the OpenAI API call:

```json
{
  "model": "...",
  "messages": [...],
  "temperature": 0.0,
  "session_done": true  // ← This is in the JSON body, NOT as HTTP headers
}
```

MetaClaw's proxy **does not read these from the body** - it expects them as HTTP headers. Therefore, the memory ingestion feature (`session_done`) is silently ignored.

---

## Affected Files (Need Fixing)

### 1. **langChain-application/my_agent/utils/llm_factory.py**

**Lines 40-52:**
```python
# Default model for MetaClaw if None provided,
# though MetaClaw often ignores this if it has its own logic.
# We use the configured metaclaw model from PROVIDER_CONFIG as a placeholder.
target_model = PROVIDER_CONFIG.get("METACLAW_MODEL", "qwen/qwen3-next-80b-a3b-instruct")

print(f"[LLM-Factory] Using MetaClaw proxy at {base_url}")
# Merge session_done into model_kwargs to enable memory ingestion
model_kwargs = kwargs.get("model_kwargs", {})
model_kwargs["session_done"] = True  # ❌ WRONG
kwargs["model_kwargs"] = model_kwargs
return ChatOpenAI(
    model=target_model,
    base_url=base_url,
    api_key=SecretStr(api_key),
    temperature=temperature,
    top_p=API_CONFIG.get("metaclaw_top_p", 0.5),
    max_tokens=API_CONFIG.get("metaclaw_max_tokens", 100000),
    **kwargs
)
```

**Fix:**
```python
print(f"[LLM-Factory] Using MetaClaw proxy at {base_url}")
# Use default_headers for MetaClaw-specific custom headers
default_headers = {"X-Session-Done": "true"}
return ChatOpenAI(
    model=target_model,
    base_url=base_url,
    api_key=SecretStr(api_key),
    temperature=temperature,
    top_p=API_CONFIG.get("metaclaw_top_p", 0.5),
    max_tokens=API_CONFIG.get("metaclaw_max_tokens", 100000),
    default_headers=default_headers,  # ✅ CORRECT
    **kwargs
)
```

### 2. **mcp-gen/src/utils/genai.ts**

**Lines 82-98:**
```typescript
// Route through MetaClaw if enabled
let llm;
if (metaclawConfig.enabled) {
  console.log("[GenAI] 🧠 Routing through MetaClaw proxy");
  llm = new ChatOpenAI({
    configuration: {
      baseURL: metaclawConfig.baseUrl,
    },
    apiKey: metaclawConfig.apiKey,
    model: selectedModel,
    temperature: temperature ?? currentConfig.temperature,
    topP: metaclawConfig.topP,
    maxTokens: maxTokens ?? metaclawConfig.maxTokens,
    maxRetries: 2,
    modelKwargs: {
      session_done: sessionDone,  // ❌ WRONG
    },
  });
}
```

**Fix:**
```typescript
// Route through MetaClaw if enabled
let llm;
if (metaclawConfig.enabled) {
  console.log("[GenAI] 🧠 Routing through MetaClaw proxy");
  llm = new ChatOpenAI({
    configuration: {
      baseURL: metaclawConfig.baseUrl,
    },
    apiKey: metaclawConfig.apiKey,
    model: selectedModel,
    temperature: temperature ?? currentConfig.temperature,
    topP: metaclawConfig.topP,
    maxTokens: maxTokens ?? metaclawConfig.maxTokens,
    maxRetries: 2,
    defaultHeaders: {
      "X-Session-Done": sessionDone ? "true" : "false",  // ✅ CORRECT
    },
  });
}
```

**Note:** The TypeScript LangChain wrapper uses `defaultHeaders` (camelCase) instead of `default_headers` (snake_case).

---

## Additional Observations

### 1. **Inconsistent Session-Done Semantics**

- **chatbot_mcp_client** (correctly): `session_done=True` ONLY on the final MetaClaw invocation to trigger memory ingestion.
- **langChain-application**: Always sets `session_done=True` for ALL requests.
- **mcp-gen**: Passes `sessionDone` parameter but defaults to `true` in the function signature.

**Recommendation:** Follow chatbot_mcp_client's pattern:
- `session_done=False` for normal conversation turns
- `session_done=True` only on the final turn to trigger ingestion

### 2. **Missing Tool Binding for MetaClaw**

When using MetaClaw with the `create_mcp_server` tool, the ChatOpenAI instance should have tools bound:

```python
# ✅ Correct pattern from chatbot_mcp_client
metaclaw_llm = ChatOpenAI(...).bind_tools([create_mcp_server_tool])
```

Neither langChain-application nor mcp-gen bind tools to their MetaClaw ChatOpenAI instances, which means:
- MetaClaw cannot call `create_mcp_server` tool
- Tool calls must be detected via text parsing (less reliable)

### 3. **Manual Client Instantiation Anti-Pattern**

The original bug (already fixed in chatbot_mcp_client) was:

```python
# ❌ DON'T DO THIS
openai_client = AsyncOpenAI(api_key=..., base_url=...)
chat = ChatOpenAI(client=openai_client)  # Wrapper conflict!
```

Both langChain-application and mcp-gen use the correct pattern of passing parameters directly to ChatOpenAI constructor, but they misuse `model_kwargs` instead of `default_headers`.

---

## Root Cause Analysis

The mistake stems from **confusing two different concepts**:

1. **`model_kwargs`**: Parameters that go in the LLM API request body (e.g., `temperature`, `top_p`, `max_tokens`). These are model inference parameters.

2. **`default_headers`**: HTTP headers added to every request. These are for **transport-layer metadata** like custom proxy headers, authentication tokens, or in this case, MetaClaw's `X-Session-Done`.

MetaClaw's architecture requires **HTTP-level signaling** (`X-Session-Done` header) separate from the model's inference parameters. The `session_done` flag is **not a model parameter** - it's a **proxy control signal**.

---

## Verification Steps

To verify the fix works:

1. **Check request headers** - Use a proxy or logging middleware to confirm `X-Session-Done` appears in HTTP headers, not in JSON body.

2. **Check MetaClaw logs** - The memory system should show `hits` > 0 and proper session ingestion when `X-Session-Done: true` is received.

3. **Functional test**:
   ```bash
   # With session_done header working:
   1. Send multiple chat messages
   2. On final message, MetaClaw should ingest to memory
   3. New conversation should retrieve from memory (hits > 0)
   ```

---

## Recommended Fixes Priority

| Repository | File | Severity | Lines | Fix Complexity |
|------------|------|----------|-------|----------------|
| langChain-application | `my_agent/utils/llm_factory.py` | HIGH | 40-52 | 2 lines |
| mcp-gen | `src/utils/genai.ts` | HIGH | 95-97 | 2 lines |

Both are **simple, surgical fixes** that will make MetaClaw memory work correctly.

---

## References

- **chatbot_mcp_client/history.md** - Original bug report and fix (2026-04-27)
- **chatbot_mcp_client/backend/metaclaw_client.py** - Correct implementation reference
- **LangChain OpenAI documentation** - `default_headers` parameter description
- **MetaClaw Architecture** - Requires `X-Session-Done` for memory ingestion

---

## Action Items

1. ✅ Fix `langChain-application/my_agent/utils/llm_factory.py` line 42-43
2. ✅ Fix `mcp-gen/src/utils/genai.ts` line 95-97 (use `defaultHeaders`)
3. ✅ Consider adding tool binding: `.bind_tools([create_mcp_server_tool])`
4. ✅ Review session_done semantics: only true on final turn
5. ✅ Add integration test that verifies `X-Session-Done` header is sent
