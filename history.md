# MetaClaw History

## [2026-05-12] - Cross-Repo MCP Pipeline Integration Notes

### Summary

No MetaClaw source files were changed for this implementation pass. The pipeline fixes were made in the calling services so MetaClaw now receives explicit request context instead of relying on fallback `tui-{model}` sessions.

### Integration Impact

- Chatbot MetaClaw calls now send `X-Session-Id`, `X-Turn-Type`, `X-Session-Done`, `X-Memory-Scope`, `X-User-Id`, and `X-Workspace-Id`.
- LangGraph agent LLM calls now construct MetaClaw clients with request-scoped headers for side-task routing and evaluation.
- mcp-gen LLM calls now support MetaClaw as the selected provider and send session/build context headers when available.

### Result

- MetaClaw memory scoping should now be driven by stable user/workspace/session context from callers.
- The existing MetaClaw API contract remains unchanged.
