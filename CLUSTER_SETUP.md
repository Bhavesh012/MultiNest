# MultiNest + PyMultiNest forks: cluster setup and maintenance guide

Written for a Claude Code session on an HPC cluster (Linux). The owner (Bhavesh Rajpoot) uses patched forks
of MultiNest and PyMultiNest on their laptop. This file tells you how to install the same versions on the
cluster, how to check they behave correctly, and the rules to follow if you need to fix something.
Last verified: 2026-10-11 on macOS (serial and MPI, 2 ranks). Not yet run on the cluster.

## 1. The two repositories

| Package | Fork (push here) | Upstream it is based on | Last code change (minimum required) |
|---|---|---|---|
| MultiNest (Fortran library) | https://github.com/Bhavesh012/MultiNest | JohannesBuchner/MultiNest master `a06c192` | `03e8e41` |
| PyMultiNest (Python wrapper) | https://github.com/Bhavesh012/PyMultiNest | JohannesBuchner/PyMultiNest master `920f725` | `ea94325` |

Both are tracked on branch `master`. This guide was added to the MultiNest fork in the commit after `03e8e41`.
Run `git log --oneline` and compare with the table: newer commits that change code are the owner's and take
precedence over this file.

## 2. What the forks change, and why

**MultiNest.** The only change from upstream is the output format: every `E28.18` became `E28.18E4` in
`src/nested.F90` and `src/posterior.F90`.

- Why: with plain `E28.18`, Fortran drops the `E` when an exponent needs 3 digits. A likelihood of `-5e99` is then
  written as `-0.500000000000000008+100`, which numpy, ForMoSA's `RAW_ev.dat` reader and PyMultiNest cannot parse.
  This happens whenever a likelihood returns about `-1e99` or less for an invalid model (pRT does that).
  Official MultiNest v3.12 (farhanferoz/MultiNest) made the same fix with `E3`.
- The sampling itself is untouched. Verified: lnZ, posterior and number of likelihood calls are bit-identical to
  stock upstream, official v3.12 and conda-forge multinest 3.10 on every test problem and seed. Resume is identical too.
- History: commit `4619331` once also ported part of the v3.12 changes to the `evDataAll` indexing in
  `posterior.F90` (stride `nPar+4`). That was wrong because v3.12 also changes the writer in `nested.F90` to
  `totPar+4`, which was never ported. With `write_output=False` it read the in-memory posterior at the wrong stride
  and past the end of the array. The result was a wrong lnZ, NaN rows and non-reproducible runs. Commit `03e8e41`
  reverted it.

**PyMultiNest.** Two changes:

- `pymultinest/analyse.py`, `Analyzer._read_error_line`: it now splits on `+/-` and uses
  `_parse_fortran_float`, which puts back a missing `E` (`0.12-100` -> `0.12E-100`). This works with `E28.18`,
  `E3` and `E4` output, and on both the global lines (`... Log-Evidence : x +/- y`) and the per-mode lines, which
  have no colon (`Strictly Local Log-Evidence x +/- y`).
  - Upstream's parser fails on E4 output. That is PyMultiNest issue #194, and it is why upstream MultiNest reverted
    the same E4 fix in 2021.
  - An earlier version in this fork split on `:` instead. It crashed `get_stats()` when
    `importance_nested_sampling=False`.
- `pymultinest/solve.py`: prints the full traceback when the user's log-likelihood raises, instead of just the
  error message.

**Rules that follow from this:**

- The two forks must be used together. Fork MultiNest with upstream or conda-forge PyMultiNest makes `get_stats()`
  fail; pRT calls it unguarded and crashes. The fork parser itself also reads stock output fine.
- Never port pieces of v3.12. Port it completely (writer and reader layout together) or not at all.
- Do not install conda-forge `multinest` or `pymultinest`, or PyPI `pymultinest`, into an env that uses the forks.
  They shadow the forks quietly. The conda-forge build is the 2015 v3.10 tag. It still has the E-less exponent bug,
  and it crashes on output paths over 100 characters (`File already opened in another unit`); the forks allow 1000.

