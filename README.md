# coawst-conda
A quick guide to installing the libraries necessary to run COAWST via the conda package manager. This install was successfully tested on MacOS Monterey 12.4 on an Apple M1 Max chip.

## Outline
1. Create a conda environment w/ requisite libraries
2. Set environmental variables
3. MCT
4. SCRIP_COAWST
5. COAWST

## Create a conda environment w/ requisite libraries
Here we will use *conda*, which is often associated with managing python packages, but can really be used for any software 
(it's very worth checking out this [article](https://jakevdp.github.io/blog/2016/08/25/conda-myths-and-misconceptions/) by @jakedvp for more information). Conda is a good way to install the prerequisite libraries for COAWST - namely MPI and netCDF to save the hassle of building the code from source, which can be intimidating and difficult
for people like me who aren't super comfortable at the command line. Managing software within environments also allow us to keep software and dependencies organized, which faciltates reproducibility and portability.

I recommend using the lightweight Miniconda, and to install it following the excellent [guide](https://medium.com/@rabernat/custom-conda-environments-for-data-science-on-hpc-clusters-32d58c63aa95) by @rabernat. 
Instead of using his example environment.yml, I've provided a coawst.yml file in this repository that we can use to create an environment for COAWST. This environment by default just contains the bare essentials for running COAWST - the Open MPI software and compiler wrappers as well as netCDF-C and netCDF-fortran. 

Anyway, once you are in the same directory as coawst.yml...
``` 
conda env create -f coawst.yml
conda activate coawst
```
will create and activate an environment named "coawst". The name is specified at the top of the .yml file, so feel free to change that to whatever you want.
 
 ## Set environment variables
COAWST requires some environment variables to be set so that the software knows which libraries and compilers to use. 
In the "config" script", we use the convenient environmental variable $CONDA_PREFIX to point to the Open MPI Fortran and C wrappers, also modifying the PATH environment variable so that the conda version of mpirun is set to default in case of conflicting installations.
```
export PATH=${CONDA_PREFIX}/bin:$PATH
export CC=${CONDA_PREFIX}/bin/mpicc
export FC=${CONDA_PREFIX}/bin/mpif90
 ```
Indicate that we want to compile the code with MPI/MPIf90
```
export USE_MPI=ON
export USE_MPIF90=ON
```
Allow for the "nc-config"/"nf-config" command to specify the paths to the netCDF libraries.
```
export USE_NETCDF4=ON
```
Specify a location for the MCT libraries to installed.
```
export MCT_PATH=${HOME}/COAWST/Lib/MCT
export MCT_INCDIR=${MCT_PATH}/include
export MCT_LIBDIR=${MCT_PATH}/lib
```
Use system default AR
```
export AR="$(which ar) cq"
```
All of these commands are in the "config" text file in this repository which you can call with
```
source config
```
You might think about automatically sourcing "config" upon environment activation 
(see https://stackoverflow.com/questions/34606196/create-a-post-activate-script-in-conda)
We're now ready to build the libraries that come with COAWST, which is already pretty well covered in the COAWST_User_Manual.docx, 
but there are a few tweaks that will make our lives easier.

### gfortran
We need to modify the compiler (COAWST/Compilers/Darwin-gfortran.mk in my case) so that the installation is smooth. First, add the
following flag to FFLAGS so the line looks like
```
FFLAGS := -frepack-arrays -fallow-argument-mismatch
```
We also need to ensure that our netCDF libraries are correctly pointed to.
It should be very convenient to use
```
export USE_NETCDF4
```
so that the libraries are automatically found via the shell commands nc-config or nf-config, but unfortunately in the conda distribution of netcdf-fortran
```
nf-config --prefix
```
returns nothing even if 
```
nf-config
```
clearly lists the path to the prefix directory. 

Below are the edits I've made to the relevant .mk file 

*original*

```
ifdef USE_NETCDF4
        NF_CONFIG ?= nf-config
    NETCDF_INCDIR ?= $(shell $(NF_CONFIG) --prefix)/include
             LIBS += $(shell $(NF_CONFIG) --flibs)
           INCDIR += $(NETCDF_INCDIR) $(INCDIR)
```
*modified*

```
ifdef USE_NETCDF4
        NF_CONFIG ?= nf-config
        NC_CONFIG ?= nc-config
    NETCDF_INCDIR ?= $(shell $(NC_CONFIG) --prefix)/include
             LIBS += $(shell $(NF_CONFIG) --flibs)
           INCDIR += $(NETCDF_INCDIR) $(INCDIR)
```

Notice all I did was add a variable, NC_CONFIG, that calls the shell command "nc-config", for which the --prefix option DOES work and correctly points to the netcdf directory in our environment when USE_NETCDF4=ON. Hopefully in future versions of netcdf-fortran "nf-config --prefix" will be more reliable. Note: if "nf-config --flibs" does not return "-lnetcdf" and "-lnetcdff" you'll need to add them to libs as follows
```
LIBS += $(shell $(NF_CONFIG) --flibs) -lnetcdf -lnetcdff
```

### MCT
Now we are ready to build the libraries. First, enter the Lib/MCT directory. When building MCT it's good to specify the prefix and compiler flags in the ./configure command itself to minimize editing Makefile.conf
```
./configure --prefix=$MCT_PATH CC=$CC FC=$FC AR=$AR
```
make
make install
```
should successfully build MCT

### SCRIP_COAWST
```
Now in the SCRIP_COAWST directory, after making sure FORT = gfortran in the makefile and comment out
```
make
```
should work.

## Running COAWST
Test the installation by first editing coawst.bash so that 
```
USE_NETCDF4=ON
USE_MPI=ON
USE_MPIF90=ON
which_mpi=openmpi
FORT = gfortran
```
after which
```
./coawst.bash -j
```
should succesfully compiles the code. note: I prefer to set the the MY_ROOT_DIR as the location of the COAWST directory, and specify the project using MY_PROJECT_DIR then modify 
```
   export     MY_HEADER_DIR=${MY_PROJECT_DIR}/Projects/Inlet_test/Coupled
   export MY_ANALYTICAL_DIR=${MY_PROJECT_DIR}/Projects/Inlet_test/Coupled
```
to 
```
   export     MY_HEADER_DIR=${MY_PROJECT_DIR}
   export MY_ANALYTICAL_DIR=${MY_PROJECT_DIR}
```
and let BINDIR=${MY_PROJECT_DIR} and SCRATCH_DIR=${MY_PROJECT_DIR}/Build so that executable and Build directory are in the project folder. This makes it easier for me to manage multiple projects at once, especially if I expect to have to recompile from time to time.

Now after modifying NtileI/NtileJ in the ROMS input file and NnodesOCN= / NnodesWav= in the coupling input file of the test case of your choice depending on the number of processors you want to use - you can test the code execution with
```
mpirun -np x ./coawstM path/to/coupling_file.in
```
Please let me know if it doesnt work for you, or if I've overlooked/glossed over anything!

### to do
-can't get parallel netcdf to work in conda (see coawst-ncparallel.yml). I'm relying on the spectraldns distribution of libnetcdf-parallel, but it doesn't get the library prefix right when I conda install it. Looks like an environment variable is set incorrectly or something b/c the path is like 
```
/Users/wit/anaconda3/envs/coawst//Users/wit/anaconda3/envs/coawst/lib
```
I'll get around to putting in an issue soon.
