# Instrumentation

Prefer lightweight observation:

- XHR/Fetch breakpoints
- Function-text breakpoints
- Reading the call stack and local variables after a pause

Only escalate to heavier source rewriting or local instrumentation when lightweight observation is insufficient.