## 3. If MultiNest/PyMultiNest already exist on the cluster

Older copies may still be at fork commits `4619331` (MultiNest, broken `write_output=False`) or `0d278dc`
(PyMultiNest, `get_stats()` crashes with INS off). Stale copies shadowing the fork were the main problem on the
laptop. Before installing anything, take an inventory and show it to the owner:

```bash
find ~ /scratch* $SCRATCH $WORK -name 'libmultinest*.so*' 2>/dev/null     # every built library
grep -n -E "MultiNest|LD_LIBRARY_PATH" ~/.bashrc ~/.bash_profile ~/.profile 2>/dev/null
ls -d ~/.local/lib/python3.*/site-packages/pymultinest* 2>/dev/null       # --user installs shadow ALL envs of that Python version
for e in $(conda env list | awk '!/^#/ && NF {print $NF}'); do
  echo "$e: $($e/bin/python -c 'import importlib.util as u; s=u.find_spec("pymultinest"); print(s.origin if s else "-")' 2>/dev/null)"
done                                                                     # run this from a directory that is NOT a PyMultiNest checkout
```

Then:

- **Existing clone of the fork:** `git pull` to at least the commits in the table, then run `make` in `MultiNest/build`.
- **Copy from somewhere else** (upstream clone, PyPI, conda-forge, a hand-copied `.so`): leave it on disk,
  but make sure nothing loads it. Remove it from `LD_LIBRARY_PATH`, and in envs that use the fork run
  `pip uninstall pymultinest` or `conda remove multinest pymultinest`.
- **Non-editable or `--user` pymultinest copies:** replace them with `pip install --no-deps -e <PyMultiNest clone>`.
  Use `pip install --user ...` only when a `--user` copy was already there and other envs relied on it.
  Remember that the user site comes before env site-packages.

## 4. Installing on the cluster

Use the GNU toolchain. Build MultiNest with the same MPI that mpi4py will use.

```bash
# 0. modules: names differ per cluster, check `module avail`
module load gcc cmake openmpi openblas      # or: intel-mkl / mpich, etc.

# 1. sources (pick a directory on a shared filesystem visible from compute nodes)
mkdir -p ~/packages && cd ~/packages
git clone https://github.com/Bhavesh012/MultiNest.git
git clone https://github.com/Bhavesh012/PyMultiNest.git

# 2. build MultiNest; produces lib/libmultinest.so and lib/libmultinest_mpi.so
cd ~/packages/MultiNest/build
cmake .. && make                             # NOT `make install` (targets /usr/local, makes a second hidden copy)
ls ../lib                                    # libs always land in MultiNest/lib (LIBRARY_OUTPUT_PATH)
# full rebuild later (other compiler/MPI): remove everything in build/ except README and .gitignore, rerun cmake

# 3. make the library findable: add to ~/.bashrc AND to every Slurm/PBS job script
export LD_LIBRARY_PATH=$HOME/packages/MultiNest/lib:$LD_LIBRARY_PATH

# 4. Python env (conda/mamba or venv), then the wrapper as an editable install
pip install numpy scipy matplotlib
pip install --no-deps -e ~/packages/PyMultiNest
#    mpi4py must be compiled against the cluster MPI, not a conda/pip-bundled one:
pip install --no-binary=mpi4py --no-cache-dir mpi4py

# 5. nothing may shadow the forks
conda list | grep -i -E "multinest|mpich|openmpi"   # expect no conda multinest/pymultinest
python -c "import pymultinest; print(pymultinest.__file__)"   # must be ~/packages/PyMultiNest/...
```

Notes:

- `CMakeLists.txt` already adds `-ffree-line-length-none` and, for gfortran 10 and newer, `-fallow-argument-mismatch`.
  If cmake does not find LAPACK, load a BLAS/LAPACK module or pass `-DLAPACK_LIBRARIES=...`.
