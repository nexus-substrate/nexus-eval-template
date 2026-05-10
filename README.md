# nexus-eval-template

Scaffold for building a new nexus-agents evaluation / benchmark harness.

Copy this repo, implement the adapter methods against your benchmark, publish as `nexus-eval-<name>`. Any benchmark you can express as `load instances → produce prediction → evaluate verdict` fits.

## What you get

- `src/adapter.ts` — `BenchmarkAdapter` stub with all 4 required methods and inline "replace this" comments
- `src/cli.ts` — CLI entry point that invokes `runBenchmark()` from nexus-agents
- `src/index.ts` — library export so your adapter can be composed by other tools
- `src/adapter.test.ts` — smoke tests proving the scaffold runs
- `tsconfig.json`, `package.json` — TypeScript strict, vitest, Node 22+
- MIT license, peer dependency on `nexus-agents >= 2.33.0`

## Quick start

```sh
# 1. Copy this repo
gh repo create yourname/nexus-eval-<bench> --template williamzujkowski/nexus-eval-template --public

# 2. Clone + install
gh repo clone yourname/nexus-eval-<bench>
cd nexus-eval-<bench>
npm install

# 3. Sanity check — the template tests pass out of the box
npm test
```

## The contract

Every `nexus-eval-*` package implements one interface from `nexus-agents`:

```ts
interface BenchmarkAdapter<TInstance, TPrediction, TEvalResult> {
  readonly name: string;
  readonly variant?: string;
  loadInstances(config): Promise<readonly TInstance[]>;
  runInstance(instance, ctx): Promise<TPrediction>;
  evaluate(instance, prediction): Promise<TEvalResult>;
  isPass(result): boolean;
  summarize(results, runTimeMs): BenchmarkRunSummary;
}
```

The orchestrator (`runBenchmark` in nexus-agents) handles concurrency, timeouts, progress, and partial failure for you — you don't reimplement the harness.

## Implementation steps

1. **Rename** `nexus-eval-BENCHMARK` to your benchmark name in `package.json` (name, bin, description).
2. **Replace `BenchmarkInstance` / `BenchmarkPrediction` / `BenchmarkEvalResult`** in `src/adapter.ts` with your benchmark's actual shapes.
3. **Implement `loadInstances`** — read your dataset from disk or fetch from an API.
4. **Implement `runInstance`** — call your solver (usually a CLI subprocess or API call).
5. **Implement `evaluate`** — run tests / diff against ground truth / grade with an LLM.
6. **Customize `summarize`** — add benchmark-specific breakdowns in `metadata` (pass-by-category, dataset version, etc.).
7. **Customize the CLI** — most of `src/cli.ts` stays the same; update flags for variant names specific to your benchmark.
8. **Tag your repo** — `gh repo edit --add-topic nexus-agents-eval` so `ECOSYSTEM.md` discovery works.

## Tips

- **No HTTP server needed.** Adapters are libraries + CLIs. nexus-agents is a peer dependency; you don't need to run its MCP server to exercise the contract.
- **Per-instance failures don't abort the run.** If one instance throws, `runBenchmark` records it in `summary.metadata.failureCount` and continues.
- **Honor `ctx.signal`** in your `runInstance` so long runs can be cancelled.
- **Put variants into `config` or the constructor**, not CLI flags passed through to every instance. Example: `new MyBenchAdapter({ variant: 'lite' })`.
- **Keep pure evaluation separate from network calls.** Makes the tests reproducible and fast.

## Recommended v0.1 architecture (matches the four shipped eval repos)

The five `nexus-eval-*` repos that have shipped (`swebench`, `swebench-pro`, `aider-polyglot`, `livecodebench`, `atbench`) all converged on the same internal layout. New harnesses are easier to review when they follow it:

