# Tool Reference & Known Limitations

Reference material for the intel-agent skill. Consult when you need tool signatures, caveats, or usage guidance.

Targets **proxy-mcp ≥ 2.0.0** (cloakbrowser + Playwright). For older proxy-mcp (1.x, `interceptor_chrome_*` + CDP sidecar), use the `v1` branch of this skill.

---

## Known Limitations & Tool Caveats

### Body preview truncation (the single most important caveat)

- `proxy_search_traffic(query)` searches body **previews** only, not full response bodies
- `proxy_get_exchange(exchange_id)` returns `bodyPreview` capped at 4 KB
- `bodySize` field shows the true size — if it's larger than the preview, the preview is incomplete
- **A search returning 0 results does NOT prove the content is absent from the full response**
- Mitigation: always start the proxy with `persistence_enabled: true, capture_profile: "full"` (Step 1). Then use `proxy_search_session_bodies(session_id, query: "search term")` for **decompressed full-body search** with grep-like context snippets, and `proxy_get_session_exchange(session_id, exchange_id, include_body: true)` for one full body.
- If neither full body nor rendered DOM snapshot is available, mark the finding as **INCONCLUSIVE — body truncated**.

### Live traffic search vs session search

- `proxy_search_traffic()` operates on the live **in-memory** traffic buffer (preview only, cleared by `proxy_clear_traffic()`)
- `proxy_query_session()` searches session **metadata** (URL, hostname, path) — it does NOT search inside body content
- `proxy_search_session_bodies()` searches **decompressed full bodies** in the persisted session — this is the authoritative body search tool
- For finding text inside HTML/JSON response bodies, always use `proxy_search_session_bodies()`

### Locator vs coordinate clicks

- `humanizer_click` accepts `selector | role + name | text | label | x,y`. Prefer the locator forms (role/name/text/label) — they auto-wait for visible + enabled + stable + in-view and handle iframes / shadow DOM / offscreen elements that coordinate-based clicks fail on.
- Fall back to `selector` for elements without good ARIA, and to `x,y` only when nothing else resolves.

### No CDP surface

- proxy-mcp v2 drives the browser via Playwright. There is no CDP HTTP/WebSocket port, no DevTools sidecar, no attach/detach step. Tools take `target_id` from `interceptor_browser_launch` directly.
- If the v1 skill said "use `interceptor_chrome_devtools_*`" — translate to `interceptor_browser_*` and drop the attach step.

---

## Tool Quick Reference

### Initialization
| Tool | Purpose |
|------|---------|
| `proxy_start(persistence_enabled: true, capture_profile: "full", session_name, max_disk_mb: 2048)` | Start MITM proxy with full-body on-disk capture (REQUIRED for intel-agent) |
| `proxy_session_start(session_name, capture_profile)` | Start session separately (not needed if `proxy_start` was called with `persistence_enabled: true`) |
| `interceptor_browser_launch(url, headless?, humanize?, timezone?, locale?, viewport_width?, viewport_height?)` | Launch cloakbrowser (stealth Chromium, source-level fingerprint patches, humanize on by default) |

### Traffic Analysis (Live Buffer — header peek)
| Tool | Purpose |
|------|---------|
| `proxy_list_traffic(url_filter, method_filter, hostname_filter, status_filter)` | List captured exchanges |
| `proxy_search_traffic(query)` | Full-text search across URLs/headers/4 KB body previews (**previews only — see Known Limitations**) |
| `proxy_get_exchange(exchange_id)` | Header-level peek (**bodyPreview capped at 4 KB**) |
| `proxy_clear_traffic()` | Clear buffer before action |

### Traffic Analysis (Persisted Session — full bodies)
| Tool | Purpose |
|------|---------|
| `proxy_query_session(session_id, hostname_contains, method_filter, ...)` | Query session by **metadata** (URL, hostname, path) — does NOT search body content |
| `proxy_search_session_bodies(session_id, query, hostname_contains?, content_type_contains?)` | **Full-text search inside response/request bodies** — decompresses gzip/br/deflate and returns grep-like context snippets. Pre-filter to narrow scan. Authoritative body search tool. |
| `proxy_get_session_exchange(session_id, exchange_id, include_body: true)` | Get full exchange from session (**full decompressed body when `capture_profile` is "full"**) |

