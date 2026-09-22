# Handoff — AM-1c: proactive OpenAPI auto-regen hook

**PR:** https://github.com/davidusb-geek/emhass/pull/951
**Branch:** `feat/am-1c-openapi-autoregen-hook` (off `upstream/master`)
**Commit:** `360b932` — `chore: add prek hook to auto-regenerate openapi.json`

---

## Diff summary

Config-only. One block added to `prek.toml`.

```
 prek.toml | 17 +++++++++++++++++
 1 file changed, 17 insertions(+)
```

```diff
@@ -14,3 +14,20 @@ args = ["--fix"]

 [[repos.hooks]]
 id = "ruff-format"
+
+# Local hook: keep the committed OpenAPI spec in sync with its sources.
+# Regenerates src/emhass/static/openapi.json whenever param_definitions.json,
+# the generator, or one of the schema inputs it reads changes. Like ruff-format,
+# this is a fixer: it rewrites the file, and prek's "files were modified by this
+# hook" then fails the commit so the regenerated spec gets staged. Mirrors the CI
+# drift-check (tests/test_openapi.py::test_committed_matches_generated) proactively.
+[[repos]]
+repo = "local"
+
+[[repos.hooks]]
+id = "generate-openapi"
+name = "regenerate openapi.json from param_definitions"
+entry = "python scripts/generate_openapi.py"
+language = "system"
+pass_filenames = false
+files = '^(src/emhass/static/data/param_definitions\.json|scripts/generate_openapi\.py|docs/api/v1/last-run\.schema\.json|docs/api/healthz\.schema\.json)$'
```

No production code touched. No CI auto-commit added. `tests/test_openapi.py::test_committed_matches_generated` (the CI backstop from #921) left untouched.

---

## Teeth-check (actual command outputs)

### 1. Generator is deterministic/idempotent — clean on in-sync master

```
$ python scripts/generate_openapi.py
wrote src\emhass\static\openapi.json (7 paths)

$ git diff --stat -- src/emhass/static/openapi.json
(empty — no diff)
```

### 2. Hook passes clean on in-sync master

```
$ prek run generate-openapi --all-files
regenerate openapi.json from param_definitions...........................Passed

$ git diff --stat -- src/emhass/static/openapi.json
(empty)
```

### 3. Fail-on-edit demo — throwaway default change blocks the commit

Edited `param_definitions.json` `costfun.default_value` `"profit"` → `"cost"`, `git add`, then `prek install` + `git commit`:

```
$ git commit -m "throwaway: should be blocked by generate-openapi hook"
ruff (legacy alias)..................................(no files to check)Skipped
ruff format..........................................(no files to check)Skipped
regenerate openapi.json from param_definitions...........................Failed
- hook id: generate-openapi
- files were modified by this hook

  wrote src\emhass\static\openapi.json (7 paths)
=== exit code: 1 ===
```

The hook regenerated the spec (the diff it produced):

```diff
               "cost",
               "self-consumption"
             ],
-            "default": "profit",
+            "default": "cost",
             "title": "Cost function",
             "description": "Define the type of cost function."
```

Reverting the throwaway → clean again, hook passes:

```
$ git checkout -- src/emhass/static/data/param_definitions.json src/emhass/static/openapi.json
$ prek run generate-openapi --all-files
regenerate openapi.json from param_definitions...........................Passed
```

### 4. No false-positive on unrelated commits

Committing the `prek.toml` change itself (the actual PR commit) — generate-openapi correctly **skipped** because `prek.toml` is not in the trigger regex:

```
$ git commit -m "chore: add prek hook ..."
ruff (legacy alias)..................................(no files to check)Skipped
ruff format..........................................(no files to check)Skipped
regenerate openapi.json from param_definitions.......(no files to check)Skipped
[feat/am-1c-openapi-autoregen-hook 360b932] chore: add prek hook to auto-regenerate openapi.json
 1 file changed, 17 insertions(+)
```

### 5. Existing drift-check test still green

```
$ python -m pytest tests/test_openapi.py -q
.....................                                                    [100%]
21 passed in 3.81s
```

### 6. ruff (config-only change → no py touched)

`ruff check` / `ruff format --check` reported only pre-existing/scratch files
(`tests/test_soc_recovery_prototype.py`, `.tmp/*`) — none introduced by this change.
Note: repo's `[tool.ruff]` has `fix = true`, so bare `ruff check` auto-fixed a
pre-existing tracked file; reverted it so the PR stays config-only. No py files in
this PR.

---

## Session feedback / surprises / follow-ups

- **Task brief named the wrong file.** It said "add a hook to `.pre-commit-config.yaml`,
  same pattern as the existing ruff-format hook." There is **no `.pre-commit-config.yaml`**
  anywhere on upstream/master. The repo uses **`prek`** (the Rust pre-commit reimpl,
  https://prek.j178.dev/) with its **native `prek.toml`** config — that's where the
  ruff-format hook actually lives (per `CONTRIBUTING.md` line 5: "install prek and run
  `prek install`"). Implemented in `prek.toml` (TOML, `repo = "local"`) rather than
  creating a competing YAML file — extending the file that already holds the referenced
  ruff-format hook is the unambiguous intent. Flagged in the PR body.

- **prek/pre-commit not installed in the venv** — installed `prek 0.4.4` via
  `pip install prek` into `.venv` for the teeth-check. `pre-commit` (the Python original)
  could NOT have verified this — it only reads `.pre-commit-config.yaml`, not `prek.toml`.

- **`language: system` + `entry = "python scripts/generate_openapi.py"`** means the hook
  uses whatever `python` is on PATH. That resolves correctly only when the contributor
  runs `prek` inside the activated project venv (the generator imports `emhass.web_server`).
  Same expectation as any system-language hook; consistent with how CONTRIBUTING tells
  contributors to set up. The system `python` (3.14, no emhass) would fail — verified the
  hook against the venv python explicitly.

- **No `-m` needed** — entry is a script path invocation, not a module, so
  `python scripts/generate_openapi.py` is correct (the "pin python -m where relevant" note
  in the brief didn't apply to a path-based entry).

- **Local artifacts left in tree** (`.tmp/`, `data/last_run.json`) are untracked scratch
  from prior sessions — not added, not in scope. `prek install` wrote a local
  `.git/hooks/pre-commit` (local-only, not part of the PR).

- **Follow-up (optional):** consider documenting the new hook in `CONTRIBUTING.md` near the
  existing prek lines so contributors know `openapi.json` is auto-managed. Left out to keep
  this PR tight plumbing; raise only if a reviewer asks.
