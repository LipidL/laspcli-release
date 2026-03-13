
# LASP_CLI

`lasp_cli` is a user-friendly, cross-platform command line tool for the [LASPAI](https://www.laspai.com) cloud computing platform.
It enables efficient access to [LASPAI](https://www.laspai.com) via the command line, providing a seamless experience for users accustomed to terminal workflows.

## Overview

`lasp_cli` combines a job management system (similar to [Slurm](https://slurm.schedmd.com/overview.html)) with a file synchronization system (like [Git](https://git-scm.com/)).
It allows users to submit jobs to [LASPAI](https://www.laspai.com) from the command line and view results locally.
Key features include:

- Login to [LASPAI](https://www.laspai.com)
- Submit jobs to the platform
- Monitor job progress
- Synchronize results from [LASPAI](https://www.laspai.com) to your local machine for analysis

## Usage

`lasp_cli` is composed of several subcommands. Typical usage:

```bash
lasp_cli [subcommand] [arguments]
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
When running `lasp_cli login`, you are prompted for your username and password. Authentication is sent to the server for validation.
After a successful login, authentication information is saved for future commands.
`lasp_cli` uses the operating system's credential management service (Keyutils on Linux, Windows Credential Manager on Windows and keychain on macOS). If unavailable, credentials are stored in the program's data directory. In this case, users are responsible for keeping the file secure.

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

Printed job IDs highlight the shortest prefix needed to specify the job. For commands requiring a job ID (e.g., `update`, `cancel`), you only need to provide a sufficient prefix as long as the prefix is longer than the highlighted portion. `lasp_cli` will resolve the ID automatically.

### Submit

The `submit` command is similar to `sbatch` in [Slurm](https://slurm.schedmd.com/overview.html). It submits a job to [LASPAI](https://www.laspai.com).
You can specify the path to the job directory; if omitted, the current directory is used.
`lasp_cli` automatically detects the job type and submits it to [LASPAI](https://www.laspai.com).
After submission, a `.lasp` directory is created in the job directory, marking it as a LASPAI job directory. Related information, including the job ID, is stored there.

### Cancel

The `cancel` command is similar to `scancel` in [Slurm](https://slurm.schedmd.com/overview.html). It cancels a job on [LASPAI](https://www.laspai.com).
You only need to provide a sufficient prefix of the job ID, not the full ID. The minimum prefix length for a job ID can be found in the output of the `queue` command.

### Update

The `update` command is similar to `git pull`. It synchronizes job files from the server to your local filesystem.
You can specify the directory to synchronize. Optional arguments:

- `--job-id` or `-j`: Specify the job ID to update (prefix is sufficient)

After running, the directory is automatically linked to the corresponding job ID, as with the `submit` command.
If `-j` is not specified, `.lasp` is used to detect job information. This is useful for synchronizing a job directory that was just submitted. Note: This shortcut only works if only one job ID is associated with the directory. If multiple jobs are linked, `lasp_cli` cannot guess which job to update.

#### Incremental Update Strategy

Most files in LASP programs are updated incrementally. `lasp_cli` adopts an incremental update strategy: when updating previously synchronized files, only the new content is downloaded and appended, saving time and bandwidth.
Incremental updates are limited to append-only files and require that files are not modified after the previous update. If you modify or add lines to an updated file, incremental update is not possible, and a full download/overwrite is required.

#### User Choice & TUI

When `lasp_cli` detects the need to overwrite local files or if the total download size exceeds a threshold, it asks for your permission.
A terminal user interface (TUI) is launched, showing the default update choice for all files involved. Available choices:

- **Full download** (`[FULL]`): For new files not present in the job directory
- **Incremental update** (`[INCR]`): Most common for files
- **Skip** (`[SKIP]`): For files not modified on the server since the last update, or if you choose to skip
- **Overwrite** (`[OVER]`): Full download and overwrite of the local file

The TUI displays the estimated download size for each file. If a file is too large or not important, you can navigate and use the `[space]` key to change its update strategy. A summary block shows your choices and total download size. After finalizing your choices, press `[q]` or `[ESC]` to quit the TUI and start the download.

### Config

Running the `config` command writes the default configuration to your user config directory. You can customize default behaviors by editing this file.

#### Update Config

- `confirm_threshold`: Size threshold (MB) for asking user confirmation. If the total update size exceeds this value, the TUI is launched.
- `cdn_threshold`: Size threshold (MB) for using CDN download. CDN provides higher speed but disables incremental updates and file selection.

#### Queue Config

- `show_threshold`: Time threshold (hours) to display completed jobs. Jobs completed long ago are not shown by default.
