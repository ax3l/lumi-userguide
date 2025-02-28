# Batch jobs

[slurm-quickstart]: ../../runjobs/scheduled-jobs/slurm-quickstart.md
[slurm-doc]: https://slurm.schedmd.com/documentation.html
[slurm-man]: https://slurm.schedmd.com/man_index.html
[slurm-sbatch]: https://slurm.schedmd.com/sbatch.html

This pages covers advanced topics related to running Slurm batch jobs on LUMI.
If you are not already familiar with Slurm, you should read the [Slurm
quickstart][slurm-quickstart] guide which covers the basics. You can also refer
to the Slurm [documentation][slurm-doc] or [manual pages][slurm-man], in
particular the page about [sbatch][slurm-sbatch].

## Specifying the account

The account option (`--account=project_<id>`) is mandatory. Failing to set
it will cause the following error:

```text
Unable to allocate resources: Job violates accounting/QOS policy 
(job submit limit, user's size and/or time limits)
```

If you don't want to specify the account to use every time you submit a job, you
can add the following lines to your `.bashrc` file.

```bash
export SBATCH_ACCOUNT=project_<id>
export SALLOC_ACCOUNT=project_<id>
```

where you have to replace `project_<id>` with the ID of the project you have
been granted.

## Example batch scripts

Here we give examples of batch scripts for typical workflows on LUMI. You may
use these as templates for your own project batch scripts.

### Shared memory jobs

```bash
#!/bin/bash -l
#SBATCH --job-name=examplejob   # Job name
#SBATCH --output=examplejob.o%j # Name of stdout output file
#SBATCH --error=examplejob.e%j  # Name of stderr error file
#SBATCH --partition=small       # Partition (queue) name
#SBATCH --ntasks=1              # One task (process)
#SBATCH --cpus-per-task=128     # Number of cores (threads)
#SBATCH --time=12:00:00         # Run time (hh:mm:ss)
#SBATCH --mail-type=all         # Send email at begin and end of job
#SBATCH --account=project_<id>  # Project for billing
#SBATCH --mail-user=username@domain.com

# Any other commands must follow the #SBATCH directives

# Set the number of threads based on --cpus-per-task
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
 
./your_application
```

### MPI-based jobs

!!! Failure "Fortan MPI program fails to start"
    If a Fortran based program with MPI fails to start when utilizing a large
    number of nodes (512 nodes for instance), add
    `export PMI_NO_PREINITIALIZE=y` to your batch script.  

```bash
#!/bin/bash -l
#SBATCH --job-name=examplejob   # Job name
#SBATCH --output=examplejob.o%j # Name of stdout output file
#SBATCH --error=examplejob.e%j  # Name of stderr error file
#SBATCH --partition=standard    # Partition (queue) name
#SBATCH --nodes=50              # Total number of nodes 
#SBATCH --ntasks=6400           # Total number of mpi tasks
#SBATCH --time= 1-12:00:00      # Run time (d-hh:mm:ss)
#SBATCH --mail-type=all         # Send email at begin and end of job
#SBATCH --account=project_<id>  # Project for billing
#SBATCH --mail-user=username@domain.com

# Any other commands must follow the #SBATCH directives

# Launch MPI code 
srun ./your_application # Use srun instead of mpirun or mpiexec
```

### Hybrid MPI+OpenMP jobs

```bash
#!/bin/bash -l
#SBATCH --job-name=examplejob   # Job name
#SBATCH --output=examplejob.o%j # Name of stdout output file
#SBATCH --error=examplejob.e%j  # Name of stderr error file
#SBATCH --partition=standard    # Partition (queue) name
#SBATCH --nodes=50              # Total number of nodes 
#SBATCH --ntasks-per-node=16    # Number of mpi tasks per node
#SBATCH --cpus-per-task=8       # Number of cores (threads) per task
#SBATCH --time=1-12:00:00       # Run time (d-hh:mm:ss)
#SBATCH --mail-type=all         # Send email at begin and end of job
#SBATCH --account=project_<id>  # Project for billing
#SBATCH --mail-user=username@domain.com

# Any other commands must follow the #SBATCH directives

# Set the number of threads based on --cpus-per-task
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

# Launch MPI code 
srun ./your_application # Use srun instead of mpirun or mpiexec
```

### Serial Job

```bash
#!/bin/bash -l
#SBATCH --job-name=examplejob   # Job name
#SBATCH --output=examplejob.o%j # Name of stdout output file
#SBATCH --error=examplejob.e%j  # Name of stderr error file
#SBATCH --partition=debug       # Partition (queue) name
#SBATCH --ntasks=1              # One task (process)
#SBATCH --time=00:15:00         # Run time (hh:mm:ss)
#SBATCH --mail-type=all         # Send email at begin and end of job
#SBATCH --account=project_id    # Project ID
#SBATCH --mail-user=username@domain.com
 
./your_application
```

## Automatic requeuing

The LUMI Slurm configuration has **automatic requeuing** of jobs upon node
failure **enabled**. It means that if a node fails, your job will be
automatically resubmitted to the queue and will have the same job ID and
possibly truncate the previous output. Here are some important parameters you
can use to alter the default behavior.

- you can disable automatic requeuing using the `--no-requeue` option
- you can avoid your output file being truncated in case of requeuing by using
  the `--open-mode=append` option

You can apply these two options permanently by exporting the following
environment variables in your `.bashrc`:

- `SBATCH_NO_REQUEUE=1` to disable requeuing
- `SBATCH_OPEN_MODE=append` to avoid output truncating after requeuing

