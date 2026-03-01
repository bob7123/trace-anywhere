# Contributing

## PR Contract
- PLAN (bullets)
- DIFFS (compilable)
- TESTS (unit/integration/UI)
- DOCS (docs updated)
- OPS NOTES (env/config)

## Coding Standards
- Python: mypy strict, async I/O, Pydantic models.
- TS: strict, typed schemas, accessibility.
- Observability: OTel spans (no PII).
- Security: non-root, CAP_NET_RAW only, allow-lists, budgets.

## Review Checklist
- Tests pass in lite profile.
- Probing changes checked manually in kernel profile.
- Docs updated.
- SBOM clean.
