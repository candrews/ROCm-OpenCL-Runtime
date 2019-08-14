# ROCm OpenCL™ Compatible Runtime 

Developer preview Version 2 of the new 

* OpenCL 1.2 compatible language runtime and compiler
* OpenCL 2.0 compatible kernel language support with OpenCL 1.2 compatible runtime
* Supports offline ahead of time compilation today; during the Beta phase we will add in-process/in-memory compilation.


## GETTING THE SOURCE CODE

Main OpenCL™ Compatible Components:

* https://github.com/RadeonOpenCompute/ROCm-OpenCL-Runtime
* https://github.com/RadeonOpenCompute/ROCm-Device-Libs 
* https://github.com/RadeonOpenCompute/ROCm-OpenCL-Driver 
* https://github.com/RadeonOpenCompute/llvm 
* https://github.com/RadeonOpenCompute/clang
* https://github.com/RadeonOpenCompute/lld 
* https://github.com/KhronosGroup/OpenCL-ICD-Loader

## INSTALL ROCm

Follow the instructions at https://rocm.github.io/install.html to install ROCm.

## SETUP OpenCL

Copy the amdocl64.icd file to /etc/OpenCL/vendors

    sudo cp api/opencl/config/amdocl64.icd /etc/OpenCL/vendors/

## BUILDING

To install additional dependencies:

* ROCm-OpenCL-Runtime
* ROCm-OpenCL-Driver
* ROC versions of LLVM, Clang, and lld
* ROCm-Device-Libs
* OpenCL-ICD-Loader
* OCaml
* findlib
* A Python 2 environment or active virtualenv with the Microsoft Z3 package
* git with subversion support (git svn)

Run:

    sudo apt-get install ocaml ocaml-findlib python-z3 git-svn

Use out-of-source CMake build and create separate directory to run CMake.

The following build steps are performed:

    mkdir -p build && cd build
    cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo ..
    make
    
## RUNNING clinfo

Assuming you are in "build" directory from the previous step:

    export LD_LIBRARY_PATH=<path-to-build>/lib
    ./bin/clinfo

OpenCL™ is registered Trademark of Apple
