# pwn.college -- Program Interaction: Linux Process Execution

**Author:** Ramji R  
**Source:** [pwn.college - Program Interaction - Linux Process Execution (YouTube)](https://www.youtube.com/watch?v=Vtb5wIlthRg)  
**Lecture Series:** Program Interaction (Lecture 4 of 4)  
**Instructor:** Yan Shoshitaishvili (Zardus), Arizona State University  

---

## 1. Recap -- Where We Left Off

- Previously covered: how a program (ELF binary on disk) gets loaded into the virtual memory space of a process
- This lecture picks up from there -- execution has begun, now what?

---

## 2. Execution Begins -- `__libc_start_main`

- After the loader finishes mapping libraries and setting up memory, it jumps to the **entry point** of the binary
- You can find the entry point using `readelf`:

```bash
readelf -h ./cat
# Look for "Entry point address" -- e.g. 0x1oc0
```

- That entry point is `_start`, NOT `main`
- `_start` does some setup and then calls `__libc_start_main` -- a libc function
- `__libc_start_main` is what actually calls your `main()`

### `__libc_start_main` signature

```c
int __libc_start_main(
    int (*main)(int, char**, char**),  // function pointer to main
    int argc,
    char **argv,
    void (*init)(void),                // constructor
    void (*fini)(void),                // destructor
    void (*rtld_fini)(void),           // dynamic linker cleanup
    void *stack_end
);
```

- Takes a **function pointer to main**, argc, argv, and several function pointers for constructors/destructors
- You can LD_PRELOAD override `__libc_start_main` just like any other library symbol

### Demo -- LD_PRELOAD override of `__libc_start_main`

```bash
# Compile a custom __libc_start_main that just prints hello and exits
gcc -shared -fPIC start_main.c -o start_main.so

# Preload it
LD_PRELOAD=./start_main.so cat cat.c
# Output: Hello  (your override ran instead of real libc_start_main)
```

- This shows even the most fundamental startup mechanism can be hooked
- `objdump -M intel -d ./cat` -- disassemble and look for the `call __libc_start_main` at `_start`

---

## 3. Environment Variables

- After launch, the program reads its **arguments and environment from memory**
- The only inputs from the outside world at program start are `argv` and `envp`
- Everything else is just binary code + libraries loaded in memory

### What is an environment variable?

- A set of `KEY=VALUE` string pairs passed into every process at launch
- Separate from `argv` -- they live in the `envp` array
- List them all with the `env` command

### Accessing `envp` in C

```c
int main(int argc, char *argv[], char *envp[]) {
    for (int i = 0; envp[i] != NULL; i++) {
        printf("%s\n", envp[i]);
    }
}
```

- Iterate through `envp[]` and print each entry until you hit a `NULL` pointer -- that's the terminator

### Setting env vars from shell

```bash
# Pass a custom env var inline
MYVAR=hello ./program

# Multiple vars
VAR1=a VAR2=b ./program

# View all env vars
env
```

### Real-world impact of env vars

- `LANG` environment variable controls how `ls` sorts filenames
  - Default: locale-aware sorting (mixes upper and lowercase, looks "natural")
  - `LANG=C ls` -- forces raw ASCII byte-order sorting (uppercase before lowercase)
- `LD_PRELOAD` -- used throughout this lecture series to hook library functions
- Env vars are a clean way to pass config to LD_PRELOAD debug libraries without touching `argv`

---

## 4. Symbols -- Library Function Resolution

- Programs use library functions (e.g. `printf`, `open`, `read`, `write`) that live in shared libs like libc
- These are called **symbols** -- named references to functions or globals in a binary

### Historical context -- lazy binding (PLT/GOT)

- Used to be: symbols were resolved **on demand** (lazy binding)
  - First call to `printf` would trigger a lookup to find `printf` inside libc
  - Saved startup time on slow machines with programs using thousands of symbols
- Now: symbols are **resolved at load time** and the GOT is marked read-only (`RELRO`)
  - Too many security exploits abused lazy binding (GOT overwrite attacks)
  - Will revisit this in the memory corruption module

### Inspecting symbols with `nm`

```bash
# Symbols cat imports (undefined = imported from a library)
nm ./cat | grep ' U '

# Symbols cat exports (defined in the binary)
nm ./cat | grep ' T '

# Examples:
# U __libc_start_main  -- imported
# T main               -- exported (defined in binary)
# T _start             -- exported entry point
```

### Inspecting with `objdump`

```bash
objdump -M intel -d ./cat   # disassemble, Intel syntax
# See call to __libc_start_main inside _start
# See call to open inside main (via libc wrapper)
```

- Nowadays dynamically linked binaries are technically libraries themselves -- you could `LD_PRELOAD` a regular ELF binary (though it may cause chaos if it exports `main`)

---

## 5. System Calls

- Library functions are one layer. Underneath them, **all real interaction with the OS happens through system calls**
- Every file open, network connection, process spawn -- all initiated through syscalls
- libc provides C-function wrappers (e.g. `open()`, `read()`, `write()`) that internally trigger the actual syscall

### Tracing syscalls with `strace`

```bash
strace cat cat.c
# Shows every syscall from execve at start through:
#   - library mapping (mmap, mprotect, etc.)
#   - openat() to open the file
#   - read() -- reads 170 bytes (exact file size)
#   - write() -- writes 170 bytes to stdout
#   - read() -- returns 0 (EOF)
#   - exit_group()

# Redirect strace output (goes to stderr) separately
strace cat cat.c 2>trace.txt         # see just cat output
strace cat cat.c 1>/dev/null         # see just strace output
```

### Syscall numbers

- Every syscall has a **number** used to identify it to the kernel
- Common ones on x86-64:
  - `0` = `read`
  - `1` = `write`
  - `2` = `open`
  - `60` = `exit`
  - `57` = `fork`
  - `59` = `execve`

### Calling syscalls directly in C

```c
#include <unistd.h>

// Instead of: read(fd, buf, count)
syscall(0, fd, buf, count);   // syscall number 0 = read

// Instead of: write(fd, buf, count)
syscall(1, fd, buf, count);   // syscall number 1 = write
```

- `strace` output looks identical whether you use `read()` or `syscall(0, ...)`
- This is one step closer to writing syscalls manually in assembly (as done in shellcoding)

### Key resource

- **Ryan A. Chapman's Linux syscall table** -- google "ryan chapman x86 syscall table"
- Lists every syscall number with its name, arguments, and return value
- Essential reference for shellcoding

### `man syscalls` -- documentation

```bash
man syscalls         # overview and calling conventions
man 2 open           # specific syscall documentation
man 2 read
man 2 execve
```

### Common syscalls you'll use

| Syscall   | Purpose                                      |
|-----------|----------------------------------------------|
| `open`    | Open a file, get a file descriptor           |
| `read`    | Read bytes from a file descriptor            |
| `write`   | Write bytes to a file descriptor             |
| `fork`    | Duplicate current process                    |
| `clone`   | Create a process/thread with more control    |
| `execve`  | Replace process image with a new program     |
| `wait`    | Wait for a child process to terminate        |
| `exit`    | Terminate the current process                |
| `ptrace`  | Attach to a process as a debugger            |
| `socket`  | Open a network connection                    |

- General rule: **to do anything in Linux you open some resource and interact with it via read/write (and sometimes other syscalls)**

---

## 6. Signals

- Syscalls let the process talk to the OS -- but how does the OS (or another process) talk to YOUR process?
- Answer: **signals**
- A signal is an asynchronous notification sent to a process
- You can define **signal handlers** -- functions that run when your process receives a signal

### Default actions

- If no handler is defined, the default action is taken:
  - Some signals: **ignore**
  - Most signals: **kill the process**

### Uncatchable signals

- **SIGKILL (signal 9)** -- cannot be caught, blocked, or ignored. `kill -9 <pid>`
- **SIGSTOP (signal 19)** -- cannot be caught. Suspends the process unconditionally

### Common signals

| Signal    | Number | Default  | Triggered by                        |
|-----------|--------|----------|-------------------------------------|
| SIGHUP    | 1      | Terminate| Terminal hangup                     |
| SIGINT    | 2      | Terminate| Ctrl+C in terminal                  |
| SIGQUIT   | 3      | Core dump| Ctrl+\ in terminal                  |
| SIGSEGV   | 11     | Core dump| Segmentation fault (bad memory access)|
| SIGALRM   | 14     | Terminate| Timer set by `alarm()` expired      |
| SIGTERM   | 15     | Terminate| Default `kill` signal               |
| SIGCHLD   | 17     | Ignore   | Child process stopped or terminated |
| SIGCONT   | 18     | Continue | Resume a stopped process            |
| SIGSTOP   | 19     | Stop     | Uncatchable pause                   |
| SIGTSTP   | 20     | Stop     | Ctrl+Z (catchable version of STOP)  |
| SIGWINCH  | 28     | Ignore   | Terminal window resized             |

- `kill -l` -- lists all signal names and numbers
- `Ctrl+Z` sends SIGTSTP (20), which CAN be caught -- unlike SIGSTOP (19)

### Registering a signal handler in C

```c
#include <signal.h>

void handler(int signum) {
    printf("Got signal %d\n", signum);
}

// Register handler for SIGINT (2)
signal(SIGINT, handler);

// Loop forever (program will catch Ctrl+C instead of dying)
while (1) {}
```

### SIGALRM -- alarm-based timeouts

```c
#include <unistd.h>
#include <signal.h>

void ding(int sig) {
    printf("Ding!\n");
    exit(0);
}

signal(SIGALRM, ding);
alarm(3);   // send SIGALRM to self after 3 seconds

while (1) {}   // wait
// After 3 seconds: "Ding!" then exits
```

- **Key insight:** receiving a signal can **break a process out of blocking syscalls** like `sleep()`
  - e.g. `sleep(100000)` will be interrupted by a SIGALRM after 3 seconds
  - NOT all syscalls are interruptible (e.g. `read()` from stdin is NOT interrupted by SIGALRM)

### Segfaults are signals

- A segmentation fault = the process receives **SIGSEGV (signal 11)**
- The process doesn't "crash" on its own -- it receives a signal and terminates because there's no handler
- You CAN install a SIGSEGV handler and survive segfaults (generally a bad idea, but possible)

---

## 7. Shared Memory

- Another way to interact with other processes: **shared memory**
- `fork()` does NOT give you shared memory -- it uses copy-on-write, so each process gets its own copy when it writes
- `clone()` syscall can create processes that genuinely share memory regions

### `/dev/shm` -- shared memory filesystem

- A memory-backed filesystem (tmpfs) that exists in RAM
- Create files here, map them with `mmap()` from multiple processes -- they all see the same memory

```c
#include <sys/mman.h>
#include <fcntl.h>

// Create or open a shared memory object
int fd = open("/dev/shm/myshared", O_CREAT | O_RDWR, 0666);
ftruncate(fd, 4096);

// Map it into this process's address space
void *ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

// Any other process that maps the same file sees the same memory
```

---

## 8. Process Termination

A process can only terminate in exactly **two ways**:

1. Receiving an **unhandled signal** (e.g. SIGKILL, SIGSEGV with no handler, SIGTERM)
2. Calling the **`exit` system call** (syscall 60 on x86-64)

That's it. No other mechanism exists.

### Zombie processes

- After a process exits, it enters a **zombie state**
- It stays there until its parent calls `wait()` to collect its exit status
- If the parent exits without calling `wait()`, orphaned children get **re-parented** to init (PID 1) and may linger as zombies
- Zombies hold a PID slot -- historically this was a real problem (16-bit PIDs = max 65535 processes)
- Now PIDs are 32-bit so it's less catastrophic but still bad practice

```bash
ps aux
# Zombie processes show Z in the STAT column
# They appear in parentheses in ps output
```

### Reaping children properly

```c
#include <sys/wait.h>

int status;
pid_t child = waitpid(pid, &status, 0);

if (WIFEXITED(status)) {
    printf("Child exited with code %d\n", WEXITSTATUS(status));
}
```

- **Always call `wait()` or `waitpid()` on your child processes**
- In Python: always call `proc.wait()` after `subprocess.Popen` or `pwn.process`

---

## 9. Full Process Lifecycle Summary

```
ELF binary on disk
        |
        v
   Loader maps binary + libraries into virtual memory
        |
        v
   Jump to _start (entry point)
        |
        v
   __libc_start_main called
        |
        v
   main() called with (argc, argv, envp)
        |
        v
   Program runs -- interacts with world via:
        |-- Library functions (libc wrappers)
        |-- System calls (open, read, write, fork, execve...)
        |-- Signals (async notifications)
        |-- Shared memory (/dev/shm + mmap)
        |
        v
   Process terminates:
        |-- exit() syscall
        |-- Unhandled signal
        |
        v
   Zombie state -- waits for parent to call wait()
        |
        v
   Process table entry cleaned up
```

---

*parts of this notes were AI generated for clarity*
