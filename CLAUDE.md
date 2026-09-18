# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

An MCP server (stdio or streamable-HTTP) that exposes the Google Ads API to LLM clients.
It is a read-only surface: every tool is annotated `readOnlyHint=True` and only
`GoogleAdsService.search_stream`, `GoogleAdsFieldService`, and
`CustomerService.list_accessible_customers` are ever called. There is no mutate path.

## Commands

Tooling is `nox` + `black` (80-column). Install dev deps with `pip install -e .[dev]`.

```shell
nox -s lint                  # black --check, fails on unformatted code
nox -s format                # apply black formatting
nox -s tests                 # unit tests on every Python in 3.10-3.13 available locally
nox -s tests-3.13            # unit tests on a single Python version (as CI names them)
nox -s smoke_tests           # boots the server over stdio, diffs tools/resources vs goldens
nox -s update_smoke_golden   # regenerate tests/smoke/golden_*.json after a tool/description change
nox -s llm_tests             # Gemini tool-selection check; needs GEMINI_API_KEY, installs google-genai
```

Run a single unit test without nox (tests are `unittest`, discovered as `tests/**/*_test.py`):

```shell
python -m unittest tests.tools.search_test
python -m unittest tests.tools.search_test.TestSearch.test_search_basic
```

Run the server straight from a checkout:

```shell
python -m ads_mcp.server          # stdio transport
google-ads-mcp                    # same, via the console script
```

### Golden files are load-bearing

`tests/smoke/golden_tools_list.json` and `golden_resources_list.json` capture the full
JSON-RPC `tools/list` and `resources/list` output, **including every tool description**.
The `search` tool's description is generated at runtime from its docstring plus
`ads_mcp/gaql_resources.txt`, so editing a docstring, a hint block, or the resource list
breaks `smoke_tests`. Regenerate with `nox -s update_smoke_golden` and review the diff —
these descriptions are the prompt the host LLM sees, so a change there is a behavior change.

`ads_mcp/gaql_resources.txt` itself is refreshed from the live API by
`google-ads-mcp-update-gaql` (`ads_mcp/update_references.py`), which requires working credentials.

## Architecture

**Singleton + reflection bootstrap.** `ads_mcp/coordinator.py` owns the one `FastMCP`
instance named `mcp`. Importing it runs `initialize_and_mount_tools()`, which walks
`ads_mcp/tools/` with `pkgutil`, finds every module-level `FastMCP` sub-server, and mounts
the enabled ones onto the parent. Consequences to keep in mind:

- A new tool category = a new module in `ads_mcp/tools/` declaring its own
  `FastMCP("<category>")` sub-server. It is discovered automatically — no registration list.
  But the category name must also be added to `ALL_CATEGORIES` in `ads_mcp/config.py`,
  or it stays off by default.
- Resources are the opposite: they hang off the parent `mcp` via `@mcp.resource` and are
  only registered because `ads_mcp/server.py` imports them explicitly (the `# noqa: F401`
  imports). A new resource module must be added to that import list.
- Mounting happens at import time, so `ToolsConfig` and the OAuth env vars are read once,
  at process start.

**Tool naming.** The mounted namespace prefix comes from `tools_config.yaml`, so the
user-visible name is `<prefix>_<tool>` (`search_search`, `customers_list_accessible_customers`,
`metadata_get_resource_metadata`). Resolution order for the config file is: the
`GOOGLE_ADS_MCP_TOOLS_CONFIG` env var, then `tools_config.yaml` in the CWD, then the copy
bundled in the package. An explicitly requested file that is missing is a hard error;
a resolvable-but-corrupt file is also a hard error.

**Two auth modes, selected by env vars, which also select the transport.**
`GOOGLE_ADS_MCP_OAUTH_CLIENT_ID` + `..._CLIENT_SECRET` being set switches
`coordinator.py` to a FastMCP `GoogleProvider` and `server.py` to `streamable-http` on
`$PORT` (8080). Unset, it is Application Default Credentials over stdio. `utils._create_credentials()`
mirrors this at request time: it prefers the FastMCP access token if one is in scope, else
falls back to `google.auth.default()` with the `adwords` scope. `auth_storage.py` backs the
OAuth-proxy mode with filetree/redis/firestore/memory stores chosen from `GOOGLE_ADS_MCP_STORAGE_*`.

**`utils.py` is the only Google Ads client factory.** Every call goes through
`get_googleads_service()`, which builds a fresh `GoogleAdsClient` per call and attaches
`MCPHeaderInterceptor` (appends `google-ads-mcp/<version>` to `x-goog-api-client` for usage
telemetry). `GOOGLE_ADS_DEVELOPER_TOKEN` and `GOOGLE_ADS_LOGIN_CUSTOMER_ID` are read here
and omitted entirely when unset — do not pass `None` into `GoogleAdsClient`.

`prevent_stdio_inheritance()` wraps `google.auth.default()` because ADC can shell out to
`gcloud`, and on Windows an inherited stdin deadlocks the stdio transport. Keep that wrapper
around any new code path that may spawn a subprocess during a stdio session.

**The `search` tool builds GAQL, it does not accept it.** Callers pass `fields`, `resource`,
`conditions`, `orderings`, `limit` and `tools/search.py` assembles the query, always
appending `PARAMETERS omit_unselected_resource_names=true`. `GoogleAdsException` is
translated into a FastMCP `ToolError` carrying the request ID and per-error messages.
Responses are normalized by `utils.format_output_row` / `format_output_value`, which
unwraps proto enums to their names and messages to dicts.

**Version pinning.** Generated Google Ads types are imported from a hardcoded API version
(`google.ads.googleads.v25...` in `tools/core.py` and `utils.py`), and
`resources/discovery.py` hardcodes `version=v25` in the discovery URL. Bumping the API
version means touching all of those plus `gaql_resources.txt`.

## Conventions

- Apache license header on every new `.py` file (copy an existing one).
- 80-column `black`; CI runs `nox -s lint` on 3.13 and the test matrix on 3.10-3.13.
- Tests are `unittest` with `unittest.mock`; patch at `ads_mcp.utils.get_googleads_service`
  rather than mocking the Google Ads client itself. `pyfakefs` is available for filesystem tests.
- Tool docstrings are prompts, not just docs — they steer tool selection and are asserted
  by the golden smoke tests.
