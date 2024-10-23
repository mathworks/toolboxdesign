# MATLAB MEX Best Practice

> [!IMPORTANT]
> This page is under construction and review

![Version Number](https://img.shields.io/github/v/release/mathworks/toolboxdesign?label=version) ![CC-BY-4.0 License](https://img.shields.io/github/license/mathworks/toolboxdesign)

Welcome to the MATLAB&reg; MEX Best Practice guide. This document provides a streamlined approach to integrating MEX files into your MATLAB toolbox projects, aligning with our existing [MATLAB Toolbox Best Practices](README.md). MEX files provide a mechanism for you to call a C or C++ function from within MATLAB. This guide will help you maintain a consistent and efficient workflow.  Let's dive in!

## Overview

MEX files allow you to call C, C++, or Fortran code directly from MATLAB, offering access to the power of C and C++. However, managing these files can be complex. This guide will help you integrate MEX files effectively in both development and production environments.

To make it easier to follow, we’ve created a fictitious toolbox for doing basic arithmetic: The Arithmetic Toolbox on [GitHub](https://github.com/mathworks/arithmetic). We’ll use this throughout to show how to apply these best practices. 

## Key Concepts

- **MEX Files**: These are functions written in C, C++, or Fortran, compiled to be callable from MATLAB.  See the [MEX documentation](https://www.mathworks.com/help/matlab/cpp-mex-file-applications.html) for more information.
- **`buildtool`**: MATLAB's tool for automating build processes, including MEX file compilation.  See the [`buildtool` documentation](https://www.mathworks.com/help/matlab/ref/buildtool.html) for more information.
- **CI/CD Pipelines**: Continuous Integration and Continuous Deployment tools like GitHub Actions or GitLab CI/CD ensure your code is tested and deployed automatically.

## MEX Source Files

Put your MEX source files in a language-specific directory at the root level of your project, such as `cpp`. This structure keeps your code organized and makes it easier to manage multiple programming languages.  

Our example toolbox has two MEX functions:  `addMex` and `subtractMex` which are called by the MATLAB functions `add.m` and `subtract.m`.

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

## Building MEX Files
- **Tools**: Use MATLAB's [`buildtool`](https://www.mathworks.com/help/matlab/ref/buildtool.html), introduced in R2022b, to automate the MEX build process. The [`matlab.buildtool.tasks.MexTask`](https://www.mathworks.com/help/matlab/ref/matlab.buildtool.tasks.mextask-class.html), introduced in R2024a, automates the compilation, ensuring consistency across environments.  If you need support in earlier releases, use the [`mex` command](https://www.mathworks.com/help/matlab/ref/mex.html).
- **Compiled Binaries**: Place compiled MEX binaries in the `toolbox/private` folder. This keeps them internal and reduces risk from unhandled inputs, as MEX functions should not be directly accessible to end users.

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
 If your MEX function uses external libraries, check the binaries into the repository under a `libraries` directory with subdirectories for different platforms.  For more advanced scenarios, consider using tools like [Conan](https://conan.io/) to manage dependencies. During the build, copy necessary libraries into the `toolbox/private` folder to avoid managing path issues on Windows, or `LD_LIBRARY_PATH` issues on Linux.

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

Write tests for the MATLAB layer in the `tests/` folder. MEX functionality is indirectly tested through MATLAB scripts:

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

## Using GitHub Actions to Automate Builds
**TBD**

## Conclusion

Following these best practices ensures a robust, maintainable, and user-friendly MATLAB toolbox that leverages the power of MEX files. By organizing your project structure, automating builds, and managing dependencies effectively, you can focus on delivering high-performance solutions to your users.

For more details, suggestions, or to contribute, feel free to [open an issue](https://github.com/mathworks/toolboxdesign/issues/new/choose).

---
[![CC-BY-4.0](images/cc-by-40.png)](https://creativecommons.org/licenses/by/4.0/)

Copyright &copy; 2023-2024, The MathWorks, Inc.
