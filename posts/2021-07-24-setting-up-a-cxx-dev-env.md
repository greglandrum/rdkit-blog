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
Updated: 2025-09-19 to modernize this and get it working again.


*Note:* the instructions in this blog post currently only work on linux and Mac systems.
I haven't tried it out on Windows yet, but I will update the post if I can make that work.

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
the hashes. I called this file `tautomer_hash.cpp`:
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
cmake_minimum_required(VERSION 3.20)

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

Now I create a build dir, run `cmake` to setup the build, and run `make` to actually
build my program:
```
(rdkit_dev) glandrum@Badger:~/RDKit_blog/src/simple_cxx_example$ mkdir build
(rdkit_dev) glandrum@Badger:~/RDKit_blog/src/simple_cxx_example$ cd build
(rdkit_dev) glandrum@Badger:~/RDKit_blog/src/simple_cxx_example/build$ cmake ..
-- The C compiler identification is GNU 9.3.0
-- The CXX compiler identification is GNU 9.3.0
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: /usr/bin/cc - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/c++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Looking for pthread.h
-- Looking for pthread.h - found
-- Performing Test CMAKE_HAVE_LIBC_PTHREAD
-- Performing Test CMAKE_HAVE_LIBC_PTHREAD - Failed
-- Looking for pthread_create in pthreads
-- Looking for pthread_create in pthreads - not found
-- Looking for pthread_create in pthread
-- Looking for pthread_create in pthread - found
-- Found Threads: TRUE  
-- Found Boost: /home/glandrum/miniconda3/envs/rdkit_dev/lib/cmake/Boost-1.74.0/BoostConfig.cmake (found suitable version "1.74.0", minimum required is "1.74.0")  
-- Found Boost: /home/glandrum/miniconda3/envs/rdkit_dev/lib/cmake/Boost-1.74.0/BoostConfig.cmake (found version "1.74.0") found components: timer system 
-- Configuring done
-- Generating done
-- Build files have been written to: /home/glandrum/RDKit_blog/src/simple_cxx_example/build
(rdkit_dev) glandrum@Badger:~/RDKit_blog/src/simple_cxx_example/build$ make tautomer_hash
[ 50%] Building CXX object CMakeFiles/tautomer_hash.dir/tautomer_hash.cpp.o
[100%] Linking CXX executable tautomer_hash
[100%] Built target tautomer_hash
```
And now I can run the program:
```
(rdkit_dev) glandrum@Badger:~/RDKit_blog/src/simple_cxx_example/build$ ./tautomer_hash /scratch/RDKit_git/Code/Profiling/GraphMol/chembl23_very_active.txt
[07:51:33] read mols
read: 10000 mols.
 0.819242s wall, 0.740000s user + 0.070000s system = 0.810000s CPU (98.9%)
[07:51:33] generate hashes
 0.872662s wall, 0.870000s user + 0.010000s system = 0.880000s CPU (100.8%)
[07:51:34] done 
(rdkit_dev) glandrum@Badger:~/RDKit_blog/src/simple_cxx_example/build$ 
```

If you don't feel like copy/pasting, the source files for this post are [available from github](https://github.com/greglandrum/rdkit_blog/tree/master/src/simple_cxx_example). 

This all works so nicely because of the time and effort Riccardo Vianello invested a few years ago to improve the RDKit's cmake integration.

Next step: add this to the documentation! 
