# OWASP LLM & Agentic AI Top 10 (2025-2026)

## OWASP Top 10 for LLM Applications v2.0 (2025)

| # | Risk | Core issue | Testing direction |
|---|------|---------|---------|
| LLM01 | Prompt Injection | Manipulating model behavior through crafted input | Direct injection, indirect injection, encoding bypass |
| LLM02 | Sensitive Information Disclosure | Leakage of PII/API keys/training data | Prompt extraction, output analysis |
| LLM03 | Supply Chain | Poisoned models/libraries/datasets | Model provenance verification, dependency scanning |
| LLM04 | Data & Model Poisoning | Backdoors in training/fine-tuning data | Data provenance, anomalous behavior detection |
| LLM05 | Improper Output Handling | Output causing XSS/SQLi/RCE | Downstream system injection testing |
| LLM06 | Excessive Agency | Excessive tools/autonomy causing real-world harm | Permission audit, human-in-the-loop testing |
| LLM07 | System Prompt Leakage | Extraction of hidden instructions/keys/business logic | Cascading extraction, canary token |
| LLM08 | Vector & Embedding Weaknesses | RAG pipeline attacks, embedding inversion | Retrieval poisoning, semantic-similarity attacks |
| LLM09 | Misinformation | Hallucinations creating security risks in high-risk scenarios | Factuality verification, confidence calibration |
| LLM10 | Unbounded Consumption | DoS/Denial-of-Wallet | Token-consumption testing, rate limiting |

## OWASP Top 10 for Agentic Applications (ASI 2026)

| # | Risk | Core harm | Testing direction |
|---|------|---------|---------|
| ASI01 | Agent Goal Hijack | Malicious input/tool output hijacking the goal | Instruction override, goal tampering |
| ASI02 | Tool Misuse & Exploitation | Unintended use of legitimate tools | Tool-chain composition, parameter injection |
| ASI03 | Identity & Privilege Abuse | Agent performing unauthorized operations | Credential theft, delegation-chain testing |
| ASI04 | Agentic Supply Chain | Real-time risks from MCP descriptors/third-party tools | Dynamic supply-chain scanning |
| ASI05 | Unexpected Code Execution | Prompt → tool → script RCE chain | Multi-layer code-execution testing |
| ASI06 | Memory & Context Poisoning | Long-term memory/embedding poisoning | Persistent-memory attacks |
| ASI07 | Insecure Inter-Agent Communication | Tampering with communication between agents | Man-in-the-middle, replay attacks |
| ASI08 | Cascading Failures | A single-point failure triggering system-wide collapse | Failure-propagation testing |
| ASI09 | Human-Agent Trust Exploitation | Manipulating human operators into approving dangerous actions | Authority-bias/urgency testing |
| ASI10 | Rogue Agents | Agent self-replication/persistent malicious behavior | Persistent-backdoor detection |

## Actual data distribution

Issues found in real assessments:
- LLM01 Prompt Injection: ~45%
- LLM06 Sensitive Info Disclosure: ~20%
- LLM08 Excessive Agency: ~15%
- Remaining 7 items: ~20%

## Key defense principles

1. Separate planning and execution — the model explaining intent ≠ the model executing actions.
2. Bind identity/purpose/scope/validity period — do not use broad environment permissions.
3. Log everything — tool calls/memory/communications as first-class security telemetry.
4. Control blast radius — circuit breakers/rollback/emergency stop take priority over convenience.
5. Treat all natural-language input (including retrieved content) as untrusted.
6. Treat output as untrusted too — sanitize before rendering/executing/querying.
