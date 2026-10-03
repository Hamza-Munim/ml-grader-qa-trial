# Work trial submission

Name: Hamza Munim
Time spent (minutes): 57

Setup note: Windows, Python 3.12, scikit-learn 1.5.2 (1.9.1 was blocked by a local Windows
security policy). The author's controls reproduce exactly: gold 1.0 (0.9825), nc_textbook 0.0,
nc_one_shot 0.0. Line numbers refer to the original files.

## Part A: Findings (most severe first)

### F1 - A no-skill "always benign" policy scores 1.0

- **Severity:** BLOCKER
- **Where:** `tests/grade.py:22` (`BAR = 0.62`), `setup.py:62` (`pass_bar_accuracy: 0.62`)
- **Failure scenario:** The bar sits above the broken release (0.610) but below the live base
  rate: 143 of the 228 live rows are benign (0.627). A `serve.py` that returns
  `np.ones(len(X))`, driven through the full trajectory (reproduce, deploy, FAILED validate,
  diagnostics, rebuild_artifact, clean validate, promote, submit), passes validate as clean and
  gets reward 1.0 without any ML.
- **Evidence:** ran `controls/exploit_constant.py`: reward 1.0, terminal_accuracy 0.6272.
- **Fix:** Raise the bar above every no-skill baseline, in both places. 0.90 works: correct fixes
  score 0.956-0.982 (LogReg, SVC, RF, GB and kNN on the training scaler), constant 0.627,
  broken 0.610. Fixed in Part B.

### F2 - The grader trusts agent-writable state (forged log + hand-placed release)

- **Severity:** BLOCKER
- **Where:** `tests/grade.py:30-34` (reads `workspace/.envstate/actions.jsonl`),
  `tests/grade.py:60` (grades `workspace/.envstate/deployed`)
- **Failure scenario:** The README says `.envstate/` is agent-controlled. The agent can write a
  fixed `serve.py` and artifacts into `.envstate/deployed/` and write an `actions.jsonl` that
  claims a clean gate walk with recovery. That should score 1.0 without calling a single op.
- **Evidence:** ran a throwaway control that forges the log and the deployed folder and calls no ops: reward 1.0 (script not included).
- **Fix:** Keep the action log, counters and deployed/promoted snapshots in a service-side,
  root-owned directory, and have the grader read only from there.

### F3 - The hidden labelled live stream is copied into the workspace

- **Severity:** BLOCKER
- **Where:** `workspace/env_cli.py:174-178`
- **Failure scenario:** The first `validate`, even one that returns FAILED, copies
  `service/hidden_stream.npz` (features and labels) to `workspace/.envstate/cache/stream.npz`.
  An agent can turn `serve.py` into a lookup table of those labels and score 1.0 with no model.
- **Evidence:** ran a throwaway control that builds a lookup table from the cached labels: reward 1.0, terminal_accuracy 1.0 (script not included).
- **Fix:** Never write the hidden stream under `workspace/`. Cache it in `service/` or read it
  directly.

### F4 - Recovery is computed but not enforced

