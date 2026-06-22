# AI-Assisted Refactor: Atlas Config Loader

This is a synthetic worked example of a safe AI-assisted refactor on Atlas (fictional internal developer platform and CLI). It demonstrates the approach described in [`docs/ai-assisted-workflow.md`](../docs/ai-assisted-workflow.md) and [`docs/human-in-the-loop.md`](../docs/human-in-the-loop.md): characterization tests first, small reversible steps, AI proposes changes, human validates behavior is unchanged.

---

## Background

**System:** Atlas (fictional internal developer platform and CLI)
**Target:** `config/loader.ts` — the module responsible for loading Atlas's YAML configuration files, merging environment-variable overrides, and resolving relative paths.
**Problem:** The loader had accumulated four years of incremental additions. It was a single 480-line function with no internal structure, several ad-hoc parsing branches for deprecated config keys, and no tests. Every change to it caused anxiety. Two recent bugs traced back to unexpected interactions in the loader.

**Goal:** Refactor the loader into a set of composable, testable functions without changing any externally observable behavior. No new features. No behavior changes.

**Non-goals:** Removing deprecated key support (that is a separate, breaking change), adding new config options, changing the public API surface.

---

## The safety principle

The risk in any refactor is changing behavior accidentally. With an AI-assisted refactor, the risk is higher: AI can produce code that looks correct and passes a quick read but has subtle differences in edge-case handling.

The approach used here:

1. Write characterization tests that capture current behavior — including quirks — before touching the implementation.
2. Run all characterization tests and confirm they pass against the original code.
3. Make changes in small, reviewable steps. Each step is a separate commit.
4. After each step, run all characterization tests. Any failure means the step changed behavior.
5. AI proposes each step's implementation. The human reviews the diff before committing.
6. Do not merge until all characterization tests pass and the human has read every changed line.

This is not "trust AI to refactor safely." It is "use AI to write code faster, while the tests and human review catch anything wrong."

---

## Stage 1: Characterization tests

Characterization tests capture what the code *currently does*, not what it *should* do. They are written by running the existing code and recording its output — including edge cases and surprising behavior.

The delivery engineer spent half a day on this. The approach was to run the existing loader with varied inputs and capture the outputs.

**Inputs tested:**

| Scenario | Input | Captured behavior |
|----------|-------|-------------------|
| Minimal valid config | `{version: 1, project: "foo"}` | Returns with defaults applied |
| Missing optional keys | Omit `log_level`, `timeout` | Defaults to `"info"`, `30` |
| Deprecated key `project_name` | `{project_name: "foo"}` | Silently maps to `project`, logs deprecation warning |
| Both `project` and `project_name` | `{project: "bar", project_name: "foo"}` | `project` wins, deprecation warning still emitted |
| Relative path in `output_dir` | `{output_dir: "./dist"}` | Resolved relative to config file location, not cwd |
| ENV override of `log_level` | `ATLAS_LOG_LEVEL=debug` | Overrides config file value |
| ENV override takes precedence | Config has `timeout: 60`, `ATLAS_TIMEOUT=10` | Returns 10 |
| Invalid YAML | Malformed file | Throws `ConfigParseError` with filename in message |
| File not found | Non-existent path | Throws `ConfigNotFoundError` with path in message |
| Numeric env override | `ATLAS_TIMEOUT=abc` | Throws `ConfigValidationError` |

**Surprising behaviors captured:**

- The `project_name` deprecation warning is emitted even when `project` is also present. This was not obvious from reading the code and would be easy to accidentally remove.
- Relative path resolution uses the config file's directory, not `process.cwd()`. This behavior is relied on by all existing Atlas users but was not documented anywhere.
- The numeric validation for `ATLAS_TIMEOUT` throws rather than using the default. This is arguably wrong but is current behavior — the characterization test captures it as-is.

```typescript
// Example characterization test
describe('config loader — relative path resolution', () => {
  it('resolves output_dir relative to the config file, not cwd', () => {
    const configPath = '/tmp/test-fixtures/subdir/atlas.yml';
    writeFileSync(configPath, 'version: 1\nproject: test\noutput_dir: ./dist');
    const result = loadConfig(configPath);
    expect(result.output_dir).toBe('/tmp/test-fixtures/subdir/dist');
    // NOT process.cwd() + '/dist'
  });
});
```

