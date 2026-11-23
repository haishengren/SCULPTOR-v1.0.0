# Reproducing a SCULPTOR run

After the three release assets have been extracted into the package root, this
package is self-contained: it holds the runtime code, every trained model
and the data a production-style run needs. No file outside this folder is
required, and nothing inside refers to the machine it was built on.

## 0. Quick start (double-click)

`start_sculptor.bat` builds the environment on first use and then opens the
control panel, so no terminal is needed:

```text
start_sculptor.bat            control panel (native window)
start_sculptor.bat web        control panel in the browser
start_sculptor.bat test       run the unit tests
start_sculptor.bat cli --help command line
```

The launcher also checks that the environment is complete before it starts the
panel, so a half-finished setup is reported immediately together with the
command that fixes it.  The manual path below is only needed if you prefer to
control the environment yourself.

## 1. Environment

The package was built and validated with **CPython 3.11.9 (64-bit)**, and the
setup script reproduces that interpreter rather than reusing whatever Python
happens to be installed:

1. look for CPython 3.11.9 (an earlier run of this script, `py -3.11`, or
   `python` on PATH - the reported version is checked, not assumed);
2. if it is not there, **download the official 3.11.9 Windows build** and
   unpack it into `runtime\python311` inside this folder - no installer, no
   administrator rights, no registry entries and no PATH changes, so removing
   the folder removes the interpreter completely;
3. build `.venv` from that interpreter.

The choice is recorded in `runtime\PYTHON_VERSION.txt` (version, path, source
and the SHA256 of the downloaded package), so the interpreter used for a
reproduction can be audited afterwards.

### Interrupted downloads are safe

Every step can be repeated.  A half-transferred Python package is written to a
temporary name, checked before it is accepted and deleted if it is incomplete,
so the next run downloads it again instead of failing on a corrupt file.  A
`.venv` that exists but is incomplete (the usual result of losing the network
during `pip install`) is detected by importing the dependencies - including the
control-panel packages `webview`, `proxy_tools` and `bottle` - and the
installation is completed in place.  If the package was moved and the recorded
interpreter path no longer exists, the `.venv` is rebuilt automatically.

`start_sculptor.bat` performs the same completeness check before opening the
panel and runs the repair itself, so pressing it again after a failed setup is
the correct action - nothing has to be deleted by hand.

### Core and panel packages are treated differently

The pipeline and the panel need different things, and the setup script keeps
them apart:

* **Core** - numpy, pandas, scipy, scikit-learn, joblib, rdkit, torch,
  torch-geometric, e3nn, xgboost, h5py, openpyxl, psutil.  These must import or
  the setup stops, because nothing runs without them.
* **Native panel window** - pywebview, proxy_tools, bottle.  The browser mode
  (`start_sculptor.bat web`) is served by the Python standard library and needs
  none of them, so if a machine cannot install these three the setup only warns
  and `start_sculptor.bat` opens the panel in the browser instead of refusing to
  start.

When something is still missing, the script installs the missing packages by
name in a separate pass (so one bad package cannot block the rest), prints
whether each one is absent or installed-but-failing, and writes the complete
pip output to `runtime\pip_install.log`.

`mordred` is no longer installed: nothing in the package imports it.  It was
left over from the descriptor exploration that produced the SFS-VR feature
table; the table itself is shipped as data.

### One package is shipped inside this release

`tools\control_panel\vendor\proxy_tools\` is the `proxy_tools` 0.1.0 module
(BSD, extracted from Werkzeug).  It is vendored because at least one mirror
serves a **broken wheel** for it: the archive contains only
`proxy_tools-0.1.0.dist-info\` and no module file, and its `RECORD` lists no
module either.  `pip install` therefore reports success, `import webview`
fails with `ModuleNotFoundError: No module named 'proxy_tools'`, and
re-running pip does nothing because the metadata says the requirement is
already satisfied.

`setup_env.ps1` imports `proxy_tools`; when that fails it copies the vendored
directory into `.venv\Lib\site-packages` and verifies the import again, so the
native panel works even on a machine whose mirror cannot deliver the package.

### The browser mode and "Failed to fetch"

`start_sculptor.bat web` starts the panel server in the console window and then
opens <http://127.0.0.1:8765/>.  The page is served by that server, so the
window has to stay open: every button in the panel calls the same server, and
once it stops they report `TypeError: Failed to fetch`.  The launcher now prints
that warning, waits for the port before opening the browser, and reports a
clear message when port 8765 is already taken by another instance.

Opening `static\index.html` directly from the file system does not work for the
same reason - use the launcher, or the URL above.

rdkit, torch, torch-geometric and e3nn only publish wheels for a limited set of
interpreter versions, which is why the version is pinned instead of inherited.

One click builds the environment inside this folder:

```powershell
powershell -ExecutionPolicy Bypass -File .\setup_env.ps1          # CPU build of torch
powershell -ExecutionPolicy Bypass -File .\setup_env.ps1 -Cuda    # CUDA build of torch
powershell -ExecutionPolicy Bypass -File .\setup_env.ps1 -Check   # report only, install nothing
powershell -ExecutionPolicy Bypass -File .\setup_env.ps1 -Recreate  # delete .venv and rebuild
```

The script prints the interpreter it selected (version, 64-bit, full path and
where it came from) before anything is installed, checks with a dry run that
every dependency has a wheel for that interpreter, then creates `.venv` next to
itself, installs PyTorch from the matching index and installs everything listed
in `requirements.txt`.  Every step is checked: the script exits with an error
instead of reporting success after a failed install.

If a package cannot be found on the configured index - a partial mirror is the
usual reason on a corporate or campus network - the install is retried against
the official PyPI index automatically.

### If the setup stops

* `Could not download Python 3.11.9` - the machine cannot reach nuget.org.
  Download <https://www.nuget.org/packages/python/3.11.9> by hand, put the file
  at `runtime\_cache\python-3.11.9.nupkg` and rerun the script.
* `At least one dependency has no build for this interpreter` - only possible
  when an interpreter was forced with `-Python`.  Rerun without `-Python` so
  that 3.11.9 is provisioned and used.
* `The existing .venv was built from a different interpreter` - the package was
  moved or the interpreter changed.  Rerun with `-Recreate`.
* Anything else - the failure message lists the interpreter, the pip index and
  the commands that reproduce the problem.  Send that block back.

The manual equivalent is:

```powershell
cd <this folder>
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

