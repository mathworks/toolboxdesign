# MATLAB MEX Best Practice

> [!IMPORTANT]
> This page is under construction and review

**TODO:**

- [X] Review and revise overview.  
- [X] Review and revise single source file section.  
- [X] Review and revise buildtool section.  
- [ ] Update `buildfile.m` in buildtool section to work with 24b
- [X] Review and update multiple source file section
- [X] Review and update `mexfunction` folder scenario section
- [X] Review and update external libraries section
- [ ] Review and update CI / GitHub Actions section
- [ ] Extend Arithmetic to build on all three OS- single MEX API
- [ ] Appendix for 2024a buildfiles

![Version Number](https://img.shields.io/github/v/release/mathworks/toolboxdesign?label=version) ![CC-BY-4.0 License](https://img.shields.io/github/license/mathworks/toolboxdesign)

Welcome to the MATLAB&reg; MEX Best Practice guide, which extends [MATLAB Toolbox Best Practices](./README.md). This document focuses on integrating [MEX functions](https://www.mathworks.com/help/matlab/call-mex-file-functions.html) into your MATLAB toolbox. MEX functions enable you to harness the power of C, C++, and Fortran code within MATLAB. In this document when we say "C++", we mean "C, C++, and Fortran."

## TL;DR
- C++ code goes into the `cpp` folder.
- Each MEX functon is in its own folder with a `Mex` suffix
- Place the built MEX functions in a `private` folder inside your `toolbox` folder. MEX functions should only be called from within your toolbox in order to increase reliablity
- External libraries are placed within a platform specific folder in the `private` folder and added to the system path
- We recommend using the [`mexhost`](https://www.mathworks.com/help/matlab/ref/mexhost.html) command to increase reliablity
- Use a [MexTask](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.tasks.mextask-class.html) in your [buildfile.m](https://www.mathworks.com/help/matlab/build-automation.html) for consistent builds

## Overview

[MEX functions](https://www.mathworks.com/help/matlab/call-mex-file-functions.html) are compiled functions that bridge the gap between MATLAB and C++. They behave like a MATLAB function and you must build them for each operating system you want to run on. You can determine the MEX file extension (for example, `.mexw64` in Microsoft Windows) for your operating system using [`mexext`](https://www.mathworks.com/help/matlab/ref/mexext.html). This guide will navigate you through the process of integrating of MEX functions, ensuring smooth implementation in both development and production environments.  This makes it easier for others to understand and contribute to your toolbox.

To illustrate these best practices, we've created a sample project: The Arithmetic Toolbox, available on [GitHub](https://github.com/mathworks/arithmetic). We'll reference this toolbox throughout the guide to demonstrate practical applications of these principles.  For key concepts, refer to the [Toolbox Best Practices](./README.md).

## MEX function from a single C++ source file
The most common case is when you have a single C++ MEX source file.  You do not distribute the source file to toolbox users, only the compiled mex function.  We recommend keeping the C++ MEX source files outside of the `toolbox` folder in a folder focused on C++ code, `cpp`. 

For each MEX function in the `cpp` folder, create a folder with the name that matches the name of your MEX function.  This folder should end with `Mex` to indicate that all the files within the folder are associated with a single MEX function.

Our example toolbox has a `cpp` folder in the root folder, with an `invertMex` subfolder inside.  The C++ code `invertMex.cpp` is within this folder.  While the `.cpp` file does not have to match the folder name, it can be convenient to do this in simple cases.

``` text
arithmetic/
├───cpp/
│   └───invertMex/
│       └───invertMex.cpp
├───toolbox/
...
└───arithmetic.prj
```
### Building MEX functions

Use the [`mex`](https://www.mathworks.com/help/matlab/ref/mex.html) command to compile `invertMex.cpp` into MEX functions. Place your compiled MEX functions in a `private` folder within the `toolbox` folder (see below). You need to provide the path of `invertMex.cpp` and path of the `private` folder as inputs to the `mex` command. 

```matlab
>> source = fullfile("cpp", "invertMex", "*.cpp");
>> destination = fullfile("toolbox", "private");
>> mex(source, "-outdir", destination, "-output", "invertMex")
```
The `mex` command would create the MEX function and place it within the `private` folder.

Our example toolbox has:

1. A `invertMex.mexw64` file in the `toolbox/private` folder
2. The MATLAB function `invertNumber.m` within the `toolbox` folder calls into the `invertMex` MEX function. We recommend using an [arguments block](https://www.mathworks.com/help/matlab/ref/arguments.html) in `invertNumber` to validate the inputs before passing it to `invertMex`. 

``` text
arithmetic/
├───cpp/
│   └───invertMex/
│       └───invertMex.cpp
├───toolbox/
|   ├───private/
|   |   └───invertMex.mexw64
|   ├...
|   └───invertNumber.m
└───arithmetic.prj
``` 
 ### Additional Notes
 * **Why put the MEX functions within a private folder?** By putting it in a [`private`](https://www.mathworks.com/help/matlab/matlab_prog/private-functions.html) folder, you restrict your users from calling the MEX function directly. Even minor errors in a MEX function will crash MATLAB, especially if they receive unexpected inputs. By limiting access to MEX functions to a MATLAB function that you control, you ensure that only what you expect will be passed as input to the MEX function, preventing errors from unexpected or unhandled inputs.
 * **Out of process MEX host** We recommend [Out-of-Process Execution of C++ MEX Functions](https://www.mathworks.com/help/matlab/matlab_external/out-of-process-execution-of-c-mex-functions.html). This prevents coding errors in your C++ MEX function from crashing MATLAB and allows you to use some third-party libraries that are not compatible with MATLAB.  Use the [`mexhost`](https://www.mathworks.com/help/matlab/ref/mexhost.html) command. Note that `mexhost` is only supported for C++ MEX functions. 
 * **Using git** In git source control systems, we recommend that you do not keep compiled MEX functions under version control, as they are derived files. Add `*.mex*` to your `.gitignore` file. This is part of the [standard .gitignore file](https://github.com/mathworks/gitignore/blob/main/Global/MATLAB.gitignore) for MATLAB.

### Automation using `buildtool`

Running the `mex` command for each MEX function can be tedious and error prone. MATLAB’s [`buildtool`](https://www.mathworks.com/help/matlab/ref/buildtool.html), introduced in R2022b, can automate this and many other repetitive processes for you. The following `buildfile.m` creates a [`MexTask`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.tasks.mextask-class.html), that builds your MEX functions.

```matlab
function plan = buildfile
    % Works in 24b.

    plan = buildplan();
    
    mexOutputFolder = fullfile("toolbox","private");
    
    % Compile Cpp source code within cpp/*Mex into MEX functions
    foldersToMex = plan.files(fullfile("cpp", "*Mex")).select(@isfolder);
    for folder = foldersToMex.paths
        [~, folderName] = fileparts(folder);
        plan("mex:"+folderName) = matlab.buildtool.tasks.MexTask(fullfile(folder, "**/*.cpp"), ...
            mexOutputFolder, ...
            Filename=folderName);
    end
    plan("mex").Description = "Build MEX functions";
end
```

<!-- In the build file, we create a [`plan`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.plan-class.html) and add a [`MexTask`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.tasks.mextask-class.html) to the it. The [`matlab.buildtool.tasks.MexTask.forEachFile`](https://www-jobarchive.mathworks.com/Bdoc/latest_pass/matlab/help/matlab/ref/matlab.buildtool.tasks.mextask.foreachfile.html) API, introduced in R2025a, converts every C++ file within the specified folder into MEX functions. [`MexTask.forEachFile`](https://www-jobarchive.mathworks.com/Bdoc/latest_pass/matlab/help/matlab/ref/matlab.buildtool.tasks.mextask.foreachfile.html) takes a [`FileCollection`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.io.filecollection-class.html) as input. The  [`files`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.plan.files.html) API can be used to create a `FileCollection`. -->

Our example toolbox adds a `buildfile.m` to automate the building of the MEX functions, on executing `buildtool mex`, the `invertMex.mexw64` gets created within the `private` folder.

``` text
arithmetic/
├───cpp/
│   └───invertMex/
│       └───invertMex.cpp
├───toolbox/
|   ├───private/
|   |   └───invertMex.mexw64
|   ├...
|   ├───buildfile.m
|   └───invertNumber.m
└───arithmetic.prj
``` 

## MEX function from multiple C++ source files

The above pattern extends to MEX functions that has multiple C++ source files. One of the source files must contain the [gateway](https://www.mathworks.com/help/matlab/matlab_external/gateway-routine.html) function. Place all the source files under a single folder.

<!-- - Create a folder for each MEX function within the `cpp` folder 
- Name of the folder should be same as that of the MEX function 
- Folder name is suffixed with 'Mex' to indicate that the contents of the folder should be compiled into a single MEX function.  -->

Our example toolbox adds two .cpp files in the  `subtractMex` folder, which builds to the `subtractMex.mexw64` binary in the `private` folder.  `subtractNumber.m` provides a user accessible version that error checks before calling `SubtractMex`:

``` text
arithmetic/
├───cpp/
│   ├───subtractMex/
│   │   ├───subtract.cpp % Implements gateway function
│   │   └───subtractImpl.cpp % other features
│   └───invertMex/
│       └───invertMex.cpp
├───toolbox/
|   ├───private/
|   |   ├───invertMex.mexw64
|   |   └───subtractMex.mexw64
|   ├─...
|   ├───subtractNumber.m
|   └───invertNumber.m
├───arithmetic.prj
└───buildfile.m
```

The `buildfile.m` is the same as before.

## Handling a large number of MEX functions
If you have many MEX functions, each in its own C++ source file, the approach of placing each C++ source file in a separate folder is cumbersome. In such scenarios we recommend an alternate pattern: move the source files within a `mexfunctions` subfolder under the `cpp` folder.

``` text
arithmetic/
├───cpp/
│   └───mexfunctions/
|       ├───powerMex.cpp
│       └───squareRootMex.cpp
├───toolbox/
|   ├───private/
|   |   ├───powerMex.mexw64
|   |   └───squareRootMex.mexw64
|   ├─...
|   ├───powerNumber.m
|   └───divideNumber.m
├───arithmetic.prj
└───buildfile.m
```
The build file for building these MEX functions is slightly different. 

```matlab
function plan = buildfile
% !!Revise to work in 24b!!
    plan = buildplan();
    mexOutputFolder = fullfile("toolbox","private");

    % Compile all the folders inside cpp/*Mex.cpp into MEX functions
    filesToMex = plan.files(fullfile("cpp", "mexfunctions", "*.cpp"));
    for cppFile = filesToMex.paths
        [~, fileName] = fileparts(cppFile);
        plan("mex:"+fileName) = matlab.buildtool.tasks.MexTask(cppFile, ...
                                mexOutputFolder);
    end
    plan("mex").Description = "Build MEX functions";
end
```

## Incorporating External Libraries

<!-- - **Compile time binaries**: Static libraries are a good examples of compile time binaries. These library binaries are required only at build time and you need not ship them to your users. -->
<!-- - **Run time binaries**: These are platform dependent binaries, that the users need to run your toolbox, shared object libraries (.so files) in Linux and dynamic link libraries (.dll files) in Windows are good examples. -->
<!-- RP: save this till we use it -->
<!-- - **CI/CD Pipelines**: Continuous Integration and Continuous Deployment tools like GitHub Actions or GitLab CI/CD ensure your code is tested and deployed automatically. -->

You can call libraries implemented in C++ using MEX functions. Since MEX source files are just C++ source code, they use standard C++ syntax to access external libraries. 

### External Library Header Files (`.h`,`.hpp`)
Create an `include` folder within the `cpp` folder and put the external library [header files](https://www.learncpp.com/cpp-tutorial/header-files/) within this folder. Use the [`-I`](https://www.mathworks.com/help/matlab/ref/mex.html#btw17rw-1-option1optionN) argument to the [mex function](https://www.mathworks.com/help/matlab/ref/mex.html) to specify that header files are in the `include` folder.

### Incorporating a Static Library (`.lib`)
Some MEX functions incorporate [static libraries](https://www.learncpp.com/cpp-tutorial/a1-static-and-dynamic-libraries/) that are compiled into your MEX function. Place these binaries under platform specific folders within the `library` folder. Use the names from the [`computer('arch')`](https://www.mathworks.com/help/matlab/ref/computer.html) command in MATLAB for the folder names. The table below provides a summary of the folder names and file extensions used for static libraries for popular operating systems.

| Platform          | Folder name | Binary Extension | 
| :---------------- | :------     | :------        |
| Linux             | glnxa64     | .a             |
| Windows           | win64       | .lib           |
| ARM / Intel  Mac  | maca64      | .dylib         |

``` text
zlibStatic/
:
├───cpp/
|   ├───include/
|   │   ├───zlibconf.h
|   │   └───zlib.h
|   ├───library/
|   │   ├───glnxa64
|   |   |   └───libz.a
|   │   ├───win64
|   |   |   └───libz.lib
|   │   └───maca64
|   |       └───libz.dylib
│   └───mexfunctions/
│       └───deflateMex.cpp
├───toolbox/
|   ├───deflate.m
|   └───private/
|       ├───deflateMex.mexw64
|       ├───deflateMex.mexa64
|       └───deflateMex.mexmaca64
├───zlibStatic.prj
└───buildfile.m 
```

### Calling a Dynamic Library
[Dynamic libraries](https://www.learncpp.com/cpp-tutorial/a1-static-and-dynamic-libraries/) are required for running the MEX functions and must ship to the users. Place the binaries within the `private` folder under the `toolbox` folder to ensure the library gets shipped to the user. Use the [`-L`](https://www.mathworks.com/help/matlab/ref/mex.html#btw17rw-1-option1optionN) argument to the [mex function](https://www.mathworks.com/help/matlab/ref/mex.html) to specify the location and the name of the runtime library.

**Note:** If you have a choice between using a static or dynamic library with your MEX function, we recommend using a static library.  Static libraries are incorprorated inside your MEX function, making your MEX function more robust and reliable.


``` text
zlibShared/
:
├───cpp/
|   ├───include/
|   │   └───zlib.h
│   └───mexfunctions/
│       └───deflateMex.cpp
├───toolbox/
|   ├───deflate.m
|   └───private/
|       ├───libz.so
|       ├───libz.lib
|       ├───libz.dylib
|       ├───deflateMex.mexw64
|       ├───deflateMex.mexa64
|       └───deflateMex.mexmaca64
├───zlibShared.prj
└───buildfile.m 
```
<!-- * When your MEX function relies on external libraries, store the binaries in a `libraries` directory with platform-specific subdirectories, as defined by the [`computer('arch')`](https://www.mathworks.com/help/matlab/ref/computer.html) command in MATLAB.  -->
* For projects with complex dependencies, consider adopting dependency management tools like [Conan](https://conan.io/) which can significantly simplify library management across different platforms.
* Depending on the platform, dynamic libraries require adding their path to the operating system search path. These search paths are often set using environment variables.  In case of C++ libraries you can use [`mexhost`](https://www.mathworks.com/help/matlab/ref/mexhost.html)'s `EnvironmentVariables` option to set-up the loader's path. Here is a summary of 
loader's search path for different platforms.

| Platform  | Environment variable for loader's search path |
| :-------- | :-------------------------------------------- |
| Linux     | `LD_LIBRARY_PATH`                             |
| Windows   | `PATH`                                        |
| Mac       | `DYLD_LIBRARY_PATH`                           |           


<!-- Our example toolbox adds a library called `complex` to the `subtractMex` function: -->

<!-- ``` text
arithmetic/
:
├───toolbox/
|   ├───add.m
|   ├───subtract.m
|   └───private/
|       ├───addMex.mexw64 (derived)
|       ├───addMex.mexa64 (derived)
|       ├───addMex.mexmaca64 (derived)
|       ├───subtractMex.mexw64 (derived)
|       ├───subtractMex.mexa64 (derived)
|       └───subtractMex.mexmaca64 (derived)
├───cpp/
│   ├───addMex/
│   │   ├───firstFile.cpp
│   │   └───secondFile.cpp
│   └───subtractMex/
│       ├───subtract.cpp
|       └───libraries
|           ├───complex.hpp
|           ├───glnxa64
|           |   └───libcomplex.so
|           ├───maci64
|           |   └───libcomplex.dylib
|           └───win64
|               └───complex.dll
├───arithmetic.prj
└───buildfile.m 
``` -->



## Automating Builds with GitHub Actions
You can use [MATLAB Actions](https://github.com/matlab-actions) for configuring MATLAB within a [GitHub Action](https://docs.github.com/en/actions). It can be used to build, test and deploy your toolbox. MathWorks offers free licenses for configuring MATLAB within a GitHub Action for public GitHub repositories. If your GitHub repository is private, you will need a [batch license token](https://github.com/mathworks-ref-arch/matlab-dockerfile/blob/main/alternates/non-interactive/MATLAB-BATCH.md#matlab-batch-licensing-token) to run MATLAB on [GitHub Runners](https://docs.github.com/en/actions/using-github-hosted-runners/using-github-hosted-runners). GitHub Hosted Runners offer support for all the three major operating systems Windows, Mac and Linux.

Within a GitHub Action, you can invoke MATLAB's buildtool using [matlab-actions/run-command@v2](https://github.com/matlab-actions/run-command/). The build tasks that you already configured for local development like building MEX functions, running tests and packaging the toolbox can all be reused within your GitHub Workflow. 

```yml
name: releaseWorkflow

# This workflows creates a new (draft) release when a new git tag is pushed to GitHub.
on:
  push:
    tags:
      - '*'
  workflow_dispatch:

jobs:
  buildPackageAndRelease:
    runs-on: windows-latest

    steps:
      - name: Check out repository
        uses: actions/checkout@v4
      - name: Set up MATLAB
        uses: matlab-actions/setup-matlab@v2
        with:
          release: R2024b
      - name: Run script
        uses: matlab-actions/run-command@v2
        with:
          command: buildtool mex test package
      - name: Create GitHub Release
        uses: ncipollo/release-action@v1
        with:
          draft: true        
          artifacts: "release/Arithmetic_Toolbox.mltbx"  
```

### Multi platform MEX functions build using CI systems
You can use GitHub Actions' [matrix strategy](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/running-variations-of-jobs-in-a-workflow) to build MEX functions on different operating systems. We recommend splitting the workflow into two jobs, the first job creates the MEX functions for all the three platforms and publishes the MEX functions as a artifacts. The second job executes on a single operating system, creates a MLTBX file incorporating the platform dependent MEX functions and publishes the MLTBX to the release section on GitHub.

```yml
name: releaseMultiplePlatform

on:
  push:
    tags:
      - '*'
  workflow_dispatch:

jobs:
  mexBuild:
  # Build MEX function on different operating systems using matrix strategy for operating systems.
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, mac-latest]
        MATLABVersion: [R2024b]
    runs-on: ${{ matrix.os }}

    steps:
      - name: Check out repository
        uses: actions/checkout@v4
      - name: Set up MATLAB
        uses: matlab-actions/setup-matlab@v2
        with:
          release: ${{ matrix.MATLABVersion }}
      - name: Run script
        uses: matlab-actions/run-command@v2
        with:
          command: buildtool mex
      - name: Upload MEX
        uses: actions/upload-artifact@v4
        with:
          name: MEX_${{ matrix.os }}
          path: toolbox/private
    
  packageAndRelease:
    needs: mexBuild
    runs-on: windows-latest
    steps:
      - name: Check out repository
        uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          name: MEX_ubuntu-latest
          path: toolbox/private
      - uses: actions/download-artifact@v4
        with:
          name: MEX_windows-latest
          path: toolbox/private
      - name: Set up MATLAB
        uses: matlab-actions/setup-matlab@v2
        with:
          release: R2024b
      - name: Run script
        uses: matlab-actions/run-command@v2
        with:
          command: buildtool test package
      - name: Create GitHub Release
        uses: ncipollo/release-action@v1
        with:
          draft: true        
          artifacts: "release/Arithmetic_Toolbox.mltbx"
```

<!-- ## Testing

Develop tests for the MATLAB layer in the `tests` folder. While MEX functionality is indirectly tested through MATLAB scripts, this approach ensures comprehensive validation of your toolbox's capabilities.

Our example toolbox adds `testAdd.m` and `testSubtract.m`to validate the functionality of `add.m` and `subtract.m` which, in turn, use `addMex` and `subtractMex`.

``` text
arithmetic/
:
├───tests/
|   ├───testAdd.m
|   └───testSubstract.m
├───cpp/
│   ├───substractMex/
│   │   ├───substract.cpp
│   │   └───substractImp.cpp
│   └───mexfunctions/
│       └───addMex.cpp
├───toolbox/
|   ├───add.m
|   ├───subtract.m
|   └───private/
|       ├───addMex.mexw64 (derived)
|       ├───addMex.mexa64 (derived)
|       ├───addMex.mexmaca64 (derived)
|       ├───subtractMex.mexw64 (derived)
|       ├───subtractMex.mexa64 (derived)
|       └───subtractMex.mexmaca64 (derived)
├───arithmetic.prj
└───buildfile.m 
``` -->


## Conclusion

By following to these best practices, you'll create a reliable, maintainable, and user-friendly MATLAB toolbox that harnesses the full potential of MEX functions. Through effective project structure organization, build automation, and dependency management, you can focus on delivering great solutions to your users.

We welcome your input! For further details, suggestions, or to contribute, please [open an issue](https://github.com/mathworks/toolboxdesign/issues/new/choose).

## Appendix: Buildfiles for MATLAB R2024a 

```matlab
function plan = buildfile
    % !! Works in 24a, introduced a dummy task called mex and made all the mex compile task as dependency. Not a great way to make it work. 24a does not have task collection. !!
    plan = buildplan();
    
    mexOutputFolder = fullfile("toolbox","private");
    
    % Compile Cpp source code within cpp/*Mex into MEX functions
    foldersToMex = plan.files(fullfile("cpp", "*Mex")).select(@isfolder);
    mexTasks = string([]);
    for folder = foldersToMex.paths
        [~, folderName] = fileparts(folder);
        plan("mex_"+folderName) = matlab.buildtool.tasks.MexTask(fullfile(folder, "**/*.cpp"), ...
            mexOutputFolder, ...
            Filename=folderName);
        mexTasks(end+1) = "mex_" + folderName;
    end
    plan("mex") = matlab.buildtool.Task;
    plan("mex").Dependencies = mexTasks;
    plan("mex").Description = "Build MEX functions";
end
```


```matlab
function plan = buildfile
% !!Revise to work in 24a!!
% No task group in 24a, need to introduce a dummy mex task and make the actual tasks as a dependency
    plan = buildplan();
    mexOutputFolder = fullfile("toolbox","private");

    % Compile all the folders inside cpp/*Mex.cpp into MEX functions
    mexTasks = string([]);
    filesToMex = plan.files(fullfile("cpp", "mexfunctions", "*.cpp"));
    for cppFile = filesToMex.paths
        [~, fileName] = fileparts(cppFile);
        plan("mex_"+fileName) = matlab.buildtool.tasks.MexTask(cppFile, ...
                                mexOutputFolder);
        mexTasks(end+1) = "mex_" + fileName;
    end
    plan("mex") = matlab.buildtool.Task;
    plan("mex").Dependencies = mexTasks;
    plan("mex").Description = "Build MEX functions";
end
```

---
[![CC-BY-4.0](images/cc-by-40.png)](https://creativecommons.org/licenses/by/4.0/)

Copyright &copy; 2023-2025, The MathWorks, Inc.
