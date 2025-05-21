# MATLAB MEX Best Practice

> [!IMPORTANT]
> This page is under construction and review

![Version Number](https://img.shields.io/github/v/release/mathworks/toolboxdesign?label=version) ![CC-BY-4.0 License](https://img.shields.io/github/license/mathworks/toolboxdesign)

Welcome to the MATLAB&reg; MEX Best Practice guide, this document extends [MATLAB Toolbox Best Practices](./README.md) by offering advise on integrating [MEX files](https://www.mathworks.com/help/matlab/call-mex-file-functions.html) into your MATLAB toolbox projects. MEX functions enable you to harness the power of _C/C++/Fortran_ functions within MATLAB.In this document when we say "C++", we mean "C, C++, and Fortran."

## Overview

[MEX functions](https://www.mathworks.com/help/matlab/call-mex-file-functions.html) are compiled code that bridges the gap between MATLAB and C++. They behave like a MATLAB function and you must build them for each operating system you want to run on. You can determine the MEX extension for your operating system using [`mexext`](https://www.mathworks.com/help/matlab/ref/mexext.html). While the integration of MEX files can be intricate, this guide will navigate you through the process, ensuring smooth implementation in both development and production environments.  This makes it easier for others to understand and contribute to your project.

To illustrate these best practices, we've created a sample project: The Arithmetic Toolbox, available on [GitHub](https://github.com/mathworks/arithmetic). We'll reference this project throughout the guide to demonstrate practical applications of these principles.  For key concepts please refer to the [Toolbox Best Practices](./README.md).

<!-- - **Compile time binaries**: Static libraries are a good examples of compile time binaries. These library binaries are required only at build time and you need not ship them to your users. -->
<!-- - **Run time binaries**: These are platform dependent binaries, that the users need to run your toolbox, shared object libraries (.so files) in Linux and dynamic link libraries (.dll files) in Windows are good examples. -->
<!-- RP: Maybe introduce this later 
BP: Introduced the buildtool later as suggested-->
<!-- - **`buildtool`**: MATLAB's tool for automating build processes, including MEX file compilation.  See the [`buildtool`](https://www.mathworks.com/help/matlab/matlab_prog/overview-of-matlab-build-tool.html) for more information. -->
<!-- RP: save this till we use it -->
<!-- -  **MEX compiler**: Converts MEX source files into MEX functions. The MEX compiler can be accessed from MATLAB via the [`mex`](https://www.mathworks.com/help/matlab/ref/mex.html) command, it can be invoked for the system terminal via the same command. You can configure C/ C++/ Fortran compilers using the [mex -setup](https://www.mathworks.com/help/matlab/matlab_external/changing-default-compiler.html). -->
<!-- - **Project Root**: The folder under which the toolbox source code is organized. A git repository is initialized under this folder. -->
<!-- - **CI/CD Pipelines**: Continuous Integration and Continuous Deployment tools like GitHub Actions or GitLab CI/CD ensure your code is tested and deployed automatically. -->

## Creating a MEX function from a single C++ source file
<!-- RP: let's focus on a single C++ / MEX scenario first (add), then have a closing section that talks about multiple single C++ file MEX functions.
BP: Added the single source scenario, first.
 -->
Suppose you have a C++ MEX source file that is self contained (has a MEX gateway and does not depend on other files). You want to organize this file in a way that makes building, integrating and sharing the MEX function easy. MEX source files need not be distributed to the toolbox users, since they are not required to run the toolbox. We recommend keeping the MEX source file outside of the `toolbox` folder. 

For the Arithmetic toolbox, create: 
1. `cpp` folder under the root folder
2. `mexfunctions` subfolder within the `cpp` folder and save `invertMex.cpp` within this folder
3. `private` folder within the `toolbox` folder for storing the compiled MEX functions
``` text
arithmetic/
├───cpp/
│   └───invertMex/
│       └───invertMex.cpp
├───toolbox/
|   ├───private/
|   └───...
└───arithmetic.prj
```
### Building MEX functions
<!-- RP: I think we're introducing buildtool too early. Show the basic mex solution first.
BP: Added the mex command based build and introduced the buildtool lated  -->
You can use the [`mex`](https://www.mathworks.com/help/matlab/ref/mex.html) command, to compile `invertMex.cpp` into MEX functions. You need to provide the path of `invertMex.cpp` and path of the `private` folder as inputs to the `mex` command. 

```matlab
>>source = fullfile("cpp", "invertMex", "*.cpp");
>>destination = fullfile("toolbox", "private");
>>mex(source, destination) % Add output file name
```
The `mex` command would create the MEX function and place it within the `private` folder.
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
 ## Why put the MEX functions within a private folder?
 By putting it in a [private](https://www.mathworks.com/help/matlab/matlab_prog/private-functions.html) folder, you restrict your users from  calling the MEX functions directly. It is very common for MEX files to crash MATLAB if they get unexpected inputs. By limiting access to MEX functions from a MATLAB function, you can control what gets passed as input to the MEX function, there by preventing errors from unexpected or unhandled inputs.

The MATLAB function `invertNumber` within the `toolbox` folder makes a call to the `invertMex` MEX function. We suggest adding an [arguments block](https://www.mathworks.com/help/matlab/ref/arguments.html) to `invertNumber` to validate the inputs before passing it to `invertMex`. 

<!-- Rewrite this -->
You’ll notice that we deliberately name the source file and the MEX function the same way. For example, on Windows, `invertMex.cpp` gets compiled into `invertMex.mexw64`, with `invertMex` being the name of the MEX function. This naming convention is helpful for the following reasons:

* It makes it easy to match each MEX function to its source code.
* By adding the "Mex" suffix, it’s immediately clear that this is a MEX function—not a regular MATLAB function.

If you are using git, we recommend leaving the compiled MEX files out of version control. You can add `*.mex*` to you `.gitignore` file. This is part of the [standard .gitignore](https://github.com/mathworks/gitignore/blob/main/Global/MATLAB.gitignore) for MATLAB.

### Out of process MEX host
For MEX functions written using the C++, we recommend calling the MEX function from an out of process MEX host. You can create an out of process MEX host using the [mexhost](https://www.mathworks.com/help/matlab/matlab_external/out-of-process-execution-of-c-mex-functions.html) command. It protects MATLAB from crashing due to unexpected errors originating from the MEX function execution.

**Note**: `mexhost` is only supported for C++ MEX functions. 

<!-- Create an Example for mexhost -->

### Automation using `buildtool`
Running the `mex` command for each MEX file can be tedious and error prone. That’s where MATLAB’s [`buildtool`](https://www.mathworks.com/help/matlab/ref/buildtool.html), introduced in R2022b can be useful. The following `buildfile.m` creates a `mex` task, that builds your MEX files.

``` matlab
function plan = buildfile

% Create a plan 
plan = buildplan();

% Compile all the .cpp files inside cpp/mexfunctions into MEX functions
mexSourceFiles = files(plan, fullfile("cpp", "mexfunctions", "*.cpp"));
mexOutputFolder = fullfile("toolbox","private");
plan("mex") = matlab.buildtool.tasks.MexTask(mexSourceFiles, mexOutputFolder);
% Make this work in 24a
end
```
<!-- In the build file, we create a [`plan`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.plan-class.html) and add a [`MexTask`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.tasks.mextask-class.html) to the it. The [`matlab.buildtool.tasks.MexTask.forEachFile`](https://www-jobarchive.mathworks.com/Bdoc/latest_pass/matlab/help/matlab/ref/matlab.buildtool.tasks.mextask.foreachfile.html) API, introduced in R2025a, converts every C++ file within the specified folder into MEX functions. [`MexTask.forEachFile`](https://www-jobarchive.mathworks.com/Bdoc/latest_pass/matlab/help/matlab/ref/matlab.buildtool.tasks.mextask.foreachfile.html) takes a [`FileCollection`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.io.filecollection-class.html) as input. The  [`files`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.plan.files.html) API can be used to create a `FileCollection`. -->



## Creating a MEX function from multiple C++ source files

Tha above pattern naturally extends to MEX functions that has multiple C++ source files. One of the source files must contain the [gateway](https://www.mathworks.com/help/matlab/matlab_external/gateway-routine.html) function. Place a the source files under a single folder.

<!-- - Create a folder for each MEX function within the `cpp` folder 
- Name of the folder should be same as that of the MEX function 
- Folder name is suffixed with 'Mex' to indicate that the contents of the folder should be compiled into a single MEX function.  -->

``` text
arithmetic/
├───cpp/
│   ├───substractMex/
│   │   ├───substract.cpp % Implements gateway function
│   │   └───substractImp.cpp % other features
│   └───invertMex/
│       └───invertMex.cpp
├───toolbox/
|   ├─...
|   ├───subtract.m
|   └───invertNumber.m
├───arithmetic.prj
└───buildfile.m
```

The `buildfile.m` is the same as before.

## Special case: Many MEX functions with a single C++ source file


## Incorporating External Libraries
You can call libraries implemented in C++ using MEX functions. Since MEX source files are just C++ source code, they use the syntax of C++ to access external libraries. You may be wondering where to store these external libraries?

The answer to this question depends on the external library and the operating system with which you are working with. There are two broad categories of libraries: compile time and execution time libraries. 
 
We assume that the external library is available to you as include headers and binaries. We will not dwell into the details of how these headers and binaries are created. Let as talk a bit about it organization.

### External library headers
These files are required only at compile time, your users do not want them to run the MEX functions. Having a standard location to store these headers makes it easier for you to manage them and pass it to the compiler.

Create an `include` folder within the `cpp` folder and move the external library headers within this folder. You can use [`mex`](https://www.mathworks.com/help/matlab/ref/mex.html) APIs [optional argument](https://www.mathworks.com/help/matlab/ref/mex.html#btw17rw-1-option1optionN) to ask the compiler to look for header files within the `include` folder.

If you want to include a header only library, copy the library headers into the `include` folder within the `cpp` folder.

### Interfacing with a compile time library
Toolbox users do not need these libraries for running the MEX functions, they are required only at compile time, these libraries are often referred to as static libraries. You can place these binaries under platform specific folders within the `library` folder. We recommend using standard names for the platform folders as defined by the [`computer('arch')`](https://www.mathworks.com/help/matlab/ref/computer.html) command in MATLAB. The table below provides a summary of the folder names and file extensions used for static libraries for popular operating systems.

| Platform          | Folder name | Binary Extension | 
| :---------------- | :------     | :------        |
| Linux             | glnxa64     | .a             |
| Windows           | win64       | .lib           |
| ARM Mac           | maca64      | .dylib         |
| Intel Mac         | maci64      | .dylib         |

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


### Execution time libraries
These type of libraries are often referred to as shared object libraries or dynamic link libraries. These libraries are required for running the MEX functions and need to be shipped to the users. You can place the execution time binaries within the `private` folder under the `toolbox` folder, this makes sure that the library gets shipped to the user. You can use the -L and -l flags during compile time to specify the location and the name of the runtime library.


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
* Depending on the platform, execution time libraries require their path to be added to the loader's search path. These search paths are often establish using platform dependent environment variables.  In case of C++ libraries you can use [`mexhost`](https://www.mathworks.com/help/matlab/ref/mexhost.html)'s `EnvironmentVariables` option to set-up the loader's path. Here is a summary of 
loader's search path for different platforms.

| Platform  | Environment variable for loader's search path |
| :-------- | :-------------------------------------------- |
| Linux     | LD_LIBRARY_PATH                               |
| Windows   | PATH                                          |
| Mac       | DYLD_LIBRARY_PATH                             |           

For non C++ libraries, you need to start MATLAB from an environment where the loaders's search path is already established.

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


## Multi platform MEX functions build using CI systems

## Testing

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
```


## Automating Builds with GitHub Actions
**TBD**

## Summary: Organizing a toolbox with MEX functions


## Conclusion

By following to these best practices, you'll create a robust, maintainable, and user-friendly MATLAB toolbox that harnesses the full potential of MEX files. Through effective project structure organization, build automation, and dependency management, you can focus on delivering great solutions to your users.

We welcome your input! For further details, suggestions, or to contribute, please [open an issue](https://github.com/mathworks/toolboxdesign/issues/new/choose).

---
[![CC-BY-4.0](images/cc-by-40.png)](https://creativecommons.org/licenses/by/4.0/)

Copyright &copy; 2023-2025, The MathWorks, Inc.
