# PythonQt / PyQt GUI Smoke Workflow

Date captured: 2026-09-24

Context: AutoCleanEEG Exclude GUI work for the `Rerun ICA After Epochs` feature. The GUI uses PyQt6, MNE, Matplotlib, and EEGLAB/MNE artifacts. The debugging session exposed several repeatable failure modes that can look like "PythonQt is broken" even when the underlying feature code is mostly correct.

## Observed Issue

The Exclude GUI could launch and the new `Rerun ICA After Epochs` button could be clicked, but repeated smoke attempts failed at different layers:

1. GUI launched but the reprocess subprocess failed immediately because the smoke fixture did not have a schema-valid task config.
2. After config was fixed, report generation failed because the synthetic file had no usable montage/head fiducials.
3. After montage was fixed, ICA failed with `array must not contain infs or NaNs`.
4. After ICA data quality was fixed, the generated smoke task failed because it called a method (`generate_reports`) that the synthetic task did not define.
5. After the generated task completed, copy-back failed because the original task root did not already contain an `ica/` folder.
6. Throughout the process, the environment also emitted a stale workspace warning pointing at a pytest temp workspace. That warning was noisy, but not the blocking error once the main failure was addressed.

The most important lesson: the visible Qt popup was often only the delivery mechanism for a deeper pipeline or fixture issue. Do not assume every popup means a Qt runtime defect.

## Reproduction Steps

From the AutoCleanEEG working tree:

```bash
cd /Users/syex8x/Documents/Github/autocleaneeg_pipeline
QT_QPA_PLATFORM=cocoa make gui-smoke-exclude
```

Then in the GUI:

1. Select `exclude_gui_smoke_epo.set`.
2. Mark a manual bad epoch in the time-series view.
3. Open the `Reprocess` tab.
4. Click `Rerun ICA After Epochs`.
5. Confirm the warning dialog.

Expected final behavior:

- Reprocessing completes.
- Manual bad epoch is dropped before ICA.
- ICA is refit after epoch removal.
- Component classification and ICA application run on the new decomposition.
- Results copy back into the original smoke task folder.
- Original artifacts are backed up under `exports/backups/`.

## Root Causes Found

### 1. Empty Smoke Task Config

The fixture initially wrote a source task with:

```python
config = {}
```

The current pipeline schema requires keys such as `schema_version`, `montage`, `resample_step`, `filtering`, `ICA`, `component_rejection`, and `epoch_settings`. Reprocess failed before it could validate the new feature.

Fix: make the smoke fixture write a minimal schema-valid task config.

### 2. Missing Montage / Fiducials

The synthetic file used standard channel names but did not provide a montage. Report/topomap code failed with a head-coordinate-frame error requiring nasion and left/right preauricular landmarks.

Fix: configure the smoke task with `standard_1020`, matching the fixture channel names.

### 3. Synthetic Data Was Too Degenerate For ICA

The original synthetic signal was pure sinusoids plus drift. After epoch manipulation, ICA could become numerically unstable and fail with:

```text
Failed to fit ICA: array must not contain infs or NaNs
```

Fix: make the fixture more EEG-like while remaining deterministic:

- deterministic RNG seed
- small broadband noise
- slight phase variation
- small shared blink-like source
- longer epochs

### 4. Smoke Task Cleaned All Channels

`clean_bad_channels()` on a four-channel toy fixture marked 100% of channels bad. ICA then tried to fit on unusable data.

Fix: remove `clean_bad_channels()` from the synthetic smoke task. This is not a production behavior change; it keeps the smoke focused on the GUI/reprocess workflow.

### 5. Duplicate ICA Apply In Generated Reprocess Task

For `manual_epochs_before_ica`, the generator rewrote ICA/classification calls but also preserved an old `apply_ica_component_rejection()` call. That could produce duplicate apply calls.

Fix: when using the manual-epochs-before-ICA strategy, drop existing ICA-apply calls and emit exactly one fresh `apply_ica_component_rejection()` after the new classification.

### 6. Nonexistent Fixture Method

The smoke task called `self.generate_reports()`, but the synthetic class did not define that method.

Fix: remove the fake report call from the smoke task. The pipeline still creates the normal run report during processing; the fixture should not pretend to implement a full project-specific report method.

### 7. Missing Destination ICA Folder On Copy-Back

The reprocess output correctly produced:

```text
reprocess/<run>/ica/<stem>-ica.fif
```

But the original smoke root did not already contain:

```text
ica/
```

The copy-back code tried to copy into the missing folder and raised `Errno 2`.

Fix: create the destination parent directory before copying ICA artifacts back.

## Final Fix Pattern

The durable pattern was not to change PyQt dependencies or bypass Qt. The fix was to make the smoke workflow realistic enough for the pipeline while keeping it isolated:

1. Keep Qt/PyQt dependencies unchanged.
2. Launch GUI from a normal macOS Terminal when validating interactive Qt behavior.
3. Use `QT_QPA_PLATFORM=cocoa` for local macOS GUI launch.
4. Keep the smoke fixture deterministic and schema-valid.
5. Avoid running full production cleaning steps that are invalid for a tiny synthetic file.
6. Validate generated reprocess task order:

```text
import_raw
create_regular_epochs
drop_manual_bad_epochs
run_ica(use_epochs=True)
classify_ica_components(reject=False)
apply_ica_component_rejection
```

7. Make copy-back tolerant of missing optional destination folders.

## Useful Checks

Focused unit test:

```bash
PYTHONPATH=src \
MPLCONFIGDIR=/private/tmp/mplconfig-autoclean \
MNE_DONTWRITE_HOME=true \
.venv/bin/python -m pytest \
tests/unit/utils/test_reprocess_overrides.py::test_generate_manual_epochs_before_ica_task_preserves_channels_and_refits_ica \
tests/unit/utils/test_reprocess_overrides.py::test_generate_manual_epochs_before_ica_task_requires_epochs_before_ica \
tests/unit/utils/test_exclude_route.py::test_reprocess_start_and_status \
-q --no-cov
```

Smoke setup:

```bash
MPLCONFIGDIR=/private/tmp/mplconfig-autoclean \
MNE_DONTWRITE_HOME=true \
make gui-smoke-exclude-setup
```

GUI smoke launch:

```bash
QT_QPA_PLATFORM=cocoa make gui-smoke-exclude
```

## Do Not Repeat

- Do not upgrade PyQt/Qt just because a GUI smoke fails. First inspect the pipeline log and generated task.
- Do not treat every macOS popup as a Qt runtime crash. Many were ordinary pipeline exceptions surfaced through Qt dialogs.
- Do not use tiny synthetic EEG data with full production channel cleaning unless the test is explicitly about bad-channel detection.
- Do not validate interactive Qt from a headless Codex subprocess when a normal Terminal launch is available.
- Do not assume a generated reprocess task is correct. Inspect call order and duplicate calls.

## Outcome

The final smoke run completed successfully:

- reprocessed `exclude_gui_smoke`
- copied results to the original task folder
- backed up originals to `exports/backups/`
- confirmed that the `Rerun ICA After Epochs` workflow can complete end-to-end on the local smoke fixture
