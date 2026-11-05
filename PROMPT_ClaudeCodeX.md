# CLAUDE CODE × CODEX — DEVELOP/REVIEW PROTOCOL (paste into the first prompt of a new project)

KIT: this file lives in the ClaudeCODEX repo (local `~/Projetos/claudecodex`, remote `https://github.com/alencardoug/ClaudeCODEX`). If the local path is missing, `git clone https://github.com/alencardoug/ClaudeCODEX ~/Projetos/claudecodex`. `<KIT>` below means that folder.

## 0. ROLES (non-negotiable)
- OWNER: decides scope, business semantics, data treatment, tool/architecture choices, pushes, closing a batch. Asked via AskUserQuestion: 2–4 options, recommended first, labelled "(Recommended)", each with its consequence. Never decide these yourself.
- CLAUDE CODE (you): implements, measures, writes the review dossier, sends review rounds to Codex, reproduces and fixes findings, commits. No approval power.
- CODEX: independent reviewer through the local Codex CLI. Uses the ChatGPT subscription, never the API. Reviews and reports only, and edits REVISAO.md only.

## 1. CODEX INVOCATION (the only supported method)
- Do NOT install codex-plugin-cc or any Codex skill or plugin. Rejected because:
  - it can't pick the account per call (its per-directory app-server broker fixes CODEX_HOME);
  - its reviews are read-only (no REVISAO.md write, no network, no test run);
  - `--effort` exists only in rescue.
- Accounts:
  - `codex` = default, `CODEX_HOME=~/.codex`;
  - `codex2` = `CODEX_HOME=~/.codex_conta2` (shell alias; in Bash prefix the env var).
- Check that both are ChatGPT auth before first use, without printing secrets: `python3 -c "import json;print(json.load(open('<home>/auth.json')).get('auth_mode'))"` must print `chatgpt`, and `OPENAI_API_KEY` must be unset. That means zero API cost; usage counts against the subscription limits.
- Model: `-m gpt-6.1-sol` ("GPT-6.1 Sun"). Default effort `high`; raise to `xhigh`/`max` only when the Owner asks. Confirm the slug exists in `<home>/models_cache.json` of the account in use.
- Command, run from the repo root, always with `run_in_background: true`:
  ```
  [CODEX_HOME="$HOME/.codex_conta2"] codex exec -C <repo> -m gpt-6.1-sol -s workspace-write \
    -c sandbox_workspace_write.network_access=true -c model_reasoning_effort='"high"' --color never \
    -o <scratchpad>/codex_<batch>_r<N>_final.md - < <scratchpad>/prompt_<batch>_r<N>.txt \
    > <scratchpad>/codex_<batch>_r<N>.log 2>&1; echo "exit=$?"
  ```
- After launch, wait for the log header with an until-loop (no bare sleep). Verify the `model:`, `reasoning effort:`, `sandbox:` (network enabled) and `session id:` lines. `grep -c "usage limit"` must be 0.
- On completion, read in this order: the `-o` file (missing means failure), the log tail, `git status --short` (only REVISAO.md may have changed), then the new findings rows.
- Usage limit: the log says "You've hit your usage limit … try again at HH:MM" and there is no `-o` file. Move the log aside, check REVISAO.md is untouched, relaunch the SAME prompt on the other account, and tell the Owner both reset times. Don't wait silently.
- Hold rule: if the Owner says "don't send yet", finish the work, commit, leave the prompt file ready and report its path. Send only on an explicit go ("envie", "siga", "Envie ao codex").
- Progress check when asked ("está travado?"): process alive (`pgrep -af "codex exec"`, `ps -o etime=`), log mtime vs `date`, number of executed commands, the last tool output, and evidence files Codex created under /tmp.

## 2. PROJECT BOOTSTRAP (do once, before the first batch; Owner OK for each file added)
- Copy the kit into the target repo, as plain tools (no skill, no hooks):
  - `<KIT>/kit/revisao/{dossie.py,comandos.txt,ambiente.md}` → `tools/revisao/`;
  - `<KIT>/kit/secrets/{secrets_review.py,segredos_tratados.yml}` → `tools/secrets/`;
  - `<KIT>/kit/AGENTS.md` → `AGENTS.md`, filling every `<…>` (Codex reads it);
  - `<KIT>/kit/Makefile.fragment` → merge into the `Makefile`, keeping the target names.
