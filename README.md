#Introduction

This document is a terse tutorial/reference on using the UCI High Performance Cluster, or [HPC](http://hpc.oit.uci.edu).

This is intended to me more up to date and easier to update than a previous [document](http://hpc.oit.uci.edu/~krthornt/BioClusterGE.pdf) that I have provided.  This document will not be as entry-level, and will assume some comfort in a Linux-like environment.

Currently, it is a list of useful commands for how to do various things.

##Running jobs

Good info [here](http://hpc.oit.uci.edu/running-jobs).

In general a job script needs to look like this:

```
#!/bin/sh

#Specify your queues
#$ -q queue1,queue2

#Optional: request resources
#$ -pe openmp 8-32
$# -R y

#Optional: define the job as an array job
#$ -t 1-10

#Often needed: load necessary modules that your job needs
module load module1
module load module2

#run the commands
command1 (arguments)
#etc.
```

##Queues

###Queues available to your user

```
q
```

###Submitting to a particular queue

Add the following lines to your jobs:
```
#$ -q queue1,queue2
```

Or add the -q to your qsub command:

```
qsub -q queue1,queue2 job.sh
```

Concrete example:
```
#$ -q krt,bio,abio,pub64,free64
```

###What is going on with a queue?

All the info:
```
qstat -q queuename
```

Just running jobs:
```
qstat -q queuename -s r
```

Just pending jobs:
```
qstat -q queuename -s p
```

Pending jobs submitted by a particular user:
```
qstat -u username
```

The above may all be mixed and matched

##Job resources

###Selecting multiple cores on one node.

To request 32 cores, add this to your job script:

```
#$ -pe openmp 32
#$ -R y
```

To request 16 to 64 cores:

```
#$ -pe openmp 16-64
#$ -R y
```

Then, when your job starts running, use the $CORES shell variable to tell your program how many cores to use.  For exampe, with the [bwa](http://bio-bwa.sourceforge.net/) aligner, a job may look like:

```
#!/bin/sh

#$ -q bio
#$ -pe openmp 8-64
#$ -R y 

module load load bwa/0.7.7

bwa aln -t $CORES in.fq > in.sai
```

##Modules

###List available modules
```
module avail
```

###List loaded modules
```
module list
```

###Load a module
```
module load modulename
```

For example,
```
module load boost/1.53.0
```

Some modules are contributed by users like myself (username krthornt), and are available to everyone (they show up in module avail described above):
```
module load krthornt/libsequence/1.7.8
```

###Remove a loaded module
```
module rm modulename
```

###Installing your own modules

```
mkdir /data/apps/user_contributed_software/$USER/modulename
mkdir /data/apps/user_contributed_software/$USER/modulename/version
```

Install the "stuff" for the module in /data/apps/user_contributed_software/$USER/modulename/version.

Concrete example using [libsequence](https://github.com/molpopgen/libsequence):
```
#get the code
git clone https://github.com/molpopgen/libsequence
cd libsequence
#checkout particular release
git checkout 1.8.0
#load dependency
module load boost/1.53.0
#make directories
mkdir /data/apps/user_contributed_software/krthornt/libsequence/1.8.0 
./configure --prefix /data/apps/user_contributed_software/krthornt/libsequence/1.8.0 
make
make install
```

We can then see the it is installed:
```
ls -1 /data/apps/user_contributed_software/krthornt/libsequence/1.8.0 
include
lib
```

And the header files are in include and the libraries are in lib.

You then need to make a module file:

```
mkdir /data/modulefiles/user_contributed_software/krthornt/modulename
```

Then a file called /data/modulefiles/user_contributed_software/krthornt/modulename/version must be made which does the "right things".  For libsequence 1.7.8, this module file looks like the following, adding the correct paths to the CPPFLAGS and LDFLAGS variables:

```
cat /data/modulefiles/user_contributed_software/krthornt/libsequence/1.8.0
#%Module1.0
# ---------------------------------------------------------------------------
# This module file will load to install a specific software mentioned below
# on the HPC cluster. Path names are relative and are defined below. The rest
# is self explanatory. Original work based off of Joseph's module files with
# logger support from Harry.
#
# @author           Adam Brenner        <aebrenne@uci.edu>
# @version              2.0
# @date                 09/2012


module-whatis "A C++ class library for population genetic analysis.  From www.molpopgen.org"

exec /bin/logger -p local6.notice -t module-hpc $env(USER) "krthornt/libsequence/1.8.0"

set ROOT /data/apps/user_contributed_software/krthornt/libsequence/1.8.0

module load boost/1.53.0
module load gcc/4.7.3
module load zlib/1.2.7

#setenv          GSLDIR             $ROOT
prepend-path    PATH               $ROOT/bin
prepend-path    LD_LIBRARY_PATH    $ROOT/lib
prepend-path    LD_RUN_PATH        $ROOT/lib
#prepend-path    PKG_CONFIG_PATH    $ROOT/lib/pkgconfig

## for dev/lib installs
prepend-path -d " "   LDFLAGS           "-L${ROOT}/lib"
prepend-path -d " "   LIBS              "-lsequence -lz"
prepend-path -d " "   CPPFLAGS          "-I$ROOT/include"
prepend-path          INCLUDE           "$ROOT/include"


# prepend-path  PATH              $ROOT/bin
# prepend-path  LD_LIBRARY_PATH   $ROOT/lib
# prepend-path  LDFLAGS           "-L $ROOT/lib"
# prepend-path  LIBS              "-l____"
# prepend-path  CPPFLAGS          "-I $ROOT/include"
```

Now, if I say

```
module load krthornt/libsequence/1.8.0
```

And then:

```
echo $LDFLAGS
```

I see:

```
-L/data/apps/user_contributed_software/krthornt/libsequence/1.8.0/lib -L/data/apps/gcc/4.7.3/lib64 -L/data/apps/gcc/4.7.3/lib -L/data/apps/boost/1.53.0/lib
```


