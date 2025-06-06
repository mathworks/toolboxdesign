# MATLAB MEX Best Practice

> [!IMPORTANT]
> This page is under construction and review

**TODO:**

- [X] Review and revise overview.  
- [X] Review and revise single source file section.  
- [X] Review and revise buildtool section.  
- [ ] Update `buildfile.m` in buildtool section to work with 24a(24a does not have task collection)
- [X] Review and update multiple source file section
- [X] Review and update `mexfunction` folder scenario section
- [ ] Review and update external libraries section
- [ ] Review and update CI / GitHub Actions section

![Version Number](https://img.shields.io/github/v/release/mathworks/toolboxdesign?label=version) ![CC-BY-4.0 License](https://img.shields.io/github/license/mathworks/toolboxdesign)

Welcome to the MATLAB&reg; MEX Best Practice guide, which extends [MATLAB Toolbox Best Practices](./README.md) focusing on  integrating [MEX functions](https://www.mathworks.com/help/matlab/call-mex-file-functions.html) into your MATLAB toolbox. MEX functions enable you to harness the power of C, C++, and Fortran code within MATLAB. In this document when we say "C++", we mean "C, C++, and Fortran."

## Overview

[MEX functions](https://www.mathworks.com/help/matlab/call-mex-file-functions.html) are compiled functions that bridge the gap between MATLAB and C++. They behave like a MATLAB function and you must build them for each operating system you want to run on. You can determine the MEX file extension (for example, `.mexw64` in Microsoft Windows) for your operating system using [`mexext`](https://www.mathworks.com/help/matlab/ref/mexext.html). This guide will navigate you through the process of integrating of MEX functions, ensuring smooth implementation in both development and production environments.  This makes it easier for others to understand and contribute to your toolbox.

To illustrate these best practices, we've created a sample project: The Arithmetic Toolbox, available on [GitHub](https://github.com/mathworks/arithmetic). We'll reference this toolbox throughout the guide to demonstrate practical applications of these principles.  For key concepts please refer to the [Toolbox Best Practices](./README.md).

## MEX function from a single C++ source file
Suppose you have a single C++ MEX source file, this MEX source file need not be distributed to the toolbox users, since they are not required to run the toolbox. Only the compiled function needs to be distributed.  We recommend keeping the C++ MEX source files outside of the `toolbox` folder in a folder focused on C++ code, `cpp`. 

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

You can use the [`mex`](https://www.mathworks.com/help/matlab/ref/mex.html) command, to compile `invertMex.cpp` into MEX functions. We suggest you place your compiled MEX functions in your `private` folder within the `toolbox` folder (see below). You need to provide the path of `invertMex.cpp` and path of the `private` folder as inputs to the `mex` command. 

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
 * **Why put the MEX functions within a private folder?** By putting it in a [private](https://www.mathworks.com/help/matlab/matlab_prog/private-functions.html) folder, you restrict your users from calling the MEX function directly. Even minor errors in a MEX function will crash MATLAB, especially if they receive unexpected inputs. By limiting access to MEX functions to a MATLAB function that you control, you ensure that only what you expect will be passed as input to the MEX function, preventing errors from unexpected or unhandled inputs.
 * **Out of process MEX host** We recommend [Out-of-Process Execution of C++ MEX Functions](https://www.mathworks.com/help/matlab/matlab_external/out-of-process-execution-of-c-mex-functions.html). This isolates the MATLAB process from crashes in you C++ MEX function and allows you to use some third-party libraries that are not compatible with MATLAB.  Use the [mexhost](https://www.mathworks.com/help/matlab/ref/mexhost.html) command to do this. Note that `mexhost` is only supported for C++ MEX functions. 
 * **Using git** In git source control systems, we recommend that you do not keep compiled MEX functions under version control, as they are derived files. Add `*.mex*` to your `.gitignore` file. This is part of the [standard .gitignore file](https://github.com/mathworks/gitignore/blob/main/Global/MATLAB.gitignore) for MATLAB.

### Automation using `buildtool`

Running the `mex` command for each MEX function can be tedious and error prone. MATLAB’s [`buildtool`](https://www.mathworks.com/help/matlab/ref/buildtool.html), introduced in R2022b, can automate this and many other repetative processes for you. The following `buildfile.m` creates a [`MexTask`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.tasks.mextask-class.html), introduced in R2024a, that builds your MEX functions.

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
<!-- In the build file, we create a [`plan`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.plan-class.html) and add a [`MexTask`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.tasks.mextask-class.html) to the it. The [`matlab.buildtool.tasks.MexTask.forEachFile`](https://www-jobarchive.mathworks.com/Bdoc/latest_pass/matlab/help/matlab/ref/matlab.buildtool.tasks.mextask.foreachfile.html) API, introduced in R2025a, converts every C++ file within the specified folder into MEX functions. [`MexTask.forEachFile`](https://www-jobarchive.mathworks.com/Bdoc/latest_pass/matlab/help/matlab/ref/matlab.buildtool.tasks.mextask.foreachfile.html) takes a [`FileCollection`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.io.filecollection-class.html) as input. The  [`files`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.plan.files.html) API can be used to create a `FileCollection`. -->

Our example toolbox adds a `buildfile.m` to automate the building of the MEX functions:

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

Tha above pattern naturally extends to MEX functions that has multiple C++ source files. One of the source files must contain the [gateway](https://www.mathworks.com/help/matlab/matlab_external/gateway-routine.html) function. Place a the source files under a single folder.

<!-- - Create a folder for each MEX function within the `cpp` folder 
- Name of the folder should be same as that of the MEX function 
- Folder name is suffixed with 'Mex' to indicate that the contents of the folder should be compiled into a single MEX function.  -->

Our example toolbox adds two .cpp files in the  `subtractMex` folder, which builds to the `subtractMex.mexw64` binary in the `private` folder.  `subtractNumber.m` provides a user accessible version that error checks:

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
│       └───divideMex.cpp
├───toolbox/
|   ├───private/
|   |   ├───powerMex.mexw64
|   |   └───divideMex.mexw64
|   ├─...
|   ├───powerNumber.m
|   └───divideNumber.m
├───arithmetic.prj
└───buildfile.m
```
The build file for building these MEX functions is slightly different. 

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

## Incorporating External Libraries

<!-- - **Compile time binaries**: Static libraries are a good examples of compile time binaries. These library binaries are required only at build time and you need not ship them to your users. -->
<!-- - **Run time binaries**: These are platform dependent binaries, that the users need to run your toolbox, shared object libraries (.so files) in Linux and dynamic link libraries (.dll files) in Windows are good examples. -->
<!-- RP: save this till we use it -->
<!-- - **CI/CD Pipelines**: Continuous Integration and Continuous Deployment tools like GitHub Actions or GitLab CI/CD ensure your code is tested and deployed automatically. -->

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

By following to these best practices, you'll create a robust, maintainable, and user-friendly MATLAB toolbox that harnesses the full potential of MEX functions. Through effective project structure organization, build automation, and dependency management, you can focus on delivering great solutions to your users.

We welcome your input! For further details, suggestions, or to contribute, please [open an issue](https://github.com/mathworks/toolboxdesign/issues/new/choose).

---
[![CC-BY-4.0](images/cc-by-40.png)](https://creativecommons.org/licenses/by/4.0/)

Copyright &copy; 2023-2025, The MathWorks, Inc.