```
src/
├── types.ts                         # Public {Instance, Prediction, EvalResult, AdapterConfig}
├── runner/
│   ├── instance-loader.ts           # Bundled fixture + .jsonl/HF source (with field normalisation)
│   ├── prompt-template.ts           # Compose system + user prompts
│   ├── <output>-extractor.ts        # Parse model response (patch / code / edits / answer)
│   └── agent-invoker.ts             # IModelAdapter wrapper; never throws (returns Result)
├── adapter.ts                       # `<Bench>Adapter` implementing BenchmarkAdapter
├── cli.ts                           # `parseArgs` + `createOpenAIAdapter` + `runBenchmark`
├── index.ts                         # Public exports (adapter + types + runner building blocks)
└── adapter.test.ts                  # Smoke tests (mocked IModelAdapter)
```

Adapter constructor signature is consistent: `constructor(modelAdapter: IModelAdapter, config: AdapterConfig = {})`. The `IModelAdapter` is supplied by the consumer; the harness never instantiates it directly. Operators can swap in `createOpenAIAdapter`, the Anthropic SDK, Ollama, etc. without changing the harness.

### v0.1 → v0.2 → v0.3 progression

The shipped repos use a consistent versioning ladder:

- **v0.1 — model-only baseline**. Bundled fixture + single-round-trip model call + extract-output. Pass/fail = "did the model produce extractable output". No real evaluation.
- **v0.2 — real evaluation**. HuggingFace / GitHub fetch loader. Sandboxed test execution. Pass/fail = "does the output pass the hidden tests".
- **v0.3 — agentic flow**. `ICliAdapter` instead of `IModelAdapter`. Multi-turn iteration on test failures. Workspace clone at base commit (where the benchmark provides one).

Land v0.1 first to validate the pipeline shape and prompt template; the expensive infrastructure (Docker eval, GPU sandbox, etc.) goes in v0.2 once the pattern is proven.

### v0.2 patterns (concrete, from shipped harnesses)

`nexus-eval-aider-polyglot` and `nexus-eval-livecodebench` both shipped v0.2 with the same shape — copy this:

**Loader (always async; PR-split into pieces):**

```
src/runner/<source>-loader.ts        # github-loader.ts, hf-loader.ts, ...
```

- Returns `Promise<readonly TInstance[]>` — never sync. Even cache-only paths are async so the public signature stays stable.
- Accepts a `fetchImpl?: typeof fetch` injection point so unit tests don't monkey-patch globals. Default `globalThis.fetch`.
- Caches normalised instances to `~/.nexus-eval-<bench>/cache/<source-slug>/<ref-slug>/<extra>.json`. Second run hits cache, skips network.
- Pins a default `<ref>` (commit SHA / dataset config / release slice) and lets operators override via `--source <kind>:<ref>`. Reproducibility ≠ "use upstream main".
- Reads `GITHUB_TOKEN` / `HF_TOKEN` env vars when present; fails closed with a "set the token if rate-limited" error message rather than retrying anonymously.
- Filters at the fetch boundary (`platforms`, `languages`, `min-release-date`) so heavy paginations don't pull all-then-throw-away.

**Sandboxed test runner (`runner/test-runner.ts` or `runner/<lang>-runner.ts`):**

- Materialises the model's emitted output + the instance's hidden tests into a tmpdir, then `child_process.spawn`s the language toolchain. No shell, no string concatenation, no env inheritance for secrets.
- Accepts a `spawnImpl?: SpawnImpl` injection point so unit tests don't actually run pytest / cargo / etc in CI. Default `child_process.spawn`.
- Per-instance timeout via `setTimeout` + `child.kill('SIGKILL')` + a `timedOut` flag in the result.
- Surfaces a `toolchainMissing: true` flag on `ENOENT` so operators get a clear "you need pytest in PATH" error instead of a confusing exit-code-1 verdict.
- Caps stdout + stderr at 4 KB each. Exercise output is third-party; metadata stays bounded.
- Refuses to materialise paths containing `..` (parent-directory traversal).
- Scrubs the env passed to the subprocess: drop `OPENAI_*`, `ANTHROPIC_*`, `NEXUS_*`, `GITHUB_TOKEN`, `NPM_TOKEN`, `HF_TOKEN`, `AWS_*`. Exercise code is semi-trusted; env-var inspection blocked.

**Adapter wiring:**

