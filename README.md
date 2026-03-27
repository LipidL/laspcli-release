
# LASPCLI

`laspcli` is a user-friendly, cross-platform command line tool for the [LASPAI](https://www.laspai.com) cloud computing platform.
It enables efficient access to [LASPAI](https://www.laspai.com) via the command line, providing a seamless experience for users accustomed to terminal workflows.

## Overview

`laspcli` combines a job management system (similar to [Slurm](https://slurm.schedmd.com/overview.html)) with a file synchronization system (like [Git](https://git-scm.com/)).
It allows users to submit jobs to [LASPAI](https://www.laspai.com) from the command line and view results locally.
Key features include:

- Login to [LASPAI](https://www.laspai.com)
- Submit jobs to the platform
- Monitor job progress
- Synchronize results from [LASPAI](https://www.laspai.com) to your local machine for analysis

## Usage

`laspcli` is composed of several subcommands. Typical usage:

```bash
laspcli [subcommand] [arguments]
```

Available subcommands:

- `login`: Manages user authentication
- `queue`: Lists jobs submitted by the user
- `submit`: Submits a job to [LASPAI](https://www.laspai.com)
- `cancel`: Cancels a job on [LASPAI](https://www.laspai.com)
- `update`: Synchronizes job files from [LASPAI](https://www.laspai.com) to the local filesystem
- `config`: Writes default config to the user's config directory

### Login

The `login` subcommand is the foundation for all other functionalities. It helps you log in to [LASPAI](https://www.laspai.com).
When running `laspcli login`, you are prompted for your username and password. Authentication is sent to the server for validation.
After a successful login, authentication information is saved for future commands.
`laspcli` uses the operating system's credential management service (Keyutils on Linux, Windows Credential Manager on Windows and keychain on macOS). If unavailable, credentials are stored in the program's data directory. In this case, users are responsible for keeping the file secure.

### Queue

The `queue` command is similar to `squeue` in [Slurm](https://slurm.schedmd.com/overview.html), displaying the current user's job information.
By default, it shows all running and pending jobs, plus jobs completed within a configurable time range (default: 3 days).
This behavior can be modified in the configuration file (see the config section).

Optional arguments:

- `--all` or `-a`: Print a full list of all jobs, including completed ones

Job information includes:

- Job ID: Unique hexadecimal identifier
- Job name: Human-readable name
- Job status: pending/running/done
- Job progress: Estimated progress
- Job type: Type of job

Printed job IDs highlight the shortest prefix needed to specify the job. For commands requiring a job ID (e.g., `update`, `cancel`), you only need to provide a sufficient prefix as long as the prefix is longer than the highlighted portion. `laspcli` will resolve the ID automatically.

### List

The `list` command lists all availible models for the current user.
Currently, there are two types of models:

- GGNN model, the official model from laspai
- GGNN-FT model, the finetuned model from the finetune job.

For GGNN-FT model, a job ID is associated with it.
You can declare that you want to use your finetuned model when submitting a job by specifying this ID in the configuration file.
See [Configuration file](#configuration-file) for detail.

### Submit

The `submit` command is similar to `sbatch` in [Slurm](https://slurm.schedmd.com/overview.html). It submits a job to [LASPAI](https://www.laspai.com).
You can specify the path to the job directory; if omitted, the current directory is used.
`laspcli` automatically detects the job type and submits it to [LASPAI](https://www.laspai.com).
After submission, a `.lasp` directory is created in the job directory, marking it as a LASPAI job directory. Related information, including the job ID, is stored there. This directory is therefore bonded with the submitted job.

#### Configuration file

The `submit` command tries to find a `laspai.config` file in the job directory.
The file contains necessary configurations for this job.
If the file is not found, `laspcli` automatically writes a default config file.
*DO NOT* modify most of the fields, as it is not yet supported for customization.
We're actively developing the project to support more options.
There are several fields that you can modify.

##### Potential type

To use your customized (finetuned) potential, simply modify the `model_type` field to "GGNN-FT" and `model_version` to the job ID of your finetune (or upload) job.
The availible models and corresponding job IDs can be seen using the `list` command.
See [List](#list) subcommand for details.
Different from the [update](#update) and [cancel](#cancel) command, you must specify *full* job ID in the configuration file.
If you specify a prefix of the ID, it might become ambiguous when you submit more jobs, even if it was the only prefix at the time you write the config file.
As the job ID of the model won't be frequently modified, we decide to force you specifying the full job ID for your convenience in the future.

#### Multiple jobs in one directory

If you try to submit a job at a directory that has been bonded with an existing job, you need to decide what to do with the previous job.
In that case, we will prompt for your choice. Availible choices are:

- Overwrite the existing job metadata. This will unbond the job with the directory, and incremental update of this job's files will be disabled.
- Append to the existing job metadata. This will bond this directory to multile jobs, and fast update (without specifying job ID, see [Update section](#quick-update)) will be disabled.

Generally, we don't recommend bonding multiple jobs with one directory. In cases where you submitted a bad job and want to submit another one with modification, overwriting the existing job metadata should be a good choice.
You can bypass the confirmation process by editing the config file. See [Config section](#config) for more information.

### Cancel

The `cancel` command is similar to `scancel` in [Slurm](https://slurm.schedmd.com/overview.html). It cancels a job on [LASPAI](https://www.laspai.com).
You only need to provide a sufficient prefix of the job ID, not the full ID. The minimum prefix length for a job ID can be found in the output of the `queue` command.

### Update

The `update` command is similar to `git pull`. It synchronizes job files from the server to your local filesystem.
You can specify the directory to synchronize. Optional arguments:

- `--job-id` or `-j`: Specify the job ID to update (prefix is sufficient)
- `--update-input` or `-u`: Also update input files even if the job hasn't completed yet
- `--force` or `-f`: Skip confirm process even if conflict is detected, directly update the files. Existing files might be overwritten

By default, this command skips input files when the job is still running.
After running this command, the directory is automatically linked to the corresponding job ID, as with the `submit` command.

#### Quick update

If `-j` is not specified, `.lasp` is used to detect job information. This is useful for synchronizing a job directory that was just submitted. Note: This shortcut only works if only one job ID is associated with the directory. If multiple jobs are linked (see [Submit section](#multiple-jobs-in-one-directory)), `laspcli` cannot guess which job to update.

#### Incremental Update Strategy

Most files in LASP programs are updated incrementally. `laspcli` adopts an incremental update strategy: when updating previously synchronized files, only the new content is downloaded and appended, saving time and bandwidth.
Incremental updates are limited to append-only files and require that files are not modified after the previous update. If you modify or add lines to an updated file, incremental update is not possible, and a full download/overwrite is required.

#### User Choice & TUI

When `laspcli` detects the need to overwrite local files or if the total download size exceeds a threshold, it asks for your permission.
A terminal user interface (TUI) is launched, showing the default update choice for all files involved. Available choices:

- **Full download** (`[FULL]`): For new files not present in the job directory
- **Incremental update** (`[INCR]`): Most common for files
- **Skip** (`[SKIP]`): For files not modified on the server since the last update, or if you choose to skip
- **Overwrite** (`[OVER]`): Full download and overwrite of the local file

The TUI displays the estimated download size for each file. If a file is too large or not important, you can navigate and use the `[space]` key to change its update strategy. A summary block shows your choices and total download size. After finalizing your choices, press `[q]` or `[ESC]` to quit the TUI and start the download.

#### Backup strategy

When `update` command overwrites an existing file, it automatically backs up the file to `.lasp` directory at job directory. You can find all overwritten files in `.lasp/backup` directory.
Note that this only stores files overwritten in the nearest update. For example, if you run `laspcli update` twice and overwrites `lasp.out` twice, only the `lasp.out` before the last overwrite can be found at `.lasp/backup`.

### Config

Running the `config` command writes the default configuration to your user config directory. You can customize default behaviors by editing this file.

#### Submit config

- `multiple_job_handling`: The default strategy when submitting job at a directory that already has a job bonded. Optional values:
  - "Ask": ask for user confirmation every time (default)
  - "Append": directly append to the job metadata list. Fast update will be disabled, as we can't determine which job you want to update anymore.
  - "Remove": remove the existing job metadata. Previously tracked job metadata will be lost and incremental update of that job is disabled.

#### Update Config

- `confirm_threshold`: Size threshold (MB) for asking user confirmation. If the total update size exceeds this value, the TUI is launched.
- `skip_confirm`: Use default stragety without asking for confirmation. If set to true, `confirm threshold` will be ignored.

#### Queue Config

- `show_threshold`: Time threshold (hours) to display completed jobs. Jobs completed long ago are not shown by default.