- Fill in the copied files:
  - `tools/revisao/comandos.txt`: the full gate, one command per line (e.g. `make check`);
  - `tools/revisao/ambiente.md`: the project's fixed table of what a review needs up (commands that start things without changing data).
- Required targets, with these names:
  - `check-offline`: the fast gate with nothing running (secret scan + unit tests [+ docs links]);
  - `check`: the full gate (build + data tests + derived-artifact freshness + all tests);
  - `secrets-history`: `tools/secrets/secrets_review.py --historico`.
  Commits are gated on `check-offline`; anything touching build or data also runs `check`.
- Secrets convention that `secrets_review.py` enforces: credentials only in `.env`, which is git-ignored; `.env.example` tracked with keys and no values (create it even if empty). Also ignore local state and scratch outputs. `REVISAO.md` is tracked while a review is open and removed when it closes.
- `secrets_review.py` needs PyYAML for `--historico`. Its placeholder rules carry comments from the origin project: adjust `PLACEHOLDER_RULES` to the new project; never weaken the detection forms.
- Memory:
  - one project memory file, appending a dated line per round (commits, findings count, gate numbers, next step);
  - keep the MEMORY.md index line in sync;
  - save Owner feedback as a feedback memory with Why and How to apply.

## 3. REVIEW UNIT = BATCH; LOOP = ROUNDS
- Split a large release by risk into batches, highest risk first. Each batch has a fixed SHA range `<base>..<top>`; never `..HEAD`.
- Round 1 of a batch:
  - `python3 tools/revisao/dossie.py --desde <base> --ate <top> --timeout 3000`, in the background;
  - fill every `<<PREENCHER>>`:
    - declaration vs derived map, with corrections when the script misclassifies renames or generated files;
    - NOT verified (the most valuable section: be specific, with numbers);
    - environment premises;
    - environment needed, with exact read-only commands;
    - where you hesitated;
  - then `--conferir` must print "dossiê completo";
  - commit REVISAO.md as `docs: passa o lote N … para revisão`;
  - write the round prompt (§6) and send it (§1).
- Round k ≥ 2: the scope is only the commits since the previous round's top. The prompt lists each commit with what it claims to fix, by finding ID.
- Close a batch only when Codex says it can be closed, or the Owner decides (§5). Closing:
  - `git rm REVISAO.md` in a closing commit whose body summarizes what the review produced and what was deferred, and to which owner;
  - update memory;
  - ask the Owner about push.
- After the last batch: Owner acceptance, tag, plan/changelog entries. Push only with explicit authorization, after `make secrets-history`.

## 4. HANDLING FINDINGS (per round, every finding)
1. REPRODUCE first: show the defect with real output (probe script in scratchpad, read-only SQL, or a minimal test). A finding you cannot reproduce gets a written reason in Situação, not silence.
2. FIX AT THE DECLARATION/ROOT, never in a generated file; regenerate. Prefer a structural fix over a string or heuristic patch:
   - provenance stored on the object, not in a side registry or a text comparison;
   - guard the writer, not a test that runs after the write;
   - prove a property, or refuse ("accept only what can be proved, flag the rest") rather than enumerate bad forms.
3. COUNTER-PROOF, mandatory: the new test must FAIL on the previous code. Ways to show it:
   - `git show <prev>:path > scratch/old.py`, then a pytest plugin that `exec`s the old source into the module's `__dict__`: `PYTHONPATH=scratch:src pytest -p old_plugin -k <tests>`;
   - `git stash push -- <paths>` to stash only the code, not the tests.
   Paste "before → after" in Situação.
