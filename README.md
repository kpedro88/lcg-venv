# lcg-venv

## Usage

```sh
source /cvmfs/sft.cern.ch/lcg/views/LCG_109a/x86_64-el9-gcc13-opt/setup.sh
lcg-venv myenv          # default name: venv
. myenv/bin/activate
```

## Explanation

An LCG view's `setup.sh` exports the view's directories on **`PYTHONPATH`** (and
sets `PYTHONHOME`):

```sh
PYTHONPATH=${thisdir}/lib:$PY_PATHS:$PYTHONPATH   # PY_PATHS = view .../site-packages
```

CPython places `PYTHONPATH` entries on `sys.path` **ahead** of a venv's own
`site-packages`. So even with `python3 -m venv --system-site-packages`, anything
you `pip install` into the venv is shadowed by the LCG copy.

To make local installs win, we control things *inside the interpreter*.
At creation time, `lcg-venv` first **snapshots the LCG view directories**
(the dirs that the view puts on `PYTHONPATH`) into `_lcgvenv_lcg_paths.txt`, then
drops **two cooperating hooks** into the venv's `site-packages`, split this way
because of *when* Python's `site` module runs each kind of hook at startup
(necessary for editable installs and build isolation to work):

1. **`_lcgvenv_pathfix.pth`** (`import _lcgvenv_pathfix`) → runs early, during
   `.pth` processing. It **promotes** the venv's `purelib`/`platlib` to the
   front of `sys.path`. Its job is to make sure the venv's *own*
   `sitecustomize.py` is the one `site` discovers (otherwise the LCG view's
   `sitecustomize.py`, earlier on the path, shadows it). It uses
   **`_lcgvenv_pathfix.py`** under the hood.
2. **`sitecustomize.py`** → runs last, after *all* `.pth` files
   (`site.main()` calls `execsitecustomize()` at the end). It does the
   authoritative reordering: it **demotes the snapshotted LCG view directories**
   to the tail of `sys.path`. Everything left in front — the venv site-packages
   *and* any source dirs that editable installs appended — keeps priority, while
   all LCG packages stay importable from the tail. It also chains to the next
   `sitecustomize.py` on the path (the LCG view's) so that behavior is
   preserved.

Both act only inside a venv (`sys.prefix != sys.base_prefix`), only touch the
snapshotted LCG dirs, and run at **every** interpreter startup — no matter how
Python was launched.

### `PYTHONHOME` and embedded interpreters (e.g. `gdb`)

Programs that **embed their own libpython** — most notably the LCG `gdb` — do
not read `pyvenv.cfg` or `.pth` files. They locate the stdlib from the
build-time prefix compiled into the binary, and for LCG that path
(`/build/jenkins/.../Python/3.11.9/...`) does not exist on cvmfs. LCG's
`setup.sh` papers over this by exporting `PYTHONHOME`, which points the
embedded interpreter at the real cvmfs stdlib.

Stock venv `activate` **unsets `PYTHONHOME`**. With it gone, `gdb`'s embedded
Python can't find `encodings` and dies at startup.

`lcg-venv` therefore appends a small restore step to the end of `bin/activate`
(and `bin/activate.fish`) that puts `PYTHONHOME` back after venv activation. It
points at the venv's own base interpreter, so it is safe for the venv python
(which still reports the venv `sys.prefix` and still lets local packages win),
and `deactivate` restores/round-trips it cleanly. `activate.csh` never touches
`PYTHONHOME`, so it needs no change.

### Editable installs (`pip install -e`)

An editable install does **not** put the package in the venv's
site-packages. It drops a `.pth` file (e.g. `_editable_impl_<pkg>.pth`,
`__editable__.<pkg>-<ver>.pth`) whose content is a **bare directory path** —
the project's source dir. `site` **appends** bare-path `.pth` entries to the
end of `sys.path`. So the editable source lands *after* the LCG view's
`site-packages`, and LCG wins.

An early `.pth` hook that only *promotes the venv site-packages* can't fix
this, because the editable source dir isn't in site-packages, and worse: `.pth`
files are processed in **sorted filename order**, so an editable `.pth` that
sorts *after* our hook is added to `sys.path` only *after* our hook has already
run.

The robust fix is the **`sitecustomize.py`** piece described above: `site` runs
it *after every* `.pth` file, and it demotes the LCG view directories to the
tail — so anything a `.pth` appended (editable sources included) ends up ahead
of LCG regardless of `.pth` sort order.

### Build isolation

pip creates a temporary build environment, installs the
build requirements into it (e.g. `hatchling` **and its deps**, e.g. `pathspec`),
sets `PYTHONPATH` to *that* dir, and expects it **first** on `sys.path`.

Therefore, both hooks must key off the *specific* LCG view directories, not
"whatever is on `PYTHONPATH` now." `lcg-venv` snapshots the real LCG view dirs
at creation time into `_lcgvenv_lcg_paths.txt`, and:

* the promoter runs **only if** one of those snapshot dirs is actually on
  `sys.path` (false inside the build subprocess → no-op);
* the demoter moves **only** those snapshot dirs to the tail (pip's build-env
  dir is not among them → untouched).

## Caveats

* **`PATH` name resolution.** If you re-source the LCG view *after* activating,
  the view prepends its `bin` to `$PATH`, so the bare name `python` resolves to
  LCG's interpreter, not the venv's. This is a `$PATH` quirk, **not** a
  `sys.path` failure — the venv interpreter itself always behaves correctly
  (`<venv>/bin/python` gives the right answer in every ordering). Guidance is the
  same as cvmfs-venv's: activate last. Neither tool can stop a later `setup.sh`
  from reordering `$PATH`.
* **Isolation is not total.** `--system-site-packages` is required so LCG
  packages remain visible; consequently a `pip install`/upgrade can still see LCG
  versions when resolving dependencies. Use `pip install --ignore-installed` and
  inspect with `pip list --local`.
* **Python version is tied to the view.** If the LCG view (hence its Python)
  changes, recreate the venv. Pin everything with a `requirements`/lock file for
  reproducibility.

## Sources

This approach was inspired by combining [`scram-venv`](https://github.com/cms-sw/cms-common/blob/master/common/scram-venv) and [`cvmfs-venv`](https://github.com/matthewfeickert/cvmfs-venv).
Code and documentation were generated by Claude Opus 4.8.
