---
aliases:
- /tutorial/technical/2021/07/24/setting-up-a-cxx-dev-env.html
categories:
- tutorial
- technical
date: '2021-07-24'
description: It's actually pretty easy
layout: post
title: Using the RDKit in a C++ program
toc: true

---
Updated: 2025-10-22 to include Windows instructions.

Updated: 2025-09-19 to modernize this and get it working again.



*Note:* the instructions in this blog post have been tested on Linux, Mac, and Windows with Visual Studio installed. It will also work fine under WSL on Windows (since that's just Linux). I haven't tried it with other compilers on Windows.

Last week I (re)discoverered that it's pretty easy to use the RDKit in other C++
projects. This is obviously somthing that's possible, but I thought of it as
being something of a pain. It turns out that it's not, as I hope to show you in
this post.

I started by setting up a fresh conda environment and grabbing an RDKit build
from conda-forge, this is much easier than doing your own build. I'm not
installing the entire RDKit (there are no python packages), just the libraries
and headers needed to build C++ programs that use the RDKit.

The first thing is to create and activate a new conda environment that has the
required dependencies installed:
```
mamba create -n rdkit_dev cmake librdkit-dev eigen libboost-devel compilers
conda activate rdkit_dev
```
Note: I'm doing tihs with [mamba](https://github.com/mamba-org/mamba) here
because it makes doing conda installs much faster. I've also installed the
conda-forge compiler package since that makes sure I have a recent enough
compiler and c++ libraries to work with the up-to-date version of boost we get
with the RDKit from conda-forge.


Here's a simple demo program which reads in a set of molecules from an input
file and generates tautomer hashes for them. It uses the `boost::timer` library
in order to separately time how long it takes to read the molecules and generate
the hashes. With modern C++ there are other ways to do this, but it's worthwhile to show how to include boost dependencies. I called this file `tautomer_hash.cpp`:
```
//
//  Copyright (C) 2021-2025 Greg Landrum
//

#include <GraphMol/FileParsers/MolSupplier.h>
#include <GraphMol/MolHash/MolHash.h>
#include <GraphMol/RDKitBase.h>
#include <algorithm>
#include <boost/timer/timer.hpp>
#include <iostream>
#include <vector>

using namespace RDKit;

void readmols(std::string pathName, unsigned int maxToDo,
              std::vector<std::unique_ptr<RWMol>> &mols) {
  boost::timer::auto_cpu_timer t;
  // using a supplier without sanitizing the molecules...
  v2::FileParsers::SmilesMolSupplierParams params;
  params.parseParameters.sanitize = false;
  params.smilesColumn = 1;
  params.nameColumn = 0;
  v2::FileParsers::SmilesMolSupplier suppl(pathName, params);
  unsigned int nDone = 0;
  while (!suppl.atEnd() && (maxToDo <= 0 || nDone < maxToDo)) {
    auto m = suppl.next();
    if (!m) {
      continue;
    }
    m->updatePropertyCache();
    // the tautomer hash code uses conjugation info
    MolOps::setConjugation(*m);
    nDone += 1;
    mols.push_back(std::move(m));
  }
  std::cerr << "  read: " << nDone << " mols." << std::endl;
}

void generatehashes(const std::vector<std::unique_ptr<RWMol>> &mols) {
  boost::timer::auto_cpu_timer t;
  for (const auto &mol : mols) {
    auto hash =
        MolHash::MolHash(mol.get(), MolHash::HashFunction::HetAtomTautomer);
  }
}
int main(int argc, char *argv[]) {
  std::vector<std::unique_ptr<RWMol>> mols;
  std::cerr << "reading molecules" << std::endl;
  readmols(argv[1], 10000, mols);
  std::cerr << "generating hashes" << std::endl;
  generatehashes(mols);
  std::cerr << "done" << std::endl;
}
```
This is a pretty crappy program since it doesn't do much error checking, but the
purpose here is to demonstrate how to get the environment setup, not to teach
how to write nice C++ programs. :-)

The way to make the build easy is to use cmake to set everything up, so I need a
`CMakeLists.txt` file that defines my executable and its RDKit dependencies:
```
cmake_minimum_required(VERSION 3.30)

project(simple_cxx_example)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED True)


find_package(RDKit REQUIRED)
find_package(Boost CONFIG COMPONENTS timer system REQUIRED)
add_executable(tautomer_hash tautomer_hash.cpp)
target_link_libraries(tautomer_hash  RDKit::MolHash RDKit::FileParsers RDKit::SmilesParse
   Boost::timer)
```
This tells cmake to find the RDKit and boost builds we have installed (which
"just works" since cmake, boost, and the RDKit were all installed from conda),
defines the executable I want to create, and then lists the RDKit and boost
libraries I use. And that is pretty much that.

