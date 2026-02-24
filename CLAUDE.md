## CRITICAL: File Editing on Windows
### ⚠️ MANDATORY: Always Use Backslashes on Windows for File Paths
**When using Edit or MultiEdit tools on Windows, you MUST use backslashes (`\`) in file paths, NOT forward slashes (`/`).**
#### ❌ WRONG - Will cause errors:
```
Edit(file_path: "D:/repos/project/file.tsx", ...)
MultiEdit(file_path: "D:/repos/project/file.tsx", ...)
```
#### ✅ CORRECT - Always works:
```
Edit(file_path: "D:\repos\project\file.tsx", ...)
MultiEdit(file_path: "D:\repos\project\file.tsx", ...)
```

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development commands

The project is a Bun/TypeScript server.  Most common workflows use the `bun` CLI.

```sh
# install deps
bun install

# compile to `dist/` (used by the published package)
bun run build            # runs `tsdown` as defined in package.json

# run in development mode with file watching
bun run dev              # equivalent to `bun run --watch ./src/main.ts`

# run the production entrypoint from source
bun run start            # sets NODE_ENV=production and executes ./src/main.ts

# linting
bun run lint             # eslint with cache
bun run lint:all         # lint entire repo

# type check
bun run typecheck        # invokes tsc

# tests
env BUN_INSTALL && bun test   # Bun’s test runner executes files in tests/**/*.ts
```

There are a few other helper scripts (knip, release) used by the repo but the above are the ones you will use most often.

### Docker

The README already documents building (`docker build -t copilot-api .`) and running the container.  A `start.bat` script is provided for Windows.  The image exposes port `4141` and persists the GitHub token under `/root/.local/share/copilot-api` when a host volume is mounted.

## High‑level architecture

- **CLI entrypoint**: `src/main.ts` defines a `citty` command with four subcommands: `start`, `auth`, `check-usage`, and `debug`.  Each command lives in its own module (`src/auth.ts`, `src/start.ts`, etc.) and exports a `defineCommand` object.

- **State & helpers**: `src/lib/*` contains lightweight utilities and shared state:
  - `state.ts` is a simple singleton holding tokens, rate‑limit settings, cached models, etc.
  - `token.ts` and `auth.ts` manage GitHub/Copilot authentication flows.
  - `paths.ts` ensures the filesystem layout for storing tokens.
  - `proxy.ts`, `rate-limit.ts`, `utils.ts`, and other modules implement miscellaneous helpers used across commands and services.

- **Server and routing**: `src/server.ts` builds a Hono application.  The bulk of the HTTP API is defined under `src/routes/*`:
  - `chat-completions`, `embeddings`, and `models` expose OpenAI‑compatible endpoints.
  - `messages` implements Anthropic‑style `/v1/messages` and token counting.
  - `token.ts` and `usage.ts` provide usage‑monitoring endpoints used by the web dashboard.
  - Each route’s handler usually delegates to a corresponding service in `src/services/copilot`.

- **Copilot services**: `src/services/copilot/*` contains the code that translates incoming requests to the reverse‑engineered GitHub Copilot API, adding proper headers, token rotation, and streaming support.  There are helper services for creating chat completions, embeddings, fetching available models, etc.

- **GitHub services**: `src/services/github/*` call GitHub’s OAuth/device‑flow endpoints to obtain a Copilot token, and fetch usage stats.

- **Entrypoint logic**: `src/start.ts` wires everything together.  It processes CLI flags (rate limits, manual approval, account type, `--claude-code` helper), initializes state, caches available models and VSCode version, optionally generates environment variables for launching Claude Code, and finally starts the HTTP server with `srvx`.

- **Tests**: The `tests/` directory contains a handful of unit tests written against Bun’s test runner (`bun:test`).  They mock `fetch` and exercise the service modules.

- **Packaging**: `tsdown` compiles TypeScript to `dist/` which is published to npm; `package.json` declares the CLI binary as `./dist/main.js`.

### Important notes for future instances

- The project targets Bun and assumes ESM (`"type": "module"` in package.json).
- Routes and services avoid complex dependency injection; state is read from the singleton and mutated directly.
- Rate‑limit, manual approval, and other runtime flags are stored in `state` and consulted by middleware in the route handlers.
- The codebase is intentionally small; most of the logic lives in a few dozen TypeScript files under `src/`.

By referencing this file, future Claude Code sessions should quickly understand how to build, run, and navigate the repository.

## Known runtime quirks

* When using GPT‑5 mini (or other unsupported model names) with Claude Code through the proxy, the GitHub Copilot backend may respond with a 400 `model_not_supported` error even though the request may still complete successfully.  Example log output:

  ```
  ERROR  Failed to create chat completions Response { status: 400,
    statusText: 'Bad Request',
    ...
    url: 'https://api.githubcopilot.com/chat/completions' }

  ERROR  Error occurred: Failed to create chat completions

    at createChatCompletions (.../dist/main.js:783:9)
    …

  ERROR  HTTP error: { error: { message: 'The requested model is not supported.',
    code: 'model_not_supported',
    param: 'model',
    type: 'invalid_request_error' } }
  ```

  The underlying API still returns a response and Claude Code continues to work.  There are no open pull requests currently addressing this; the only active PR (`coderabbitai/docstrings/0ea08fe`) adds API key authentication and docstrings, which is unrelated.  Future changes may need to add model name validation or mapping if GitHub starts rejecting these names more strictly.
