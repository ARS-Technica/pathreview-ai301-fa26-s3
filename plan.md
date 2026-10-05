# Implementation Plan - Issue #66: structlog output is not captured by pytest caplog

## Diagnosis

The application logs through `structlog`, which is not configured to capture or route log records into standard library (`logging`) handlers during test execution. As a result, pytest's `caplog` fixture (which captures `logging` module output) remains empty during assertions even when the code under test correctly emits log events.

This root cause was verified during local reproduction on commit `2f4e82f`. 

### Quoted Reproduction Evidence
**Environment:**
 - OS: Windows 11 (`win32`)
 - Python: 3.13.5
 - pytest: 9.1.1
 - structlog: 26.1.0

**Command executed:**
```powershell
python -m pytest "tests/unit/test_batch_processor.py::TestBatchEmbeddingProcessor::test_empty_chunks_list_returns_empty" -vv -s --runxfail
```

**Observed Output:**
The warning message was emitted to stderr during the test run:
```text
Empty chunks list provided to BatchEmbeddingProcessor
```

The test then failed specifically at the `caplog` assertion:
```text
       assert "Empty chunks list" in caplog.text or any(
            "empty" in record.message.lower() for record in caplog.records
         )
E       AssertionError: assert ('Empty chunks list' in '' or False)
E        +  where '' = <_pytest.logging.LogCaptureFixture object at 0x000001F4BBDFCC20>.text
E        +  and   False = any(...)
```

**Conclusion:**
`structlog` emits the warning during execution, but because it is not propagated to stdlib logging in `tests/conftest.py`, pytest's `caplog` fixture fails to capture the output.

---

## Scope

### In Scope
- Configure `structlog` inside `tests/conftest.py` so that its outputs propagate to `logging.stdlib` during pytest runs.
- Ensure existing suite-wide log assertions using `caplog` capture `structlog` messages.
- Remove the `@pytest.mark.xfail(strict=True, reason="issue #66...")` decorator on `test_empty_chunks_list_returns_empty` in `tests/unit/test_batch_processor.py` so the test passes natively.

### Not in Scope
- Modifying production logging logic or changing `structlog` behavior outside the test suite setup in `tests/conftest.py`.
- Altering existing application business logic or adding new test cases beyond fixing the `caplog` assertion bridge.

---

## Files to Modify

1. `tests/conftest.py`: Add `structlog.configure()` or standard library integration processors (such as `structlog.stdlib.filter_by_level` and `structlog.stdlib.add_logger_name`) so `structlog` formats and emits records through `logging.getLogger()`.
2. `tests/unit/test_batch_processor.py`: Remove the temporary `xfail` marker from `test_empty_chunks_list_returns_empty`.

---

## Approach

1. Inspect `tests/conftest.py` to see current fixture setups.
2. Add a session-level pytest fixture or top-level configuration in `tests/conftest.py` using `structlog.configure()` configured with stdlib processors:
   - `structlog.stdlib.add_logger_name`
   - `structlog.stdlib.add_log_level`
   - `structlog.stdlib.PositionalArgumentsFormatter()`
   - `structlog.processors.StackInfoRenderer()`
   - `structlog.processors.format_exc_info()`
   - `structlog.stdlib.ProcessorFormatter.wrap_for_formatter`
   - Set `logger_factory=structlog.stdlib.LoggerFactory()`
3. Ensure standard library logging handler captures these events during testing.
4. Remove the `xfail` decorator on `test_empty_chunks_list_returns_empty`.

---

## Test Plan

### Pre-Fix Verification Command (Reproduction Step Re-run)
Run the unit test target:
```powershell
python -m pytest "tests/unit/test_batch_processor.py::TestBatchEmbeddingProcessor::test_empty_chunks_list_returns_empty" -vv -s

---

## Expected Output After Fix
1. The test executes without requiring the --runxfail flag.

2. caplog.text contains "Empty chunks list" (or caplog.records contains the log record).

3. The test output reports a clean pass:
```
PASSED tests/unit/test_batch_processor.py::TestBatchEmbeddingProcessor::test_empty_chunks_list_returns_empty
1 passed in 0.25s
```

4. Re-run the full unit test suite (python -m pytest tests/unit/) to ensure no existing tests regress.

## Risks and Unknowns
Log Formatting Interference: Adding stdlib logging integration in conftest.py might affect console output formatting for other tests if standard handlers are attached multiple times.

Fixture Ordering: Ensuring structlog is configured before any module-level loggers are instantiated during test collection.

---

## Deviations
No deviations were made from the original plan. The fix was implemented exactly as proposed by configuring `structlog` standard library logging integration in `tests/conftest.py` and removing the `xfail` marker from `test_empty_chunks_list_returns_empty` in `tests/unit/test_batch_processor.py`.