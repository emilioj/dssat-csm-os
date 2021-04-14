# DSSAT + PDI proof of concept

I am setting up the cmake project like this:
(PDI was installed in /opt/pdi)

    CMAKE_BUILD_TYPE                 DEBUG
    CMAKE_Fortran_FLAGS_TESTING       -O2
    CMAKE_INSTALL_PREFIX             /opt/my_dssat-pdi
    PDI_DIR                          /opt/pdi/share/pdi/cmake
    paraconf_DIR                     /opt/pdi/share/paraconf/cmake

`make` builds the project flawlessly, once avoided the default static
linkage (apparently PDI is built as a shared/dynamic library
exclusively).

`make install` gets my custom DSSAT-CSM-OS installed in
/opt/my_dssat-pdi.

Before using DSSAT, we need some data! Just clone this repo:
https://github.com/DSSAT/dssat-csm-data

I have no idea where to put it to be able to execute DSSAT
experiments. I run DSSAT using the wrapper `run_dssat` in `Utilities`
and I tried different locations and both relative and absolute paths
in the invocation with no success:

    ./run_dssat C dssat-csm-data/Maize/UFGA8201.MZX 1

So in the end I copy the files DSSAT needs for the experiment in
`Utilities` to make it work... ¯\_(ツ)_/¯

    cp ./dssat-csm-data/Maize/UFGA8201.MZX .
    cp ./dssat-csm-data/Weather/UFGA8201.WTH .
    cp ./dssat-csm-data/Soil/SOIL.SOL .

that way a simple

    ./run_dssat C UFGA8201.MZX 1

just works and returns the expected DSSAT output, preceded by some PDI
stuff (Initialization, debugging messages, etc.) along with some
Fortran prints of the variables exposed to PDI:

    [...] PDI stuff [...]

    RUN    TRT FLO MAT TOPWT HARWT  RAIN  TIRR   CET  PESW  TNUP  TNLF   TSON TSOC
               dap dap kg/ha kg/ha    mm    mm    mm    mm kg/ha kg/ha  kg/ha t/ha
      1 MZ   1  76 127  6433  2020   661    13   348   144   102    54   8662   87

Ah, I also need to copy the YAML file with the PDI Specification tree to
the `Utilities` directory:

    cp ../Coupling/dssat-pdi.yml .

So far... so good (sort of).

To complete the first part of my proof of concept I want to be able to
access DSSAT contents from Python through PDI, though. I am not
succeeding with that for the moment, using the `on_data` pattern in
the `pycall` plugin :-/

This simple test I can manage to work in a simple Fortran program +
PDI (even accessing real data, not just a Hello World like here):

    pycall:
      on_data:
        nfert: print('Hello World from Python!')

breaks with what seems a Fortran exception:

    % PYTHONPATH=/opt/pdi/lib/python3/dist-packages ./run_dssat C UFGA8201.MZX 1
    [PDI][Trace-plugin] *** info: Welcome!
    [PDI][15:52:00] *** debug: Metadata is not defined in specification tree
    [PDI][15:52:00] *** info: Initialization successful
     DSSAT->NFERT val:            3
     DSSAT->IFERI val: R

    Program received signal SIGFPE: Floating-point exception - erroneous arithmetic operation.

After some struggling I found that the compiler flag
`-ffpe-trap=overflow` was causing the issue... :-?  (apparently, the
issue only arises when the source file that calls PDI_Init/PD_Finalize
is compiled with that flag).


# dssat-csm-os
DSSAT Cropping System Model

The Decision Support System for Agrotechnology Transfer (DSSAT) Version is a software 
application program that comprises crop simulation models for over 42 crops (as of Version 4.7).

For DSSAT to be functional it is supported by data base management programs for soil, 
weather, and crop management and experimental data, and by utilities and application 
programs. The crop simulation models simulate growth, development and yield as a 
function of the soil-plant-atmosphere dynamics.

DSSAT and its crop simulation models have been used for many applications ranging from 
on-farm and precision management to regional assessments of the impact of climate 
variability and climate change. It has been in use for more than 20 years by researchers, 
educators, consultants, extension agents, growers, and policy and decision makers 
in over 100 countries worldwide.

