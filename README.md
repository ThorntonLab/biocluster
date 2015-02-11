#Introduction

This document is a terse tutorial/reference on using the UCI High Performance Cluster, or [HPC](http://hpc.oit.uci.edu).

This is intended to be more up to date and easier to update than a previous [document](http://hpc.oit.uci.edu/~krthornt/BioClusterGE.pdf) that I have provided.   I suggest that users look there first for the basics.

This document will not be as entry-level, and will assume some comfort in a Linux-like environment.

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

##Array jobs

"Array jobs" let you submit multiple copies of the same task using one script.  A common application in my work is simulation: one wishes to simulate 1,000 replicates of the same parameters.  Rather than write 1,000 scripts that differ only by the random number seed, I can do the following instead:

```
#!/bin/sh

#$ -q bio
#$ -t 1-1000

SEED=`echo "$SGE_TASK_ID*$RANDOM"|bc -l`
simulation param1 param2 $SEED | gzip > outfile.$SGE_TASK_ID.gz
```

The script shown above will spawn one job ID number with 1,000 task ID numbers.  Each instance has its own shell variable called SGE\_TASK\_ID.  The script uses this variable to write each replicate to a different file.  It also uses the task ID to make sure that each replicate gets a unique seed, to the best of our ability (two tasks starting on the same second will get the same value from \$RANDOM, but multiplying by \$SGE\_TASK\_ID will result in two different values for that case).  Another option is to store 1,000 unique seeds in a file. If we call that file "seedfile.txt", we can do this:

```
SEED=`head -n $SGE_TASK_ID seedfile.txt | tail -n 1`
```

This will pull the i-th seed out for the i-th replicate.

###The above example is very, very bad!

For large simulation projects, the above example is a disaster for distributed file systems, and it __SHOULD NOT__ be done. The reason why is described [here](http://moo.nac.uci.edu/~hjm/Job.Array.ZOT.html).

 For programs that write to stdout (aka, "print to screen"), their output should be collected and written to a file using some form of file locking.  KRT has provided code for this [here](https://github.com/molpopgen/atomic_locker).  The UCI team that provides invaluable cluster support have provided solutions [here](http://moo.nac.uci.edu/~hjm/zotkill.pl), which can also be found from [here](http://moo.nac.uci.edu/~hjm/Job.Array.ZOT.html).

For custom programs where the output file name is an option, I recommend some sort of POSIX file locking procedure to avoid writing zillions of files.  

If you are using someone else's code that cannot be modified and will write to zillions of files in an array context, then you risk bringing your cluster's file system to its knees.

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

###Selecting nodes with a certain amount of memory.

Some HPC nodes have 256GB memory.  This may not be enough.  You can select the 512GB nodes only:

```
#$ -l mem_size=512
```

##Giving jobs names and pipelining jobs

Sometimes, you wish to run job A, followed by job B.  Yes, you could issue the commands for jobs A and B in the same script, but that may waste resources if they require, for example, very different numbers of cores.  For example, job A could be an array job, and job B is a very fast job that goes through the output of A to do some calculation.

We can "pipeline" our workflows using job names.

For job A,

```
#!/bin/sh

#$ -N JOBA

##Do the rest of A's work
```

Then, for job B:

```
#!/bin/sh

#$ -hold_jid JOBA

##Do rest of B's work
```

Then, we submit to the queue:

```
qsub A.sh
qsub B.sh
```

The result is that A will run as soon as its necessary resources are available.  Job "B" will remain held on the queue (status hqw) until job A completes, at which point the hold will release (changing B's status to qw) and B will execute as soon as sufficient resources become available.  This can be extended to job C, D, etc.

The -hold_jid flag can take comma-separate, quoted wildcards as options, e.g.

```{shell}
qsub -hold_jid 'additive*','multiplicative*' ../h2.sh
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

##&Checkpointing (caveat emptor)

```
qsub -notify -r y -ckpt blcr -l kernel=blcr job.sh
```

Or, this, which will just "kill and resubmit", and works quite well:

~~~
#$ -ckpt restart
~~~


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

##Changing the relative priorities of your jobs

Let's say that you have jobs A and B in the queue, and both is waiting for resources with A ahead of B.  You want to change things so that B runs first.  You can alter the relative priority of B like this:

```
qalter -js 100 B
```

The above is a hint from [here](http://www.ace-net.ca/wiki/FAQ#I_need_to_change_the_order_of_my_waiting_jobs)
