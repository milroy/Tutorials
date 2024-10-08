# Python Asynchronous Bulk Job Submission

Parts (a) and (b) demonstrate different implementations of the same basic use-case---submitting large numbers of jobs to Flux. For simplicity, in these examples all of the jobs are identical.

In part (a), we use the `flux.job.submit_async` and `flux.job.wait` functions to submit jobs and wait for them.
In part (b), we use the `FluxExecutor` class, which offers a higher-level interface. It is important to note that
these two different implementations deal with very different kinds of futures.
The executor's futures fulfill in the background and callbacks added to the futures may
be invoked by different threads; the `submit_async` futures do not fulfill in the background, callbacks are always
invoked by the same thread that added them, and sharing the futures among threads is not supported.

## Setup - Downloading the Files

If you haven't already, download the files and change your working directory:

```bash
$ cd flux-workflow-examples/async-bulk-job-submit
```

## Part (a) - Using `submit_async`

### Description: Asynchronously submit jobspec files from a directory and wait for them to complete in any order

1. Allocate three nodes from a resource manager:

```bash
salloc -N3 -ppdebug
```

2. Make a **jobs** directory:

```bash
mkdir /tmp/jobs
```

3. If you are running Slurm, launch a Flux instance on the current allocation by running `flux start` once per node, redirecting log messages to the file `out` in the current directory:

```bash
srun --pty --mpi=none -N3 flux start -o,-S,log-filename=out
```

4. Store the jobspec of a `sleep 0` job in the **jobs** directory:

```bash
flux run --dry-run -n1 sleep 0 > /tmp/jobs/0.json
```

5. Copy the jobspec of **job0** 1024 times to create a directory of 1025 `sleep 0` jobs:

```bash
for i in `seq 1 1024`; do cp /tmp/jobs/0.json /tmp/jobs/${i}.json; done
```

6. Run the **bulksubmit.py** script and pass all jobspec in the **jobs** directory as an argument with a shell glob `jobs/*.json`:

```bash
./bulksubmit.py /tmp/jobs/*.json
```
```console
bulksubmit: Starting...
bulksubmit: submitted 1025 jobs in 0.43s. 2392.93job/s
bulksubmit: First job finished in about 0.521s
|██████████████████████████████████████████████████████████| 100.0% (274.3 job/s)
bulksubmit: Ran 1025 jobs in 3.7s. 274.3 job/s
```

### Notes to Part (a)

- `h = flux.Flux()` creates a new Flux handle which can be used to connect to and interact with a Flux instance.

- `job_submit_async(h, jobspec.read(), waitable=True).then(submit_cb)` submits a jobspec, returning a future which will be fulfilled when the submission of this job is complete.

`.then(submit_cb)`, called on the returned future, will cause our callback `submit_cb()` to be invoked when the submission of this job is complete and a jobid is available. To process job submission RPC responses and invoke callbacks, the flux reactor for handle `h` must be run:

```python
if h.reactor_run() < 0:
    h.fatal_error("reactor start failed")
```

The reactor will return automatically when there are no more outstanding RPC responses, i.e., all jobs have been submitted.

- `job.wait(h)` waits for any job submitted with the `FLUX_JOB_WAITABLE` flag to transition to the **INACTIVE** state.


### Part (b) - Using FluxExecutor

#### Description: Asynchronously submit a single command repeatedly

If continuing from part (a), skip to step 3.

1. Allocate three nodes from a resource manager:

```bash
salloc -N3 -ppdebug
```

2. Launch a Flux instance on the current allocation by running `flux start` once per node, redirecting log messages to the file `out` in the current directory:

```bash
srun --pty --mpi=none -N3 flux start -o,-S,log-filename=out
```

3. Run the **bulksubmit_executor.py** script and pass the command (`/bin/sleep 0` in this example) and the number of times to run it (default is 100):

```bash
./bulksubmit_executor.py -n200 /bin/sleep 0
```
```console
bulksubmit_executor: submitted 200 jobs in 0.18s. 1087.27job/s
bulksubmit_executor: First job finished in about 0.248s
|██████████████████████████████████████████████████████████| 100.0% (229.8 job/s)
bulksubmit_executor: Ran 200 jobs in 1.0s. 199.6 job/s
```
