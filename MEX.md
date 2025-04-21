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
- **CI/CD Pipelines**: Continuous Integration and Continuous Deployment tools like GitHub Actions or GitLab CI/CD ensure your code is tested and deployed automatically.

## Organizing MEX Source Files
Organize your MEX source files in language-specific folder at the root level of your project (e.g., `cpp`). This structure enhances code organization and simplifies management across multiple programming languages. We recommend using the `cpp` folder at the root of the toolbox repo for both C and C++ source code.

MEX functions are classified into two groups: 
1. Single source MEX function 
2. Multiple source MEX function 

Our folder structure within the language-specific folder (e.g., `cpp`) for the MEX source files is largely based on this classification. For a single source MEX function, the source code file implements the MEX gateway function and other functionalities that are called within the gateway function. A multiple source MEX function on the other hand, has one of its source code files implement the gateway function while the other source files implement features and functionalities that are used in the gateway function. 


For C and C++ single source MEX functions, place the source code within the `mexfunctions` folder within the `cpp` folder. In case of Fortran, create a `fortran` folder within the toolbox root and move the source code for Fortran single source MEX function within the `mexfunctions` folder. Note that a toolbox project can have both `cpp` and `fortran` folders if it has MEX functions written using C/ C++ and Fortran. We also recommend the names of the MEX functions to be same as their  source file (of course with different extensions), this ensures traceability. Moreover, we also recommend the file names be suffixed with 'Mex', this acts as an indication that the function being called is not any MATLAB function but a MEX function.


For multiple source MEX functions, create a folder for each MEX function. The name of the folder should be same as that of the MEX function. In addition, we recommend suffixing the folder name with 'Mex' to indicate that the contents of the folder should be compiled into a single MEX function.    


Our Arithmetic Toolbox example features two MEX functions: `addMex` and `subtractMex`, we have left the MEX function's extension to make the discussion platform agnostic. `addMEX` is implemented as a single source MEX function and placed under the `mexfunctions` folder within the `cpp` folder, while `substractMEX` is implemented as a MEX function with multiple source and the source code that compiles to `substractMEX` is place within the `substractMex` under the `cpp` folder. `substract.cpp` implements the MEX gateway function and `substractImp.cpp` contains implementations that are required for the functionality of `substractMEX` function.

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
- **Tools**: Use MATLAB's [`buildtool`](https://www.mathworks.com/help/matlab/ref/buildtool.html), introduced in R2022b, to automate the MEX build process. The [`matlab.buildtool.tasks.MexTask`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.tasks.mextask-class.html), introduced in R2024a, automates the compilation, ensuring consistency across environments.  If you need support in earlier releases, use the [`mex` command](https://www.mathworks.com/help/matlab/ref/mex.html).


## Organizing MEX Functions
[MATLAB Toolbox Best Practices](README.md) advocates for placing the user files under the `toolbox` folder, contents of this folder gets shipped to the user. For a toolbox that uses MEX, place the MEX functions under a `private` folder within the `toolbox` folder, thus making them private [Private functions](https://www.mathworks.com/help/matlab/matlab_prog/private-functions.html).  

Our motivation for making MEX functions private is to restrict the toolbox user from  calling the MEX function directly. We recommend accessing the MEX functions always from a MATLAB script, this approach gives toolbox authors control over what gets passed as input to the MEX functions, their by elimination unexpected MATLAB crashes.

We recommend ignoring the MEX functions with `private` folder from version control (add the private folder to the `.gitignore`) since MEX functions are derived artifacts.

<!-- - **Compiled Binaries**: Store compiled MEX binaries in the `toolbox/private` folder. This approach maintains internal accessibility while mitigating the risk of crashes from unhandled inputs by preventing direct user access to MEX functions. -->

Our example toolbox shows the built mex functions for several platforms:

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

Here is a `buildfile` with a single task called `mex`, it compile the single and multiple source MEX functions. Note that the `buildfile` was tested on Windows, with MATLAB R2025a.

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


## Incorporating External Libraries
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
