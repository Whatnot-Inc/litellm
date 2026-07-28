# MCP Named Authorization Server Issuer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make named MCP authorization-server metadata report the same path-scoped issuer advertised by protected-resource metadata while preserving the root issuer.

**Architecture:** Keep the existing metadata builder and route structure. Derive the issuer from `request_base_url` and the resolved optional server name, then add focused regression assertions around named, DCR-bridge, and root metadata behavior.

**Tech Stack:** Python, FastAPI, pytest

---

## File Structure

- Modify `litellm/proxy/_experimental/mcp_server/discoverable_endpoints.py`: derive the client-facing OAuth issuer for named and root metadata
- Modify `tests/test_litellm/proxy/_experimental/mcp_server/test_discoverable_endpoints.py`: prevent regressions in named, bridge, and root issuer behavior

### Task 1: Add Issuer Regression Coverage

**Files:**
- Test: `tests/test_litellm/proxy/_experimental/mcp_server/test_discoverable_endpoints.py`

- [ ] **Step 1: Assert the named issuer is path-scoped**

Extend `test_oauth_authorization_server_returns_empty_scopes_when_none` after its existing scope assertion:

```python
assert response["issuer"] == "https://litellm.example.com/atlassian_mcp"
```

- [ ] **Step 2: Assert DCR bridge metadata uses its advertised issuer**

Extend `test_oauth_authorization_server_metadata_served_for_bridge_server` before the endpoint assertions:

```python
assert result["issuer"] == "https://litellm.example.com/bridge_srv"
```

- [ ] **Step 3: Assert root metadata keeps the gateway origin**

Add this focused test beside the existing root-level OAuth endpoint tests:

```python
def test_oauth_authorization_server_root_metadata_keeps_origin_issuer():
    from fastapi import Request

    from litellm.proxy._experimental.mcp_server.discoverable_endpoints import (
        _build_oauth_authorization_server_response,
    )
    from litellm.proxy._experimental.mcp_server.mcp_server_manager import (
        global_mcp_server_manager,
    )

    server = _create_oauth2_server()
    global_mcp_server_manager.registry.clear()
    global_mcp_server_manager.registry[server.server_id] = server
    request = MagicMock(spec=Request)
    request.base_url = "https://litellm.example.com/"
    request.headers = {}

    try:
        result = _build_oauth_authorization_server_response(request=request, mcp_server_name=None)
        assert result["issuer"] == "https://litellm.example.com"
    finally:
        global_mcp_server_manager.registry.clear()
```

- [ ] **Step 4: Run the focused tests and verify the named assertions fail**

Run:

```bash
uv run pytest \
  tests/test_litellm/proxy/_experimental/mcp_server/test_discoverable_endpoints.py::test_oauth_authorization_server_returns_empty_scopes_when_none \
  tests/test_litellm/proxy/_experimental/mcp_server/test_discoverable_endpoints.py::test_oauth_authorization_server_metadata_served_for_bridge_server \
  tests/test_litellm/proxy/_experimental/mcp_server/test_discoverable_endpoints.py::test_oauth_authorization_server_root_metadata_keeps_origin_issuer \
  -v
```

Expected: the two named metadata tests fail because the issuer lacks the server path; the root test passes

### Task 2: Correct the Named Issuer

**Files:**
- Modify: `litellm/proxy/_experimental/mcp_server/discoverable_endpoints.py:1739-1763`
- Test: `tests/test_litellm/proxy/_experimental/mcp_server/test_discoverable_endpoints.py`

- [ ] **Step 1: Derive the issuer from the optional server name**

After root server-name resolution and before endpoint construction, add:

```python
issuer = f"{request_base_url}/{mcp_server_name}" if mcp_server_name else request_base_url
```

Change the metadata response to use it:

```python
return {
    "issuer": issuer,
```

- [ ] **Step 2: Run the focused tests and verify they pass**

Run:

```bash
uv run pytest \
  tests/test_litellm/proxy/_experimental/mcp_server/test_discoverable_endpoints.py::test_oauth_authorization_server_returns_empty_scopes_when_none \
  tests/test_litellm/proxy/_experimental/mcp_server/test_discoverable_endpoints.py::test_oauth_authorization_server_metadata_served_for_bridge_server \
  tests/test_litellm/proxy/_experimental/mcp_server/test_discoverable_endpoints.py::test_oauth_authorization_server_root_metadata_keeps_origin_issuer \
  -v
```

Expected: all three tests pass

- [ ] **Step 3: Run the mapped test file**

Run:

```bash
uv run pytest tests/test_litellm/proxy/_experimental/mcp_server/test_discoverable_endpoints.py -q
```

Expected: all tests pass

- [ ] **Step 4: Commit the implementation**

```bash
git add litellm/proxy/_experimental/mcp_server/discoverable_endpoints.py
git add tests/test_litellm/proxy/_experimental/mcp_server/test_discoverable_endpoints.py
make pre-commit
git commit -m "fix(mcp): scope named authorization issuer"
```

### Task 3: Verify and Publish

**Files:**
- Modify: `.github/pull_request_template.md` content only through the PR body, not the file itself

- [ ] **Step 1: Verify the final diff and repository state**

Run:

```bash
git status --short
git diff whatnot/main...HEAD --check
git diff --stat whatnot/main...HEAD
```

Expected: clean worktree, no whitespace errors, and only the design, plan, implementation, and regression test files differ

- [ ] **Step 2: Push the feature branch**

Run:

```bash
git push -u git@github.com:Whatnot-Inc/litellm.git litellm_mcp_named_issuer
```

Expected: branch is published to `Whatnot-Inc/litellm`

- [ ] **Step 3: Create a draft pull request**

Create a draft PR against `Whatnot-Inc/litellm:main` with conventional title:

```text
fix(mcp): scope named authorization issuer
```

Use `.github/pull_request_template.md`, mark meaningful tests and isolated scope complete, report the exact test commands under proof of fix, and leave the Linear section blank
