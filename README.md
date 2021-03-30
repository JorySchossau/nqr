
## nqr
**nqr** is an enqueuer for pleasantly parallel computation of variation and replication.
In other words, it uses simple language and native python mechanisms ~like autocomplete~* to help deploy experiments, so you can spend more time doing science and less time writing and managing pipelines. However, nqr does not have any mechanism to check on your jobs and know when they are done, as such it has an "enqueue and forget" philosophy.

nqr creates a unique directory for each run and all required files and output files are there, but what that actually looks like depends on the backend target to which you submit the jobs.

*turns out SimpleNamespace doesn't support autocomplete, but Prodict does, so I'll engineer the replacement

## How to Use
Suppose you are performing a stochastic analysis on various chunks of data, and aggregating statistics on this method.
**1)** Create a variable with any default value
```python
g.dataChunk = 0
```

**2)** List the variations for any subset of variables
```python
with Group("analysis"):
    Vary(g.dataChunk, list(range(0,20,5)))
```
**3)** Set the replication power

```python
g.setreps(32)
```

**4)** Define any file requirements and commands to run
```python
g.requires('analysis.py')
g.requires('data.csv')
g.addcommand(exe='python analysis.py data.csv',template='{exe} -r {rep} {parameters}')
```
**5)** Lastly, launch the jobs on a supported target, such as localhost or SLURM, or write your own (more to come)
```python
for run in g.Runs():
    local.launch(run)
    # modify any information in run:dict as needed before launch
    # or skip certain conditions or replicates if you need
```
## Installation
This is not yet on pipy, but will be when it exits beta. Until then, install as a local package.
```bash
git clone https://github.com/joryschossau/nqr
cd nqr
pip install --user -e nqr
```

## Full Example
Here is a complete example experiment showcasing complex variations, setup, and analysis.
```python
import nqr
from nqr import generic as g, Vary, Group
from numpy import arange

# these will be discovered and used as parameters
g.allowedTime = 100
g.MEMORY = nqr.NS()
g.MEMORY.nitems = 10
g.MEMORY.noise = 0.01

# create a set of conditions
with Group("memory"): # name optional
    Vary(g.allowedTime, [1000,5000], name='time') # name & labels optional
    Vary(g.MEMORY.nitems, [10,50], name='nItems')
    Vary(g.MEMORY.noise, arange(0.00,0.1,0.04), name='noise')
# ... more groups

# set replication power
g.setreps(32) # alt: setreps(range(101,110))

g.requires('mabe')
g.requires('analysis.py')

g.addcommand(exe='module load pytorch python') # a supercomputer command
g.addcommand(exe='mabe',template='{exe} -p GLOBAL-randomSeed {rep} {parameters}')
g.addcommand(exe='python analysis.py results.csv',template='{exe} -r {rep} {parameters}')

# launch the jobs on the localhost
from nqr.targets import local
for run in g.Runs(priority=g.REPS):
    local.launch(run)
```
Running this script without any arguments does a dry-run and shows what would happen:
```
Jobs
384 total submission(s) from
12 condition(s)
32 replicate(S) each condition

Directories
results/memory_C00_time=1000_nItems=10_noise=0.0/{00..31}
results/memory_C01_time=1000_nItems=10_noise=0.04/{00..31}
results/memory_C02_time=1000_nItems=10_noise=0.08/{00..31}
results/memory_C03_time=1000_nItems=50_noise=0.0/{00..31}
results/memory_C04_time=1000_nItems=50_noise=0.04/{00..31}
results/memory_C05_time=1000_nItems=50_noise=0.08/{00..31}
results/memory_C06_time=5000_nItems=10_noise=0.0/{00..31}
results/memory_C07_time=5000_nItems=10_noise=0.04/{00..31}
results/memory_C08_time=5000_nItems=10_noise=0.08/{00..31}
results/memory_C09_time=5000_nItems=50_noise=0.0/{00..31}
results/memory_C10_time=5000_nItems=50_noise=0.04/{00..31}
results/memory_C11_time=5000_nItems=50_noise=0.08/{00..31}

Local Target
 parallelism: serial
output delay: no
```
Perform job submissions (not a dry-run) with the `--run` flag

## Features
### General
* **Custom nqr experience** The nqr experience can be customized for different tools if you always run the same tool. See `custom/mabe.py` which you import and use instead of `generic as g` and it discovers all the namespace parameter defaults for you.
* **Custom NS to CLI string** `Runs(arg_transform=your_function)` lets you customize how parameter name-value pairs are converted into CLI parameter strings for your specific tool.
* **Prioritize reps or conditions** `Runs(priority='reps')` or `Runs(priority='conditions')`
### Localhost Target (nqr.targets.local)
* **Serial** Normal operation, one run after the other
* **Parallel** Use as many cores as you want!
* **Reduce Spam** Optionally scrape N latest lines of process output (serial mode only)
### SLURM/SBATCH Target (nqr.targets.slurm)
* **Supercomputers** Submits to the locahost's SLURM job system
* **.finished** Conditions are file-semaphore-marked when they complete successfully
* **DMTCP** Optionally submit infinite-running jobs in 4hr chunks, until they finish or you kill them
### All Targets (nqr.targets.targets)
* **CLI Target Selection** Pick among loaded targets from the command line
* **Custom Target** Easily write your own target - it only needs to support Runs() similar to easy examples
## Planned Features
* **AWS/S3 Target**
* **Google Cloud Target??**
* **Azure Target??**
* **Cleanup UX** Change a few things around for a more fluid, simpler, and flexible workflow

## Help
```
NQR: the Enqueuer for Pleasantly Parallel Computation
    
    Usage: python nqr_script.py [options]
    
    Should be run from within target directory.
    
    Help - General ===============================
    
    (no flags)     Dry-run, report what WOULD happen
    -h,--help      This help message
    --run          Actually perform actions, not dry-run
    

    Help - SLURM Target ==========================

    (no flags)  Normal submission
    -i          Indefinite mode (4hr chunks w/ checkpointing until done)
    -sN         When -i, limit indefinite mode to N 4hr chunks


    Help - Local Target ==========================

    (no flags)  Run jobs in serial (like -j1)
    -jN         Run jobs in parallel up to N
    -dS         Only show latest line(s) of output every S:float seconds
                (default 1, for empty -d)
    -lN         When -d, show N latest lines of output (default 1)
```
