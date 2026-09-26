# FileOps & ProcOps Tools — Linux Project (OS 2026)

This repository contains the incremental implementation of the **FileOps & ProcOps Tools** project for the Operating Systems course (OS 2026). The project consists of a complex set of Linux utilities that combine directory-tree processing and indexing (**FileOps**) with monitoring and control of active processes (**ProcOps**).

The applications are written exclusively in **C** (C11 standard, compiled with `-Wall -Wextra -Werror`), using only standard **POSIX** system calls and the C standard library, with no external dependencies, ensuring maximum portability and performance in Linux environments.

## Project structure and evolution

The project is structured as independent modules for each stage of the lab (T3, T4, T5). Each module introduces a distinct architecture and specific technical complexity:

### Directory organization

For each assignment, the required directory structure is as follows:
- `bin/` — Contains the compiled executables.
- `src/` — C source code (`.c`).
- `include/` — C header files (`.h`).
- `data/` — Persistent data files (binary databases `.db`, the `ipc.mmap` protocol).
- `logs/` — Debug and execution logs.
- `reports/` — Generated text reports (e.g., diffs between databases).
- `tmp/` — Temporary files needed to ensure atomic writes.
- `tests/` — Automated bash test scripts.
- `doc/` — Detailed technical documentation of the binary formats and protocols.
- `tools/` — The `fileops.sh` orchestration script.

## Implementation modules (T3, T4, T5)

### [Assignment T3](./T3/) — Binary databases and concurrent updates (SPMD)
- **fileops_indexer**: Recursively traverses a directory tree and saves the metadata (absolute path, type, size, `mtime`, inode, device, and a deterministic XOR checksum of the content) into a versioned binary database (`data/index.db`).
- **proc_snapshot**: Captures the state of current processes from `/proc` (PID, PPID, state, name, command line, RSS, and CPU time in clock ticks) and writes them into a binary database (`data/proc.db`).
- **Concurrent synchronization (SPMD)**: Multiple instances of the programs can run in parallel, writing to the same database files. Synchronization is implemented directly on the file using exclusive and shared locks via the `fcntl(2)` call (no external lock files).
- **db_diff**: Compares two snapshots (old vs. new) of the same type and generates detailed text reports (`reports/T3_filediff.txt` or `reports/T3_procdiff.txt`) highlighting added, deleted, or significantly modified entries.

*For detailed T3 documentation, see the [T3 README](./T3/README.md).*

### [Assignment T4](./T4/) — Multi-process inventory in C (`fork`/`exec` & `mmap`)
- **Manager-Worker architecture**: A central process (`fileops_manager`) coordinates `N` child processes (`fileops_worker`) started via `fork()` and `exec()`.
- **IPC via shared memory**: Communication between processes happens extremely fast in RAM through a file mapped with `mmap(..., MAP_SHARED, ...)`, which contains:
  - An IPC header with global configuration and state.
  - A circular job queue for the directories to be scanned (supports dynamic jobs added by workers).
  - Circular result channels for file records.
  - A per-worker active statistics area.
- **Synchronization and backpressure**: Synchronization of shared resources is ensured by POSIX semaphores shared between processes (`sem_t` in `mmap`). A backpressure strategy is used to prevent record loss or overwriting of the circular buffers.
- **Atomic writes**: The manager aggregates all results from shared memory and atomically writes the final binary database (`data/inventory.db`) using the temp-file technique (`tmp/data_base_tmp.db`) followed by `rename(2)`. Includes the CLI modes `--verify` and `--dump`.

*For detailed T4 documentation, see the [T4 README](./T4/README.md).*

### [Assignment T5](./T5/) — Control Plane, Signals, and Graceful Shutdown
- **Plane separation**: Completely separates the *Data Plane* (job queue, results in `mmap`) from the *Control Plane* (communication via a unidirectional anonymous pipe from workers to the manager) and the *Signal Plane* (system signals).
- **Pipe protocol (`T5MSG`)**: Workers send short, atomic, asynchronous messages (progress: `JOB_DONE`, completion: `WORKER_EXITING`, or errors: `ERROR`) to the Manager via the pipe. The Manager reads asynchronously in non-blocking mode (`O_NONBLOCK`).
- **Signal handling (Signal Plane)**:
  - `SIGUSR1`: The manager prints a stable, aggregated status line to the console in real time (`STATUS queued_jobs=... active_jobs=...`).
  - `SIGINT` / `SIGTERM`: Initiates a graceful shutdown. The manager stops allocating jobs, notifies the children, grants them a graceful timeout (`--graceful-timeout`), and as a last resort cleans up the processes via `SIGKILL`.
  - `SIGCHLD`: The manager asynchronously collects worker statuses via `waitpid()` to prevent zombie processes.
- **Incomplete DB semantics**: If the inventory is interrupted in a controlled way by the user via signals, the manager ensures a structurally valid database is written, but explicitly marked in the header with the `complete=0` flag.

*For detailed T5 documentation, see the [T5 README](./T5/README.md).*

## Build, run, and test

Build, run, and test orchestration is done uniformly through the centralizing script `./tools/fileops.sh`.

### 1. Initialize the structure and compile the sources
```bash
# Create the required directories
./tools/fileops.sh init

# Compile the C source code with -Wall -Wextra -Werror -std=c11
./tools/fileops.sh build
```

### 2. Running the utilities (examples)

**Concurrent SPMD mode (T3):**
```bash
# Start indexing a directory (multiple concurrent instances can run on the same DB)
./tools/fileops.sh run -- fileops_indexer --root /path/to/dir --db data/index.db

# Capture a process snapshot from /proc
./tools/fileops.sh run -- proc_snapshot --db data/proc.db

# Compare two snapshots
./tools/fileops.sh run -- db_diff --old data/index_old.db --new data/index_new.db --out reports/T3_filediff.txt
```

**Multi-process Manager-Worker mode (T4 & T5):**
```bash
# Start the inventory manager with 4 workers and a 5-second timeout
./tools/fileops.sh run -- fileops_manager --root /path/to/dir --workers 4 --ipc data/ipc.mmap --db data/inventory.db --graceful-timeout 5 --pid-file tmp/manager.pid
```

**Verifying and dumping the databases:**
```bash
# Validate the structural integrity of the database
./tools/fileops.sh run -- fileops_manager --db data/inventory.db --verify

# Print the database metadata as key=value pairs
./tools/fileops.sh run -- fileops_manager --db data/inventory.db --dump
```

### 3. Running the automated tests
Each module contains non-interactive tests meant to validate complex scenarios of concurrent execution, mmap synchronization, and signal handling.
```bash
./tools/fileops.sh test
```

## Detailed technical documentation

For in-depth technical details at the protocol and binary-format level, please refer to the files in the documentation directories:
- [T3/doc/Format_DB.md](./T3/doc/Format_DB.md) — The exact structure of the headers and records for the T3 databases (`index.db` and `proc.db`).
- [T4/doc/MMAP_PROTOCOL.md](./T4/doc/MMAP_PROTOCOL.md) — The detailed shared-memory layout and the structure of the circular job and result queues in T4.
- [T4/doc/T4_DB_FORMAT.md](./T4/doc/T4_DB_FORMAT.md) — The binary format of the final `inventory.db` database produced by the manager.
- [T5/doc/T5_CONTROL_PLANE.md](./T5/doc/T5_CONTROL_PLANE.md) — The structure of the control-plane pipe channels, the `T5MSG` message format, and the asynchronous signaling mechanisms.

*Project developed as part of the Operating Systems lab, 2026.*
