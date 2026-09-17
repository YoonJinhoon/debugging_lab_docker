Debugging Lab

A hands-on lab for debugging C memory bugs using gdb and logs: trace back from the crash point → analyze the cause → fix it.

There are 20 independently executable challenges, organized by difficulty. Each challenge contains exactly one representative memory bug.

All bug programs in this lab are designed to always crash when executed (SIGSEGV / SIGABRT / SIGBUS).

Structure
memory-debugging-lab/
├── README.md
├── Makefile                     # Build/run/debugging helpers (gdb only)
├── Dockerfile                   # gcc/gdb Linux lab environment
├── .devcontainer/               # VS Code Dev Containers configuration
├── .vscode/                     # launch.json / tasks.json (F5 debugging)
├── scripts/check.sh             # Run all challenges → summarize whether they crash
├── challenges/                  # Buggy code (analyze the cause here)
│   ├── 01_use_after_free/bug.c
│   ├── ...
│   └── 20_vector_stale_pointer/bug.c


Each bug.c has comments at the top describing the scenario / expected behavior / symptoms / how to catch it with gdb / how to catch it with printf.

If you are confident in your skills, it is recommended that you try to fix the problem yourself without referring to the comments at the top.

Note: solutions/ is excluded by .gitignore and is not included in the distributed version for students (it is maintained locally by the instructor only). Therefore, solution-related commands such as make solutions and make check-all work only in the instructor's environment.

Environment

Linux, macOS — The lab is run using Docker / Dev Containers.

Windows — Make sure the Docker backend is WSL2. (Docker Desktop → Settings → General → Check "Use the WSL 2 based engine") An Ubuntu + glibc container must run on the WSL2 (Linux) kernel so that crashes are reproduced the same way as on Linux. The Hyper-V backend also works because it runs a Linux VM, but do not run Docker Desktop in Windows container mode.

Crash types (stack smashing, glibc double-free/invalid-pointer detection, etc.) are based on Linux glibc.
The same code may produce different crash signals on other libc implementations / operating systems, so use Docker (Ubuntu) for verification.

Verified environment (container): Ubuntu 24.04 LTS · gcc 13 · glibc 2.39 · gdb 15. (The host OS/distribution does not matter — the code runs inside the container using this toolchain.)

Run the Lab with Docker (gcc + gdb)
docker build -t memdbg .
docker run --rm -it --cap-add=SYS_PTRACE --security-opt seccomp=unconfined \
    -v "$PWD":/work memdbg
# ── Inside the container ──
make check                                  # Run all 20 → summarize whether they crash
gdb ./build/06_null_deref                   # run → bt → frame N → print variable


--cap-add=SYS_PTRACE and --security-opt seccomp=unconfined allow gdb
to attach to processes inside the container.

VS Code Integration (Mac / Windows) — Dev Containers

By connecting VS Code to the Docker container (Linux toolchain), you can use gdb through a GUI even on Mac/Windows.

This repository includes .devcontainer/devcontainer.json, .vscode/launch.json, and .vscode/tasks.json.

Prerequisites (Common)

Install and run Docker Desktop

Windows must use the WSL2 backend — Docker Desktop → Settings → General → Make sure "Use the WSL 2 based engine" is enabled. (This lab is for Linux containers only, and running on WSL2 ensures that crashes are reproduced the same way as on Linux.)

VS Code + the "Dev Containers" extension (ms-vscode-remote.remote-containers)

Method A — Reopen in Container (Recommended, One Click)

Open the project folder in VS Code

Click "Reopen in Container" in the bottom-right corner (if it does not appear, press F1 → Dev Containers: Reopen in Container)

On the first run, the image is built (1–3 minutes), then VS Code connects to Linux inside the container

Run the lab directly from the integrated terminal:

make check                                  # Summarize whether all 20 crash
gdb ./build/06_null_deref                   # run → bt


Press F5 for GUI debugging: select a challenge to inspect breakpoints, backtraces, and variables with gdb.

Method B — Attach to a Running Container
cd <this project folder>
docker build -t memdbg .
docker run -d --name memdbg-dev --cap-add=SYS_PTRACE \
  --security-opt seccomp=unconfined -v "$PWD":/work -w /work memdbg sleep infinity


In VS Code: F1 → Dev Containers: Attach to Running Container → memdbg-dev → Open the folder /work

In Windows PowerShell, replace "$PWD" with ${PWD}.

Things to Keep in Mind

The folder is bidirectionally mounted at /work inside the container (edits made on Mac/Windows are immediately reflected in the container).

build/ contains Linux binaries, so do not run make directly on the host. Build and run from the container terminal.

The --cap-add=SYS_PTRACE and --security-opt seccomp=unconfined options in runArgs are required for gdb debugging to work properly inside the container.

Apple Silicon Macs run an arm64 Linux container, and gcc/gdb work normally in that environment as well.

Build / Run Summary
make list                     # List challenges
make all                      # Build all bug.c files without sanitizers → build/<name>
make run  NAME=01_use_after_free     # Run a challenge (crashes if the bug is present)
make gdb  NAME=06_null_deref         # Debug with gdb (run → bt)
make check                    # Run all challenges → summarize whether they crash
make clean                    # Clean build/

