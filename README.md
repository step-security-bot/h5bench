# About
H5bench: a benchmark suite for parallel HDF5 (H5bench) Copyright (c) 2021,
The Regents of the University of California, through Lawrence Berkeley National
Laboratory (subject to receipt of any required approvals from the U.S. Dept. of
Energy) and North Carolina State University. All rights reserved.

If you have questions about your rights to use or distribute this software,
please contact Berkeley Lab's Intellectual Property Office at
IPO@lbl.gov.

NOTICE.  This Software was developed under funding from the U.S. Department
of Energy and the U.S. Government consequently retains certain rights.  As
such, the U.S. Government has been granted for itself and others acting on
its behalf a paid-up, nonexclusive, irrevocable, worldwide license in the
Software to reproduce, distribute copies to the public, prepare derivative 
works, and perform publicly and display publicly, and to permit others to do so.


# H5bench: a Parallel I/O Benchmark suite for HDF5
H5bench is a suite of parallel I/O benchmarks or kernels representing I/O patterns that are commonly used in HDF5 applications on high performance computing systems. H5bench measures I/O performance from various aspects, including the I/O overhead, observed I/O rate, etc.

# Instructions to build H5bench
## Build with CMake (**recommended**)
### Dependency and environment variable settings
H5bench depends on MPI and Parallel HDF5. 

#### Use system provided HDF5
For instance on the Cori system at NERSC:

- `module load cray-hdf5-parallel` 

or 

load any parallel HDF5 provided on your system, and you are good to go.

#### Use your own installed HDF5
Make sure to unload any system provided HDF5 version, and set an environment variable to specify the HDF5 install path:

- **HDF5_HOME**: the location you installed HDF5. It should point to a path that look like /path_to_my_hdf5_build/hdf5 and contains include/, lib/ and bin/ subdirectories. 

### Compile with CMake
Assume that the repo is cloned and now you are in the source directory h5bench, run the following simple steps:

- `mkdir build`
- `cd build`
- `cmake ..`
- `make`

### Build to run in async
To run h5bench_vpicio or h5bench_bdcatsio in async mode, you need the `develop` branchs of BOTH HDF5 and Async-VOL and build H5bench separately. 
- `mkdir build`
- `cd build`
- `cmake .. -DUSE_ASYNC_VOL:BOOL=ON -DCMAKE_C_FLAGS="-I/$YOUR_ASYNC_VOL/src -L/$YOUR_ASYNC_VOL/src"`
- `make`

Necessary environment variable settings:
```
export HDF5_HOME="$YOUR_HDF5_DEVELOP_BRANCH_BUILD/hdf5"
export ASYNC_HOME="$YOUR_ASYNC_VOL/src"
export HDF5_VOL_CONNECTOR="async under_vol=0;under_info={}"
export HDF5_PLUGIN_PATH="$ASYNC_HOME"
export DYLD_LIBRARY_PATH="$HDF5_HOME/lib:$ASYNC_HOME"
```
And all the binaries will be built to the build / directory.

# Benchmark suite usage
## Basic I/O benchmark
Major refactoring is in progress, this document may be out of date. Both h5bench_vpicio and h5bench_bdcats take config and data file path as command line arguments.

- ./h5bench_vpicio my_config.cfg my_data.h5
- ./h5bench_bdcats my_config.cfg my_data.h5

This set of benchmarks contains an I/O kernel developed based on a particle physics simulation's I/O pattern (VPIC-IO for writing data in a HDF5 file) and on a big data clustering algorithm (BDCATS-IO for reading the HDF5 file VPIC-IO wrote).

## Settings in the config file

