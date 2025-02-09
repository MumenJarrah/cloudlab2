# Lab 3: Fuzzing lab
## Bugs That Are Hard to Catch


### Overview

Program analysis is used both by attackers and defenders to hunt for potential issues in the software.
However, there are always code patterns that pose unique challenges for these tools to analyze. The
goal of this lab is to help you understand the limitations of state-of-the-art program
analysis tools and get a feeling on their advantages and disadvantages with some hands-on experience.
Your goal is to craft programs **with a single bug intentionally embedded** and check
whether different program analysis tools can find the bug (evident by a program crash). If the program
analysis tool fails to find the bug *within a computation bound*, you have found a potential limitation
in this program analysis tool and will be awarded full marks for that component.

In this lab, the computation bound is **15 minutes per analyzer × 2 analyzers**:

   - AFL++: as a representative fuzzer

To confine the scope of the analysis and standardize the auto-grading process, we will NOT accept
arbitrary programs for this lab. Instead, you are encouraged to produce minimal programs
and prepare a package for each program. The package should contain not only the program code but
also information that bootstraps analysis and shows evidence that a bug indeed exists.

Specifically, each package MUST be prepared according to the requirements and restrictions below.
Failing to do so will result in an invalid package that can’t be used to score this assignment component.

  * **Directory Structure.** Each submitted package MUST follow the directory structure below:
    ```
    <package>/
        |-- main.c       // mandatory: the only code file to submit
        |-- input/       // mandatory: the test suite with 100% gcov coverage
        |   |-- <*>      // test case name can be arbitrary valid filename
        |-- crash/       // mandatory: sample inputs that crash the program
        |   |-- <*>      // sample name can be arbitrary valid filename
        |-- README.md    // optional: an explanatory note on the code and embedded bug
    ```
  * Include all source code of the program in one and only one `main.c` file. This `main.c` is the only code file you need to include in the package. DO NOT include `interface.h`.
    - Only valid C programs are allowed. DO NOT code in C++.
    - The size of `main.c` should NOT exceed 256KB (i.e., 256 × 2^10 bytes).
   
  * Your program can only invoke three library calls, all provided in `interface.h` available below.
    - `ssize_t in(void *buffer, size_t count)` which reads in at max count bytes from stdin and store then in buffer. The return value indicates the actual number of bytes read in or a negative number indicating failure.
    - `int out(const char *buffer)` which prints the buffer string to stdout. The return value indicates the actual number of
bytes written out.
    - `void abort(void)` which forces a crash of program. Note that this is NOT the only way to crash a program.

```
//interface.h

#include <stddef.h>
#include <sys/types.h>

/* external interfaces */
ssize_t read(int fd, void *buffer, size_t count);
int puts(const char *buffer);

/* exposed interfaces */
inline __attribute__((always_inline))
ssize_t in(void *buffer, size_t count) {
    return read(0, buffer, count);
}

inline __attribute__((always_inline))
int out(const char *buffer) {
    return puts(buffer);
}

void abort(void);
```

 * Your program can only take input from `stdin` using the provided `in()` function in `interface.h`. It should NOT take input from command line arguments nor environment variables. The size of input acquired from `stdin` should NOT exceed 1024 bytes.
 * Your program MUST be *compatible* with all program analysis tools used in this lab
without special modification to these tools. In other words, your program can be analyzed using
the tool invocation command provided in later part of the lab.
 * Your program should have **one and only one bug** that is intentionally planted. If any of the tool finds a crash in your program, even the crash is not caused by the intended bug, the tool is considered successful and this package cannot be used to claim victory over that tool.
 * Provide a set of test cases that achieves 100% coverage (see details on `gcov` below).
    - Each test case should NOT crash the program, i.e., exiting with a non-zero status code.
    - Each test case should complete its execution in 10 seconds.
    - Each test case should NOT exceed 1024 bytes in size.
 * Provide **at least one** sample input that can cause the program to crash by triggering the planted bug — this is to provide evidence on the existence of the bug.
 * Avoid using features that are known limitations for most program analysis tools, including:
    - DO NOT use any other library functions (including libc functions). The only permitted
library routines are given in the header file `interface.h`.
    - DO NOT use floating-point operations in your program.
    - DO NOT use inline assemblies in your program.
    - DO NOT use multi-threading. The entire program logic must be able to execute end-to-end
