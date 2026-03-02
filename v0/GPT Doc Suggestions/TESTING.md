# Testing

## Profiles
- lite: mocks, runs in CI
- kernel: Linux+netem, raw sockets
- full: real probes on privileged runners

## Categories
- Unit tests: fallback logic, Paris stability, EWMA, CI math.
- Integration: netem scenarios, fallback checks.
- UI: Playwright, screenshots quarantinable, event preservation.
- Docs: link validation, snippet execution.

## CI/CD Gates
- Coverage: 90%+ networking, 80%+ overall.
- Budgets respected in CI.
- Fail on GPL/AGPL deps.
- Every bug fix must include test.

## Example
```python
@pytest.mark.kernel_profile
def test_icmp_fallback():
    # Block ICMP, expect TCP:443 fallback and UI badge
```
