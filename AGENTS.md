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

- PHP target: `^8.3` (`composer.json` also pins platform PHP to `8.3`)
- Test framework: PHPUnit 12
- Static analysis: PHPStan level 4
- Key extensions used by the library: `yaml`, `pdo`, `posix`, `pcntl`
- Job queue integration tests use Valkey via `predis/predis`

## Repository layout

- `src/ThomasInstitut/`: library code, PSR-4 namespace root `ThomasInstitut\`
- `test/ThomasInstitut/`: PHPUnit tests, generally mirroring source areas
- `test/.env.local`: env vars for running tests with local PHP against host Valkey
- `test/.env.container`: env vars for running tests inside container PHP
- `docker/`: Dockerfile and compose setup for PHP + Valkey
- `scripts/composer`: run Composer inside the long-running PHP container
- `scripts/test`: shortcut for running the Composer `test` script in the container

## Local development commands

Install dependencies locally:

```bash
composer install
```

Run tests locally with host PHP:

```bash
set -a && source test/.env.local && set +a && composer test
```

Run PHPStan locally:

```bash
composer phpstan
```

Run coverage locally:

```bash
set -a && source test/.env.local && set +a && composer test:coverage
```

## Container development commands

Start the Docker environment from the repository root:

```bash
docker compose -f docker/compose.yaml up -d --build
```

This starts:

- `backend-common-php` (service `shared-php`)
- `backend-common-valkey` (service `valkey`)

Use the helper script to run Composer inside the PHP container:

```bash
./scripts/composer install
./scripts/composer test
./scripts/composer phpstan
```

There is also a dedicated test shortcut:

```bash
./scripts/test
```

Notes:

- `scripts/composer` uses `docker exec`, so the PHP container must already be running.
- Inside the container, tests should use the container Valkey host/port (`valkey:6379`). Those values are documented in `test/.env.container`.
- In practice, the Valkey integration test already defaults to `valkey:6379`, so explicit sourcing is usually only needed when running with local host PHP.

## Test environment rules

There are **two supported test environments** and they are easy to mix up:

1. **Local PHP**
   - Must load `test/.env.local`
   - Uses `VALKEY_HOST=localhost` and `VALKEY_PORT=6333`
   - Recommended command:

   ```bash
   set -a && source test/.env.local && set +a && composer test
   ```

2. **Container PHP**
   - Run through `./scripts/composer ...` (or `./scripts/test`)
   - Uses the Docker network host `valkey` on port `6379`
   - Typical flow:

   ```bash
   docker compose -f docker/compose.yaml up -d --build
   ./scripts/composer test
   ```

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

- Add or update PHPUnit coverage alongside behavior changes.
- For job queue changes, consider both pure unit behavior and Valkey-backed behavior.
- For API helper changes, keep PSR-7 response semantics stable.
- For config loading changes, preserve the current deep-merge behavior and error-message patterns because tests assert them.

