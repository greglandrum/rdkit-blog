---
categories:
- tutorial
- technical
date: '2023-03-17'
title: Setting up an RDKit development environment 1
description: It's surprisingly straightforward
layout: post
toc: true

---

This post walks provides a series of step-by-step instructions for getting a
local build of the RDKit working on your machine. This is potentially useful if
you want to contribute to the RDKit C++ code, make Python contributions (an
alternative approach for this is described
[here](https://greglandrum.github.io/rdkit-blog/posts/2020-03-30-setting-up-an-environment.html))
or just try out/work with the pre-release version of the RDKit.

The instructions in this blog post currently work on linux and the Mac.
I will do a second post with instructions for Windows in the next few days. In
the meantime, David Cosgrove has put together some instructions in a [thread on
the RDKit discussions tab](https://github.com/rdkit/rdkit/discussions/6148).

Thanks to Felix Pultar and Jessica Braun for testing and proofreading these
instructions.

# Part 0: getting ready

This assumes that you have a local install of conda. That could be
[miniconda](https://docs.conda.io/en/latest/miniconda.html),
[miniforge](https://github.com/conda-forge/miniforge), or one of the larger
conda installations. If you're just getting started, I would suggest miniforge.

You also need to have a working C++17 compiler on your machine. This is not a
problem on modern linux distributions where you can install either g++ or
clang++ directly from the distribution. On the Mac you need to install the xcode
command line tools, the command to do that is `xcode-select --install`.

Finally, you need a working version of git. If you haven't done this already:
there are a number of easy ways to install git, you can decide which makes most
sense for you. The simplest is probably to just install it using conda.

Now that you've got everything set up, go to the directory where you want to
keep the RDKit source code and clone the github repo:
```
git clone https://github.com/rdkit/rdkit.git RDKit_git
```
This will create a directory called RDKit_git with the RDKit source.

# Part 1: setup conda environment

Conda makes it very, very easy to install all of the dependencies for building
the RDKit, here's the command:
```
mamba create -n py310_rdkit_build -c conda-forge python=3.10 boost-cpp boost cairo pandas pillow freetype cmake numpy eigen matplotlib 
```

Note that I use `mamba` instead of `conda` for installing things. `mamba` is
often significantly faster than `conda`. It's installed by default with
miniforge, or you can install it manually with `conda install -c conda-forge
mamba`

# Part 2: run cmake and build

Now we're going to actually build the RDKit. These instructions build the code
and installs it in the RDKit source tree (instead of trying to install it
centrally on your computer).

From inside the RDKit source directory you checked out above:
```
conda activate py310_rdkit_build
mkdir build_demo
cd build_demo
cmake -DRDK_BUILD_INCHI_SUPPORT=ON -DRDK_BUILD_YAEHMOP_SUPPORT=ON -DRDK_BUILD_XYZ2MOL_SUPPORT=ON ..
```

That should run without errors. It’ll produce a bunch of warnings, but should
end with something like:
```
-- Generating done
-- Build files have been written to: /localhome/glandrum/RDKit_git/build_demo
```

Now actually build it:
```
make -j4 install
```
The `-j4` here controls the number of jobs which run simultaneously. If you have
a lot of memory on your machine, you can definitely increase the value from 4 to
8 (or whatever).

That will take a while and generate a few warnings, but shouldn’t have any
errors. The last line of the output should be something like this:
```
-- Installing: /localhome/glandrum/RDKit_git/rdkit/RDPaths.py
```


# Part 3: setup your environment and then run the tests

At this poing you have a completed build of the RDKit that is installed within
the source tree. Now we're going to set up your enviroment so that you can
actually use that build.

Do this from inside the RDKit source directory (*not* the build directory).
On linux:
```
export RDBASE=`pwd`
export PYTHONPATH=$RDBASE
export LD_LIBRARY_PATH=$RDBASE/lib:$CONDA_PREFIX/lib
```
on the Mac, that is:
```
export RDBASE=`pwd`
export PYTHONPATH=$RDBASE
export DYLD_LIBRARY_PATH=$RDBASE/lib:$CONDA_PREFIX/lib
```

Now you can run the tests:
```
cd build_demo
ctest -j4 –output-on-failure
```

This will take a bit of time and generate a lot of output, but the last bit
should look something like this:
```  
100% tests passed, 0 tests failed out of 219

Total Test time (real) =  63.92 sec
```

That’s it! You have now built the RDKit and can work with it locally. 

If you make changes to the C++ code and want to test them out, you just need to
run `make -j4 install`. After making Python changes you don't need to do
anything in order to have those changes available. I do, however, strongly
suggest running the tests again (`ctest -j4 --output-on-failure`) after making
any change. It's good to find out quickly if you've broken anything!

# Part 4: Using the environment
In order to use the code you just need to set up your environment by repeating
the export commands shown above. I find the following aliases really useful on
linux (the Mac form just needs `LD_LIBRARY_PATH` changed to `DYLD_LIBRARY_PATH`).
```
alias remote_rdkit='unset RDBASE;export PYTHONPATH="";export LD_LIBRARY_PATH=""'
alias local_rdkit='conda activate py310_rdkit_build;export RDBASE="/localhome/glandrum/RDKit_git";export PYTHONPATH="$RDBASE";export LD_LIBRARY_PATH="$RDBASE/lib:$CONDA_PREFIX/lib"'
alias rdkit_vers='python -c "import rdkit;print(rdkit.__version__,rdkit.__file__)"'
```
`local_rdkit` sets up the shell to use my local RDKit build. 
`remote_rdkit` clears that stuff out so that you can use a conda rdkit install
`rdkit_vers` shows the current rdkit version and where the files are coming from

I hope this was useful. If you have suggestions for how to make these instructions clearer or find any mistakes, please comment!