Read more about DSSAT at http://dssat.net/about

See also: [The DSSAT Crop Modeling Ecosystem](https://dssat.net/wp-content/uploads/2020/03/The-DSSAT-Crop-Modeling-Ecosystem.pdf)

and: [Non-threatening best practice DSSAT Fortran coding guidelines](https://dssat.net/non-threatening-best-practice-dssat-fortran-coding-guidelines). 


## The directory structure ##

DSSAT cropping system model (CSM) design is a modular structure in which components 
separate along scientific discipline lines and are structured to allow easy replacement 
or addition of modules. It has one Soil module, a Crop Template module which can simulate 
different crops by defining species input files, an interface to add individual crop 
models if they have the same design and interface, a Weather module, and a module for 
dealing with competition for light and water among the soil, plants, and atmosphere. 
It is also designed for incorporation into various application packages, ranging from 
those that help researchers adapt and test the CSM to those that operate the DSSAT /CSM 
to simulate production over time and space for different purposes.
[The DSSAT cropping system model](https://dssat.net/jones_2003_the_dssat_cropping_system_model).

## Compiling the code ##

The code is compatible with the CMake utility for generating make files
and setting up projects for a variety of IDEs and compilers. To use this feature, 
first download and install CMake. Then set up a CMake project by pointing to the
source code directory and the build directory.

## Structure of the code ##
    .
    ├── <source files>
    ├── CMakeLists.txt
    ├── distclean.cmake
    ├── README.md
    ├── ...
    ├── cmake
    │   └── Modules
    │       ├── SetCompileFlag.cmake
    │       └── SetFortranFlags.cmake
    ├── build
    │   └── ...
    └── Data
        ├── Documentation
        ├── Cotton
        ├── ... 
        └── Wheat
         
### CMakeLists.txt ###

This file contains all the configuration needed to set up the project.  
Edit this file to make your own configuration and add new projects. 
Comment/Uncomment any lines pertaining to options you may need. 

### distclean.cmake ###

This is a CMake script that will remove all files and folder that are created after running `make`.  You can run this code in one of two ways:

* Execute `cmake -P distclean.cmake`. (The `-P` option to `cmake` will execute a CMake script)
* Execute `make distclean` after your Makefile has been generated.

You shouldn't need to edit this file.

### README.md ###

This File.

### [...] ###

Inside the main directory you will find all subdirectories and source files for your project. All .for, f90, etc.

### cmake/Modules/ ###

This directory contains CMake scripts that aid in configuring the build system.

###### SetCompileFlag.cmake ######

This file defines a function that will test a set of compiler flags to see which one works and adds that flag to a list of compiler flags.  This is used to set compile flags when you don't know which compiler will be used.

###### SetFortranFlags.cmake ######

This file uses the function from `SetCompilerFlag.cmake` to set the DEBUG, TESTING, and RELEASE compile flags for your build.  You might want to inspect this file and edit the flags to your liking.

### build ###

This folder is created to organize all working files inside it, avoiding messing up your source folder. During compilation and linking, working folders are created automatically inside this folder. Any libraries created end up in `mod/`, as well as compiled Fortran `.mod` files.  The executable will end up in `bin/`.  

### Data ###

This folder contains data, documentation, DSSAT configuration files, and crop-specific experiment files.

## Configuring the build ##

It is usually preferred that you do an out-of-source build.  To do this, create a `build/` directory at the top level of your project and build there.  

    $ mkdir build
    $ cd build
    $ cmake ..
    $ make
    
When you do this, temporary CMake files will not be created in your `src/` directory.  

As written, this template will allow you to specify one of three different sets of compiler flags.  The default is RELEASE.  You can change this using to TESTING or DEBUG using

    $ cmake .. -DCMAKE_BUILD_TYPE=DEBUG
    
or

    $ cmake .. -DCMAKE_BUILD_TYPE=TESTING

You can provide all kind of information CMake. Some examples can be find at [[CMake Command-Line Options](https://cmake.org/cmake/help/cmake-2.4.html)].

One usable examples could be:

    $ cmake -G "Unix Makefiles" -DCMAKE_Fortran_COMPILER=ifort ..

In this example we are specifying the fortran compiler and the kind of project we want as result (make file project). 
