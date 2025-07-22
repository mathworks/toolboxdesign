# MATLAB MEX Best Practice

> [!IMPORTANT]
> This page is under construction and review

![Version Number](https://img.shields.io/github/v/release/mathworks/toolboxdesign?label=version) ![CC-BY-4.0 License](https://img.shields.io/github/license/mathworks/toolboxdesign)

Welcome to the MATLAB&reg; MEX Best Practice guide, which extends [MATLAB Toolbox Best Practices](./README.md). This document focuses on integrating [MEX functions](https://www.mathworks.com/help/matlab/call-mex-file-functions.html) into your MATLAB toolbox. MEX functions enable you to harness the power of C, C++, and Fortran code within MATLAB. In this document when we say "C++", we mean "C, C++, and Fortran."

## TL;DR
- C++ code goes into the `cpp` folder.
- Each MEX function is in its own folder with a `Mex` suffix
- Place the built MEX functions in a `private` folder inside your `toolbox` folder. MEX functions should only be called from within your toolbox in order to increase reliability
- External libraries are placed within a platform specific folder in the `private` folder and added to the system path
- We recommend using the [`mexhost`](https://www.mathworks.com/help/matlab/ref/mexhost.html) command to increase reliability
- Use a [`MexTask`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.tasks.mextask-class.html) in your [`buildfile.m`](https://www.mathworks.com/help/matlab/build-automation.html) for consistent builds

## Overview

[MEX functions](https://www.mathworks.com/help/matlab/call-mex-file-functions.html) are compiled functions that bridge the gap between MATLAB and C++. They behave like a MATLAB function and you must build them for each operating system you want to run on. You can determine the MEX file extension (for example, `.mexw64` in Microsoft Windows) for your operating system using [`mexext`](https://www.mathworks.com/help/matlab/ref/mexext.html). This guide will navigate you through the process of integrating of MEX functions, ensuring smooth implementation in both development and production environments.  This makes it easier for others to understand and contribute to your toolbox.

To illustrate these best practices, we've created a sample project: The Arithmetic Toolbox, available on [GitHub](https://github.com/mathworks/arithmetic). We'll reference this toolbox throughout the guide to demonstrate practical applications of these principles.  For key concepts, refer to the [Toolbox Best Practices](./README.md).

## MEX function from a single C++ source file
The most common case is when you have a single C++ MEX source file.  You do not distribute the source file to toolbox users, only the compiled MEX function.  We recommend keeping the C++ MEX source files outside of the `toolbox` folder in a folder focused on C++ code, `cpp`. 

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

``` matlab
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
 * **Out of process MEX host:** We recommend [Out-of-Process Execution of C++ MEX Functions](https://www.mathworks.com/help/matlab/matlab_external/out-of-process-execution-of-c-mex-functions.html). This prevents coding errors in your C++ MEX function from crashing MATLAB and allows you to use some third-party libraries that are not compatible with MATLAB.  Use the [`mexhost`](https://www.mathworks.com/help/matlab/ref/mexhost.html) command. Note that `mexhost` is only supported for C++ MEX functions. 
 * **Using git:** In git source control systems, we recommend that you *do not* keep compiled MEX functions under version control, as they are derived files. Add `*.mex*` to your `.gitignore` file. This is part of the [standard .gitignore file](https://github.com/mathworks/gitignore/blob/main/Global/MATLAB.gitignore) for MATLAB.

### Automation using `buildtool`

Running the `mex` command for each MEX function can be tedious and error prone. MATLAB’s [`buildtool`](https://www.mathworks.com/help/matlab/ref/buildtool.html), introduced in R2022b, can automate this and many other repetitive processes for you. The following `buildfile.m` uses [`MexTask`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.tasks.mextask-class.html), for building your MEX functions. 

We have defined the `mex` task as a [task groups](https://www.mathworks.com/help/matlab/matlab_prog/create-groups-of-similar-tasks.html), this allows us to build MEX functions out of multiple folders.

*This works in R2024b and later -- see the end of this document for a version supported in R2022a-R2024a.*
``` matlab
function plan = buildfile

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
    plan("clean") = matlab.buildtool.tasks.CleanTask;
end
```

Our example toolbox adds a `buildfile.m` to automate the building of the MEX functions. When you run "`buildtool mex`", the `invertMex.mexw64` gets created within the `private` folder.

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

Our example toolbox adds two `.cpp` files in the  `subtractMex` folder, which builds to the `subtractMex.mexw64` binary in the `private` folder.  `subtractNumber.m` provides a user accessible version that error checks before calling `SubtractMex`:

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

``` matlab
function plan = buildfile
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
    plan("clean") = matlab.buildtool.tasks.CleanTask;
end
```

## Incorporating External Libraries

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
[Dynamic libraries](https://www.learncpp.com/cpp-tutorial/a1-static-and-dynamic-libraries/) are required for running the MEX functions and must ship to the users. Place the binaries within the `private` folder under the `toolbox` folder to ensure the library gets shipped to the user. Use the [`-L`](https://www.mathworks.com/help/matlab/ref/mex.html#btw17rw-1-option1optionN) argument to the [`mex`](https://www.mathworks.com/help/matlab/ref/mex.html) command to specify the location and the name of the runtime library.

**Note:** If you have a choice between using a static or dynamic library with your MEX function, we recommend using a static library.  Static libraries are incorporated inside your MEX function, making your MEX function more robust and reliable.


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
* For projects with complex dependencies, consider adopting dependency management tools like [Conan](https://conan.io/) which can significantly simplify library management across different platforms.
* Depending on the platform, dynamic libraries require adding their path to the operating system search path. These search paths are often set using environment variables.  In case of C++ libraries you can use the `EnvironmentVariables` option of [`mexhost`](https://www.mathworks.com/help/matlab/ref/mexhost.html) to set-up the loader's path. The table below shows the environment variable for different operating systems.

| Operating system  | Environment variable for loader's search path |
| :-------- | :-------------------------------------------- |
| Linux     | `LD_LIBRARY_PATH`                             |
| Windows   | `PATH`                                        |
| Mac       | `DYLD_LIBRARY_PATH`                           |           

## Automating Builds with GitHub Actions
[MATLAB Actions](https://github.com/matlab-actions) build, test and deploy your toolbox as a part of [GitHub Action](https://docs.github.com/en/actions).  MathWorks offers free licenses for public GitHub repositories, including support for Windows, Mac and Linux. If your GitHub repository is private, visit [this webpage](https://github.com/matlab-actions/setup-matlab?tab=readme-ov-file#use-matlab-batch-licensing-token). You can also refer to [this GitHub repository](https://github.com/mathworks/advanced-ci-configuration-examples) for examples on advanced CI configurations from MathWorks.

Within a GitHub Action, you can invoke MATLAB's buildtool using [matlab-actions/run-build](https://github.com/matlab-actions/run-build). The build tasks that you already configured for local development like building MEX functions, running tests and packaging the toolbox can all be reused within your GitHub Workflow. Our example uses a [matrix strategy](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/running-variations-of-jobs-in-a-workflow) in the GitHub Action to build MEX functions on different operating systems. 

```yaml
# Excerpt from the YAML file
Jobs:
  mexBuild:
  # Build MEX function on different operating systems using matrix strategy for operating systems.
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, mac-latest]
    runs-on: ${{ matrix.os }}
  ...

  steps:
    - name: Check out repository
        uses: actions/checkout@v4
    - name: Set up MATLAB
        uses: matlab-actions/setup-matlab@v2
    - name: Run build
        uses: matlab-actions/run-build@v2
        with:
          tasks: mex 
      ...
```
For the full YAML file refer to [`mexbuild.yml`](https://github.com/mathworks/arithmetic/blob/main/.github/workflows/mexbuild.yml).



## Conclusion

By following to these best practices, you'll create a reliable, maintainable, and user-friendly MATLAB toolbox that harnesses the full potential of MEX functions. Through effective project structure organization, build automation, and dependency management, you can focus on delivering great solutions to your users.

We welcome your input! For further details, suggestions, or to contribute, please [open an issue](https://github.com/mathworks/toolboxdesign/issues/new/choose).

## Appendix: Buildfiles for MATLAB R2022a-R2024a 

``` matlab
function plan = buildfile
    % Create a plan using local functions
    plan = buildplan(localfunctions);
    plan("mex").Inputs.MexFolders = files(plan, fullfile("cpp", "*Mex"));
    plan("mex").Inputs.Destination = fullfile("toolbox", "private");
    plan("mex").Outputs = fullfile("toolbox", "private", "*.mexw64");
    plan("clean") = matlab.buildtool.tasks.CleanTask;
end

function mexTask(context)
    % Build multiple source MEX function within cpp/*Mex folders
    for mexSourceFolder =context.Task.Inputs.MexFolders.paths
        mex(fullfile(mexSourceFolder, "*.cpp"), "-outdir", context.Task.Inputs.Destination);
    end
end
```


``` matlab
function plan = buildfile
    % Create a plan using local functions
    plan = buildplan(localfunctions);
    plan("mex").Inputs.MexFiles = files(plan, fullfile("cpp","mexfunctions/", "*.cpp"));
    plan("mex").Inputs.Destination = fullfile("toolbox", "private");
    plan("mex").Outputs = fullfile("toolbox", "private", "*.mexw64");
    plan("clean") = matlab.buildtool.tasks.CleanTask;
end

function mexTask(context)
    % Build single source MEX functions within cpp/mexfunctions
    for mexSourceFiles =context.Task.Inputs.MexFiles.paths
        mex(mexSourceFiles, "-outdir", context.Task.Inputs.Destination);
    end
end
```

---
[![CC-BY-4.0](images/cc-by-40.png)](https://creativecommons.org/licenses/by/4.0/)

Copyright &copy; 2025, The MathWorks, Inc.
