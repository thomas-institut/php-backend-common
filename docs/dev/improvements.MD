# Recommended improvements

This list is based on a review of the current code in `src/ThomasInstitut/` and the existing tests in `test/ThomasInstitut/`.

The codebase is already small and fairly clean, but if the goal is maximum robustness, these are the improvements I would prioritize.

## Highest priority

1. **Add CI with a real Valkey integration job**
   - The current test suite passes locally, but `ValkeyJobQueueManagerTest` is skipped unless Valkey is available.
   - Robustness depends heavily on the queue implementation, so those tests should run automatically in CI on every change.
   - Recommended:
     - add a CI workflow
     - run PHPUnit and PHPStan on PHP 8.3
     - run queue integration tests in a service container with Valkey
     - optionally add a second job on PHP 8.4 to catch forward-compat issues
   - Affected areas: `test/ThomasInstitut/JobQueue/ValkeyJobQueueManagerTest.php`, `docker/compose.yaml`, `scripts/`

2. **Harden JSON encoding/decoding everywhere**
   - Several paths assume `json_encode()` and `json_decode()` always succeed.
   - If payloads contain invalid UTF-8 or malformed stored data, methods can return bad data or trigger notices.
   - Recommended:
     - use `JSON_THROW_ON_ERROR`
     - catch `JsonException` at boundaries
     - validate decoded structures before indexing into them
   - Most important in:
     - `src/ThomasInstitut/JobQueue/ValkeyJobQueueManager.php`
     - `src/ThomasInstitut/StandardApi/ApiResponseFactory.php`

3. **Validate Valkey-stored job payloads before using them**
   - `ValkeyJobQueueManager` reads arrays from Valkey and immediately assumes keys like `attempts`, `max_attempts`, `secs_between_retries`, and `started_at` exist.
   - Corrupt, partial, or legacy queue data could break processing or recovery.
   - Recommended:
     - introduce a small internal validator/hydrator for job records
     - reject or quarantine malformed records
     - log clearly when stored queue data is invalid
   - Especially important for:
     - `rescheduleJob()`
     - `getJobsByState()`
     - `fetchJob()`
     - `failJob()`
     - `runRecovery()`

4. **Replace the current job signature hashing strategy**
   - `getJobSignature()` currently uses `hash('md2', ...)` over concatenated strings plus `json_encode($payload)`.
   - That is fragile for a few reasons:
     - `md2` is obsolete
     - array key order changes can produce different signatures for equivalent payloads
     - concatenation without structure is harder to reason about than hashing a canonical serialized object
   - Recommended:
     - recursively sort associative payload keys before serialization
     - hash a structured JSON representation
     - use `sha256`
   - Affected area: `src/ThomasInstitut/JobQueue/ValkeyJobQueueManager.php`

5. **Define and enforce queue semantics that are currently implied but not implemented**
   - `JobHandlerInterface` exposes `mustBeUnique()` and `minTimeBetweenSchedules()`, but the queue manager does not use them.
   - That creates a gap between the public API and actual behavior.
   - Recommended:
     - either implement those semantics in `scheduleJob()`
     - or remove them from the interface if they are intentionally unused
   - This is important for correctness and to avoid false assumptions by library consumers.
   - Affected areas:
     - `src/ThomasInstitut/JobQueue/JobHandlerInterface.php`
     - `src/ThomasInstitut/JobQueue/ValkeyJobQueueManager.php`

## Medium priority

6. **Make queue state transitions more defensive and idempotent**
   - Some methods assume jobs are in the expected state when called.
   - For robustness in distributed workers, methods like `finishJob()`, `failJob()`, and recovery logic should tolerate duplicated calls and partial state drift.
   - Recommended:
     - remove stale entries from all relevant structures where appropriate
     - explicitly handle missing processing entries
     - guard against requeueing broken jobs forever during recovery
   - Affected area: `src/ThomasInstitut/JobQueue/ValkeyJobQueueManager.php`

7. **Remove error suppression and tighten YAML/file handling in `ConfigLoader`**
   - `ConfigLoader` uses `@file_get_contents` and `@yaml_parse`, which hides useful failure details.
   - It also relies on a static error message side channel.
   - Recommended:
     - stop suppressing warnings
     - distinguish missing, unreadable, and invalid files
     - validate that parsed YAML is actually an array
     - reset the static error state on success
     - consider an exception-based API in addition to the current nullable-return API
   - Affected area: `src/ThomasInstitut/ConfigLoader/ConfigLoader.php`

