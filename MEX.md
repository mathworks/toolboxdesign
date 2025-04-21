# MATLAB MEX Best Practice

> [!IMPORTANT]
> This page is under construction and review

![Version Number](https://img.shields.io/github/v/release/mathworks/toolboxdesign?label=version) ![CC-BY-4.0 License](https://img.shields.io/github/license/mathworks/toolboxdesign)

Welcome to the MATLAB&reg; MEX Best Practice guide. This document offers a refined approach to seamlessly integrating MEX files into your MATLAB toolbox projects, complementing our established [MATLAB Toolbox Best Practices](README.md). MEX files enable you to harness the power of C /C++/ Fortran functions within MATLAB, and this guide will equip you with the knowledge to maintain a consistent and efficient workflow. Let's explore the world of MEX integration!

## Overview

MEX functions bridge the gap between MATLAB and C, C++, or Fortran, allowing you to leverage the strengths of these languages directly within MATLAB. While the integration of MEX files can be intricate, this guide will navigate you through the process, ensuring smooth implementation in both development and production environments.  This organized structure enhances maintainability and makes it easier for others to understand and contribute to your project.

To illustrate these best practices, we've created a sample project: The Arithmetic Toolbox, available on [GitHub](https://github.com/mathworks/arithmetic). We'll reference this project throughout the guide to demonstrate practical applications of these principles.

## Key Concepts
- **MEX Source Files**: These are functions written in C, C++, or Fortran, compiled to be callable from MATLAB.  See the [MEX documentation](https://www.mathworks.com/help/matlab/cpp-mex-file-applications.html) for more information.
- **MEX Functions**: Platform dependent binaries that can be called directly from MATLAB, they behave almost like a MATLAB function. MATLAB uses a platform dependent file extension for MEX functions, you can determine the MEX extension for your platform using [`mexext`](https://www.mathworks.com/help/matlab/ref/mexext.html).
- **MEX Gateway Function**: Written in a MEX supported language, MATLAB accesses this function when a call is made to the MEX function. Each MEX function can have one gateway function only. [MEX gateway functions written in C](https://www.mathworks.com/help/matlab/apiref/mexfunction.html) 
- **Compile time binaries**: Static libraries are a good examples of compile time binaries. These library binaries are required only at build time and you need not ship them to your users.
- **Run time binaries**: These are platform dependent binaries, that the users need to run your toolbox, shared object libraries (.so files) in Linux and dynamic link libraries (.dll files) in Windows are good examples.
- **`buildtool`**: MATLAB's tool for automating build processes, including MEX file compilation.  See the [`buildtool` documentation](https://www.mathworks.com/help/matlab/ref/buildtool.html) for more information.
-  **MEX compiler**: Converts MEX source files into MEX functions. The MEX compiler can be accessed from MATLAB via the [`mex`](https://www.mathworks.com/help/matlab/ref/mex.html) command, it can be invoked for the system terminal via the same command. You can configure C/ C++/ Fortran compilers using the [mex -setup](https://www.mathworks.com/help/matlab/matlab_external/changing-default-compiler.html).
- **Project Root**: The folder under which the toolbox source code is organized. A git repository is initialized under this folder.
- **CI/CD Pipelines**: Continuous Integration and Continuous Deployment tools like GitHub Actions or GitLab CI/CD ensure your code is tested and deployed automatically.

## Organizing MEX Source Files
Organize your MEX source files in language-specific folder at the root level of your project (e.g., `cpp`). This structure enhances code organization and simplifies management across multiple programming languages. We recommend using the `cpp` folder at the root of the toolbox repo for both C and C++ source code.

MEX functions are classified into two groups: 
1. Single source MEX function 
2. Multiple source MEX function 

Our folder structure within the language-specific folder (e.g., `cpp`) for the MEX source files is largely based on this classification. For a single source MEX function, the source code file implements the MEX gateway function and other functionalities that are called within the gateway function. A multiple source MEX function on the other hand, has one of its source code files implement the gateway function while the other source files implement features and functionalities that are used in the gateway function. 

For C and C++ single source MEX functions, place the source code within the `mexfunctions` folder within the `cpp` folder. In case of Fortran, create a `fortran` folder within the toolbox root and move the source code for Fortran single source MEX function within the `mexfunctions` folder. Note that a toolbox project can have both `cpp` and `fortran` folders if it has MEX functions written using C/ C++ and Fortran. We also recommend the names of the MEX functions to be same as their  source file (of course with different extensions), this ensures traceability. Moreover, we also recommend the file names be suffixed with 'Mex', this acts as an indication that the function being called is not any MATLAB function but a MEX function.


For multiple source MEX functions, create a folder for each MEX function within the respective language folder. The name of the folder should be same as that of the MEX function that will be created out for source code with the folder. The folder name can be suffixed with 'Mex' to indicate that the contents of the folder should be compiled into a single MEX function, with the same name as the folder.    


Our [Arithmetic Toolbox]() example features two MEX functions: `addMex` and `subtractMex`. `addMex` is implemented as a single source MEX function and the source code is placed under the `mexfunctions` folder. On the other hand, `substractMex` is implemented as a multiple source MEX function with source code placed under the `substractMex` folder, `substract.cpp` implements the MEX gateway function for the `substractMex` MEX function and `substractImp.cpp`  implementations functionalities that are used in `substract.cpp`. The organization of the MEX source files is shown below:

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
## Building MEX Functions
- **Tools**: Use MATLAB's [`buildtool`](https://www.mathworks.com/help/matlab/ref/buildtool.html), introduced in R2022b, to automate the MEX build process. The [`matlab.buildtool.tasks.MexTask`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.tasks.mextask-class.html), introduced in R2024a, automates the compilation, ensuring consistency across environments.  If you need support in earlier releases, use the [`mex`](https://www.mathworks.com/help/matlab/ref/mex.html) command directly. 


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

### Call MEX functions from a mexhost 


## Incorporating External Libraries
One of the key strengths of MEX functions is their ability to call  libraries implemented in languages like C, C++ and Fortran. Since MEX source files are just source code written in a MEX supported language, they use the source language syntax to access functionality implemented in other libraries. One of the common questions with using external libraries in you toolbox work is, where to store these external libraries?

The answer to this question depends on if the library is required at **compile** or **run** time. For the purposes of this best practices we will assume that the external library is available to you as a include headers and binaries. We do not dwell into the details of how these binaries are created.

### External library headers
These files are required only at compile time, your toolbox users do not want them to run the MEX functions. Having a standard location to store these headers makes it easier for you (the author) to manage them and pass it to the compiler.

Create an `include` folder within the the language specific folder (say `cpp`) and move the external library headers within this folder. The [`mex`](https://www.mathworks.com/help/matlab/ref/mex.html) MATLAB API takes in an [optional argument](https://www.mathworks.com/help/matlab/ref/mex.html#btw17rw-1-option1optionN) that can be used to inform the compiler to look for header files within the `include` folder.



### Managing compile time library binaries
The toolbox user does not need these libraries to run the MEX functions  they are required only at compile time, these libraries are often referred to as static libraries. You can place these binaries under platform specific folders within the `library` folder. We recommend using standard names for the platform folders. The table below provides a summary of the folder names and file extensions used for static libraries for popular platforms

| Platform          | Folder name | File Extension | 
| :---------------- | :------     | :------        |
| Linux             | glnx64      | .a             |
| Windows           | win64       | .lib           |
| Mac ARM           | maca64      | .dylib         |


* When your MEX function relies on external libraries, store the binaries in a `libraries` directory with platform-specific subdirectories, as defined by the [`computer('arch')`](https://www.mathworks.com/help/matlab/ref/computer.html) command in MATLAB. 
* For projects with complex dependencies, consider adopting dependency management tools like [Conan](https://conan.io/) which can significantly simplify library management across different platforms.
* During the build process, copy the external libraries into the `toolbox/private` folder to circumvent path management issues on Windows and `LD_LIBRARY_PATH` concerns on Linux.

Our example toolbox adds a library called `complex` to the `subtractMex` function:

``` text
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
```

## Testing

Develop tests for the MATLAB layer in the `tests/` folder. While MEX functionality is indirectly tested through MATLAB scripts, this approach ensures comprehensive validation of your toolbox's capabilities.

Our example toolbox adds `testAdd.m` and `testSubtract.m`to validate the functionality of `add.m` and `subtract.m` which, in turn, use `addMex` and `subtractMex`.

``` text
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
├───tests/
|   ├───testAdd.m
|   └───testSubstract.m
├───arithmetic.prj
└───buildfile.m 
```


## Automating Builds with GitHub Actions
**TBD**

## Conclusion

By following to these best practices, you'll create a robust, maintainable, and user-friendly MATLAB toolbox that harnesses the full potential of MEX files. Through effective project structure organization, build automation, and dependency management, you can focus on delivering great solutions to your users.

We welcome your input! For further details, suggestions, or to contribute, please [open an issue](https://github.com/mathworks/toolboxdesign/issues/new/choose).

---
[![CC-BY-4.0](images/cc-by-40.png)](https://creativecommons.org/licenses/by/4.0/)

Copyright &copy; 2023-2024, The MathWorks, Inc.
