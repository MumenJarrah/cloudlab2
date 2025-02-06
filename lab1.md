# Lab 3: Fuzzing lab

## Bugs That Are Hard to Catch

![](images/lab5-00-u.png)

## Overview

Program analysis is used both by attackers and defenders to hunt for potential issues in the software.
However, there are always code patterns that pose unique challenges for these tools to analyze. The
goal of this lab is to help you understand the limitations of state-of-the-art program
analysis tools and get a feeling on their advantages and disadvantages with some hands-on experience.
Your goal is to craft programs **with a single bug intentionally embedded** and check
whether different program analysis tools can find the bug (evident by a program crash). If the program
analysis tool fails to find the bug *within a computation bound*, you have found a potential limitation
in this program analysis tool and will be awarded full marks for that component.

In this lab, the computation bound is **15 minutes per analyzer × 2 analyzers**:

    • AFL++: as a representative fuzzer

To confine the scope of the analysis and standardize the auto-grading process, we will NOT accept
arbitrary programs for this lab. Instead, you are encouraged to produce minimal programs
and prepare a package for each program. The package should contain not only the program code but
also information that bootstraps analysis and shows evidence that a bug indeed exists.

Specifically, each package MUST be prepared according to the requirements and restrictions below.
Failing to do so will result in an invalid package that can’t be used to score this assignment component.

### Directory Structure
  * Each submitted package MUST follow the directory structure below:
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





## 2 Lab Environment

### Step 1: Download and Extract the Lab Files
Download Labsetup.zip from the lab’s website and unzip it. This will provide you with the necessary files, including the docker-compose.yml for setting up the lab environment.


### You have successfully completed the lab