```ts
export class FooAdapter implements BenchmarkAdapter<Instance, Prediction, EvalResult> {
  // runInstance: model call only — caches a "model-only verdict" so
  // evaluate() has a sensible fallback when test execution is skipped.
  async runInstance(...) { ... }

  // evaluate: when runTests is on (default) AND tests exist AND code/edits
  // were emitted, materialise tmpdir + run toolchain. Otherwise return
  // the model-only verdict.
  async evaluate(...) { ... }

  // isPass: prefer test-based verdict when present; fall back to v0.1.
  isPass(result) {
    if (result.testsPassed !== undefined) return result.testsPassed;
    return result.editsProduced /* or whatever the v0.1 signal was */;
  }

  // Test-only spawn injection.
  setSpawnImplForTests(impl) { this.spawnImplForTests = impl; }
}
```

**Type extensions (additive, non-breaking from v0.1):**

```ts
interface FooAdapterConfig {
  // existing v0.1 fields...
  readonly runTests?: boolean;        // default true; --no-run-tests opts out
  readonly testTimeoutMs?: number;    // default 60_000 or 15_000 per test
}

interface FooEvalResult {
  // existing v0.1 fields...
  readonly testsPassed?: boolean;     // canonical pass/fail when present
  readonly testRunner?: string;       // "pytest", "go test", ...
  readonly testStderr?: string;       // truncated to 4 KB
  readonly toolchainMissing?: boolean;
}
```

**Don't break v0.1:** v0.2 `EvalResult` extensions are all optional. Old fixture-based runs that have no hidden tests still work — the test-runner skips, the v0.1 verdict stands.

### CI / release scaffolding

Each shipped repo has the same two GitHub Actions workflows:

- `.github/workflows/ci.yml` — typecheck + test + build on Ubuntu / Node 22, runs on PR + push to main.
- `.github/workflows/release.yml` — tag-triggered (`v*`) `npm publish --provenance`. Requires `NPM_TOKEN` repo secret.

Copy these from `nexus-eval-swebench-pro` or `nexus-eval-aider-polyglot` when scaffolding a new harness — they're identical except for the package name in the release-job comment.

## Why a separate repo?

The nexus-agents core stays lean — benchmark harnesses are evaluation-only code that 99% of consumers don't run. Concentrating them in dedicated `nexus-eval-*` repos lets each harness:

- Evolve on its own cadence (dataset bumps, harness rewrites, model-API churn) without forcing nexus-agents minor releases.
- Pull in its own dependency tree (Docker SDKs, dataset libs, eval-specific Python tooling) without bloating the npm-installable core.
- Be peer-tested in isolation — the BenchmarkAdapter contract at the boundary is the only API surface either side has to maintain.

This is policy, not a suggestion: nexus-agents' [`benchmark-extraction-gate`](https://github.com/williamzujkowski/nexus-agents/blob/main/.github/workflows/benchmark-extraction-gate.yml) workflow fails CI on any PR that adds files under `packages/nexus-agents/src/swe-bench/` or `packages/nexus-agents/src/benchmarks/atbench/`. If you're proposing a new benchmark, this template is the right starting point. See [nexus-agents epic #2514](https://github.com/williamzujkowski/nexus-agents/issues/2514) for the rationale.

## Existing benchmarks using this pattern

- [nexus-eval-swebench](https://github.com/williamzujkowski/nexus-eval-swebench) — SWE-bench Lite / Verified / Full (clean-room rewrite, v0.2)
- [nexus-eval-swebench-pro](https://github.com/williamzujkowski/nexus-eval-swebench-pro) — SWE-bench Pro (731 multi-language instances)
- [nexus-eval-aider-polyglot](https://github.com/williamzujkowski/nexus-eval-aider-polyglot) — Aider polyglot (six-language code edits, v0.1)
- [nexus-eval-livecodebench](https://github.com/williamzujkowski/nexus-eval-livecodebench) — LiveCodeBench (competitive programming, v0.1)
- [nexus-eval-atbench](https://github.com/williamzujkowski/nexus-eval-atbench) — atbench (agent-trajectory safety, v0.1)

## Ecosystem

See [nexus-agents ECOSYSTEM.md](https://github.com/williamzujkowski/nexus-agents/blob/main/ECOSYSTEM.md) for the full registry.

## License

MIT.