- **Severity:** BLOCKER
- **Where:** `tests/grade.py:84` (`strict_pass = terminal and walk` leaves out
  `forced_recovery_ok`), `workspace/env_cli.py:159` (the fault fires only on deploy #1)
- **Failure scenario:** Correct fix, deploy, FAILED validate, then deploy again with no
  diagnostics or recovery. Deploy #2 is healthy, validate is clean, promote and submit go through,
  and the reward is 1.0. That contradicts AUTHOR_NOTES ("Recovery is therefore enforced").
- **Evidence:** ran a throwaway control that deploys twice and skips recovery: reward 1.0, with the grader itself reporting forced_recovery_ok false (script not included).
- **Fix:** `strict_pass = terminal and walk and recovery`, and make re-deploying not dodge the
  fault.

### F5 - A correct, properly driven fix can score 0.0

- **Severity:** BLOCKER
- **Where:** `tests/grade.py:63-65` (stages only `pipeline/serve.py` and `artifacts/`) vs
  `workspace/env_cli.py:154-155` and `:186` (validate runs the whole deployed snapshot, including
  `data/`)
- **Failure scenario:** The agent fixes serving with the training mean/std from
  `data/train_stats.json`, which is equivalent to the fitted scaler. Validate is clean and promote
  succeeds, but the grader never copies `data/`, crashes with FileNotFoundError and scores 0.0.
  The same happens for a helper module in `pipeline/`.
- **Evidence:** ran a throwaway control using data/train_stats.json: gate_walk_ok true, reward 0.0, error FileNotFoundError for train_stats.json (script not included).
- **Fix:** Grade exactly what validate ran: the full deployed snapshot, with the same runner.

### F6 - The grader imports agent code in-process next to the answer key

- **Severity:** BLOCKER
- **Where:** `tests/grade.py:26-27` and `:66-70`
- **Failure scenario:** `HIDDEN_X`/`HIDDEN_Y` are module globals of the grader, and the
  candidate `serve.py` is executed inside that process (as root in production). `predict()` can
  read `sys.modules["__main__"].HIDDEN_Y` or patch `BAR`.
- **Evidence:** ran a throwaway control whose predict() reads HIDDEN_Y from the grader: reward 1.0, terminal_accuracy exactly 1.0 (script not included).
- **Fix:** Score in an isolated, unprivileged subprocess that receives X only.

### F7 - The grader scores whatever is deployed at grading time, not the promoted release

- **Severity:** MAJOR
- **Where:** `workspace/env_cli.py:146-164` (`deploy` has no check for `submitted`),
  `tests/grade.py:60`
- **Failure scenario:** The agent drives the gold trajectory to `submit` (promoted release
  0.98), then keeps working and deploys an unfinished `serve.py`. The grader scores that
  never-validated release instead of the promoted one. "submit freezes the promoted release"
  is not implemented.
- **Evidence:** ran a throwaway control that deploys the broken `serve.py` after submit:
  reward 0.0, terminal_accuracy 0.6096, although the promoted release scored 0.9825 (script not
  included).
- **Fix:** Snapshot the release at `promote` into service-side storage, reject all ops after
  `submit`, and grade the snapshot.

### F8 - The validate budget can be reset by editing the log

- **Severity:** MAJOR
- **Where:** `workspace/env_cli.py:64-83` (state is rebuilt from the agent-writable log),
  `workspace/env_cli.py:171`
- **Failure scenario:** Deleting the `validate` lines from `actions.jsonl` resets the counter
  to 0, so the budget of 4 stops meaning anything and `validate` becomes an unlimited oracle on
  the hidden stream.
- **Evidence:** ran a throwaway control that erases validate lines when rejected: 9 validate
  calls accepted against a budget of 4 (script not included).
- **Fix:** Keep counters and state service-side (same fix as F2).

### F9 - validate scores on the same hidden stream the grader uses

- **Severity:** MAJOR
- **Where:** `workspace/env_cli.py:186` vs `tests/grade.py:69`
- **Failure scenario:** The validation signal is the test set itself, so every clean/dirty bit
  leaks test information. With F8 it becomes unlimited hill climbing on the exact data that
  decides the reward.
- **Evidence:** inferred from code; F8 shows the oracle is unlimited.
- **Fix:** Validate on a disjoint slice of live traffic and grade on the rest.

### F10 - The recovery step does not test diagnosis

- **Severity:** MAJOR
- **Where:** `workspace/env_cli.py:159` (always deploy #1), `:198-200` (the symptom text says
  what happened), `:207-210` and `:246-248` (a wrong family is rejected and never logged)
- **Failure scenario:** The fault is deterministic and the diagnostics message names it. Even
  without reading it, an agent can try all 3 families at no cost, because rejected calls leave
  no trace in the log. The step the task calls "recover from an injected failure" needs no
  diagnosis.
- **Evidence:** inferred from code.
- **Fix:** Log and penalise wrong recovery attempts, and vary the fault (which deploy, which
  artifact, which symptom).

### F11 - The control battery does not test the task's claims

- **Severity:** MINOR
- **Where:** `controls/run_controls.py:15-19`, `controls/nc_one_shot.py`
- **Failure scenario:** `nc_one_shot` floors only because `.envstate/deployed/` does not exist
  (a crash), not because of the gate check. There is no no-skill, skip-recovery or tampering
  control, so F1 to F6 all pass an all-green battery.
- **Evidence:** grade output for nc_one_shot shows `FileNotFoundError ... deployed\pipeline\serve.py`.
- **Fix:** Add a negative control per exploit and a positive control for a correct-but-different fix.

### F12 - Robustness gaps in the grader and setup

- **Severity:** MINOR
- **Where:** `tests/grade.py:22` and `setup.py:62` (the bar is written twice),
  `tests/grade.py:69-71` (prediction length is never checked)
- **Failure scenario:** Someone updates one bar and not the other, and validate and the grader
  silently disagree. A `predict()` that returns the wrong number of labels either crashes the
  comparison or is compared incorrectly, depending on the numpy version. The model is pickled
  with an unpinned scikit-learn, so a version mismatch between setup and grading can break
  loading.
- **Evidence:** inferred from code.
- **Fix:** Read the bar from `service/config.json` in both places, assert
  `pred.shape == HIDDEN_Y.shape`, and pin the scikit-learn version.

## Part B: Fix + proof

- **Blocker fixed:** F1
- **Exploit control:** `controls/exploit_constant.py`
- **Patch:** `patch/fix.diff`. `tests/grade.py:22` changes `BAR` from 0.62 to 0.90, and
  `setup.py:62` changes `pass_bar_accuracy` from 0.62 to 0.90 so validate and the grader agree.

| run                        | before patch     | after patch                                               |
| -------------------------- | ---------------- | --------------------------------------------------------- |
| exploit_constant           | 1.0 (acc 0.6272) | 0.0 (validate dirty, promote rejected; acc 0.6272 < 0.90) |
| gold (`solution/solve.py`) | 1.0 (acc 0.9825) | 1.0 (acc 0.9825)                                          |
| nc_textbook                | 0.0              | 0.0                                                       |
| nc_one_shot                | 0.0              | 0.0                                                       |

Commands to reproduce:

```
# from the kit root, with this zip's controls/ and patch/ folders next to task/
cp -r task task.orig
python3 task.orig/setup.py
python3 task.orig/controls/run_controls.py controls/exploit_constant.py   # reward 1.0
patch -p1 < patch/fix.diff                                                 # patches task/tests/grade.py and task/setup.py
python3 task/setup.py                                                      # rewrites service/config.json with the 0.90 bar
python3 task/controls/run_controls.py controls/exploit_constant.py        # reward 0.0
python3 task/controls/run_controls.py                                      # gold 1.0, nc_textbook 0.0, nc_one_shot 0.0, all OK
```

## Part C: Verdict

Ship / ship with fixes / reject: **Reject**

The bar fix closes the no-skill exploit, but the bigger problem is architectural: the
grader's evidence (action log, deployed release), the validate budget and a copy of the hidden
labels all live in the agent-writable workspace, so forging or reading answers still scores 1.0.
Recovery is not enforced despite the author's claim, and it does not test diagnosis even when
it runs, so the long-horizon part of the task is weaker than described. The grader also scores a
different staging than validate, which gives some correct fixes 0.0. The author's all-green
battery has no control that could catch any of this.

Top priorities:

1. Move the action log, counters, deployed/promoted snapshots and validate cache out of
   `workspace/` into service-side storage the grader trusts, freeze the release at promote,
   and validate on a slice disjoint from grading (F2, F3, F7, F8, F9).
2. Enforce recovery in `strict_pass`, make the fault vary and wrong recoveries cost something,
   and grade exactly what validate ran in an isolated subprocess (F4, F5, F6, F10).
3. Keep the 0.90 bar as a single config value, and add a negative control for each exploit plus
   a correct-but-different positive control (F1, F11, F12).