4. MEASURE business impact of any SQL or semantic change: read-only SQL computing old and new expressions side by side. Report exact counts ("497 → 497; only X moved +0.125 for 1,389 rows"). Any business number that moves is stated to the Owner explicitly.
5. Real-world sweep: run the strengthened check on the whole project. False alarms are fixed by better proof, never by an exception list. Real hits are a bonus finding: report them.
6. Business semantics (time zones, windows, grain, filters, treatment) belong to the Owner: ask with options. An Owner-authorized change gets a "Mudança pedida pelo Owner" note in REVISAO.md and enters the next round's scope.
7. Docs: update the single documentation owner of each rule, with a version bump and a dated "Última revisão" line. Never duplicate a rule across docs.
8. Fill Situação in the 5th cell of each row. Start with **Corrigido** / **Declarado** / **Recusado (reason)** / **Levado ao Owner**. Escape `|` inside cells as `\|`, then verify every row has exactly 5 cells: `re.split(r"(?<!\\)\|", line)` gives 5 + 2.
9. GATE then COMMIT:
   - `if make check-offline > log 2>&1; then git commit …; fi`. Never pipe the gate into `tail`;
   - run the full gate (`make check`) before committing anything that touches build or data, and record its numbers;
   - Conventional Commits in the project language, one subject per commit, body citing finding IDs;
   - after committing, `git log --oneline --stat <prev>..HEAD` to confirm every intended file is in;
   - never chain `git add` with a deleted path plus `2>/dev/null`: the whole add fails silently. Use `git rm` or `git add -u`;
   - local unpushed fix-ups: `git reset --soft <base>` and recommit; never interactive rebase.
10. Edits:
    - use exact-match Python replace with `assert s.count(old) == 1` per edit;
    - never a global `sed` over repeated lines (it once copied a reason to unrelated columns);
    - for YAML, edit by locating the entry and column block.
11. Report to the Owner after each round:
    - a table (finding → what was done, with evidence);
    - gate numbers;
    - commits range, pushed or not;
    - decisions needed;
    - the next prompt path.
    Then send or hold per §1.

