# Tool Reference & Known Limitations

Reference material for the intel-agent skill. Consult when you need tool signatures, caveats, or usage guidance.

---

## Known Limitations & Tool Caveats

### Body preview truncation

- `proxy_search_traffic(query)` searches body **previews** only, not full response bodies
- `proxy_get_exchange(exchange_id)` returns `bodyPreview` which is truncated for large responses
- Always compare `bodySize` to the actual preview length; if `bodySize` significantly exceeds the preview, the preview is incomplete
- **A search returning 0 results does NOT prove the content is absent from the full response**
- Mitigation: use `proxy_search_session_bodies(session_id, text: "search term")` to search **decompressed** full bodies with grep-like context snippets. Pre-filter by hostname, content-type, status code to narrow the search.
- Mitigation: use `proxy_get_session_exchange(session_id, exchange_id, include_body: true)` to retrieve one full decompressed body from a persisted session
- If neither full body nor rendered DOM snapshot is available, mark the finding as **INCONCLUSIVE — body truncated, no snapshot available**

### DevTools sidecar dependency

- `interceptor_chrome_devtools_attach()` may return a warning containing `"native-fallback mode"` if the chrome-devtools-mcp sidecar binary is not installed
- In fallback mode, `screenshot()`, `snapshot()`, `list_network()`, and `list_console()` are **unavailable**
- Remedy: call `interceptor_chrome_devtools_pull_sidecar()` to install the sidecar, then detach and re-attach
- If the sidecar cannot be installed, the workflow must degrade — see Step 1a capability check in SKILL.md

### Live traffic search vs session search

- `proxy_search_traffic()` operates on the live **in-memory** traffic buffer (preview only, cleared by `proxy_clear_traffic()`)
- `proxy_query_session()` searches session **metadata** (URL, hostname, path) — it does NOT search inside body content
- `proxy_search_session_bodies()` searches **decompressed full bodies** in the persisted session — this is the authoritative body search tool
- For finding text inside HTML/JSON response bodies, always use `proxy_search_session_bodies()`
- Always start a session with `capture_profile: "full"` in Step 1 to enable full body search

---

## Tool Quick Reference

### Initialization
| Tool | Purpose |
|------|---------|
| `proxy_start(persistence_enabled, capture_profile, session_name)` | Start MITM proxy with session (use `persistence_enabled: true, capture_profile: "full"`) |
| `proxy_session_start(session_name, capture_profile)` | Start session separately (not needed if `proxy_start` was called with `persistence_enabled: true`) |
| `interceptor_chrome_launch(url, stealthMode: true)` | Launch stealth Chrome |
| `interceptor_chrome_devtools_attach(target_id)` | Attach DevTools bridge |
| `interceptor_chrome_devtools_pull_sidecar()` | Install DevTools sidecar if missing |

### Traffic Analysis (Live Buffer)
| Tool | Purpose |
|------|---------|
| `proxy_list_traffic(url_filter, method_filter)` | List captured exchanges |
| `proxy_search_traffic(query)` | Full-text search across traffic (**previews only — see Known Limitations**) |
| `proxy_get_exchange(exchange_id)` | Request/response details (**bodyPreview may be truncated**) |
| `proxy_clear_traffic()` | Clear buffer before action |

### Traffic Analysis (Persisted Session)
| Tool | Purpose |
|------|---------|
| `proxy_query_session(session_id, text, url_contains, ...)` | Query session by **metadata** (URL, hostname, path) — does NOT search body content |
| `proxy_search_session_bodies(session_id, text, ...)` | **Full-text search inside response/request bodies** — decompresses and searches with grep-like context snippets. Pre-filter by hostname, content-type, status code. This is the authoritative body search tool. |
| `proxy_get_session_exchange(session_id, exchange_id, include_body)` | Get full exchange from session (**full decompressed body when capture_profile is "full"**) |

### Browser Inspection
| Tool | Purpose |
|------|---------|
| `interceptor_chrome_devtools_navigate(url)` | Navigate (preserves DevTools) |
| `interceptor_chrome_devtools_screenshot()` | Capture screenshot (FULL_MODE only) |
| `interceptor_chrome_devtools_snapshot()` | Accessibility tree / rendered DOM (FULL_MODE only) |
| `interceptor_chrome_devtools_list_network(resource_types)` | Browser network view |
| `interceptor_chrome_devtools_list_cookies(domain_filter)` | Get cookies |

### Human Interaction
| Tool | Purpose |
|------|---------|
| `humanizer_click(target_id, selector)` | Click with human-like behavior |
| `humanizer_type(target_id, text)` | Type with realistic timing |
| `humanizer_scroll(target_id, delta_y)` | Smooth scroll (delta_y in pixels, positive = down) |
| `humanizer_idle(target_id, duration_ms)` | Idle with micro-movements |

### Protection Testing
| Tool | Purpose |
|------|---------|
| `proxy_set_upstream(url)` | Chain to upstream proxy |
| `proxy_set_host_upstream(hostname, url)` | Per-host upstream proxy |
| `proxy_clear_upstream()` | Remove all upstream proxies |
| `proxy_list_tls_fingerprints(hostname_filter)` | List unique JA3/JA4 fingerprints across traffic |
| `proxy_get_tls_fingerprints(exchange_id)` | Get TLS fingerprints for a specific exchange |

### Session Management
| Tool | Purpose |
|------|---------|
| `proxy_session_stop()` | Stop recording and finalize session |
| `proxy_export_har(session_id, output_file, include_bodies)` | Export session as HAR |

---

## Important Rules

- Always use `interceptor_chrome_devtools_navigate()` for navigation — NOT `interceptor_chrome_navigate()` (loses DevTools session)
- Always use `stealthMode: true` when launching Chrome
- Always `proxy_clear_traffic()` before an interaction to isolate the traffic it generates
- Always start session recording in Step 1 with `capture_profile: "full"` — this is the only way to get full response bodies for authoritative searches
- Use `proxy_search_session_bodies()` to search inside response/request bodies (not `proxy_search_traffic()` which only searches previews, and not `proxy_query_session()` which only searches metadata)
