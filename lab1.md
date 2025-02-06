# Lab 3: Fuzzing lab (Bugs That Are Hard to Catch)

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
## 2 Lab Environment

### Step 1: Download and Extract the Lab Files
Download Labsetup.zip from the lab’s website and unzip it. This will provide you with the necessary files, including the docker-compose.yml for setting up the lab environment.


### You have successfully completed the lab