All characterization tests were run against the original code. Every test passed. This established the baseline.

---

## Stage 2: Identify the refactor steps

The delivery engineer read the 480-line function and identified the distinct concerns:

1. **File loading** — read the YAML file from disk, throw on not-found or parse error
2. **Deprecated key migration** — map old keys to new keys, emit warnings
3. **Environment variable merging** — apply `ATLAS_*` env vars over config file values
4. **Validation** — check types and required fields, throw on invalid values
5. **Path resolution** — resolve relative paths in path-typed fields

Each concern was roughly 80–120 lines in the original function, entangled with the others. The refactor would extract each concern into its own function with a clear input and output.

The delivery engineer documented these five steps before asking AI to implement any of them. Each step was designed so it could be done, tested, and committed independently — no step required completing another to be testable.

---

## Stage 3: AI-assisted implementation — step by step

### Step 1: Extract `loadYamlFile()`

**Prompt given to AI:**

```text
I'm refactoring Atlas's config loader. First step: extract the file-loading concern.
Here is the relevant section of the original function (lines 12–67, pasted).
Extract it into a function called loadYamlFile(filePath: string): unknown.
It should throw ConfigNotFoundError if the file doesn't exist,
and ConfigParseError if the YAML is invalid. Both error classes already exist —
I'll paste their signatures.
Do not change any behavior. Do not add error handling that isn't already there.
Return only the extracted function. I will integrate it.
```

AI produced `loadYamlFile()`. The delivery engineer reviewed the diff:

- Error class usage was correct.
- The original code checked `existsSync` before reading. AI preserved this.
- One difference found: AI added a `try/catch` around the `existsSync` call that wasn't in the original. The original let `existsSync` throw natively (it doesn't throw, but the check was redundant). Removed the extra `try/catch` to stay exactly behavior-equivalent.

All characterization tests: pass. Committed.

### Step 2: Extract `applyDeprecatedKeys()`

**Prompt:**

```text
Next step: extract the deprecated key migration.
Here is the relevant section (lines 68–142, pasted).
Extract into applyDeprecatedKeys(raw: Record<string, unknown>): Record<string, unknown>.
It should return a new object with deprecated keys mapped to their replacements,
and emit the same deprecation warnings using the existing logger.warn() calls.
Do not change which warnings are emitted or when.
```

AI produced the function. The delivery engineer checked:

- The "both `project` and `project_name` present" case: AI emitted the warning only when `project_name` was present and `project` was absent. The original emits the warning whenever `project_name` is present, regardless of `project`. This was a behavioral difference.
- The delivery engineer corrected the condition before committing.

**This is the exact risk AI-assisted refactoring carries:** the logic looked right at a glance and would have passed most tests, but the characterization test for "both keys present" caught it.

All characterization tests: pass after correction. Committed.

### Step 3: Extract `mergeEnvOverrides()`

**Prompt:**

```text
Next: extract environment variable merging.
Here is the section (lines 143–220, pasted).
Extract into mergeEnvOverrides(config: AtlasConfig): AtlasConfig.
It should read ATLAS_* env vars and apply them over the config values.
Numeric fields should be parsed with parseInt; throw ConfigValidationError
on non-numeric values for numeric fields. Same behavior as today.
```

AI produced the function. Review found no differences from the original. All characterization tests: pass. Committed.

### Step 4: Extract `validateConfig()`

**Prompt:**

```text
Next: extract validation.
Here is the section (lines 221–340, pasted).
Extract into validateConfig(config: AtlasConfig): void.
It should throw ConfigValidationError for each invalid field.
Same error messages as today — I'll need to compare the messages exactly.
```

AI produced the function. The delivery engineer compared the error message strings in the original with those in the AI output:

- `timeout` out-of-range message: original used `"timeout must be between 1 and 3600"`, AI produced `"timeout must be a number between 1 and 3600"`. Corrected to match original exactly (characterization tests check error messages).
- All other messages matched.

All characterization tests: pass after correction. Committed.

### Step 5: Extract `resolveConfigPaths()`

**Prompt:**

```text
Final step: extract path resolution.
Here is the section (lines 341–420, pasted).
Extract into resolveConfigPaths(config: AtlasConfig, configFilePath: string): AtlasConfig.
Paths must be resolved relative to the directory of configFilePath using path.dirname(),
not process.cwd(). That is the existing behavior — do not change it.
```

