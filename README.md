# Debugging Lab

A hands-on practice lab designed to trace crash points, analyze root causes, and fix C memory bugs using GDB and logs. It consists of 20 standalone source files organized by difficulty, each intentionally injected with exactly one representative memory bug.

Every buggy program in this lab is guaranteed to crash (`SIGSEGV`, `SIGABRT`, or `SIGBUS`) upon execution.

**Project Structure**

memory-debugging-lab/
├── README.md
├── Makefile                     # Build/run/debug helper (GDB focused)
├── Dockerfile                   # Linux lab environment with GCC and GDB
├── .devcontainer/               # VS Code Dev Containers configuration
├── .vscode/                     # launch.json / tasks.json (F5 debugging support)
├── scripts/check.sh             # Run all challenges and summarize crash statuses
├── challenges/                  # Buggy source codes (analyze root causes here)
│   ├── 01_use_after_free/bug.c
│   ├── ...
│   └── 20_vector_stale_pointer/bug.c

Each bug.c contains header comments detailing the scenario, expected behavior, symptoms, how to debug with GDB, and how to debug with printf.

If you are confident in your debugging skills, it is recommended to solve and fix the bugs directly without reading the top comments.

Note: The solutions/ directory is excluded via .gitignore and is not included in the student distribution (maintained locally by coaches only). Therefore, solution-related commands such as make solutions and make check-all only function in the coach environment.

**Environment Setup**
Linux, macOS — Practice using Docker or Dev Containers.

Windows — Ensure your Docker backend is set to WSL 2. (Docker Desktop → Settings → General → Check "Use the WSL 2 based engine"). Running an Ubuntu + glibc container on top of WSL 2 (Linux kernel) ensures that crash reproduction behaves identically to Linux. While the Hyper-V backend also runs a Linux VM and works, running in "Windows containers" mode is strictly unsupported.

Crash types (stack smashing, glibc double-free, invalid pointer detection, etc.) are standardized around Linux glibc. Because the crash signals and behaviors may vary across different libc implementations or operating systems, verification should always be done inside Docker (Ubuntu).

Reference Environment (Inside Container): Ubuntu 24.04 LTS · GCC 13 · glibc 2.39 · GDB 15. (Host OS or distribution does not matter—it runs within this exact toolchain inside the container.)

**Practice with Docker (GCC + GDB)**
docker build -t memdbg .
docker run --rm -it --cap-add=SYS_PTRACE --security-opt seccomp=unconfined \
    -v "$PWD":/work memdbg

# ── Inside the container ──
make check                                  # Run all 20 challenges and summarize crash results
gdb ./build/06_null_deref                   # run → bt → frame N → print <variable>

## VS Code Integration (Mac / Windows) — Dev Containers

You can attach VS Code directly to the Docker container (Linux toolchain) to use GDB with a GUI on macOS or Windows. This repository includes `.devcontainer/devcontainer.json`, `.vscode/launch.json`, and `.vscode/tasks.json`.

### Prerequisites (Common)

* Install and run **Docker Desktop**.
* **Windows users must use the WSL 2 backend:** Check `Docker Desktop` → `Settings` → `General` → **"Use the WSL 2 based engine"**. (This lab is designed specifically for Linux containers; running on top of WSL 2 ensures that crash behavior matches native Linux.)
* Install **VS Code** along with the **Dev Containers** extension (`ms-vscode-remote.remote-containers`).

### Method A — Reopen in Container (Recommended, One-Click)

1. Open this project folder in VS Code.
2. Click **"Reopen in Container"** in the bottom-right notification popup (or press `F1` → run `Dev Containers: Reopen in Container`).
3. On first launch, the image will build (1–3 minutes), and VS Code will attach to Linux inside the container.
4. Run commands directly in the integrated terminal:
```bash
make check                                  # Summarize crash statuses across all 20 challenges
gdb ./build/06_null_deref                   # run → bt

```


5. **GUI Debugging with F5:** Select a challenge configuration to inspect breakpoints, backtraces, and variables visually with GDB.

### Method B — Attach to a Running Container

```bash
cd <path-to-this-project>
docker build -t memdbg .
docker run -d --name memdbg-dev --cap-add=SYS_PTRACE \
  --security-opt seccomp=unconfined -v "$PWD":/work -w /work memdbg sleep infinity

```

In VS Code: Press `F1` → `Dev Containers: Attach to Running Container` → select `memdbg-dev` → open folder `/work`.

*(For Windows PowerShell, replace `"$PWD"` with `${PWD}`.)*

### Important Notes

* The project directory is two-way mounted to `/work` inside the container (edits made on macOS/Windows reflect immediately inside the container).
* Executable binaries inside `build/` are compiled for Linux. **Do not run `make` directly on the host machine**; always build and run inside the container terminal.
* The flags `--cap-add=SYS_PTRACE` and `--security-opt seccomp=unconfined` configured in `runArgs` are mandatory for GDB debugging to function properly inside Docker.
* Apple Silicon Macs will run an `arm64` Linux container, where GCC and GDB function identically.

---

## Build & Run Summary

```bash
make list                     # List all challenges
make all                      # Build all bug.c files without sanitizers → build/<name>
make run NAME=01_use_after_free     # Run the specified challenge (crashes if buggy)
make gdb NAME=06_null_deref         # Debug with GDB (run → bt)
make check                    # Run all challenges and summarize crash results
make clean                    # Clean up the build/ directory

# ── Coach / Local only (requires solutions/ directory) ──
make solutions                # Build solution codes → build/sol_<name>
make check-all                # Verify both challenges (crash) and solutions (exit code 0)

```

