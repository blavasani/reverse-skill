---
name: api-security
description: Use for authorized security assessment of REST, GraphQL, WebSocket, or SOAP APIs, including discovery, authentication, authorization, rate-limit, and CI/CD testing.
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.
# API Security Testing

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-pentest.md` — confirm this skill's actions are authorized routine operations
2. `NOW`: Confirm that the current task matches this skill's scope
3. `NEXT`: Read `../tool-index.md` and verify tool availability and actual paths
4. `NEXT`: Invoke bootstrap when a tool is missing; do not guess paths
5. `ACT`: Enter and execute the first workflow step; do not stop in a confirmation state

> Covers all REST / GraphQL / WebSocket / SOAP protocols
> Ten-phase methodology, from discovery through CI/CD integration

## Applicable Scenarios

- REST API security testing (OpenAPI/Swagger-driven or black-box)
- GraphQL security audits (introspection, batching, alias overloading)
- WebSocket security testing
- JWT / OAuth 2.0 authentication testing
- BOLA/IDOR/BFLA authorization vulnerability detection
- API rate-limit bypass and DoS testing

## Ten-Phase Testing Workflow

### Phase 1: API Discovery and Reconnaissance

```text
Active discovery:
□ Vespasian: headless-browser crawling → automatically generate OpenAPI 3.0 / GraphQL SDL specifications
□ Entropy --discover: extract endpoints from robots.txt + JS files
□ Kiterunner / ffuf: brute-force undocumented endpoint paths
□ Check common paths: /swagger.json, /openapi.json, /graphql, /api-docs

GraphQL introspection (three attempts):
  1. Standard introspection query
  2. Minimized query (bypass full WAF blocking)
  3. Query only __schema { types { name } } (minimal probe)
```

### Phase 2: Authentication Testing

```text
JWT analysis (jwt_tool / Burp):
□ alg:none attack: change the header to "alg":"none", clear the signature
□ Key confusion: RS256 public key → HS256 symmetric key
□ Brute-force weak HMAC keys: jwt_tool -C -d wordlist.txt
□ Expiration/claim tampering: modify exp/iat/sub/role claims
□ kid injection: ../../etc/passwd → HMAC signature bypass

OAuth 2.0:
□ redirect_uri manipulation → authorization-code leakage
□ CSRF via missing state parameter
□ Token leakage in the Referer header
□ Detect missing PKCE

GraphQL authentication:
□ mutation bypasses authentication through a GET request (CSRF)
□ Batch-query authentication bypass
```

### Phase 3: Authorization Testing (BOLA/IDOR/BFLA)

```text
BOLA (broken object-level authorization):
□ Enumerate numeric IDs: /user/1 → /user/2 → /user/3
□ Enumerate UUIDs
□ Enumerate usernames/email addresses
□ Burp Autorize: replay and compare two sessions

BFLA (broken function-level authorization):
□ Regular user executes an administrator API
□ HTTP method switching: GET → PUT → PATCH → DELETE
□ API version downgrade: /v2/admin → /v1/admin
□ Bulk-operation injection: {"users": [1,2,3]} → {"users": [1,2,3,admin_id]}

Tools: Burp Autorize, AuthMatrix, Entropy (malicious_insider persona)
```

### Phase 4: GraphQL Special Tests

```text
Introspection leakage → information-exposure detection
Alias overloading → DoS with 100+ aliases
Batch queries → DoS with 10+ simultaneous queries
Field repetition → __typename × 500
Directive overloading → recursive @skip/@include
Circular queries → deeply nested introspection recursion
Field suggestions → information leakage in error messages
GraphiQL/Playground exposure → public IDE risk
GET mutation → CSRF risk
Tracing/debug mode → metadata leakage

Tools: FireTail, Escape DAST, api.sh (Phases 1-3)
```

### Phase 5: REST Input Validation

```text
□ HTTP method switching: GET→POST→PUT→DELETE→OPTIONS→PATCH
□ Content-Type tampering: JSON→XML→multipart
□ NoSQL injection: {"username": {"$gt": ""}}
□ SSRF via URL parameters: webhook URL/avatar URL/import URL
□ XXE in XML endpoints
□ Parameter pollution: /api?role=user&role=admin
□ Mass assignment: add is_admin: true to the request body
```

### Phase 6: Business Logic and Differential Testing

```text
□ Entropy compare: diff v1 vs v2 API → status-code changes/field removal/latency regressions
□ Multi-role workflow testing: admin/user/readonly permission matrix
□ Coupon/points/price manipulation
□ Race condition: test concurrent requests for TOCTOU
```

### Phase 7: WebSocket Testing

```text
□ Endpoint discovery
□ Message injection (inject payloads, prototype pollution)
□ Oversized-message handling
□ Type confusion
□ Cross-site WebSocket hijacking (CSWH)
```

### Phase 8: Rate Limiting and DoS

```text
□ Rate-limit bypass via headers: X-Forwarded-For, X-Real-IP
□ Path variants: /api/ → /api → /Api/ → /API/
□ Slowloris low-bandwidth exhaustion
□ GraphQL deeply nested batch-query DoS
□ IP rotation testing (ProxyCat proxy pool)
```

### Phase 9: Data Exposure

```text
□ Excessive response exposure: compare API responses vs UI display
□ Pagination enumeration: ?page=1&limit=10000
□ Information leakage in error messages: stack traces/internal paths/SQL errors
□ GraphQL nested traversal to access unauthorized data
□ OpenAPI specification exposes sensitive endpoints
```

### Phase 10: CI/CD Integration

```text
□ Entropy --ci --watch: rerun automatically when the spec changes
□ Escape DAST: automatically block builds at the severity threshold
□ Persist findings as regression tests
□ StackHawk (developer-first, ZAP-based)
```

## Toolchain

| Tool | Purpose | Source |
|------|---------|--------|
| Vespasian | Traffic → OpenAPI/GraphQL specification | GitHub: praetorian-inc/vespasian |
| Entropy | LLM-generated attack scenarios, 5 personas | GitHub: arjinexe/entropy-chaos |
| Escape DAST | Business-logic security testing | escape.tech |
| api.sh | 8-phase all-protocol attack pipeline | GitHub: Sharon-Needles/api |
| FireTail | 12 GraphQL specialized tests | firetail.ai |
| jwt_tool | Comprehensive JWT testing | GitHub: ticarpi/jwt_tool |
| Burp Autorize | Two-session authorization comparison | Burp BApp Store |

## References

- `references/rest-graphql-testing.md` — in-depth REST + GraphQL testing
- `references/jwt-oauth-testing.md` — JWT + OAuth security testing

## Task Completion Checklist (MUST pass before claiming completion)

- [ ] Did I execute every workflow step rather than only read it?
- [ ] Did I use real tool paths based on `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/reports)?
- [ ] Did I complete and write back the Checklist items required by RULES?
