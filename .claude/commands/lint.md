# /lint

Run linters and formatters, then report results.

## Steps
1. Run type check: `npx tsc --noEmit`
2. Summarize all findings and group by severity

## Constraints
- Fix auto-fixable issues automatically
- For non-auto-fixable issues, explain the problem and suggest a manual fix
- Do not disable lint rules to suppress warnings
