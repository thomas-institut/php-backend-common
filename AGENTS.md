# AGENTS.md

## Project summary

`thomas-institut/php-backend-common` is a small shared PHP library for backend code.

Main areas in `src/ThomasInstitut/`:

- `ConfigLoader/`: YAML config loading and deep array merge helpers
- `Http/`: HTTP status constants
- `JobQueue/`: Valkey-backed job queue, job stats, and job handler interfaces
- `Profiler/`: lightweight static system profiler
- `Settable/`: reflection-based `fromArray()` trait for DTO/config objects
- `StandardApi/`: Slim/PSR-7 response helpers and route builder

## Tech/runtime expectations

- PHP target: `^8.3` \
- Test framework: PHPUnit 12
- Static analysis: PHPStan level 4
- Key extensions used by the library: `yaml`, `pdo`, `posix`, `pcntl`
- Job queue integration tests use Valkey via `predis/predis`

## Repository layout

- `src/ThomasInstitut/`: library code, PSR-4 namespace root `ThomasInstitut\`
- `test/ThomasInstitut/`: PHPUnit tests, generally mirroring source areas
- `test/.env.local`: env vars for running tests with local PHP against host Valkey
- `docker/`: Dockerfile and compose setup for development Valkey and other PHP versions
- `scripts/test-local`: normal way to run tests using local PHP and local Valkey
- `scripts/test-php8x`: scripts to run tests with specific PHP versions (8.3, 8.4, 8.5)

## Local development commands

Install dependencies locally:

```bash
composer install
```

Run tests locally with host PHP (this is the normal and preferred way for Agents):

```bash
./scripts/test-local
```

Note: If running `php` or `composer` locally does not work right away, try adding `/usr/local/bin` to your `PATH`:
```bash
export PATH=$PATH:/usr/local/bin
```

Run PHPStan locally:

```bash
composer phpstan
```

Run coverage locally:

```bash
set -a && source test/.env.local && set +a && composer test:coverage
```

## Test environment rules

There are several ways to run the tests, and they are easy to mix up:

1. **Local PHP (Normal/Preferred)**
   - Uses local PHP and a local Valkey container.
   - **Agents must use this way** to ensure tests pass after changes.
   - Command: `./scripts/test-local`


2. **Version-Specific PHP Containers**
   - Scripts: `./scripts/test-php83`, `./scripts/test-php84`, `./scripts/test-php85`
   - **Agents must not normally use these** unless specifically instructed to do so.

The integration-sensitive test is `test/ThomasInstitut/JobQueue/ValkeyJobQueueManagerTest.php`. It reads `VALKEY_HOST` and `VALKEY_PORT`, and skips if Valkey is unavailable.

## Code conventions inferred from the codebase

- Follow the existing PSR-4 structure under `src/ThomasInstitut/...`.
- Put tests in the matching namespace/path under `test/ThomasInstitut/...`.
- Preserve existing public APIs unless the task explicitly requires a breaking change.
- Match the existing style:
  - braces on their own line
  - typed properties and return types where already used
  - small utility classes with lightweight PHPDoc
- No formatter config is present; keep edits minimal and style-consistent.

## Change guidance

When changing code:

- Ensure that tests pass by running `./scripts/test-local` after any PHP code changes.
- Add or update PHPUnit coverage alongside behavior changes.
- For job queue changes, consider both pure unit behavior and Valkey-backed behavior.
- For API helper changes, keep PSR-7 response semantics stable.
- For config loading changes, preserve the current deep-merge behavior and error-message patterns because tests assert them.

