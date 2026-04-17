# VS Code Copilot Instructions

## Project Overview

Visual Studio Code is built with a layered architecture using TypeScript, web APIs and Electron, combining web technologies with native app capabilities. The codebase is organized into key architectural layers:

### Root Folders
- `src/`: Main TypeScript source code with unit tests in `src/vs/*/test/` folders
- `build/`: Build scripts and CI/CD tools
- `extensions/`: Built-in extensions that ship with VS Code
- `test/`: Integration tests and test infrastructure
- `scripts/`: Development and build scripts
- `resources/`: Static resources (icons, themes, etc.)
- `out/`: Compiled JavaScript output (generated during build)

### Core Architecture (`src/` folder)
- `src/vs/base/` - Foundation utilities and cross-platform abstractions
- `src/vs/platform/` - Platform services and dependency injection infrastructure
- `src/vs/editor/` - Text editor implementation with language services, syntax highlighting, and editing features
- `src/vs/workbench/` - Main application workbench for web and desktop
  - `workbench/browser/` - Core workbench UI components (parts, layout, actions)
  - `workbench/services/` - Service implementations
  - `workbench/contrib/` - Feature contributions (git, debug, search, terminal, etc.)
  - `workbench/api/` - Extension host and VS Code API implementation
- `src/vs/code/` - Electron main process specific implementation
- `src/vs/server/` - Server specific implementation
- `src/vs/sessions/` - Agent sessions window, a dedicated workbench layer for agentic workflows (sits alongside `vs/workbench`, may import from it but not vice versa)

The core architecture follows these principles:
- **Layered architecture** - from `base`, `platform`, `editor`, to `workbench`
- **Dependency injection** - Services are injected through constructor parameters
    - If non-service parameters are needed, they need to come after the service parameters
- **Contribution model** - Features contribute to registries and extension points
- **Cross-platform compatibility** - Abstractions separate platform-specific code

### Built-in Extensions (`extensions/` folder)
The `extensions/` directory contains first-party extensions that ship with VS Code:
- **Language support** - `typescript-language-features/`, `html-language-features/`, `css-language-features/`, etc.
- **Core features** - `git/`, `debug-auto-launch/`, `emmet/`, `markdown-language-features/`
- **Themes** - `theme-*` folders for default color themes
- **Development tools** - `extension-editing/`, `vscode-api-tests/`

Each extension follows the standard VS Code extension structure with `package.json`, TypeScript sources, and contribution points to extend the workbench through the Extension API.

### Finding Related Code
1. **Semantic search first**: Use file search for general concepts
2. **Grep for exact strings**: Use grep for error messages or specific function names
3. **Follow imports**: Check what files import the problematic module
4. **Check test files**: Often reveal usage patterns and expected behavior

## Validating TypeScript changes

MANDATORY: Always check for compilation errors before running any tests or validation scripts, or declaring work complete, then fix all compilation errors before moving forward.

- NEVER run tests if there are compilation errors
- NEVER use `npm run compile` to compile TypeScript files

### TypeScript compilation steps
- If the `#runTasks/getTaskOutput` tool is available, check the `VS Code - Build` watch task output for compilation errors. This task runs `Core - Build` and `Ext - Build` to incrementally compile VS Code TypeScript sources and built-in extensions. Start the task if it's not already running in the background.
- If the tool is not available (e.g. in CLI environments) and you only changed code under `src/`, run `npm run compile-check-ts-native` after making changes to type-check the main VS Code sources (it validates `./src/tsconfig.json`).
- If you changed built-in extensions under `extensions/` and the tool is not available, run the corresponding gulp task `npm run gulp compile-extensions` instead so that TypeScript errors in extensions are also reported.
- For TypeScript changes in the `build` folder, you can simply run `npm run typecheck` in the `build` folder.

### TypeScript validation steps
- Use the run test tool if you need to run tests. If that tool is not available, then you can use `scripts/test.sh` (or `scripts\test.bat` on Windows) for unit tests (add `--grep <pattern>` to filter tests) or `scripts/test-integration.sh` (or `scripts\test-integration.bat` on Windows) for integration tests (integration tests end with .integrationTest.ts or are in /extensions/).
- Use `npm run valid-layers-check` to check for layering issues

## Running a local rebuild (desktop + web dev server)

This is the canonical sequence to rebuild this repo and see changes reflected in the localhost:8080 web dev server (and any other run target).

### Gotcha: Node 22 is required
The repo's `.nvmrc` pins **Node 22.22.1**. The gulpfiles are authored in TypeScript (`build/gulpfile.ts`, `build/next/index.ts`), and Node 22+ is required for native `.ts` ESM support. Under Node 20 the build fails immediately with `ERR_UNKNOWN_FILE_EXTENSION` on the gulpfiles.

Claude's shell starts on the system default Node (often 20). Fix by prepending Node 22 to `PATH` for the whole command — NOT just by invoking npm via its absolute path. `npm-run-all2` (used by `npm run watch`) spawns children that resolve `node` via `PATH`, so child processes will still fall back to Node 20 if only the parent has Node 22.

```bash
PATH="$HOME/.nvm/versions/node/v22.22.1/bin:$PATH" npm run watch
```

### What each watch target actually does
- **`npm run watch`** — the canonical one. Runs four tasks in parallel (via `npm-run-all2 -lp`): `watch-client-transpile`, `watch-client`, `watch-extensions`, `watch-copilot`.
- **`npm run watch-client-transpile`** — esbuild transpile `src/` → `out/*.js`. **This is the task that actually emits the JS the dev server serves.** Initial build ~5 seconds.
- **`npm run watch-client`** — TypeScript typecheck only (`noEmit: true` under `useEsbuildTranspile = true` in `build/buildConfig.ts`). Initial typecheck ~45-50 seconds. Does NOT emit.
- **`npm run watch-extensions`** — builds files under `extensions/`.
- **`npm run watch-web`** — misleading name: only builds **web extensions** (extensions that run in the browser), NOT the workbench core. Don't rely on this for `src/` changes.

