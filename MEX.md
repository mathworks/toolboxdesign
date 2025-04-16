# MATLAB MEX Best Practice

> [!IMPORTANT]
> This page is under construction and review

![Version Number](https://img.shields.io/github/v/release/mathworks/toolboxdesign?label=version) ![CC-BY-4.0 License](https://img.shields.io/github/license/mathworks/toolboxdesign)

Welcome to the MATLAB&reg; MEX Best Practice guide. This document offers a refined approach to seamlessly integrating MEX files into your MATLAB toolbox projects, complementing our established [MATLAB Toolbox Best Practices](README.md). MEX files enable you to harness the power of C /C++/ Fortran functions within MATLAB, and this guide will equip you with the knowledge to maintain a consistent and efficient workflow. Let's explore the world of MEX integration!

## Overview

MEX files bridge the gap between MATLAB and C, C++, or Fortran, allowing you to leverage the strengths of these languages directly within MATLAB. While the integration of MEX files can be intricate, this guide will navigate you through the process, ensuring smooth implementation in both development and production environments.  This organized structure enhances maintainability and makes it easier for others to understand and contribute to your project.

To illustrate these best practices, we've created a sample project: The Arithmetic Toolbox, available on [GitHub](https://github.com/mathworks/arithmetic). We'll reference this project throughout the guide to demonstrate practical applications of these principles.

## Key Concepts
- **MEX Source Files**: These are functions written in C, C++, or Fortran, compiled to be callable from MATLAB.  See the [MEX documentation](https://www.mathworks.com/help/matlab/cpp-mex-file-applications.html) for more information.
- **MEX Functions**: Platform dependent binaries that can be called directly from MATLAB, they behave almost like a MATLAB function. MATLAB uses a platform dependent file extension for MEX functions, these extensions can be determined using [`mexext`](https://www.mathworks.com/help/matlab/ref/mexext.html).
- **MEX Gateway Function**: Written in a MEX supported language, MATLAB accesses this function when a call is made to a MEX function. Each MEX function has only one gateway function. [MEX gateway functions written in C](https://www.mathworks.com/help/matlab/apiref/mexfunction.html) 
- **Compile time binaries**: Static libraries are a good example of these binaries. These library binaries are required only at build time and you need not ship them to your users.
- **Run time binaries**: These are platform dependent binaries, that the users need to run your toolbox, shared object libraries (.so files) in Linux and dynamic link libraries (.dll files) in Windows are good examples of run time binaries.
- **`buildtool`**: MATLAB's tool for automating build processes, including MEX file compilation.  See the [`buildtool` documentation](https://www.mathworks.com/help/matlab/ref/buildtool.html) for more information.
- **CI/CD Pipelines**: Continuous Integration and Continuous Deployment tools like GitHub Actions or GitLab CI/CD ensure your code is tested and deployed automatically.

## MEX Source Files

Organize your MEX source files in language-specific directories at the root level of your project (e.g., `cpp`). This structure enhances code organization and simplifies management across multiple programming languages. We recommend using the `cpp` folder for both C and C++ source code.


Our Arithmetic Toolbox example features two MEX functions: `addMex` and `subtractMex`, which are called by the MATLAB functions `add.m` and `subtract.m`.

``` text
arithmetic/
├───toolbox/
|   ├───add.m
|   └───subtract.m
├───cpp/
│   ├───addMex/
│   │   ├───firstFile.cpp
│   │   └───secondFile.cpp
│   └───subtractMex/
│       └───subtract.cpp
├───arithmetic.prj
└───buildfile.m
```
## MEX functions and runtime binaries
[MATLAB Toolbox Best Practices](README.md) advocates for placing the files needed by the user to run the toolbox under the `toolbox` folder. Contents of this folder is what gets shipped to the user. For a toolbox that uses MEX, we recommend the MEX functions be placed under a `private` folder within the `toolbox` folder, thus making these MEX functions [Private](https://www.mathworks.com/help/matlab/matlab_prog/private-functions.html).  The content of the `private` folder can be ignored from version control (add the private folder to the .gitignore) since MEX functions are derived artifacts. In addition to MEX functions, the `private` folder can also have other binary files that are required at runtime.

Improper use of MEX functions can lead to MATLAB crashing. An effective safeguard against this failure is to prevent direct access to MEX functions. We recommended calling the MEX function from a MATLAB script, by doing so the author can validate the inputs from a MATLAB script before passing it to the MEX function. Placing the the MEX functions within a private folder only facilitates this, since a private folder cannot be added to MATLAB path directly and only MATLAB scripts placed within the parent folder of a private folder can access its contents.



## Building MEX Files
- **Tools**: Use MATLAB's [`buildtool`](https://www.mathworks.com/help/matlab/ref/buildtool.html), introduced in R2022b, to automate the MEX build process. The [`matlab.buildtool.tasks.MexTask`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.tasks.mextask-class.html), introduced in R2024a, automates the compilation, ensuring consistency across environments.  If you need support in earlier releases, use the [`mex` command](https://www.mathworks.com/help/matlab/ref/mex.html).
- **Compiled Binaries**: Store compiled MEX binaries in the `toolbox/private` folder. This approach maintains internal accessibility while mitigating the risk of crashes from unhandled inputs by preventing direct user access to MEX functions.

Our example toolbox shows the built mex functions for several platforms:

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
│       └───subtract.cpp
├───arithmetic.prj
└───buildfile.m 
```

Example buildfile.m:
``` matlab
TBD
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