### Browser Inspection
| Tool | Purpose |
|------|---------|
| `interceptor_browser_navigate(target_id, url, wait_until?)` | Navigate; `wait_until: "networkidle"` for SPA hydration settle |
| `interceptor_browser_screenshot(target_id, file_path?, full_page?)` | Screenshot (saves to disk if `file_path` given) |
| `interceptor_browser_snapshot(target_id, selector?, mode?)` | YAML ARIA tree (role/name/text); `selector` scopes the tree; `mode: "ai"` adds refs |
| `interceptor_browser_list_console(target_id, types?, text_filter?)` | Buffered console messages since launch — useful for site profiling |
| `interceptor_browser_list_cookies(target_id, domain_filter?)` | Browser context cookies |
| `interceptor_browser_list_storage_keys(target_id, storage_type)` | local/sessionStorage key listing |
| `interceptor_browser_get_storage_value(target_id, storage_type, item_id)` | Full storage value |
| `interceptor_browser_close(target_id)` | Close the browser instance |

### Human Interaction
| Tool | Purpose |
|------|---------|
| `humanizer_click(target_id, selector? \| role+name? \| text? \| label? \| x+y?)` | Click; prefer locator forms — auto-waits for visible + enabled + stable + in-view |
| `humanizer_type(target_id, text, wpm?, error_rate?)` | Type with realistic timing (WPM + bigram + typo model) |
| `humanizer_scroll(target_id, delta_y, delta_x?, duration_ms?)` | Smooth eased scroll (delta_y in pixels, positive = down) |

(cloakbrowser already humanizes mouse/keyboard dispatch by default; the `humanizer_*` timing profile layers on top. Explicit idle waits are unnecessary — `humanizer_click` auto-waits for stability and `interceptor_browser_navigate(..., wait_until: "networkidle")` waits for XHR settle.)

### Protection Testing
| Tool | Purpose |
|------|---------|
| `proxy_set_upstream(url)` | Chain to upstream proxy |
| `proxy_set_host_upstream(hostname, url)` | Per-host upstream proxy |
| `proxy_clear_upstream()` | Remove all upstream proxies |
| `proxy_list_tls_fingerprints(hostname_filter)` | List unique JA3/JA4 fingerprints across traffic |
| `proxy_get_tls_fingerprints(exchange_id)` | Get TLS fingerprints for a specific exchange |
| `proxy_set_fingerprint_spoof(preset)` | Outbound TLS+HTTP/2 spoofing (impit) — for non-browser clients only |

### Session Management
| Tool | Purpose |
|------|---------|
| `proxy_session_stop()` | Stop recording and finalize session |
| `proxy_export_har(session_id, file_path, include_bodies: true)` | Export session as HAR (REQUIRED for Raw Evidence section) |
| `proxy_get_session_handshakes(session_id)` | JA3/JA4/JA3S handshake metadata coverage report |

---

## Important Rules

- Tools take `target_id` from `interceptor_browser_launch` directly — no attach/detach step
- Always start the proxy with `persistence_enabled: true, capture_profile: "full"` — without this, bodies cap at 4 KB and most data-point evidence is truncated
- Prefer locator-based `humanizer_click` (`role + name`, `text`, `label`) over CSS selectors
- Always `proxy_clear_traffic()` before an interaction to isolate the traffic it generates
- For geographic testing, relaunch the browser with matching `timezone` + `locale` when switching upstream proxy countries — IP geo vs browser geo mismatch is a bot signal
- Use `proxy_search_session_bodies()` to search inside response/request bodies (not `proxy_search_traffic()` which only searches previews, and not `proxy_query_session()` which only searches metadata)
- TLS fingerprint spoofing (`proxy_set_fingerprint_spoof`) is for non-browser HTTP clients only — cloakbrowser already presents an authentic Chrome fingerprint at the source level
