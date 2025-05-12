# MATLAB MEX Best Practice

> [!IMPORTANT]
> This page is under construction and review

![Version Number](https://img.shields.io/github/v/release/mathworks/toolboxdesign?label=version) ![CC-BY-4.0 License](https://img.shields.io/github/license/mathworks/toolboxdesign)

<!-- RP: I think we should assume that the user is coming from the main best practice. Position this as a sub-document. We should avoid LLM excitement "Let's explore the world of MEX integration!" -->

_This is a sub document of the Toolbox Best practices._
Welcome to the MATLAB&reg; MEX Best Practice guide. This document offers a refined approach to seamlessly integrating MEX files into your MATLAB toolbox projects, complementing our established [MATLAB Toolbox Best Practices](README.md). MEX files enable you to harness the power of C/C++/Fortran functions within MATLAB, and this guide will equip you with the knowledge to maintain a consistent and efficient workflow. _We're going show you how to...._

## Overview

[MEX functions](https://www.mathworks.com/help/matlab/call-mex-file-functions.html) are compiler code that bridges the gap between MATLAB and C, C++, or Fortran. They behave like a MATLAB function and are platform dependent. You can determine the MEX extension for your platform using [`mexext`](https://www.mathworks.com/help/matlab/ref/mexext.html). While the integration of MEX files can be intricate, this guide will navigate you through the process, ensuring smooth implementation in both development and production environments.  This makes it easier for others to understand and contribute to your project.

To illustrate these best practices, we've created a sample project: The Arithmetic Toolbox, available on [GitHub](https://github.com/mathworks/arithmetic). We'll reference this project throughout the guide to demonstrate practical applications of these principles.  To make things clearer, when we say "C++", we mean "C, C++, and Fortran." For Key Concepts please refer to [Toolbox Best Practices](./README.md).

<!-- RP: Move this to the intro
     BP: Moved MEX functions to Overview -->
_See [Toolbox Best Practices] for other concepts._

<!-- - **MEX Functions**: Compiled code that can be called directly from MATLAB. They behave like a MATLAB function. MEX files are platform dependent binaries, so there is a platform dependent file extension. You can determine the MEX extension for your platform using [`mexext`](https://www.mathworks.com/help/matlab/ref/mexext.html). -->
<!-- - **MEX Gateway Function**: Written in a MEX supported language, MATLAB accesses this function when a call is made to the MEX function. Each MEX function can have one gateway function only. [MEX gateway functions written in C](https://www.mathworks.com/help/matlab/apiref/mexfunction.html)  -->
<!-- - **Compile time binaries**: Static libraries are a good examples of compile time binaries. These library binaries are required only at build time and you need not ship them to your users. -->
<!-- - **Run time binaries**: These are platform dependent binaries, that the users need to run your toolbox, shared object libraries (.so files) in Linux and dynamic link libraries (.dll files) in Windows are good examples. -->
<!-- RP: Maybe introduce this later -->
<!-- - **`buildtool`**: MATLAB's tool for automating build processes, including MEX file compilation.  See the [`buildtool`](https://www.mathworks.com/help/matlab/matlab_prog/overview-of-matlab-build-tool.html) for more information. -->
<!-- RP: save this till we use it -->
<!-- -  **MEX compiler**: Converts MEX source files into MEX functions. The MEX compiler can be accessed from MATLAB via the [`mex`](https://www.mathworks.com/help/matlab/ref/mex.html) command, it can be invoked for the system terminal via the same command. You can configure C/ C++/ Fortran compilers using the [mex -setup](https://www.mathworks.com/help/matlab/matlab_external/changing-default-compiler.html). -->
<!-- - **Project Root**: The folder under which the toolbox source code is organized. A git repository is initialized under this folder. -->
<!-- - **CI/CD Pipelines**: Continuous Integration and Continuous Deployment tools like GitHub Actions or GitLab CI/CD ensure your code is tested and deployed automatically. -->

## Creating a MEX function from a single C++ source file
<!-- RP: Recast this titles to be user centric. Answer the question "What goal does this section help the user achieve?" -->
<!-- RP: let's focus on a single C++ / MEX scenario first (add), then have a closing section that talks about multiple single C++ file MEX functions. -->
Suppose you have several C++ MEX source files, each implementing its own MEX gateway and not depending on an external library or other source files, you want to organize these files in a way that makes building, integrating and sharing the MEX functions easy. Since the C++ source files need not be distributed to the toolbox users, we recommend placing them outside the `toolbox` folder. Create a folder named `cpp` under the repository root.  Within this folder, create a subfolder called `mexfunctions` and place the single source MEX functions source code within this folder. 

<!-- RP: For clarity's sake, let's not get mixed up in add and subtract -- let's create something new, like multiply.

Be able to say "We are going to create the invertMex function, which on Windows will be invertMex.mexw64." -->

To illustrate this organization, we will use the arithmetic repository as an example. For demonstration purposes we will reimplement the internals of `add()` and `substruct()` using MEX functions. We have created two MEX source files: `addMex.cpp` and `substractMex.cpp`. The names of the C/C++ source files are chosen to match the  the MEX functions complied out of the source files. For example, on Windows platform, `addMex.cpp` would be complied to `addMex.mexw64` and `addMex` will be the name of the MEX function . 
This naming convention gives us
*  makes it easy to associate each MEX function with its source code.  
* the "Mex" suffix in the MEX function file name clearly indicates that the function is a MEX function rather than a standard MATLAB function.

The organization of the MEX source files for the arithmetic toolbox relative to the toolbox folder is shown below:
<!-- RP: Follow the langauge of the best practice -->
``` text
arithmetic/
├───cpp/
│   └───mexfunctions/
|       ├───substractMex.cpp
│       └───addMex.cpp
├───toolbox/
|   ├───add.m
|   └───subtract.m
├───arithmetic.prj
└───buildfile.m
```

### Building MEX functions
<!-- RP: I think we're introducing buildtool too early. Show the basic mex solution first. -->
We recommend using MATLAB's [`buildtool`](https://www.mathworks.com/help/matlab/ref/buildtool.html), introduced in R2022b for automating MEX functions build. 

[MATLAB Toolbox Best Practices](README.md) advocates for placing the user files under the `toolbox` folder, contents of this folder gets shipped to the user. Create a `private` folder within the `toolbox` folder and place the MEX functions within this folder, thus making them [Private functions](https://www.mathworks.com/help/matlab/matlab_prog/private-functions.html).  

Our motivation for making MEX functions private is to restrict the toolbox user from  calling them directly. We recommend accessing MEX functions always from a MATLAB script, this approach gives toolbox authors control over what gets passed as input to the MEX functions, their by elimination failures due to unhandled inputs.

We recommend ignoring the MEX functions from version control (add the `private` folder to the `.gitignore`) since MEX functions are derived artifacts. Here is the organization of the toolbox repository after building the MEX functions.

``` text
arithmetic/
├───cpp/
│   └───mexfunctions/
|       ├───substractMex.cpp
│       └───addMex.cpp
├───toolbox/
|   ├───private/
|   |   ├───substractMex.mexw64
│   |   └───addMex.mexw64
|   ├───add.m
|   └───subtract.m
├───arithmetic.prj
└───buildfile.m
```
<!-- RP: Introduce Buildtool here. -->
## Expanding to create multiple MEX functions
_Insert the add and subtract example here_

### Automating using `buildtool`
<!-- RP: I think we need a summary of why this is a good idea. -->
Here is a simple buildfile that can be used to build single source MEX functions: 
``` matlab
function plan = buildfile

% Create a plan 
plan = buildplan();

% Compile all the .cpp files inside cpp/mexfunctions into MEX functions
mexSourceFiles = files(plan, fullfile("cpp", "mexfunctions", "*.cpp"));
mexOutputFolder = fullfile("toolbox","private");
plan("mex") = matlab.buildtool.tasks.MexTask.forEachFile(mexSourceFiles, mexOutputFolder);

end
```
<!-- If you want to build mex pre 25a look at arith -->

In the build file, we create a [`plan`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.plan-class.html) and add a [`MexTask`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.tasks.mextask-class.html) to the it. The [`matlab.buildtool.tasks.MexTask.forEachFile`](https://www-jobarchive.mathworks.com/Bdoc/latest_pass/matlab/help/matlab/ref/matlab.buildtool.tasks.mextask.foreachfile.html) API, introduced in R2025a, converts every C++ file within the specified folder into MEX functions. [`MexTask.forEachFile`](https://www-jobarchive.mathworks.com/Bdoc/latest_pass/matlab/help/matlab/ref/matlab.buildtool.tasks.mextask.foreachfile.html) takes a [`FileCollection`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.io.filecollection-class.html) as input, which can be created using the [`files`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.plan.files.html) API. 

<!-- RP: put this motivation earlier -- we need to advocate for buildtool first, before we show how. -->
The buildfile has some advantages,

1. *Incremental build:* By using `MexTask` instead of the `mex` command, you automatically get support for incremental build, recompiling only the files that have changed.
2. *No hardcoding of source files:* Since the build file references to the folder containing the MEX source files and not the source files directly, the build file can scale to handle any number of single-source MEX functions, as long as the source files are placed within the cpp/mexfunctions folder.
3. *Platform independence:* Although MEX functions themselves are platform dependent, the build file does not rely on any platform-specific information to compile them, making it possible to use the same build file across different platforms.

### Multi platform MEX functions build using CI systems


## Creating a MEX function from multiple C++ source files

It is common to have MEX functions generated from multiple C++ source files, we call these MEX functions multiple source MEX functions. For such MEX functions one of the source files implement the gateway function and other source files implement methods that are called within the gateway function. For multiple source MEX functions, create a folder for each MEX function within the `cpp` folder. The name of the folder should be same as that of the MEX function that will be created out for source code with the folder. The folder name can be suffixed with 'Mex' to indicate that the contents of the folder should be compiled into a single MEX function. 


For the sake of discussion let us implement substractMex MEXfunction as a Multiple source MEX function. The organization of the source files is shown below.

``` text
arithmetic/
├───cpp/
│   ├───substractMex/
│   │   ├───substract.cpp % Implements gateway function
│   │   └───substractImp.cpp % other features
│   └───mexfunctions/
│       └───addMex.cpp
├───toolbox/
|   ├───add.m
|   └───subtract.m
├───arithmetic.prj
└───buildfile.m
```


## Organizing MEX Source Files
Organize your MEX source files in language-specific folder at the root level of your project (e.g., `cpp`). This structure enhances code organization and simplifies management across multiple programming languages. We recommend using the `cpp` folder at the root of the toolbox repo for both C and C++ source code.

MEX functions are classified into two groups: 
1. Single source MEX function 
2. Multiple source MEX function 

Our folder structure within the language-specific folder (e.g., `cpp`) for the MEX source files is largely based on this classification. For a single source MEX function, the source code file implements the MEX gateway function and other functionalities that are called within the gateway function. A multiple source MEX function on the other hand, has one of its source code files implement the gateway function while the other source files implement features and functionalities that are used in the gateway function. 

For C and C++ single source MEX functions, place the source code within the `mexfunctions` folder within the `cpp` folder. In case of Fortran, create a `fortran` folder within the toolbox root and move the source code for Fortran single source MEX function within the `mexfunctions` folder. Note that a toolbox project can have both `cpp` and `fortran` folders if it has MEX functions written using C/ C++ and Fortran. We also recommend the names of the MEX functions to be same as their  source file (of course with different extensions), this ensures traceability. Moreover, we also recommend the file names be suffixed with 'Mex', this acts as an indication that the function being called is not any MATLAB function but a MEX function.


For multiple source MEX functions, create a folder for each MEX function within the respective language folder. The name of the folder should be same as that of the MEX function that will be created out for source code with the folder. The folder name can be suffixed with 'Mex' to indicate that the contents of the folder should be compiled into a single MEX function, with the same name as the folder.    


Our [Arithmetic Toolbox]() example features two MEX functions: `addMex` and `subtractMex`. `addMex` is implemented as a single source MEX function with the source code placed under the `mexfunctions` folder. On the other hand, `substractMex` is implemented as a multiple source MEX function with source code placed under the `substractMex` folder, `substract.cpp` implements the MEX gateway function for the `substractMex` MEX function and `substractImp.cpp`  implementations functionalities that are used in `substract.cpp`. The organization of the MEX source files is shown below:

``` text
arithmetic/
├───cpp/
│   ├───substractMex/
│   │   ├───substract.cpp % Implements gateway function
│   │   └───substractImp.cpp % other features
│   └───mexfunctions/
│       └───addMex.cpp
├───toolbox/
|   ├───add.m
|   └───subtract.m
├───arithmetic.prj
└───buildfile.m
```




## Organizing MEX Functions
[MATLAB Toolbox Best Practices](README.md) advocates for placing the user files under the `toolbox` folder, contents of this folder gets shipped to the user. For a toolbox that uses MEX, place the MEX functions under a `private` folder within the `toolbox` folder, thus making them private [Private functions](https://www.mathworks.com/help/matlab/matlab_prog/private-functions.html).  

Our motivation for making MEX functions private is to restrict the toolbox user from  calling them directly. We recommend accessing the MEX functions always from a MATLAB script, this approach gives toolbox authors control over what gets passed as input to the MEX functions, their by elimination failures due to unhandled inputs.

We recommend ignoring the MEX functions from version control (add the `private` folder to the `.gitignore`) since MEX functions are derived artifacts.

<!-- - **Compiled Binaries**: Store compiled MEX binaries in the `toolbox/private` folder. This approach maintains internal accessibility while mitigating the risk of crashes from unhandled inputs by preventing direct user access to MEX functions. -->

<!-- Our example toolbox shows the MEX functions for several platforms: -->

The layout of the files with the `private` folder is shown below, for the sake of clarity we have shown the MEX functions for all the major platforms. Not that the GitHub repository for the Arithmetic toolbox will not contain the MEX functions since they are ignored by the version control system. You can create the MEX functions by running the following command from MATLAB terminal:
``` matlab
>> buildtool mex
```

``` text
arithmetic/
:
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

Here is a `buildfile` with a single task called `mex`,  this task compiles the single and multiple source MEX functions and places them within the `private` folder. Note that the `buildfile` was tested on MATLAB R2025a.

``` matlab
function plan = buildfile

import matlab.buildtool.tasks.*
import matlab.buildtool.io.*

% Create a plan 
plan = buildplan();

% Compile all the .cpp files inside cpp/mexfunctions into MEX functions
mexSourceFiles = files(plan, fullfile("cpp", "mexfunctions", "*.cpp"));
mexOutputFolder = fullfile("toolbox","private")l;
plan("mex") = MexTask.forEachFile(mexSourceFiles, mexOutputFolder);

end
```

### Out of process MEX host
If you are using MEX functions written using the C++, we recommend calling the MEX function from an out of process MEX host. You can create an out of process MEX host using the [mexhost](https://www.mathworks.com/help/matlab/ref/mexhost.html) command. It protects MATLAB from crashing due to unexpected errors originating from the MEX function execution.


## Incorporating External Libraries
One of the key strengths of MEX functions is their ability to call  libraries implemented in languages like C, C++ and Fortran. Since MEX source files are just source code written in a MEX supported language, they use the source language syntax to access functionality implemented in external libraries. One of the common questions with using external libraries in you toolbox work is, where to store these external libraries?

The answer to this question depends on the external library and the platform with which you are working with. For the purposes of this best practices we classify the external libraries into compile time and execution time libraries. 
 
We will assume that the external library is available to you as include headers and binaries. We do not dwell into the details of how these headers and binaries are created. First let as talk a bit about organizing 

### External library headers
These files are required only at compile time, your toolbox users do not want them to run the MEX functions. Having a standard location to store these headers makes it easier for you (the author) to manage them and pass it to the compiler.

Create an `include` folder within the the language specific folder (say `cpp`) and move the external library headers within this folder. The [`mex`](https://www.mathworks.com/help/matlab/ref/mex.html) MATLAB API takes in an [optional argument](https://www.mathworks.com/help/matlab/ref/mex.html#btw17rw-1-option1optionN) that can be used to inform the compiler to look for header files within the `include` folder.

If you want to include a header only library in your work, copy the library headers into the `include` folder within the respective language folders.



### Compile time libraries
The toolbox user does not need these library binaries to run the MEX functions  they are required only at compile time, these libraries are often referred to as static libraries. You can place these binaries under platform specific folders within the `library` folder. We recommend using standard names for the platform folders as defined by the [`computer('arch')`](https://www.mathworks.com/help/matlab/ref/computer.html) command in MATLAB. The table below provides a summary of the folder names and file extensions used for static libraries for popular platforms.

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
