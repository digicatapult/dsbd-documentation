# An example workflow of compiling/testing

This example will use the LLVM toolchain. We’ll be using GitHub repositories and self-hosted runners.

Pre-requisites:

- cheribsd-morello vm

- repository and a runner created on the vm which is linked to the repository

## Adding and executing a file and workflow

In this walkthrough, will create a file, compile and execute it via a github workflow. After that, we will add tests and also compile and execute them using github workflow. (The code for this example will be written in `C`.)

Firstly, create a new `src` directory and in it a file called `get-pointer.c` with an example body:

```
/* SPDX-License-Identifier: BSD-2-Clause-DARPA-SSITH-ECATS-HR0011-18-C-0016 */
/* Copyright (c) 2020 SRI International */
#include <stdio.h>
#include "functions.h"

// Function to get the size of a pointer
size_t get_pointer_size(void) {
    return sizeof(void *);
}

// Function to get the size of an address
size_t get_address_size(void) {
    return sizeof(size_t);
}

int main(void)
{

    printf("size of pointer: %zu\n", get_pointer_size());
    printf("size of address: %zu\n", get_address_size());

    return 0;
}
```

Next create a file called `functions.h` (also in `src` directory) and paste this in it:

```
#ifndef FUNCTIONS_H
#define FUNCTIONS_H

#include <stddef.h>

// Function declarations
size_t get_pointer_size(void);
size_t get_address_size(void);

#endif

```

NOTE: A header file (such as `functions.h`) typically contains declarations of functions, types, and other elements that are used across multiple source files. They are separated out as we will be using them in a test file later on.

Next we will add a new workflow under `.github/workflows` called `myworkflow.yaml` with body:

```
name: myWorkflow
on:
  push:

jobs:
  workflow:
    runs-on: cheribsd-23.11
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v2

      - name: Install LLVM
        run: pkg64 install -y llvm-base

      - name: Compile C File
        run: |
          cc -Wall -o get-pointer src/get-pointer.c

      - name: Run Test File
        run: ./get-pointer

      - name: List Contents
        run: ls -al .
```

NOTE: There are 3 stages in our workflow. Installing packages, compiling and execution of the binary.

- We need to install LLVM first by doing `pkg64 install -y llvm-base` (With regards to packages: the repository that we're using is [pkg.cheribsd.org][pkg.cheribsd.org]. We have several options for ABIs to compile against, but in this case we want to be using the hybrid ABI. The most relevant list is therefore: [CheriBSD packages: CheriBSD:20230804:aarch64][CheriBSD packages: CheriBSD:20230804:aarch64].)

- Then we compile our `get-pointer.c` file using `cc -Wall -o get-pointer src/get-pointer.c` (The format for this command is: `cc -g -Wall -o BINARY_NAME FILE_NAME.c` so we are providing output binary name.)

- Then we execute the binary via command `./get-pointer`.

Commit your changes to github and if your runner is not running start it by: `pot start -p RUNNER_NAME`(You have to be in your vm for this.)

Now you can observe the workflow execution on github and stdout on your vm.

NOTE: If you have more than one workflow you will need to start your runner once per workflow as it only runs one queued workflow and then stops.

## Adding and executing a test file and test workflow

In `src` add a new directory `__tests__` and in it a file `get-pointer.test.c`. Paste the following in that file:

```
#include <check.h>
#include "../functions.h" // Include the header file with function declarations

// Unit tests
START_TEST(test_pointer_size)
{
    ck_assert_int_eq(get_pointer_size(), sizeof(void *));
}
END_TEST

START_TEST(test_address_size)
{
    ck_assert_int_eq(get_address_size(), sizeof(size_t));
}
END_TEST

```

NOTE: If you want to test compilation of your test file locally, you will need to install `check` package. Use a package manager `e.g. brew, pkg, apt, etc.` to do so.

Add a test workflow under `.github/workflows` called `tests.yaml` with body:

```
name: Unit Tests

on:
  push:

jobs:
  unit-tests:
    runs-on: cheribsd-23.11

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v2

      - name: Install LLVM
        run: pkg64 install -y llvm-base

      - name: Install Check
        run: pkg64 install -y check

      - name: Find where check is installed
        run: pkg64 info check

      - name: Build binary
        run: |
          cc -g -Wall -o get-pointer-test  -I"/usr/local64/include"  src/__tests__/get-pointer.test.c src/get-pointer.c

      - name: cd into test directory
        run: cd src/__tests__

      - name: Run Unit test
        run: ./get-pointer-test
      - name: Output binary
        run: file get-pointer-test

```

NOTE: Same as in previous workflow we need to install `llvm`, but now we also need to install `check` as it is needed for execution of the tests.

Command `-I"/usr/local64/include"` includes additional directories to be searched - necessary due to `check` package being installed [-Idir][ldir] ← definition.

NOTE: In the `Build binary` step make sure to include all files the test is dependent on. (For example: `src/__tests__/get-pointer.test.c src/get-pointer.c`.)

Command `file get-pointer-test` allows you to view information about the binary.

<!-- Links -->

[ctsrd-cheri]: https://ctsrd-cheri.github.io/cheri-exercises/exercises/compile-and-run/index.html
[pkg.cheribsd.org]: https://pkg.cheribsd.org/
[CheriBSD packages: CheriBSD:20230804:aarch64]: https://pkg.cheribsd.org/CheriBSD:20230804:aarch64.html
[ldir]: https://docs.oracle.com/cd/E19957-01/806-3567/cc_options.html#:~:text=library%20file%20instead.-,%2DIdir,-Adds%20dir%20to