AI produced the function and correctly used `path.dirname(configFilePath)`. The delivery engineer verified this explicitly because it was one of the surprising behaviors captured in characterization tests. Correct.

All characterization tests: pass. Committed.

### Final integration

The delivery engineer wrote the new `loadConfig()` function, composing the five extracted functions in the correct order:

```typescript
export function loadConfig(filePath: string): AtlasConfig {
  const raw = loadYamlFile(filePath);
  const migrated = applyDeprecatedKeys(raw as Record<string, unknown>);
  const withDefaults = applyDefaults(migrated);
  const withEnv = mergeEnvOverrides(withDefaults as AtlasConfig);
  validateConfig(withEnv);
  return resolveConfigPaths(withEnv, filePath);
}
```

`applyDefaults()` was a small function the delivery engineer wrote by hand — it had been inline in the original and was simple enough not to involve AI.

All characterization tests: pass. Full test suite: pass.

---

## Stage 4: Validation

**Characterization test results:** All 10 scenarios passed, including the two surprising behaviors (deprecation warning with both keys present, path resolution relative to config file).

**Behavioral diff check:** The delivery engineer ran the original loader and the refactored loader against a set of 20 real Atlas config files from the test fixtures directory and diffed the outputs. Zero differences.

```bash
# Comparison script
for config in test/fixtures/*.yml; do
  original=$(node -e "const {loadConfigOriginal} = require('./original'); console.log(JSON.stringify(loadConfigOriginal('$config'), null, 2))")
  refactored=$(node -e "const {loadConfig} = require('./dist'); console.log(JSON.stringify(loadConfig('$config'), null, 2))")
  diff <(echo "$original") <(echo "$refactored") && echo "$config: identical" || echo "$config: DIFFERS"
done
```

All 20: identical.

**Code quality check (human judgment):**

- Each extracted function is under 60 lines.
- Each function has a clear input type and output type.
- The deprecation warning logic is now explicitly testable in isolation.
- The path resolution behavior is now documented in the function signature comment, not buried in a 480-line function.

---

## Stage 5: Review and merge

The PR description noted:

- This is a pure refactor. No behavior changes.
- Characterization tests were written before any code was touched.
- All characterization tests pass before and after.
- Behavioral diff against 20 fixture files: identical.
- Two behavioral differences found in AI output during review (Steps 2 and 4) — both corrected before committing.

A reviewer confirmed the characterization test coverage looked complete and checked the two corrected diffs. PR approved.

---

## Evidence captured in AI change log

From the AI change log (format: [`templates/ai-change-log.md`](../templates/ai-change-log.md)):

```text
## Refactor: Atlas config loader extraction

Step 2 — applyDeprecatedKeys()
Tool: Claude Code
Behavioral difference found: AI emitted deprecation warning only when
project_name was present and project was absent. Original emits when
project_name is present regardless. Corrected before commit.

Step 4 — validateConfig()
Tool: Claude Code
Behavioral difference found: error message for timeout out-of-range
had extra word "a number". Corrected to match original exactly.

Steps 1, 3, 5: no behavioral differences found. Committed as-is after review.
```

---

## What this example demonstrates

**Characterization tests are load-bearing.** Without them, the two behavioral differences in Steps 2 and 4 would not have been caught. The code looked correct in both cases. The tests are what made the refactor safe, not the AI's accuracy.

**Small steps compound.** Each step was a single commit, testable independently. If a step had failed, the revert was one commit, not a complete rollback.

**AI proposes, human approves.** Every AI-generated function was reviewed as a diff against the relevant section of the original before it was committed. The review was not a formality — it caught real differences.

**The human wrote the structure.** AI did not decide how to decompose the function. The delivery engineer identified the five concerns, sequenced the steps, wrote the integration function, and wrote the characterization tests. AI wrote the implementation of each extracted function. That division of responsibility reflects what AI is and isn't reliable at.

---

## Artifacts produced

| Artifact | Notes |
|----------|-------|
| Characterization test file | `config/loader.characterization.test.ts` — 10 test cases |
| 5 extracted functions | Each in its own file under `config/` |
| Behavioral diff script | `scripts/compare-loader-output.sh` |
| AI change log entries | Recorded deviations found in Steps 2 and 4 |
| PR with reviewer sign-off | Links to this methodology for context |