in a single process and a single thread.

Each package will be analyzed by all program analysis tools. For any tool X, if the tool finds a bug (even not the planted one), it is considered a success. Otherwise, the bug has managed to “evade” the detection from tool X and the package will be counted towards the scoring of tool X evasion.

**A maximum of 10 packages** can be submitted. You will have full points for tool X evasion as long as you have one package that “evaded” the detection from tool X. If you are confident that one package can be used to score all components, feel free to submit one package only. However, to be on the safe side, it is highly advised to submit multiple packages with
different strategies to confuse these program analysis tools.

### 1. Lab Environment Preparation

Although we provide two modes of preparation, we highly recommend you to setup a local VM (i.e., Option 1) for this lab as long as your machine can support it. There are two reasons for this recommendation:
    1. The ugster platform is limited in resources and cannot accommodate everyone.
    2. A local VM enables more agile development and a more controllable environment.

#### Option 1: local VM
You need to create a fresh VM with Ubuntu version 22.04.5 LTS. You can choose your favorite VM
management tool for this task, including but not limited to VirtualBox, VMware, Hyper-V, etc. Once
your Ubuntu VM is ready, log / SSH into your account and follow the common provision steps (shown below). You
will need sudo passwords for the provision.

#### Option 2: ugster VM
If option 1 is not feasible on your machine (e.g., Mac computers with Apple silicon chips), you can opt
to use the ugster VM. To be specific, you can use the `portal.sh` script to create a new VM, destroy an existing VM, or ssh into a running VM. The following is `portal.sh` and how to use it:

