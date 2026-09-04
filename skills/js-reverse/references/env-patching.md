# Environment Patching Rules

- Only patch objects that page evidence has already proven to be needed
- Patch one minimal causal unit at a time
- Patch values first, then function shells, then returned-object contracts
- Re-execute after every patch and record whether the first divergence moved forward
