### Issue for this PR

Closes #

### Type of change

- [x] Bug fix
- [ ] New feature
- [ ] Refactor / code improvement
- [ ] Documentation

### What does this PR do?

This PR fixes an ACP interoperability issue when OpenCode is used with the Agent Client Protocol Python SDK over `stdio`.

In this integration mode, JSON-RPC payload size is effectively constrained (observed around the 64KB boundary in practice for this path). When a single emitted JSON message exceeds this boundary, the Python SDK side can hit a **fatal error** on the stdio channel/port handling path, which can break ACP communication for the client session.

OpenCode tool updates can easily exceed that limit:

- `rawOutput.output` is often very large.
- `tool_call_update.content[].content.text` can also become large (for example, `read` on moderately large files).

To prevent oversized ACP messages from causing fatal stdio-side failures and breaking delivery, this change applies field-level truncation for those two high-risk text fields before sending `tool_call_update` payloads to ACP.

What changed:

- Added sanitization in ACP update sending for `tool_call_update` payloads.
- Capped each risky text field at **10KB** (UTF-8 byte-based limit).
- Appended an explicit truncation notice to preserve operator visibility (including omitted byte count).
- Added clear English inline comments documenting why and where truncation is applied.

Why this works:

- It keeps update payloads comfortably below transport-boundary risk while preserving useful context for ACP consumers.
- In IDE usage, ACP users typically do not inspect full raw tool output in these update frames, so truncating very large payload text is a practical tradeoff for transport reliability.
- It addresses the observed failure mode directly at the emission boundary before oversized text reaches stdio transport.

Scope of truncation (only):

1. `update.content[].content.text`
2. `update.rawOutput.output`

No other ACP update types are altered.

Notes:

- This is a transport-safety and robustness fix.
- It does not change tool execution semantics.
- It only limits oversized text payload segments before ACP emission.

### How did you verify your code works?

I validated this change with:

- Targeted ACP tests
- Type checks
- Build checks
- Local runtime verification

After this change, the ACP Python SDK can reliably receive previously oversized tool-update messages in truncated form.

### Screenshots / recordings

N/A (non-UI change).

### Checklist

- [x] I have tested my changes locally
- [x] I have not included unrelated changes in this PR