```
#!/bin/bash -e

# Welcome to The Assignment Portal of CS 489/698 - Software and Systems Security
#
# This is not a regular website for broadcasting information. Instead, it is a
# web service hidden behind the HTTP protocol and only intended for users of the
# Software and Systems Security course for assignment-related tasks.
#
# The file you are currently viewing is a Bash-based client to interact with
# the web service. This script is designed for a UNIX-like system (e.g., Ubuntu,
# MacOS, or WSL on Windows) with `bash`, `curl`, and `ssh-keygen` available.
#
# To use this file,
# - save it as a script (e.g., `portal.sh`)
# - change its permission if needed (e.g., `chmod +x portal.sh`) and
# - check its help message `./portal.sh help`
#
# Also check the FAQ section in the prompt.

# course variables (SET BY ADMIN)
host=ugster72d.student.cs.uwaterloo.ca
port=8000
GATEWAY=$host:$port
NAMESPACE=s3

# course variables (SET BY USERS)
USR_DEFAULT=
KEY_DEFAULT=key

# alternative to set user info: environment variable
if [ -z "${U}" ]; then
  USR="${USR_DEFAULT}"
else
  USR="${U}"
fi

if [ -z "${K}" ]; then
  KEY="${KEY_DEFAULT}"
else
  KEY="${K}"
fi

# command: print help message
function cmd_help() {
  cat <<EOF
This is a Bash-based Client to the Assignment Portal of CS 489/698

Before using this script, please check that you have set the course variables
correctly on top of this file, including:
- USR_DEFAULT, your UWaterloo username (currently set to "${USR_DEFAULT}")
- KEY_DEFAULT, a path prefix for your key files (currently set to "${KEY_DEFAULT}")

Alternatively, you can supply the same information via environment variables
before calling this script, e.g.,
U=<username> K=<path-prefix-to-passkey> ./portal.sh <command>

Your current username is "${USR}" and path-prefix to passkey is "${KEY}".

The following commands are available for everyone for VM management
- help: print this help message
- register: create a ed25519 key pair and register the user with the public key
- status: check the status of the virtual machine allocated to the user
- launch: launch the virtual machine allocated to the user
- destroy: destroy the virtual machine allocated to the user
- ssh: SSH into the virtual machine allocated to the user

The following command are available if you are a system administrator
- reset <uid>: reset the user to a state before registration
- shutdown: send a shutdown request to the system

**Frequently Asked Questions (FAQs)**

Q: What is the path prefix for keys?
A: This is a location on your local filesystem where you store your key pairs.
   - The public key will be stored in a file with path: <path-prefix>.pub
   - The private key will be stored in a file with path: <path-prefix>

   For example,
   - If your path prefix is "my_path", your keys will be
     - "my_path.pub" and
     - "my_path"
   - If your path prefix is "/home/my_username/my_keys/course", you keys will be
     - "/home/my_username/my_keys/course.pub" and
     - "/home/my_username/my_keys/course"

Q: I received a warning about conflicting fingerprints when on the SSH trial
   into a newly re-created VM (i.e., destroyed and launched), is this normal?
A: Yes, this is expected and is a defense mechanism against man-in-the-middle
   (MiTM) attacks. To resolve this issue, simply follow the prompt and remove
   the offending fingerprint.

EOF
}

# command (special): generate a key pair and register a user with the public key
function cmd_special_register() {
  ssh-keygen -t ed25519 -C "${USR}@${NAMESPACE}" -f "${KEY}" -P ""
  echo ""
  echo "!!! IMPORTANT !!!"
  echo "- Please keep your private key in a safe place."
  echo "- If you lose it, you will LOSE ACCESS TO THE SYSTEM FOREVER."
  echo "!!! IMPORTANT !!!"
  echo ""
  curl -w '\n' -s "${GATEWAY}/${USR}/register" -d @"${KEY}.pub"
}

# command (special): ssh into the allocated VM
function cmd_special_ssh() {
  local C=$(cmd_generic "config")
  local X=(${C//:/ })
  ssh -i ${KEY} vagrant@${X[0]} -p ${X[1]}
}

# command (generic): run a generic query
function cmd_generic() {
  local T=$(date +%s | xargs echo -n)
  local S=$(echo -n "${T}" | ssh-keygen -Y sign -f "${KEY}" -n "${NAMESPACE}" -)
  curl -w '\n' -s -X POST "${GATEWAY}/${USR}/$1" -d "${S}"
}

# command (generic): run a generic query (for admin commands)
function cmd_generic_admin() {
  local T=$(date +%s | xargs echo -n)
  local S=$(echo -n "${T}" | ssh-keygen -Y sign -f "${KEY}" -n "${NAMESPACE}" -)
  curl -w '\n' -s -X POST "${GATEWAY}/${USR}/$1/$2" -d "${S}"
}

# command (generic): run a system action (for admin commands)
function cmd_generic_system() {
  local T=$(date +%s | xargs echo -n)
  local S=$(echo -n "${T}" | ssh-keygen -Y sign -f "${KEY}" -n "${NAMESPACE}" -)
  curl -w '\n' -s -X POST "${GATEWAY}/${USR}/$1" -d "${S}"
}

# main entrypoint
if [ -n "$1" ]; then
  case "$1" in
  help)
    cmd_help
    ;;
  register)
    cmd_special_register
    ;;
  status)
    cmd_generic "status"
    ;;
  launch)
    cmd_generic "launch"
    ;;
  destroy)
    cmd_generic "destroy"
    ;;
  ssh)
    cmd_special_ssh
    ;;
  reset)
    cmd_generic_admin "reset" "$2"
    ;;
  list)
    cmd_generic_system "list"
    ;;
  shutdown)
    cmd_generic_system "shutdown"
    ;;
  *)
    echo "unknown command, please see run '$0 help' for the help message"
    ;;
  esac
else
  cmd_help
fi
```

You will need to register for the new platform with a new pair of keys. This can be done with `./portal.sh register`. However, this time, once you are registered, take care of your private key! More importantly,

   - ./portal.sh register should be called only once, at the beginning of this assignment.
   - The way to wipe out and reset your VM is through ./portal.sh destroy. Please do not use ./portal.sh register for this purpose.
   - NOTE: if you lose your private key after successful registration, you will be locked out of the
system completely. There is no way back.

After you register to the `ugster` platform and have your VM launched, please SSH into your VM and
follow the following common provision steps:

#### Common provision step for both options
After logging in / SSH into your VM, you can provision it with the following commands:

```
git clone https://github.com/meng-xu-cs/cs453-program-analysis-platform.git
cd cs453-program-analysis-platform
git submodule update --init
./scripts/ugster-up.sh
```

Upon successful completion, you will see an ==== END OF PROVISION === mark in the terminal.

### You have successfully completed the lab