- On Linux, PyMultiNest calls `ctypes.cdll.LoadLibrary('libmultinest.so')` with the bare name (`find_library`
  is only used on macOS). That makes it the dynamic loader's choice: the Python binary's RPATH comes first
  (conda Pythons point to `$CONDA_PREFIX/lib`, so a conda `multinest` package there wins), then `LD_LIBRARY_PATH`,
  then the ldconfig cache. Under `mpiexec`/`srun` with more than one rank it also loads `libmultinest_mpi.so`
  (`pymultinest/run.py`). The self-test prints the file that actually got mapped.
- If the downstream codes (ForMoSA, petitRADTRANS) are installed with pip, install them first and the
  PyMultiNest fork last, or use `--no-deps` for them. Otherwise pip may pull PyPI `pymultinest` back in.

## 5. Verifying the install

Save the script below as `fork_selftest.py` and run it from an empty scratch directory: once serially, and once
under MPI inside an interactive or batch allocation. It takes about a minute. Each check targets a specific
past bug:

| Check | Fails with |
|---|---|
| fixed stats parser present | upstream/PyPI/conda-forge PyMultiNest |
| `get_stats` (INS on / INS off) | upstream parser + fork lib; old `:`-split parser (INS off) |
| numpy reads `ev.dat` with -5e99 | stock/conda-forge MultiNest (E-less exponents) |
| `write_output=False` matches `write_output=True` | the partial v3.12 port (`4619331`) |
| output path > 100 chars | conda-forge multinest 3.10 (crashes the process) |

```bash
mkdir -p $SCRATCH/mn_selftest && cd $SCRATCH/mn_selftest
python fork_selftest.py
srun -n 4 python fork_selftest.py        # or: mpiexec -n 4 python fork_selftest.py
```

Expected: `ALL PASSED`, exit code 0, and the printed `libmultinest` path inside the fork's `lib/` directory.
Reference output on the laptop (serial, gfortran 15, Accelerate). On the cluster the numbers may differ slightly
because of compiler and BLAS differences, but they must stay within the tolerances the script checks:

```
PASS get_stats (INS on) lnZ = -0.021 +/- 0.019 (truth 0)
PASS numpy reads ev.dat with -5e99 values 2098 rows
PASS posterior mean/std mean [0.499 0.501] std [0.0502 0.0524] (truth 0.5 / 0.05)
PASS get_stats (INS off) lnZ = 0.123
PASS write_output=False posterior has no NaN 2498 rows
PASS write_output=False matches write_output=True logZ -0.046515 vs -0.046515
PASS output path > 100 chars 249 chars
ALL PASSED
```