8. **Make HTTP/JSON response writing safer**
   - `ApiResponseFactory` writes directly to the current response body without clearing or rewinding it first.
   - If callers pass a response with existing body content, payloads may append instead of replace.
   - Recommended:
     - create a fresh stream or rewind/truncate before writing
     - fail loudly when JSON encoding fails
     - consider adding JSON flags such as `JSON_UNESCAPED_UNICODE` if desired
   - Affected area: `src/ThomasInstitut/StandardApi/ApiResponseFactory.php`

9. **Reduce mutable global/static state**
   - `SystemProfiler` and `ConfigLoader` both rely on static mutable state.
   - This is manageable in short-lived request lifecycles, but it is less robust for long-running workers, parallel test execution, and reuse inside larger systems.
   - Recommended:
     - convert `SystemProfiler` to an instance-based profiler or add an explicit `reset()`
     - avoid static global error state in `ConfigLoader`
   - Affected areas:
     - `src/ThomasInstitut/Profiler/SystemProfiler.php`
     - `src/ThomasInstitut/ConfigLoader/ConfigLoader.php`

10. **Use stronger types for simple value objects and constants**
   - A few classes are effectively immutable value objects but are implemented as mutable classes with public properties or loose constants.
   - Recommended:
     - make value objects `final`
     - use `readonly` where appropriate
     - consider enums for status/state values where it improves safety
   - Good candidates:
     - `DailyJobStats`
     - `JobStats`
     - `ScheduledJobState`
     - `HttpStatus`
     - `ApiResponse` / `ErrorResponse` (possibly with controlled construction)

## Testing and verification improvements

11. **Raise static analysis strictness**
   - PHPStan level 4 is a good start, but this library would benefit from a stricter target.
   - Recommended:
     - move toward level 7 or max incrementally
     - add array-shape annotations for queue/job records
     - enable checks that catch possibly-null offsets and mixed-array access earlier
   - Affected files:
     - `phpstan.neon`
     - most files in `src/ThomasInstitut/JobQueue/`

12. **Add malformed-input and edge-case tests**
   - The current tests cover happy paths well, but robustness needs more adversarial tests.
   - Recommended additions:
     - malformed JSON stored in Valkey
     - missing fields in stored job data
     - payloads with reordered associative keys
     - invalid UTF-8 in API response payloads
     - YAML parsing edge cases in `ConfigLoader`
     - repeated `finishJob()` / `failJob()` / recovery calls
   - Affected areas:
     - `test/ThomasInstitut/JobQueue/ValkeyJobQueueManagerTest.php`
     - `test/ThomasInstitut/ConfigLoader/ConfigLoaderTest.php`
     - `test/ThomasInstitut/StandardApi/ApiResponseFactoryTest.php`

13. **Add coverage reporting to regular development flow**
   - The project already has a coverage command, but robustness work is easier when coverage is part of routine verification.
   - Recommended:
     - generate coverage in CI at least on main branches
     - track coverage specifically for queue state transitions and error handling branches

## Nice-to-have improvements

14. **Clarify `RouteBuilder` validation rules**
   - `RouteBuilder` currently validates class and method existence, which is useful, but could be more explicit about what kinds of callables are supported.
   - Recommended:
     - document supported callable formats
     - optionally reject non-public methods explicitly
     - optionally detect duplicate route definitions early
   - Affected area: `src/ThomasInstitut/StandardApi/RouteBuilder.php`

15. **Add a small top-level developer guide**
   - There is no repository `README.md` right now.
   - A short guide would reduce setup mistakes and make the test matrix easier to run consistently.
   - Recommended sections:
     - local vs container test setup
     - Valkey requirements
     - how to run PHPUnit, PHPStan, and coverage
     - what parts of the code are integration-sensitive

## Suggested implementation order

1. CI with Valkey-backed integration tests
2. JSON and stored-data validation in `ValkeyJobQueueManager`
3. Signature hashing/canonicalization
4. `ConfigLoader` error-handling cleanup
5. `ApiResponseFactory` response-body and JSON hardening
6. Static-analysis level increase and adversarial tests