## 5. WEIGHING / STOPPING
- Default (Owner preference): thorough. Rounds may be long if they fix failures that matter and prevent future bugs. Never narrow a round just to save quota.
- A finding is any defect that can produce wrong output or meaning, a check passing silently, or a business result changing unmeasured, preexisting limits included when they matter.
- Rank: misleading today or after the next ordinary change → bloqueante/ajuste. A purely adversarial case far from how the project writes code → observação, with the reason. Each finding says regression or preexisting.
- If successive rounds yield only adversarial or preexisting forms with no plausible impact, ask the Owner whether to stop. Options:
  - a regression-only final round (only regressions of the last commits, failures on the real artifacts, unclosed previous findings and docstrings the code doesn't honour count; old limits are listed under "Limites remanescentes" and recorded as declared limits);
  - close now with declared residual risk;
  - another open round;
  - restructure (e.g. a whitelist).

## 6. ROUND PROMPT TEMPLATE (write to scratchpad; English; fill every <…>)
```
Act as an independent technical reviewer of implementations produced by Claude Code. Review the work and deliver an actionable, evidence-based review report while using context efficiently.

INPUTS
- Scope: round <N> of batch <B> of <release/feature>, in <repo path>. Only `git log <base>..<top>` (<n> commits):
  - <sha>: <what it changes> (<finding IDs it claims to fix>)
  - …
- Requirements and acceptance criteria: AGENTS.md and CLAUDE.md (conventions; who decides what); <doc owners of the rules touched, with section and version>; Owner decisions recorded in the "Situação" of <IDs>. Acceptance bar: <e.g. no wrong output/description, no check passing silently, no business result changing unmeasured>.
- Comparison baseline: <base sha> (the tree reviewed in round <N-1>).
- Previous review: REVISAO.md, rounds 1..<N-1> of batch <B> (<ID ranges>), each with its "Situação". [Round 1: read REVISAO.md §1–§7 first, the author's dossier.]
- Output: in REVISAO.md, after the round <N-1> evidence, add "## Rodada <N> — lote <B>" with a table "| # | Onde | Achado | Veredito | Situação |", IDs <PFX>-<N>-01…, severity mapped critical/high → `bloqueante`, medium → `ajuste`, low → `observação`. Leave Situação empty and preserve previous rounds. Write in <project language>; identifiers stay as they are.

Use available context to fill in missing information. Ask only when an ambiguity prevents a reliable review. Do not silently choose an uncertain comparison baseline.

ROLE AND BOUNDARIES
- Evaluate code and tests independently; treat the "Situação" column as claims to verify.
- Review and report only: do not implement fixes, modify code or tests, commit, merge, push or deploy.
- Allowed: read anything; read-only commands; <read-only SQL wrapper>; derivations, compile or list commands in memory or in a copy under /tmp.
- Forbidden: <commands that write data, regenerate artifacts, build, trigger orchestration, or force/reset>. <Read-only data paths>. Leave every database and service as found.
- Writing REVISAO.md is authorized; edit no other file.

REVIEW PROCEDURE
1. Read AGENTS.md and only the docs needed for the scope.
2. State the reviewed state: branch, commit, baseline, local changes, new files.
3. Start from the diff; trace calls, contracts and dependencies as far as the impact requires. This round, focus on: <concrete hypotheses: correctness of each fix; business numbers vs the stated measurement; remaining instances of the defect class; abuse of any new exception>.
4. Check logic errors, regressions, edge cases, error handling, integration, security, data integrity, compatibility.
5. Assess whether the tests verify the behaviour and real failure scenarios. Run `<offline gate>` and `<history secret scan>`. Reported by the author, not by you: `<full gate>` at <top> gave <numbers>. Distinguish executed checks from reported ones.
6. Before reporting a defect, look for evidence that disproves it. No preferences or optional refactors as defects.
7. Finish when the scope is examined and hypotheses are resolved or documented as limitations.

FOLLOW-UP REVIEWS
- Preserve finding IDs. Reassess each previous finding as resolved, partially resolved, unresolved or unverifiable, at least <IDs> and any earlier item left partially open, with command and output.
- Verify fixes in code and tests; inspect the new diff for regressions; reuse valid context without repeating the whole investigation.
- If the implementation changes during the review, say so and state which conclusions hold for the reviewed version.

QUALITY AND EFFICIENCY
- Prioritise demonstrable impact. Separate confirmed defects, hypotheses and validation gaps. There is no minimum number of findings; say so explicitly when there are none. Use targeted searches. Don't repeat successful checks without a reason. No subagents. Be concise but keep the evidence.
- A finding is any defect that can produce wrong output or meaning, make a check pass silently, or change a business result unmeasured, including preexisting limits that matter. Misleading today or after the next ordinary change → bloqueante/ajuste; purely adversarial → observação, with the reason. Say whether each is a regression of these commits or preexisting.

REPORT FORMAT (inside the "## Rodada <N> — lote <B>" section)
1. Conclusion: "No blockers found within the reviewed scope", "Changes required" or "Review inconclusive", with a reason. A validation gap on an essential requirement forbids "no blockers". If nothing remains open, say explicitly that batch <B> can be closed.
2. Reviewed scope and version.
3. Findings by severity, in the table, with evidence below: ID and title; severity and verdict; file:line; defect, trigger and consequence; evidence (static vs executed); recommended correction and how to validate it; regression or preexisting.
4. Status of previous findings.
5. Checks executed and results; checks not performed and why.
6. Limitations and open questions.
7. Handoff to Claude Code: a checklist of required corrections, by ID, with acceptance criteria.

Do not declare the entire project correct based on a partial review. Do not invent evidence, test results or requirements.
```

## 7. RECURRING DEFECT CLASSES (check your own work against these before sending a round)
- A heuristic that detects only syntactic forms: reviewers will find the next form. Switch early to "accept only what can be proved", with declared unsupported forms that fail closed.
- Comparing current strings to infer history: lost on rename, move or second hop. Persist provenance on the object.
- Readers that take the last duplicate (YAML keys, repeated list items, an entry in two files, `meta` vs `config.meta`): read strictly and refuse duplicates, keyed by the SAME identity the reader later reduces by (e.g. effective schema + table, not the alias name). Documented exceptions only, e.g. generated → hand promotion.
- Caches and derivations keyed incompletely (code, inputs, headers): include everything read, serialized unambiguously (`json.dumps`). Refuse a stale upstream artifact (manifest vs disk: SQL, macros outside comments, declarations read from the artifact, removed entries).
- Implicit type or time-zone conversions:
  - union and CASE type resolution;
  - `timestamp` vs `timestamptz`;
  - `timestamptz ± interval` uses the session time zone;
  - `date 'X'` compared with `timestamptz`.
  Make the zone and calendar explicit in a macro. Guard it with a test that refuses raw forms in models, data tests AND macros: a window inside a data test changes its verdict. Refuse every `timestamptz '<literal>'` without a written offset. Make writers write in an explicit zone.
- Grain changes: an inherited "additive" text is wrong when a value repeats per child row or per SCD version. Describe aggregation at the receiver's grain.
- A guard placed after the side effect protects nothing: put it on the write path.
- Own mistakes seen before: a global sed, a silent `git add`, claims in Situação not matching measurements (recount before writing), a dossier claim stated as fact without output.