```python
"""Self-test for the Bhavesh012 MultiNest + PyMultiNest forks.

Run from an empty scratch directory:
    python fork_selftest.py                 # serial checks
    mpiexec -n 4 python fork_selftest.py    # also checks the MPI library
Exit code 0 = all checks passed.
"""
import os, sys
import numpy as np
import pymultinest
from pymultinest.run import lib, lib_mpi

try:
    from mpi4py import MPI
    rank, size = MPI.COMM_WORLD.Get_rank(), MPI.COMM_WORLD.Get_size()
except ImportError:
    rank, size = 0, 1

SIG = 0.05
def prior(cube, nd, np_):
    pass
def loglike(cube, nd, np_):
    # 2D normalised Gaussian in U[0,1]^2 -> lnZ = 0. x0 > 0.7 is an "invalid model"
    # region returning -5e99, so 3-digit exponents end up in the output files.
    if cube[0] > 0.7:
        return -5e99
    x = np.array([cube[0], cube[1]])
    return float(-0.5 * np.sum(((x - 0.5) / SIG) ** 2) - 2 * np.log(np.sqrt(2 * np.pi) * SIG))

if '--dump-case' in sys.argv:
    # helper mode: one run in a fresh process, posterior from dump_callback saved to .npz
    wo = sys.argv[sys.argv.index('--dump-case') + 1] == 'files'
    d = {}
    def dump(nSamples, nlive, nPar, physLive, posterior, paramConstr, maxLogLike, logZ, logZerr, ctx):
        d.update(logZ=float(logZ), post=np.array(posterior, copy=True))
    pymultinest.run(loglike, prior, 2, outputfiles_basename='st-dump-', seed=1, resume=False,
                    verbose=False, n_live_points=400, sampling_efficiency=0.8,
                    write_output=wo, dump_callback=dump)
    np.savez('st-dump-%s.npz' % ('files' if wo else 'mem'), **d)
    sys.exit(0)

failures = []
def check(name, ok, info=''):
    if rank == 0:
        print('%-4s %s %s' % ('PASS' if ok else 'FAIL', name, info))
    if not ok:
        failures.append(name)

def run(basename, ins=True):
    pymultinest.run(loglike, prior, 2, outputfiles_basename=basename, seed=1, resume=False,
                    verbose=False, n_live_points=400, sampling_efficiency=0.8,
                    importance_nested_sampling=ins)

def resolved(l):
    # on Linux ctypes only knows the bare name ('libmultinest.so'); look up the file actually mapped
    if l is None:
        return '(not loaded, serial run)'
    try:
        for line in open('/proc/self/maps'):
            if line.rstrip().endswith('/' + os.path.basename(l._name)) or os.path.basename(l._name) + '.' in line:
                return line.split()[-1]
    except OSError:
        pass
    return l._name

if rank == 0:
    print('libmultinest     :', resolved(lib))
    print('libmultinest_mpi :', resolved(lib_mpi))
    print('pymultinest      :', pymultinest.__file__)
check('analyse.py has the fixed stats parser', hasattr(pymultinest.analyse, '_parse_fortran_float'))
if size > 1:
    check('MPI library loaded under mpiexec', lib_mpi is not None)

# 1. default usage (files, INS on) + 3-digit exponents in ev.dat
run('st-ins-')
if rank == 0:
    a = pymultinest.Analyzer(n_params=2, outputfiles_basename='st-ins-', verbose=False)
    try:
        s = a.get_stats()
        lnZ, err = s['global evidence'], s['global evidence error']
        check('get_stats (INS on)', abs(lnZ) < 3 * err + 0.2, 'lnZ = %.3f +/- %.3f (truth 0)' % (lnZ, err))
    except Exception as e:
        check('get_stats (INS on)', False, repr(e))
    try:
        ev = np.loadtxt('st-ins-ev.dat', ndmin=2)
        check('numpy reads ev.dat with -5e99 values', (ev[:, 2] < -1e99).any(), '%d rows' % len(ev))
    except Exception as e:
        check('numpy reads ev.dat with -5e99 values', False, repr(e)[:120])
    eq = a.get_equal_weighted_posterior()[:, :2]
    check('posterior mean/std', np.allclose(eq.mean(0), 0.5, atol=0.01) and np.allclose(eq.std(0), SIG, rtol=0.15),
          'mean %s std %s (truth 0.5 / %.2f)' % (np.round(eq.mean(0), 3), np.round(eq.std(0), 4), SIG))

if size == 1:
    # 2. INS off: per-mode "Local Log-Evidence" lines have no colon (old parser crashed here)
    run('st-noins-', ins=False)
    try:
        s = pymultinest.Analyzer(n_params=2, outputfiles_basename='st-noins-', verbose=False).get_stats()
        check('get_stats (INS off)', abs(s['global evidence']) < 3 * s['global evidence error'] + 0.2,
              'lnZ = %.3f' % s['global evidence'])
    except Exception as e:
        check('get_stats (INS off)', False, repr(e))

    # 3. write_output=False: posterior given to dump_callback is built in memory
    #    (the partial v3.12 port in posterior.F90 broke exactly this path).
    #    Compared in fresh processes: MultiNest keeps state between calls, so the first
    #    run in a process differs from later identical runs (upstream behaviour, all versions).
    import subprocess
    for case in ('files', 'mem'):
        subprocess.check_call([sys.executable, os.path.abspath(__file__), '--dump-case', case],
                              stdout=subprocess.DEVNULL)
    f, m = np.load('st-dump-files.npz'), np.load('st-dump-mem.npz')
    check('write_output=False posterior has no NaN', np.isfinite(m['post']).all(), '%d rows' % len(m['post']))
    check('write_output=False matches write_output=True',
          m['logZ'] == f['logZ'] and m['post'].shape == f['post'].shape
          and np.allclose(m['post'], f['post'], rtol=1e-12, atol=0),
          'logZ %.6f vs %.6f' % (m['logZ'], f['logZ']))

    # 4. output paths longer than 100 characters (conda-forge v3.10 crashes here)
    deep = os.path.join(os.getcwd(), 'a_rather_long_directory_name_for_testing_' * 3)
    os.makedirs(deep, exist_ok=True)
    try:
        run(os.path.join(deep, 'st-long-'))
        check('output path > 100 chars', os.path.exists(os.path.join(deep, 'st-long-stats.dat')),
              '%d chars' % len(os.path.join(deep, 'st-long-')))
    except Exception as e:
        check('output path > 100 chars', False, repr(e))

if rank == 0:
    print('ALL PASSED' if not failures else 'FAILED: %s' % ', '.join(failures))
sys.exit(1 if failures else 0)
```

