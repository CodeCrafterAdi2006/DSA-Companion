# Custom Rules for DSA Companion

## AI Node Selection Preference
- **Rule:** Before adding, changing, or recommending any nodes related to AI (LLMs, chat models, text parsing, etc.), the agent **MUST** first ask the user if they prefer to use direct **HTTP request nodes** or **official native AI nodes** (e.g. Basic LLM Chain, Groq Chat Model, Google Gemini Chat Model). Do not assume one over the other without explicit user consent.
## AI Analys
- **Rule:** Before any sort of analysis or making change to the node after a query is asked the agent **MUST** first explain the cause of the error and what the solution to be implemented. Do not assume one over the other without explicit user consent.