The h5bench_vpicio and h5bench_bdcats take parameters in a plain text config file. The content format is **strict**. Unsupported formats :
- Blank/empty lines, including ending lines.
- Comment symbol(#) follows value immediately:
  -   TIMESTEPS=5# Not supported
  -   TIMESTEPS=5 #This is supported
  -   TIMESTEPS=5 # This is supported
- Blank space in assignment
  -   TIMESTEPS=5 # This is supported
  -   TIMESTEPS = 5 # Not supported
  -   TIMESTEPS =5 # Not supported
  -   TIMESTEPS= 5 # Not supported


A template of config file can be found `basic_io/sample_config/template.cfg`:
```
#========================================================
#   General settings
NUM_PARTICLES=16 M #16 K/G
TIMESTEPS=5
EMULATED_COMPUTE_TIME_PER_TIMESTEP=1 s #1 ms, 1 min
#========================================================
#   Benchmark data dimensionality
NUM_DIMS=1
DIM_1=16777216
DIM_2=1
DIM_3=1
#========================================================
#   IO pattern settings
IO_OPERATION=READ
#IO_OPERATION=WRITE
MEM_PATTERN=CONTIG
# INTERLEAVED STRIDED
FILE_PATTERN=CONTIG # STRIDED
#========================================================
#   Options for IO_OPERATION=READ
READ_OPTION=FULL # PARTIAL STRIDED
TO_READ_NUM_PARTICLES=4 M
#========================================================
#   Strided access parameters, required for strided access
#STRIDE_SIZE=
#BLOCK_SIZE=
#BLOCK_CNT=
#========================================================
# Collective data/metadata settings
#COLLECTIVE_DATA=NO #Optional, default for NO.
#COLLECTIVE_METADATA=NO #Optional, default for NO.
#========================================================
#    Compression, optional, default is NO.
#COMPRESS=NO
#CHUNK_DIM_1=1
#CHUNK_DIM_2=1
#CHUNK_DIM_3=1
#========================================================
#    Async related settings
DELAYED_CLOSE_TIMESTEPS=2
IO_MEM_LIMIT=5000 K
ASYNC_MODE=EXP #EXP IMP NON 
#========================================================
#    Output performance results to a CSV file
#CSV_FILE=perf_write_1d.csv
#
#FILE_PER_PROC=
```

### General settings  
- **IO_OPERATION**: required, chose from **READ** and **WRITE**.
- **MEM_PATTERN**: required, chose from **CONTIG**, **INTERLEAVED** and **STRIDED**
- **FILE_PATTERN**: required, chose from **CONTIG**, and **STRIDED**
- **NUM_PARTICLES**: required, the number of particles that each rank needs to process, can be in exact numbers (12345) or in units (format like 16 K, 128 M and 256 G are supported,  format like 16K, 128M, 256G is NOT supported).   
- **TIMESTEPS**: required, the number of iterations
- **EMULATED_COMPUTE_TIME_PER_TIMESTEP**: required, must be with units (eg,. 10 s, 100 ms or 5000 us).  In each iteration, the same amount of data will be written and the file size will increase correspondingly. After each iteration, the program sleeps for $EMULATED_COMPUTE_TIME_PER_TIMESTEP time to emulate the application computation.
- **NUM_DIMS**: required, the number of dimensions, valid values are 1, 2 and 3.
- **DIM_1, DIM_2, and DIM_3**: required, the dimensionality of the source data. Always set these parameters in ascending order, and set unused dimensions to 1, and remember that NUM_PARTICLES == DIM_1 * DIM_2 * DIM_3 **MUST** hold. For example, DIM_1=1024, DIM_2=256, DIM_3=1 is a valid setting for a 2D array when `NUM_PARTICLES=262144` or `NUM_PARTICLES=256 K`, because 1024*256*1 = 262144, which is 256 K.
#### Example for using multi-dimensional array data 
- Using 2D as the example, 3D cases are similar, the file is generated with with 4 ranks, each rank write 8M elements, organized in a 4096 * 2048 array, in total it forms a (4 * 4096) * 2048 2D array. The file should be around 1GB. 

Dimensionality part of the Config file:
```
NUM_DIMS=2
DIM_1=4096
DIM_2=2048
DIM_3=64 # Note: extra dimensions than specified by NUM_DIMS are ignored.
```

### Additional settings for READ (h5bench_bdcats)
- **READ_OPTION**: required for IO_OPERATION=READ, not allowed for IO_OPERATION=WRITE.
  - FULL: read the whole file
  - PARTIAL: read the first $TO_READ_NUM_PARTICLES particles
  - STRIDED: read in streded pattern
- **TO_READ_NUM_PARTICLES**: required, the number for particles attempt to read.

### Async related settings
- **ASYNC_MODE**: optional, the default is NON.
  - NON: the benchmark will run in synchronous mode. 
  - EXP: enable the asynchronous mode. An installed async VOL connector and coresponding environment variables are required.
- **IO_MEM_LIMIT**: optional, the default is 0, requires **ASYNC_MODE=EXP**, only works in asynchronous mode. This is the memory threshold used to determine when to actually execute the IO operations. The actual IO operations (data read/write) will not be executed until the timesteps associated memory reachs the threshold, or the application run to the end.
- **DELAYED_CLOSE_TIMESTEPS**: optional, the default is 0. The groups and datasets associated to to the timesteps will be closed later for potential caching.

### Compression settings
- **COMPRESS**: YES or NO, optional. Only applicable for WRITE(h5bench_vpicio), has no effect for READ. Used to enable compression, when enabled, chunk dimensions(CHUNK_DIM_1, CHUNK_DIM_2, CHUNK_DIM_3) are required.
To enable parallel compression feature for VPIC, add following section to the config file, and make sure chunk dimension settings are compatible with the data dimensions: they must have the same rank of dimensions (eg,. 2D array dataset needs 2D chunk dimensions), and chunk dimension size cannot be greater than data dimension size. 
```
COMPRESS=YES # to enable parallel compression(chunking)
CHUNK_DIM_1=512 # chunk dimensions
CHUNK_DIM_2=256
CHUNK_DIM_3=1 # extra chunk dimension take no effects.
```
**Note:** There is a known bug on HDF5 parallel compression that could cause the system run out of memory when the chunk amount is large (large number of particle and very small chunk sizes). On Cori Hasswell nodes, the setting of 16M particles per rank, 8 nodes (total 256 ranks), 64 * 64 chunk size will crash the system by runing out of memory, on single nodes the minimal chunk size is 4 * 4.  

### Collective operation settings
- **COLLECTIVE_DATA**: optional, set to "YES" for collective data operations, otherwise and default (not set) cases for independent operations.
- **COLLECTIVE_METADATA**: optional, set to "YES" for collective metadata operations, otherwise and default (not set) cases for independent operations.

### Other settings
- **CSV_FILE**=my_csv_file: optional CSV file output, performance results will be print to the file and the standard output as well. 

## Supported patterns
Note: not every pattern combination is covered, supported benchmark parameter settings are listed below.
### Supported write patterns(h5bench_vpicio): IO_OPERATION=WRITE 
The I/O patterns include array of structures (AOS) and structure of arrays (SOA) in memory as well as in file. The array dimensions are 1D, 2D, and 3D for the write benchmark. This defines the write access pattern, including CONTIG (contiguous), INTERLEAVED and STRIDED” for the source (the data layout in the memory) and the destination (the data layout in the resulting file). For example, MEM_PATTERN=CONTIG and FILE_PATTERN=INTERLEAVED is a write pattern where the in-memory data layout is contiguous (see the implementation of prepare_data_contig_2D() for details) and file data layout is interleaved by due to its’ compound data structure (see the implementation of data_write_contig_to_interleaved () for details).

#### 4 patterns for both 1D and 2D array write (`NUM_DIMS=1` or `NUM_DIMS=2`)
- MEM_PATTERN=CONTIG, FILE_PATTERN=CONTIG
- MEM_PATTERN=CONTIG, FILE_PATTERN=INTERLEAVED
- MEM_PATTERN=INTERLEAVED, FILE_PATTERN=CONTIG
- MEM_PATTERN=INTERLEAVED, FILE_PATTERN=INTERLEAVED 
#### 1 pattern for 3D array (`NUM_DIMS=3`)
- MEM_PATTERN=CONTIG, FILE_PATTERN=CONTIG
#### 1 strided pattern for 1D array (`NUM_DIMS=1`)
- MEM_PATTERN=CONTIG, FILE_PATTERN=STRIDED


### Supported read patterns(h5bench_bdcatsio): IO_OPERATION=READ 
#### 1 pattern for 1D, 2D and 3D read (`NUM_DIMS=1` or `NUM_DIMS=2`)
- MEM_PATTERN=CONTIG, FILE_PATTERN=CONTIG, READ_OPTION=FULL, contiguously read through the whole data file.
#### 2 patterns for 1D read
- MEM_PATTERN=CONTIG, FILE_PATTERN=CONTIG, READ_OPTION=PARTIAL, contiguously read the first $TO_READ_NUM_PARTICLES elements.
- MEM_PATTERN=CONTIG, FILE_PATTERN=STRIDED, READ_OPTION=STRIDED 

### Sample settings
The following setting reads 2048 particles from 128 blocks in total, each block consists of the top 16 from every 64 elements. See HDF5 documentation for details of using strided access.
```
#   General settings
NUM_PARTICLES=16 M
TIMESTEPS=5
EMULATED_COMPUTE_TIME_PER_TIMESTEP=1 s
#========================================================
#   Benchmark data dimensionality
NUM_DIMS=1
DIM_1=16777216
DIM_2=1
DIM_3=1
#========================================================
#   IO pattern settings
IO_OPERATION=READ
MEM_PATTERN=CONTIG
FILE_PATTERN=CONTIG
#========================================================
#    Options for IO_OPERATION=READ
READ_OPTION=PARTIAL # FULL PARTIAL STRIDED
TO_READ_NUM_PARTICLES=2048
#========================================================
#    Strided access parameters
STRIDE_SIZE=64
BLOCK_SIZE=16
BLOCK_CNT=128
```
For more exampes, please find the config files and template.cfg in basic_io/sample_config/ directory. 

### To run the h5bench_vpicio and h5bench_bdcatsio
Both h5bench_vpicio and h5bench_bdcatsio use the same command line arguments:
- Single process run: 
  - `./h5bench_vpicio sample_write_cc1d_es1.cfg my_data.h5`
- Parallel run (replace mpirun with your system provided command, for example, srun on Cori/NERSC and jsrun on Summit/OLCF):
  - `mpirun -n 2 ./h5bench_vpicio sample_write_cc1d_es1.cfg output_file`

### Understanding the output
The metadata and raw data operations are timed separately, and overserved time and rate are based on the total time.

Sample output of h5bench_vpicio:
```
==================  Performance results  =================
Total emulated compute time 4000 ms
Total write size = 2560 MB
Data preparation time = 739 ms
Raw write time = 1.012 sec
Metadata time = 284.990 ms
H5Fcreate() takes 4.009 ms
H5Fflush() takes 14.575 ms
H5Fclose() takes 4.290 ms
Observed completion time = 6.138 sec
Raw write rate = 2528.860 MB/sec
Observed write rate = 1197.592 MB/sec
===========================================================
```

Sample output of h5bench_bdcatsio:
 ```
 =================  Performance results  =================
Total emulated compute time = 4 sec
Total read size = 2560 MB
Metadata time = 17.523 ms
Raw read time = 1.201 sec
Observed read completion time = 5.088 sec
Raw read rate = 2132.200 MB/sec
Observed read rate = 2353.605225 MB/sec
```
## --------------------------------------------------------------------------------


## h5bench_exerciser
We modified this benchmark slightly so to be able to specify a file location that is writable. Except for the first argument $write_file_prefix, it's identical to the original one. Detailed README can be found int source code directory, the original can be found here https://xgitlab.cels.anl.gov/ExaHDF5/BuildAndTest/-/blob/master/Exerciser/README.md

Example run:

   - `mpirun -n 8 ./h5bench_exerciser $write_file_prefix -numdims 2 --minels 8 8 --nsizes 3 --bufmult 2 --dimranks 8 4`

## The metadata stress test: h5bench_hdf5_iotest
This is the same benchmark as it's originally found at https://github.com/HDFGroup/hdf5-iotest. We modified this benchmark slightly so to be able to specify the config file location, everything else remains untouched.

Example run:

   - `mpirun -n 4 ./h5bench_hdf5_iotest hdf5_iotest.ini`

## Streaming operation benchmark: h5bench_vl_stream_hl
This benchmark tests the performance of append operation. It supports two types of appends, FIXED and VLEN, represents fixed length data and variable length data respectively.
Note: This benchmark doesn't run in parallel mode.
#### To run the benchmark

`./h5bench_vl_stream_hl write_file_path FIXED/VLEN num_ops`

Example runs:

    - ` ./h5bench_vl_stream_hl here.dat FIXED 1000`
    - ` ./h5bench_vl_stream_hl here.dat VLEN 1000`