## 6. Known behaviour that is NOT a bug in the forks

- **The first run in a Python process differs from later identical runs** with the same seed. MultiNest keeps state
  between calls; after the first call, results are reproducible. This was verified with stock upstream, the forks
  and official v3.12, and it is why the self-test compares in fresh processes. To compare results bit-for-bit,
  run each case in its own process.
- **`summary.txt` has rows with different numbers of columns**, so `np.loadtxt` fails on it. Every MultiNest
  version writes it this way.
- **Upstream PyMultiNest `dump_callback` bugs**, not fixed in the fork. Do not fix them without asking the owner,
  because downstream code may depend on the current behaviour:
  - `paramConstr` is reshaped as `(nPar, 4)` instead of `(4, nPar)`, so the mean/std/best-fit/MAP tuples come out
    scrambled.
  - The Python dumper signature lacks `INSlogZ`, so the argument named `logZerr` actually holds INSlogZ.
- **`pycuba` imports fail** when `libcuba.so` is not built. This is unrelated and was already the case before.

## 7. Downstream code that depends on these files

- **ForMoSA** (`nested_sampling/results.py`) uses the basename `RAW_`. It reads:
  - `RAW_stats.dat` with `split()[5]` and `[7]` (global evidence and its error, via `float()`);
  - `RAW_ev.dat` and `RAW_.txt` with `float()`/numpy.
  It therefore needs the E4 output, and the `stats.dat` layout must not change.
- **petitRADTRANS 4** (`retrieval/sampler.py`, `post_run`) calls `Analyzer.get_stats()` without a try/except.
  It needs the fork parser.

## 8. If something needs fixing

1. Reproduce it with the smallest script you can (start from `fork_selftest.py`). Confirm which library and which
   `pymultinest` are actually loaded before blaming the code: an outside copy shadowing the fork is the most
   common cause.
2. Code locations:
   - Fortran output: `MultiNest/src/nested.F90` (writes `ev.dat`, `phys_live.points`, resume files) and
     `MultiNest/src/posterior.F90` (writes `.txt`, `post_equal_weights.dat`, `stats.dat`, `summary.txt`; builds the
     in-memory posterior for `write_output=False`).
   - Python wrapper: `PyMultiNest/pymultinest/run.py` (ctypes call, library loading),
     `analyse.py` (reads output files), `solve.py` (pythonic wrapper).
   - All Fortran read-backs, such as resume, use list-directed `read(u,*)`. Output format changes therefore never
     break resuming, but layout changes do.
3. Rebuild after Fortran edits: `cd MultiNest/build && make`. Python edits need no reinstall (editable install).
   Long-running Python sessions must be restarted to pick up a new `.so`.
4. Rerun `fork_selftest.py`, serial and MPI. Every check must pass. If you change sampling-relevant code, also
   compare lnZ and posterior against a build of the previous commit with the same seed, each case in a fresh process.
5. Commit in the cluster clone with a message that says what and why. Push to the fork (`origin master`) only after
   the self-test passes, and tell the owner. Their laptop has its own clones under `~/Karmabhumi/Packages/`, which
   need `git pull` plus a MultiNest `make` to stay in sync.
6. Do not modify shared or system installs (`/usr/local`, cluster-wide modules, other users' envs).