# ── The following are instructor/local-only (work only when solutions/ exists) ──
make solutions                # Build solution code → build/sol_<name>
make check-all                # Verify both challenges (crash) + solutions (normal exit 0)


The code is built with -g -O0 -fno-omit-frame-pointer, so gdb backtraces include accurate source lines and variable information.

Challenge List (20)
#	Name	Bug Type	Crash Seen in gdb
01	use_after_free	vtable widget: freed slot not cleared → function pointer call	SIGSEGV
02	stack_buffer_overflow	Stack array overflow caused by triangular indexing off-by-one	SIGABRT (stack smashing)
03	heap_buffer_overflow	Dynamic array growth bug (capacity vs. actual buffer mismatch)	SIGABRT (realloc detection)
04	double_free	Same object freed from two aliased indices	SIGABRT (double free)
05	null_return_deref	NULL returned for a missing key during config template expansion, then dereferenced	SIGSEGV
06	null_deref	Header parser: line without : → write through a NULL pointer returned by strchr	SIGSEGV
07	stack_use_after_return	Local array address escapes as a view → dereferenced after frame reuse	SIGSEGV
08	uninitialized_read	Dereferencing an uninitialized row pointer after dirty heap reuse	SIGSEGV
09	strcpy_overflow	Off-by-one in join size calculation (last fragment omitted)	SIGSEGV
10	realloc_dangling	Undo snapshot becomes dangling after realloc moves the buffer → double free	SIGABRT (double free)
11	global_overflow	Bounds check missing in global arena bump allocator	SIGSEGV
12	free_non_heap	Individually freeing CSV fields (internal pointers)	SIGABRT (invalid ptr)
13	linked_list_uaf	Job queue filter: read next after free (UAF)	SIGSEGV
14	integer_overflow_alloc	Integer overflow in image w*h*ch multiplication → under-allocation	SIGSEGV
15	dangling_in_struct	Session invokes a callback belonging to a freed User	SIGBUS/SIGSEGV
16	unused_cap_overflow	append ignores the cap argument, causing an overflow	SIGABRT (stack smashing)
17	ownership_uaf	Message broker: consumer frees object + audit log frees it again (UAF)	SIGSEGV
18	cleanup_double_free	Multi-resource goto ladder: validation failure path frees tx twice	SIGABRT (double free)
19	realloc_shrink_overflow	Iterate using the old len after trimming a signal buffer	SIGSEGV
20	vector_stale_pointer	Histogram hot pointer becomes stale after vector growth	SIGSEGV

Verification: When running make check on Linux (Docker), all 20 challenges crash.

gdb Cheat Sheet
gdb ./build/06_null_deref        # Start the debugger
(gdb) run                        # Run → the bug code crashes here
(gdb) bt                         # Backtrace: find the function/line where it crashed
(gdb) frame 1                    # Move to a specific stack frame
(gdb) print variable             # Inspect variable/pointer values (e.g. print p, print i)
(gdb) info locals                # Show all local variables in the current frame
(gdb) list                       # Show source around the crash point


Commands useful for tracking memory bugs:

(gdb) break file:line            # Set a breakpoint at a specific line
(gdb) watch variable             # Stop when the value changes
(gdb) x/8xg pointer              # Dump 8 words of hexadecimal memory at the pointer
(gdb) p (long)pointer - (long)base # Calculate the offset between two pointers (to determine out-of-bounds access)

Two Approaches: gdb vs. printf (Logging)

The comments at the top of each bug.c include both [Catch with gdb] and [Catch with printf (logging)] approaches. It is recommended to try both and compare them.

	gdb	printf (logging)
Method	Trace the location after a crash with run → bt	Add logs to the code and track value changes
Advantage	Immediate debugging without recompiling or modifying code; powerful variable/memory inspection	Observe the entire flow chronologically; easy conditional logging
Caveat	If the crash occurs inside libc, use frame/up to move back to your own code	stdout is buffered, so logs may be lost when the program crashes

Key point when using printf for debugging: to prevent the log immediately before the crash from disappearing, print to stderr (fprintf(stderr, ...)), or if you use stdout, call fflush(stdout) after each output or disable buffering with setvbuf(stdout, NULL, _IONBF, 0).

Recommended Learning Flow

Read challenges/<name>/bug.c and predict the symptom and where it will crash.

Run make gdb NAME=<name> → run to trigger the crash, then use bt to find the crash location.

Use print / info locals / x to inspect pointers, indices, and sizes and identify the cause.

(Alternatively or additionally) Following [Catch with printf (logging)] in the comments, add fprintf(stderr, ...) logs and observe chronologically where the values become inconsistent.

Modify the code to eliminate the cause, then run make run NAME=<name> to verify that the crash is gone and the program exits normally (0). (The standard solution is provided by the instructor.)

Notice

Questions about the development environment are not supported. Configure the environment to suit your own setup based on the code and documentation provided.
