#Introduction

This document is a terse tutorial/reference on using the UCI High Performance Cluster, or [HPC](http://hpc.oit.uci.edu).

This is intended to me more up to date and easier to update than a previous [document](http://hpc.oit.uci.edu/~krthornt/BioClusterGE.pdf) that I have provided.  This document will not be as entry-level, and will assume some comfort in a Linux-like environment.

Currently, it is a list of useful commands for how to do various things.

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

You then need to make a module file:

```
mkdir /data/modulefiles/user_contributed_software/krthornt/modulename
```

Then a file called /data/modulefiles/user_contributed_software/krthornt/modulename/version must be made which does the "right things".  For libsequence 1.7.8, this module file looks like the following, adding the correct paths to the CPPFLAGS and LDFLAGS variables:

```
cat /data/modulefiles/user_contributed_software/krthornt/libsequence/1.7.8 
#%Module1.0
# ---------------------------------------------------------------------------
# This module file will load to install a specific software mentioned below
# on the HPC cluster. Path names are relative and are defined below. The rest
# is self explanatory. Original work based off of Joseph's module files with
# logger support from Harry.
#
# @author	    Adam Brenner	<aebrenne@uci.edu>
# @version		2.0
# @date			09/2012


module-whatis "A C++ class library for population genetic analysis.  From www.molpopgen.org"

exec /bin/logger -p local6.notice -t module-hpc $env(USER) "krthornt/libsequence/1.16"

set ROOT /data/apps/user_contributed_software/krthornt/libsequence/1.7.8

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