Next I create a build dir and run `cmake` to setup the build. On Windows I executed these commands using the bash shell that comes with a git install, but I'm sure you can make this work with other Windows shells too.

```
(rdkit_dev) glandrum@stoat:~/RDKit_blog/src/simple_cxx_example$ mkdir build
(rdkit_dev) glandrum@stoat:~/RDKit_blog/src/simple_cxx_example$ cd build
(rdkit_dev) glandrum@stoat:~/RDKit_blog/src/simple_cxx_example/build$ cmake ..
-- The C compiler identification is GNU 14.3.0
-- The CXX compiler identification is GNU 14.3.0
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: /home/glandrum/mambaforge/envs/rdkit_dev/bin/x86_64-conda-linux-gnu-cc - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /home/glandrum/mambaforge/envs/rdkit_dev/bin/x86_64-conda-linux-gnu-c++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Performing Test CMAKE_HAVE_LIBC_PTHREAD
-- Performing Test CMAKE_HAVE_LIBC_PTHREAD - Success
-- Found Threads: TRUE
CMake Warning (dev) at /home/glandrum/mambaforge/envs/rdkit_dev/share/cmake-4.1/Modules/CMakeFindDependencyMacro.cmake:78 (find_package):
  Policy CMP0167 is not set: The FindBoost module is removed.  Run "cmake
  --help-policy CMP0167" for policy details.  Use the cmake_policy command to
  set the policy and suppress this warning.

Call Stack (most recent call first):
  /home/glandrum/mambaforge/envs/rdkit_dev/lib/cmake/rdkit/rdkit-config.cmake:42 (find_dependency)
  CMakeLists.txt:8 (find_package)
This warning is for project developers.  Use -Wno-dev to suppress it.

-- Found Boost: /home/glandrum/mambaforge/envs/rdkit_dev/lib/cmake/Boost-1.86.0/BoostConfig.cmake (found suitable version "1.86.0", minimum required is "1.86.0") found components: headers iostreams serialization
-- Configuring done (0.3s)
-- Generating done (0.0s)
-- Build files have been written to: /home/glandrum/RDKit_blog/src/simple_cxx_example/build
```
On Linux or the Mac you can now run `make`:
```
(rdkit_dev) glandrum@stoat:~/RDKit_blog/src/simple_cxx_example/build$ make tautomer_hash
[ 50%] Building CXX object CMakeFiles/tautomer_hash.dir/tautomer_hash.cpp.o
[100%] Linking CXX executable tautomer_hash
[100%] Built target tautomer_hash
```
And now I can run the program:
```
(rdkit_dev) glandrum@stoat:~/RDKit_blog/src/simple_cxx_example/build$ ./tautomer_hash /scratch/RDKit_git/Code/Profiling/GraphMol/chembl23_very_active.txt
reading molecules
  read: 10000 mols.
 0.750000s wall, 0.680000s user + 0.070000s system = 0.750000s CPU (100.0%)
generating hashes
 0.610000s wall, 0.600000s user + 0.010000s system = 0.610000s CPU (100.0%)
done
(rdkit_dev) glandrum@stoat:~/RDKit_blog/src/simple_cxx_example/build$ 
```

On Windows you can most easily build the program using cmake itself, notice that I need to tell cmake to build in Release mode (you end up with a Debug build if you don't do this):
```
glandrum@ferret MINGW64 ~/Code/rdkit_blog/src/simple_cxx_example/build (master)
$ cmake --build . --config Release
MSBuild version 17.14.23+b0019275e for .NET Framework

  1>Checking Build System
  Building Custom Rule C:/Users/glandrum/Code/rdkit_blog/src/simple_cxx_example
  /CMakeLists.txt
  Scanning sources for module dependencies...
  tautomer_hash.cpp
  Compiling...
  tautomer_hash.cpp
  tautomer_hash.vcxproj -> C:\Users\glandrum\Code\rdkit_blog\src\simple_cxx_exa
  mple\build\Release\tautomer_hash.exe
  Building Custom Rule C:/Users/glandrum/Code/rdkit_blog/src/simple_cxx_example
  /CMakeLists.txt
```
The Windows build system puts the code in the `Release` subdirectory, so you run it with: `./Release/tautomer_hash`

Note that it's also possible to use cmake to run the build on Linux or the Mac; I'm just used to doing that with make.

If you don't feel like copy/pasting, the source files for this post are [available from github](https://github.com/greglandrum/rdkit_blog/tree/master/src/simple_cxx_example). 

This all works so nicely because of the time and effort Riccardo Vianello invested a few years ago to improve the RDKit's cmake integration.

Next step: add this to the documentation! 