If you want to perform specific operations in your batch script when a job has
been requeued you can check the value of the `SLURM_RESTART_COUNT` variable.
The value of this variable will be `0` if it's the first time the job is run.
If the job has been restarted then the value will be the number of times the
job has been restarted.

## Common error messages

Below are some common error messages you may get when your job submission
fails.

### Invalid account or account/partition combination specified

The complete error message is:

```text
sbatch: error: Batch job submission failed: Invalid account or account/partition combination specified
```

This error message relates to improper use of the Slurm options
`--account=<project>` and `--partition`. The most common causes are:

- the project does not exist.
- the project exists, but you are not a member of it.
- the partition does not exist.

### Job violates accounting/QOS policy

The complete error message is:

```text
sbatch: error: AssocMaxSubmitJobLimit
sbatch: error: Batch job submission failed: Job violates accounting/QOS policy (job submit limit, user's size and/or time limits)
```

The most common causes are:

- your project has already used all of its allocated compute resources.
- job script is missing the `--account` parameter.
- your project has exceeded the limit for number of simultaneous jobs, either
  running or queuing. Note that Slurm counts each job within an array job as a
  separate job.

## Common Slurm options

Here is an overview of some of the most commonly used Slurm options.

### Basic job specification

| Option        | Description                                              |
| --------------|----------------------------------------------------------|
| `--time`      | Set a limit on the total run time of the job allocation  |
| `--account`   | Charge resources used by this job to specified project   |
| `--partition` | Request a specific partition for the resource allocation |
| `--job-name`  | Specify a name for the job allocation                    |

### Specify tasks distribution

| Option                | Description                                 |
| ----------------------|---------------------------------------------|
| `--nodes`             | Number of nodes to be allocated to the job  |
| `--ntasks`            | Set the maximum number of tasks (MPI ranks) |
| `--ntasks-per-node`   | Set the number of tasks per node            |
| `--ntasks-per-socket` | Set the number of tasks on each node        |
| `--ntasks-per-core`   | Set the maximum number of task on each core |

### Request CPU cores

| Option            | Description                              |
| ------------------|------------------------------------------|
| `--cpus-per-task` | Set the number of cores per tasks        |
| `--cpus-per-gpu`  | Set the number of CPUs per allocated GPU |

### Request GPUs

| Option            | Description                                              |
| ------------------|----------------------------------------------------------|
| `--gpus`          | Set the total number of GPUs to be allocated for the job |
| `--gpus-per-node` | Set the number of GPUs per node                          |
| `--gpus-per-task` | Set the number of GPUs per task                          |

### Request memory

| Option            | Description                            |
| ------------------|----------------------------------------|
| `--mem`           | Set the memory per node                |
| `--mem-per-cpu`   | Set the memory per allocated CPU cores |
| `--mem-per-gpu`   | Set the memory per allocated GPU       |

!!! info
    The `/tmp` directory on the compute nodes resides in memory. The memory
    used for `/tmp` is included in the job memory allocation. If you use
    `/tmp`, you must allocate memory for it in order to avoid running out of
    memory.

### Receive email notifications

!!! warning
    The email notification feature is not yet configured and does not work at
    the moment.

Email notifications from Slurm can be requested when certain events occur (job
starts, fails, ...).

| Email Type    | Send email when                                            |
| --------------|------------------------------------------------------------|
| `--mail-user` | Used to specify the email that should receive notification |
| `--mail-type` | When to send an email: `BEGIN`, `END`, `FAIL`, `ALL`       |

## Pipelining with dependencies

Job dependencies allow you to defer the start of a job until the specified
dependencies have been satisfied. Dependencies can be defined in a batch script
with the `--dependency` directive or be passed as a command-line argument to
`sbatch`.

```bash
$ sbatch --dependency=<type:job_id[:job_id]>
```

The `type` defines the condition that the job with ID `job_id` must fulfil
before the job which depends on it can start. For example,

```bash
$ sbatch job1.sh
Submitted batch job 123456

$ sbatch --dependency=afterany:123456 job2.sh
Submitted batch job 123458
```

will only start execution of `job2.sh` when `job1.sh` has finished. The available
types and their description are presented in the table below.

| Dependency type               | Description                                           |
| ------------------------------|-------------------------------------------------------|
| `after:jobid[:jobid...]`      | Begin after the specified jobs have started           |
| `afterany:jobid[:jobid...]`   | Begin after the specified jobs have finished          |
| `afternotok:jobid[:jobid...]` | Begin after the specified jobs have failed            |
| `afterok:jobid[:jobid...]`    | Begin after the specified jobs have run to completion |

The example below demonstrates a bash script for submission of multiple Slurm
batch jobs with dependencies. It also shows an example of a helper function
that extracts the job ID from the output of the `sbatch` command.  

``` bash
#!/bin/bash

submit_job() {
  sub="$(sbatch "$@")"
  
  if [[ "$sub" =~ Submitted\ batch\ job\ ([0-9]+) ]]; then
    echo "${BASH_REMATCH[1]}"
  else
    exit 1
  fi
}

# first job - no dependencies
id1=$(submit_job job1.sh)

# Two jobs that depend on the first job
id2=$(submit_job --dependency=afterany:$id1 job2.sh)
id3=$(submit_job --dependency=afterany:$id1 job3.sh)

# One job that depends on both the second and the third jobs
id4=$(submit_job  --dependency=afterany:$id2:$id3 job4.sh)
```