### CSS & non-TS resource gotcha
`watch-client-transpile` copies non-TS files (CSS, media, etc.) from `src/` → `out/` ONCE at the start of the watch (it prints `[resources] Copying all non-TS files to out...`), but does NOT re-copy them when they change. If you edit a `.css` file mid-session:
- either restart `npm run watch`, or
- `cp src/<path-to-file>.css out/<path-to-file>.css` manually.

### Dev web server
`localhost:8080` is served by `@vscode/test-web` (`node_modules/@vscode/test-web/out/server/index.js`), pointed at `--sourcesPath /Users/znewman/indeed/vscode`. It reads the built `out/` files directly, so once `watch-client-transpile` has emitted, just reload the browser to pick up changes. No separate server rebuild needed.

### Quick diagnostic: are my edits actually in `out/`?
When a change doesn't appear, check the `out/` file timestamp against the `src/` timestamp:
```bash
ls -la out/vs/<path>.js src/vs/<path>.ts
```
If `out/` is older than `src/`, the transpile watch isn't running (or is stuck) — see the Node 22 gotcha above.

## Coding Guidelines

### Indentation

We use tabs, not spaces.

### Naming Conventions

- Use PascalCase for `type` names
- Use PascalCase for `enum` values
- Use camelCase for `function` and `method` names
- Use camelCase for `property` names and `local variables`
- Use whole words in names when possible

### Types

- Do not export `types` or `functions` unless you need to share it across multiple components
- Do not introduce new `types` or `values` to the global namespace

### Comments

- Use JSDoc style comments for `functions`, `interfaces`, `enums`, and `classes`

### Strings

- Use "double quotes" for strings shown to the user that need to be externalized (localized)
- Use 'single quotes' otherwise
- All strings visible to the user need to be externalized using the `vs/nls` module
- Externalized strings must not use string concatenation. Use placeholders instead (`{0}`).

### UI labels
- Use title-style capitalization for command labels, buttons and menu items (each word is capitalized).
- Don't capitalize prepositions of four or fewer letters unless it's the first or last word (e.g. "in", "with", "for").

### Style

- Use arrow functions `=>` over anonymous function expressions
- Only surround arrow function parameters when necessary. For example, `(x) => x + x` is wrong but the following are correct:

```typescript
x => x + x
(x, y) => x + y
<T>(x: T, y: T) => x === y
```

- Always surround loop and conditional bodies with curly braces
- Open curly braces always go on the same line as whatever necessitates them
- Parenthesized constructs should have no surrounding whitespace. A single space follows commas, colons, and semicolons in those constructs. For example:

```typescript
for (let i = 0, n = str.length; i < 10; i++) {
    if (x < 10) {
        foo();
    }
}
function f(x: number, y: string): void { }
```

- Whenever possible, use in top-level scopes `export function x(…) {…}` instead of `export const x = (…) => {…}`. One advantage of using the `function` keyword is that the stack-trace shows a good name when debugging.

### Code Quality

- All files must include Microsoft copyright header
- Prefer `async` and `await` over `Promise` and `then` calls
- All user facing messages must be localized using the applicable localization framework (for example `nls.localize()` method)
- Don't add tests to the wrong test suite (e.g., adding to end of file instead of inside relevant suite)
- Look for existing test patterns before creating new structures
- Use `describe` and `test` consistently with existing patterns
- Prefer regex capture groups with names over numbered capture groups.
- If you create any temporary new files, scripts, or helper files for iteration, clean up these files by removing them at the end of the task
- Never duplicate imports. Always reuse existing imports if they are present.
- When removing an import, do not leave behind blank lines where the import was. Ensure the surrounding code remains compact.
- Do not use `any` or `unknown` as the type for variables, parameters, or return values unless absolutely necessary. If they need type annotations, they should have proper types or interfaces defined.
- When adding file watching, prefer correlated file watchers (via fileService.createWatcher) to shared ones.
- When adding tooltips to UI elements, prefer the use of IHoverService service.
- Do not duplicate code. Always look for existing utility functions, helpers, or patterns in the codebase before implementing new functionality. Reuse and extend existing code whenever possible.
- You MUST deal with disposables by registering them immediately after creation for later disposal. Use helpers such as `DisposableStore`, `MutableDisposable` or `DisposableMap`. Do NOT register a disposable to the containing class if the object is created within a method that is called repeadedly to avoid leaks. Instead, return a `IDisposable` from such method and let the caller register it.
- You MUST NOT use storage keys of another component only to make changes to that component. You MUST come up with proper API to change another component.
- Use `IEditorService` to open editors instead of `IEditorGroupsService.activeGroup.openEditor` to ensure that the editor opening logic is properly followed and to avoid bypassing important features such as `revealIfOpened` or `preserveFocus`.
- Avoid using `bind()`, `call()` and `apply()` solely to control `this` or partially apply arguments; prefer arrow functions or closures to capture the necessary context, and use these methods only when required by an API or interoperability.
- Avoid using events to drive control flow between components. Instead, prefer direct method calls or service interactions to ensure clearer dependencies and easier traceability of logic. Events should be reserved for broadcasting state changes or notifications rather than orchestrating behavior across components.
- Service dependencies MUST be declared in constructors and MUST NOT be accessed through the `IInstantiationService` at any other point in time.

## Learnings
- Minimize the amount of assertions in tests. Prefer one snapshot-style `assert.deepStrictEqual` over multiple precise assertions, as they are much more difficult to understand and to update.