`requirements.lock.txt` records the exact versions used for the paper run; the
test suite additionally needs `requirements-dev.txt`:

PyTorch is installed separately by `setup_env.ps1` so that the same lock works
on CPU-only and CUDA machines: the default is the CPU build, and `-Cuda`
selects the CUDA build.

```powershell
.\.venv\Scripts\python.exe -m pip install -r requirements-dev.txt
.\.venv\Scripts\python.exe -m pytest tests -q      # 74 tests
```

The panel and the CLI both work from any working directory; every path is
derived from the location of this file.

## 2. Compute device

| mode | behaviour |
|------|-----------|
| `cpu` | the whole workflow runs on the CPU; CUDA is never probed |
| `gpu` | the property heads use the GPU; the run stops with an error if CUDA cannot be used |
| `auto` (default) | probe CUDA with a short smoke test, then take the GPU path if it passes, otherwise the CPU path |

Only the property heads can use a GPU. Generation, screening, perturbation and
the analysis steps are CPU-only, so a `cpu` run and an `auto` run on a machine
without CUDA follow the same code path.

`auto` is resolved **once**: the first lookup in a process runs the smoke test
and caches the answer, the resolved device is exported as
`SCULPTOR_RESOLVED_DEVICE` for every child process (including the worker pools),
and the panel resolves it once per job before launching its commands. A run
therefore prints a single `[device] ...` line per process instead of repeating
the probe during property prediction. Set `SCULPTOR_DEVICE_NO_CACHE=1` only if
you want the old per-lookup probing back for comparison.

## 3. Control panel

```powershell
.\.venv\Scripts\python.exe tools\control_panel\server.py
```

Open <http://127.0.0.1:8765>. The *Compute device* selector at the top of the
form maps to the three modes above; the panel passes the choice to every
command it launches.

## 4. Trial runs shipped with this package

`result\trial_generation\` and `result\trial_env\` are two complete generation +
perturbation runs made through the control panel with the CPU device mode (75
generation requests on C10, then 15 perturbation requests). They show the
expected output layout:

```
result\trial_generation\
  C10\ ...                     per-carbon generation tables
  perturbation_seeds_T123.csv  Tier 1-3 seeds handed to the perturbation line
  analysis\generation\C10\     organised generation tables
  analysis\perturbation\       organised perturbation tables
  analysis\totals\             flow_totals.csv + final_data_report.txt
```

Their ledgers (`analysis\totals\flow_totals.csv`) read: 75 generation requests →
69/67 valid → 3/2 screened, then 15 perturbation requests → 5/4 screened →
1 Tier 1 each (`trial_env` finds 2 Tier 1 in generation as well, 3 in total).

`analysis\totals\tier1_candidates_total.csv` lists every Tier 1 molecule of a
run together with the three predicted properties and the number of properties on
which it beats the same-pipeline JP-10 reference.

## 5. What is *not* in this package

* the external raw datasets (GDB-13/17 archives, QM9, ANI-1x) used to *train*
  the models — only the trained weights are shipped;
* the model-selection candidate regressors of the perturbation iteration step
  (`Evaluation_Iteration\evaluation\model\perturbation_iteration\candidates\`);
  the deployed model `perturbation_delta_iteration.joblib` is included. A
  pre-release artifact without `ring_count_definition` is retrained from the
  current run by the iterative perturbation command;
* historical production outputs.
