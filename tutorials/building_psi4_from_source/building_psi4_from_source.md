# Building Psi4 from Source
Two major steps exist in building Psi4 from source: 
1. **Setting up the development environment**
2. **Building Psi4 with cmake**

## Setting up the Development Environment

Before building Psi4, you need to set up the development environment. This
involves installing the necessary dependencies and tools. Thankfully, Psi4
developers have simplified this process by providing a script that helps
construct the development environment. Once you are at the root of your
forked Psi4 repository, you can run the following command to learn about
different environment construction options:

```bash
python3 conda/psi4-path-advisor.py --help
```
This should provide you this output:

```bash
usage: psi4-path-advisor [-h] [-v] {env,conda,cache,cmake,bulletin,deploy} ...

Dependency, Build, and Run path advisor for Psi4.
Mediates file https://github.com/psi4/psi4/blob/master/codedeps.yaml
Run env subcommand. Conda env create and activate. Run cmake subcommand. Build.

=========================================
  (A) black-box usage (copy/paste-able)
=========================================
# (1) get code from GitHub
git clone https://github.com/psi4/psi4.git && cd psi4
# (2) generate env spec file from codedeps.yaml. "eval $(...)" creates and activates conda env.
eval $(conda/psi4-path-advisor.py env)
# (3) generate cmake cache file from conda env. "eval $(...)" configures and builds with cmake.
eval $(conda/psi4-path-advisor.py cmake)

# (4) next steps. repeat each login or add to shell's rc. your paths may vary.
eval $(objdir_p4dev/stage/bin/psi4 --psiapi)
export PSI_SCRATCH=/path/to/existing/writable/local-not-network/directory/for/scratch/files

=========================================
  (B) flexible usage
=========================================
# (1) get code from GitHub
git clone https://github.com/psi4/psi4.git && cd psi4

# (2.0) consider dependency options
conda/psi4-path-advisor.py env -h
# (2.1) generate env spec file from codedeps.yaml.
conda/psi4-path-advisor.py env -n p4dev310 --python 3.10 --disable addons --lapack openblas
#> conda env create -n p4dev310 -f /home/psi4/env_p4dev310.yaml && conda activate p4dev310
# (2.2) edit env_p4dev310.yaml to customize software packages.
# (2.3) issue suggested or customized command to create and activate conda env.
conda env create -n p4dev310 -f /home/psi4/env_p4dev310.yaml && conda activate p4dev310

# (3.0) consider compile options
conda/psi4-path-advisor.py cmake -h
# (3.1) generate cmake cache file from conda env.
conda/psi4-path-advisor.py cmake
#> cmake -S. -GNinja -C/home/psi4/cache_p4dev310.cmake -Bobjdir_p4dev310 && cmake --build objdir_p
4dev310
# (3.2) edit cache_p4dev310.cmake to customize build configuration.
# (3.3) issue suggested or customized command to configure & build with cmake.
cmake -S. -GNinja -C/home/psi4/cache_p4dev310.cmake -Bobjdir_p4dev310 -DCMAKE_INSTALL_PREFIX=/path
/to/install-psi4 && cmake --build objdir_p4dev310

# (4) next steps. repeat each login or add to shell's rc. your paths may vary.
eval $(objdir_p4dev310/stage/bin/psi4 --psiapi)
export PSI_SCRATCH=/path/to/existing/writable/local-not-network/directory/for/scratch/files

positional arguments:
  {env,conda,cache,cmake,bulletin,deploy}
                        Script requires a subcommand.
    env (conda)         Write conda environment file from codedeps file.
    cache (cmake)       Write cmake configuration cache from conda environment.
    bulletin            (Info only) Read any build messages or advice.
    deploy              (Admin only) Apply codedeps info to codebase.

options:
  -h, --help            show this help message and exit
  -v                    Use for more printing (-vv).
                        Verbosity arg is NOT compatible with bash command substitution.
```

We want to create a standard, minimal psi4 environment, so we will run the following command:

```bash
python ./conda/psi4-path-advisor.py env -n p4dev311 --python 3.11 --disable addons 
```

Then to create the environment, run:

```bash
conda env create -n p4dev311 -f /theoryfs2/ds/amwalla3/gits/psi4_amw/env_p4dev311.yaml 
```
Before any development session, you will need to activate the conda environment:
```bash
conda activate p4dev311
```

Now you are ready to build Psi4!

## Building Psi4 with cmake

With the developer environment, building from source becomes much easier. 
Many different options exist with how you might want to compile and re-compile Psi4
as you are making changes; however, I personally prefer to use a bash script 
and Ninja to build Psi4. If you would like to use this method, you can create a
`build.sh` file in the root of your Psi4 repository with the following contents:

```bash
#!/bin/bash
cat ./build.sh

start_time=$(date +%s)
branch_name=$(git branch | grep \* | cut -d ' ' -f2)
objdir_name="build_$branch_name"
echo "building $objdir_name"
export CXXFLAGS=""
if [ ! -d $objdir_name ]; then
    echo "creating $objdir_name"
    cmake  -DBUILD_SHARED_LIBS=ON -S. -B $objdir_name -G Ninja
fi
cmake --build $objdir_name
$objdir_name/stage/bin/psi4 --psiapi
eval `$objdir_name/stage/bin/psi4 --psiapi`
end_time=$(date +%s)
echo "Elapsed time: $((end_time - start_time)) seconds"
echo ""
```

You will notice that there are two different cmake commands in the script. The
first command is used to create the build directory, and the second command is
used to build Psi4. To isolate projects and not have to re-clone my fork
multiple times, I have the script create a build directory with the name of the
branch I am on. This way, I can have multiple build directories for different
branches. Additionally, we first check if the build directory exists before
creating it. This way, we can re-run the script without having to re-configure
the build directory (useful when we are editing code and re-compiling where we
don't need to re-configure every time).

To run the script, you will need to make it executable:

```bash
chmod +x build.sh
```

Then you can run the script with:

```bash
./build.sh
```
This will likely take a few minutes to build Psi4 depending on your system. Once
the build is complete, you need to add psi4 to your path and python path to execute it through:

```bash
branch_name=$(git branch | grep \* | cut -d ' ' -f2)
objdir_name="build_$branch_name"
eval `$objdir_name/stage/bin/psi4 --psiapi`
```

Now you can run Psi4 with:

```bash
psi4 --version
```

Now you have successfully built Psi4 from source! You can now make changes to
the source code (either python or cpp files) and re-run the `./build.sh` script
to re-compile Psi4. Note you do not need to re-evaluate the psi4 path each time (just once
per session like the conda environment).

# Final considerations
You can figure out a build procedure that works best for you. The above is just
one way to build Psi4 for development, but better ways probably exist (if you
have suggestions, please let me know!).

Also, before you actually run calculations is is VERY important to set your
SCRATCH directory. This is where Psi4 will write temporary files during
calculations. If you do not set this and work on a fileserver, you could
cause a lot of network traffic and slow down the fileserver. To set the scratch
directory, you can run:

```bash
export SCRATCH=/path/to/scratch
export PSI_SCRATCH=/path/to/scratch
```

Congrats! You have now built Psi4 from source and are ready to start
developing! 

# Next Tutorial

Now we will start to learn about the Psi4 codebase and the general structure of 
the code in this [tutorial](../psi4_codebase_overview/psi4_codebase_overview.md).