Binaries are compiled with `-g -O0 -fno-omit-frame-pointer`, ensuring GDB backtraces display precise source line numbers and variable information.

---

## Challenge List (20 Challenges)

| # | Name | Bug Type | Crash Signal Seen in GDB |
| --- | --- | --- | --- |
| 01 | `use_after_free` | vtable widget: Slot not cleared after free → function pointer invocation | `SIGSEGV` |
| 02 | `stack_buffer_overflow` | Triangular indexing off-by-one overflowing stack array | `SIGABRT` (stack smashing) |
| 03 | `heap_buffer_overflow` | Dynamic array growth bug (capacity vs. allocation mismatch) | `SIGABRT` (realloc detection) |
| 04 | `double_free` | Freeing the same object twice via aliased index pointers | `SIGABRT` (double free) |
| 05 | `null_return_deref` | Dereferencing `NULL` returned for a missing key in config expansion | `SIGSEGV` |
| 06 | `null_deref` | Header parser: Writing to `strchr` `NULL` return on lines missing `:` | `SIGSEGV` |
| 07 | `stack_use_after_return` | Local array pointer escaping to a view struct → dereference after frame reuse | `SIGSEGV` |
| 08 | `uninitialized_read` | Dereferencing uninitialized row pointer due to dirty heap reuse | `SIGSEGV` |
| 09 | `strcpy_overflow` | Join size calculation off-by-one (omitted trailing piece) | `SIGSEGV` |
| 10 | `realloc_dangling` | Undo snapshot pointer invalidated by `realloc` move → double free | `SIGABRT` (double free) |
| 11 | `global_overflow` | Unchecked bounds in global arena bump allocator | `SIGSEGV` |
| 12 | `free_non_heap` | Calling `free()` directly on an interior pointer in CSV fields | `SIGABRT` (invalid pointer) |
| 13 | `linked_list_uaf` | Job queue filter: Reading `node->next` after `free(node)` (UAF) | `SIGSEGV` |
| 14 | `integer_overflow_alloc` | Image `w * h * ch` multiplication overflow → under-allocation | `SIGSEGV` |
| 15 | `dangling_in_struct` | Session invokes callback on a freed User struct | `SIGBUS` / `SIGSEGV` |
| 16 | `unused_cap_overflow` | Append function ignores `capacity` parameter, overflowing buffer | `SIGABRT` (stack smashing) |
| 17 | `ownership_uaf` | Message broker: Consumer frees message, audit log double frees (UAF) | `SIGSEGV` |
| 18 | `cleanup_double_free` | Multi-resource `goto` ladder: Transaction double freed on validation failure path | `SIGABRT` (double free) |
| 19 | `realloc_shrink_overflow` | Signal buffer trimmed with `realloc`, but iterated using old `len` | `SIGSEGV` |
| 20 | `vector_stale_pointer` | Histogram hot pointer becomes stale after vector capacity reallocation | `SIGSEGV` |

*Verification:* Running `make check` inside the Linux (Docker) environment should result in all 20 challenges crashing.

---

## GDB Cheat Sheet

```bash
gdb ./build/06_null_deref        # Start debugger
(gdb) run                        # Execute program → crashes here
(gdb) bt                         # Backtrace: View call stack and crash location
(gdb) frame 1                    # Switch to stack frame 1
(gdb) print var                  # Inspect variable/pointer value (e.g., print p, print i)
(gdb) info locals                # List all local variables in the current frame
(gdb) list                       # View source code surrounding the crash location

```

Useful commands for tracing memory bugs:

```bash
(gdb) break file:line            # Set a breakpoint at a specific line
(gdb) watch var                  # Stop execution when the variable value changes
(gdb) x/8xg ptr                  # Dump 8 quadwords (64-bit words) of memory in hex
(gdb) p (long)ptr - (long)base   # Calculate offset between two pointers (bounds check)

```

---

## Two Approaches: GDB vs. `printf` (Logging)

The header comments in each `bug.c` contain both **[Debugging with GDB]** and **[Debugging with `printf`]** guides. We encourage trying both methods to compare their strengths.

| Approach | Method | Advantages | Caveats |
| --- | --- | --- | --- |
| **GDB** | Post-mortem trace: `run` → `bt` to trace the crash location backwards | Immediate inspection of variables/memory without recompilation or source changes | When crashing inside `libc`, you must navigate up stack frames (`up` / `frame`) to your code |
| **`printf` (Logs)** | Instrument code with print statements to trace variable transitions | Clear chronological view of control flow; simple conditional inspection | `stdout` is buffered and logs may be lost upon crashing |

> **Key Rule for `printf` Debugging:**
> To ensure logs printed immediately before a crash are not lost due to buffering, output to `stderr` using `fprintf(stderr, ...)`. If using `stdout`, flush the stream after each print (`fflush(stdout);`) or disable buffering entirely via `setvbuf(stdout, NULL, _IONBF, 0);`.

---

## Recommended Learning Workflow

1. Read `challenges/<name>/bug.c` and predict the symptoms and likely crash point.
2. Run `make gdb NAME=<name>`, trigger the crash with `run`, and locate the crash site with `bt`.
3. Inspect pointers, indices, and sizes using `print`, `info locals`, and `x` to pinpoint the root cause.
4. *(Optional / In addition)* Add `fprintf(stderr, ...)` logs following the **[Debugging with `printf`]** guide to trace values chronologically and see where states diverge.
5. Modify the code to eliminate the root cause, then verify with `make run NAME=<name>` that the program exits cleanly with return code `0`. *(Standard solutions are provided by the instructor.)*
