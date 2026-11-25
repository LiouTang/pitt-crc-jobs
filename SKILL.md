---
name: pitt-crc-jobs
description: Write, submit, monitor Slurm jobs on the University of Pittsburgh CRC computing cluster.
license: MIT
---

# CRC Jobs

Skill for managing Slurm jobs on the University of Pittsburgh CRC computing cluster, and operational guidance for user best practices. When the skill instructions are insufficient, consult the [CRC user manual](https://crc-pages.pitt.edu/user-manual/) for relevant sections regarding Slurm syntax, hardware, QoS, storage, or troubleshooting.

This is best used as a per-project and per-folder skill. Keep local configuration and submission state in `<local-project>/.slurm-mgmt` (written as `.slurm-mgmt` below). **This folder should not be synced with CRC or Git**. Create or ensure existence of `.slurm-mgmt/crc-config.json` using [assets/crc-config.json](assets/crc-config.json). Reuse an older `.slurm-mgmt` configuration only after confirming that it belongs to this project. Exclude local management state from source commits and synchronization; transfer rendered job scripts explicitly.

## Before Use

On first use, ask for:

1. The user's CRC username, stored in `.slurm-mgmt/crc-config.json` with key `crc_username`.

2. The user's email address, stored in `.slurm-mgmt/crc-config.json` with key `crc_useremail`.

3. The conda environment to use on CRC, stored in `.slurm-mgmt/crc-config.json` with key `conda_env`. **This setting is specific to this project and this folder only.** Ensure that this is the **full path** to the conda environment directory.

4. The full path to the Conda initialization script (`etc/profile.d/conda.sh` in the CRC Conda installation), stored in `.slurm-mgmt/crc-config.json` with key `conda_init`. This is a script path, distinct from the `conda_env` directory.

5. The working directory for the project on CRC, stored in `.slurm-mgmt/crc-config.json` with key `working_dir`. Ensure that this is the **full path** to the working directory on the CRC system.

6. A short, free-text description of how the local and remote codebase are synchronized, stored in `.slurm-mgmt/crc-config.json` with key `sync_method` (for example, `Git push/pull on main` or `SFTP upload on save`).

7. The full path to the project's active data, cache, and output storage on CRC, stored in `.slurm-mgmt/crc-config.json` with key `work_storage_dir`. For AI and I/O-intensive jobs, use the project's provisioned `/vast/` space; see "Resource Management" below.

8. **Ensure that `.slurm-mgmt/crc-config.json` contains valid JSON and nonempty strings for all required keys.** Remote paths must be absolute POSIX paths. Empty strings in the template are placeholders, not valid settings.

If any required configuration is missing or invalid, prompt the user to provide the necessary information and update `.slurm-mgmt/crc-config.json` accordingly. **Do not infer any information from local environments, system settings, or conversation contexts.**

## Verify Connection

This skill assumes that the user has configured **Passwordless SSH connection** to the CRC system. Before running any CRC-related commands, test the connection with:

```bash
ssh <crc_username>@h2p.crc.pitt.edu
```

In which `<crc_username>` should be `.slurm-mgmt/crc-config.json` key `crc_username`. Successful connection should show a welcome message without prompting for a password. If the connection fails, point the user to [the CRC manual page](https://crc-pages.pitt.edu/user-manual/getting-started/passwordless-ssh/). **This step should be entirely manual and you should never automate SSH setup.**

## Run Practices

These practices ensure a reliable workflow. Review and follow them each time carefully before any execution.

### Source Control

1. Only edit and commit code locally. Synchronize changes to the remote CRC environment using the agreed method (`.slurm-mgmt/crc-config.json` with key `sync_method`). Do not make direct edits on the remote system outside of the established workflow.

2. Before submitting any job, synchronize the remote CRC working directory (`.slurm-mgmt/crc-config.json` key `working_dir`) using the agreed method. For Git, ensure both repositories are clean, push locally, synchronize the intended branch with `git pull --ff-only` remotely, and verify matching commit IDs. For rsync, SFTP, or similar transfers, use the agreed paths and exclusions. Treat any consistency concerns as advisory warnings and recommend manual verification when relevant; do not require source snapshots, extra copies, or additional templates solely to verify synchronization.

### Resource Management

1. Run **all experiment computation on compute nodes inside Slurm allocations**, including Python, environment checks, checkpoint loading, diagnostics, validation, training, inference, and data processing. This workflow deliberately keeps these checks inside allocations. Lightweight management commands such as Git synchronization, file management, `sbatch`, `squeue`, `sacct`, and `scontrol` may run on login nodes; do not launch experiment scripts there.

2. Always request **resources and run time** that match the actual needs of the job. Over-requesting can lead to inefficient cluster usage and longer queue times, while under-requesting may cause job failures. Be mindful of GPU allocation and usage. Ensure that the job script correctly specifies the number of GPUs required and that the code utilizes them efficiently.

3. Choose CPU/GPU, memory, and partition for the actual workload and current manual, then the shortest suitable QoS tier that comfortably fits the requested time. GPU jobs must request GPU resources explicitly.

4. Separate **storage duties** between different types of data: use `/ihome/` for code, configuration, and small files. Set `work_storage_dir` to the project's provisioned storage for active datasets, caches, checkpoints, and outputs. Current [CRC storage guidance](https://crc-pages.pitt.edu/user-manual/data-management/file-systems/) directs AI and I/O-intensive work to `/vast/`; use `/ix/` or `/ix1/` for persistent project data and retained results. Confirm the actual project paths and available storage rather than assuming access.

5. Clean up temporary files and intermediate outputs to avoid unnecessary storage consumption on the CRC system.

### Time Management

Requesting appropriate job times for CRC and Slurm jobs is **extremely important**. For **each job**, explain the estimated run duration and requested `--time` briefly before submitting the script. The requested time should be sufficient to complete the job with a short buffer for any delays. 

One approach is to run a small pilot job to measure the execution time of a representative task. Use this measurement to estimate the required time for the full job, adding a buffer for potential delays. **Do not** invent precise time estimates. Explain uncertainty. **Preserve user-specified hard caps for the job even when the estimated run time exceeds them, but explain likely infeasibility.** Queued time is separate from allocated runtime.

## Write the Job Scripts

This section guides writing Slurm job scripts for specific experiments. They only outline general structure and some points, but require careful judgement specific to each experiment task.

1. **Create the job script.** Draft a new Slurm job script as `.slurm-mgmt/runs/<unique-run-id>/job.sbatch`, using [assets/job.sbatch.template](assets/job.sbatch.template). Keep the rendered script and submission record together without overwriting prior runs. Replace all template tokens with project-specific values and review the rendered script for correctness.

2. **Populate the job script.** Fill in the job script with the specific commands for the experiment, including environment setup, data staging, model training, and result handling. Render cluster, partition, CPU, memory, GPU, QoS, and time settings for the actual workload; remove the GPU request and GPU diagnostics for CPU-only jobs. Tasks are processes, CPUs per task support each process, and `--gres=gpu` requests GPUs per node. Source `conda_init`, then activate the `conda_env` directory. Set concrete working and log paths, and choose run-specific output and cache paths under `work_storage_dir`. Adapt the launch section for multi-command or distributed workloads rather than assuming that a larger allocation distributes the program automatically.

3. **Check before submission.** Check for unresolved template tokens, follow the method-specific Source Control guidance above, verify required input/log paths, and run `bash -n job.sbatch` locally. Transfer the rendered script through the agreed mechanism, then use `sbatch --test-only <remote-script-path>` on CRC when supported to validate scheduler requests without submitting a job; this does not test the application. Confirm that the requested time respects the user's hard cap and fits the selected QoS.

4. **Obtain user confirmation.** After finishing checking and validating the job script, obtain **explicit confirmation** from the user to submit with a short overview of the jobs, estimated run times, and requested run time and resources before proceeding with the actual submission.

5. **Submit and record.** Submit the validated remote script with `sbatch --parsable <remote-script-path>`. Record the returned job ID and cluster (separated by `;` when a cluster is returned), submission time, and rendered script in the run record. If submission times out or the response is ambiguous, check for an existing submission before retrying. For monitoring, refer to the "Monitoring" section of this skill document.

### Guidelines

1. The best practice to launch similar jobs is to use a [job array](https://crc-pages.pitt.edu/user-manual/slurm/job-arrays/). Consult the user manual for details.

2. Write shell files with LF endings and UTF-8 without BOM. Set explicit Slurm resources, time, job name, working directory, and output/error paths; use `%j`, or `%A_%a` for arrays. Resolve concrete paths in `#SBATCH` directives locally: shell variables do not expand there.

3. Create directories needed by Slurm's log paths before submission through an available, agreed transfer mechanism or an allocated preparation job. Use explicit log paths under the configured project storage; creating log directories only inside the batch body is too late.

4. Activate the environment explicitly selected for this request inside the allocation, then launch work with `srun`. Guard computation against accidental execution outside a compute allocation. Set the working directory and source search path explicitly when helpers live outside the source tree.

5. Send model checkpoints, dataset/framework caches, and other large active writes to `work_storage_dir`, including applications that otherwise default to the home directory. Use node scratch only with an appropriate copy-back plan to project storage.

6. Stop on environment or workload failures and preserve the workload exit status. Collect `crc-job-stats` as best-effort diagnostics without masking a failed workload or turning successful computation into failure.

7. For job arrays, make the task-to-work mapping explicit, keep output paths disjoint, and set concurrency according to actual resources and group limits. Chain stages with dependencies only when needed.

## Monitoring

After submission, record the job ID and cluster, and verify that the job appears in the queue with the expected resources and parameters. **Do not launch automatic monitoring unless specified by the user.** If monitoring is requested:

1. Monitor job status with Slurm native commands like `sacct`, `squeue`, and `scontrol`, using the recorded cluster. Use a modest polling interval (for example, 60 seconds) and stop on a terminal state or the user's requested stopping point. Download relevant logs and small output/status files for local analysis; retrieve large artifacts only when needed. Record state changes and anomalies. Do not run analysis scripts remotely on either login or compute nodes.

2. If a job fails or behaves unexpectedly, investigate the logs and output files to determine the cause. **Do not make any changes to the source code either local or remote, do not alter the execution environment, and do not relaunch the job independently**. Notify the user immediately with your findings and any relevant evidence.

3. At completion, record the terminal state, exit code, elapsed time, and output locations. Check expected artifacts through lightweight file metadata and local validation as appropriate; a successful submission alone is not a successful experiment. If accounting or output validation is incomplete, report that uncertainty rather than declaring success.


## References

These references provide guidance on running the skills, but are not required for every job submission. Read only when additional context is needed.

- [CRC user manual](https://crc-pages.pitt.edu/user-manual/).
- [SSH terminal access and setup](https://crc-pages.pitt.edu/user-manual/web-portals/terminal/).
- [Batch jobs](https://crc-pages.pitt.edu/user-manual/slurm/batch-jobs/) and [job arrays](https://crc-pages.pitt.edu/user-manual/slurm/job-arrays/).
- [GPU hardware and requests](https://crc-pages.pitt.edu/user-manual/hardware_profiles/gpu/).
- [Job limits and QoS](https://crc-pages.pitt.edu/user-manual/slurm/job-limits/) and [scheduling policy](https://crc-pages.pitt.edu/user-manual/policies/job-scheduling-policy/).
- [File systems](https://crc-pages.pitt.edu/user-manual/data-management/file-systems/).
