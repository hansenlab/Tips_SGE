---
author: Kasper D. Hansen
title: Tips for the Sub Grid Engine (SGE)
---

# Preface

Title: Tips for the Sun Grid Engine (SGE)  
Author: Kasper D. Hansen  
Home: [GitHub](https://github.com/kasperdanielhansen/Tips_SGE)  
License: CC BY  

# Dot files

In your home directory, create a file called `.sge_request`.  It contains standard parameters to any `qrsh` or `qsub` command.  Mine contains

```{bash}
-V -cwd -S /bin/bash -j yes -l h_stack=256M -l h_vmem=60G -m n -M kasperdanielhansen@gmail.com
```

- `-V`, `-cwd`: start in current directory and inherit all environment variables.
- `-S /bin/bash`: Let `/bin/bash` be the shell.
- `-j yes`: merge output and error scripts, the ones called `NAME.sh.oXX` and `NAME.sh.eXX`, into one file.
- `-l h_stack=256M`: You should always have this setting on JHPCE.  Basically, some programs won't run without it.
- `-m n -M kasperdanielhansen@gmail.com`: do not send emails, but if you do, use this email address.

Anything you type in at the command will override whatever is in your `.sge_request`.

# File headers

I start my shell scripts with the following

```{bash}
#!/bin/bash
#$ -l mf=5G,h_vmem=6G
set -e -u
```

The first line is actually ignored in SGE, but it is good style to show it is a bash script.

The second line gives arguments to `qsub` when the script is submitted.  It has to start with `#$`, and you can have several lines starting like this, if you want to spread the arguments over several lines.  This is an excellent place to keep track of script-specific settings, such as requests for memory and cores.

The last line has nothing to do with SGE per se, but it gives some options to Bash, namely "stop at an error" (`-e`) and "stop if you encounter undefined variables" (`-u`).  The later one sometimes has to be removed.

# Array jobs

Array jobs are truly awesome.  It lets you execute a script many times, the only difference being that an environment variable called `SGE_TASK_ID` is defined, which is equal to the array number.  This sounds weird, but consider the following R code:

```{r}
all_files <- list.files()
file <- all_files(Sys.getenv("SGE_TASK_ID")
```

Executing this script with `qsub -t 1-10` means it gets run 10 times and the variable takes values from 1 to 10.  You can do stuff like `qsub -t 1,4,6,10-12` to run the job for some specific values.

Here is an example of a script with of lot of checking, which takes an input directory `BASEDIR`.  The it uses `ls` to get of list of files ending with `.fastq.gz`.  It then does some bash array indexing magic so that `BASEFILE` becomes the name of the file corresponding to `SGE_TASK_ID` (subtracting 1 to deal with 1-indexing vs. 0-indexing).  The way `ls` works is that it returns the full path.  So using `basename` strips the leading apth from it and the funny construction in the last line chops off `.fastq.gz` from the name.

```{bash}
#!/bin/bash
#$ -V -j yes -cwd -S /bin/bash

if [ -z "${SGE_TASK_ID}" ]; then
  echo "Need to set SGE_TASK_ID"
  exit 1
fi

BASEDIR=/this/path

if [ ! -d ${BASEDIR} ]; then
    echo "BASEDIR=${BASEDIR} does not exist"
    exit
fi

BASEFILES=$(/bin/ls ${BASEDIR}/*.fastq.gz)
BASEFILES_ARRAY=(${BASEFILES})
BASEFILE=${BASEFILES_ARRAY[(${SGE_TASK_ID} - 1)]}

f [ ! -f ${BASEFILE} ]; then
    echo "BASEFILE=${BASEFILE} does not exist"
    exit
fi

BASENAME=$(basename ${BASEFILE})
BASENAME=${BASENAME%%.fastq.gz}
```

# Holding jobs

If you have multiple scripts which should be executed in order, you can tell SGE to hold running them.  The easiest way to do this is to give the jobs a name, and then wait for the job.  Example 

```{bash}
qsub -N align align.sh
qsub -N sam2bam -hold_jid align sam2bam.sh
```

Here `-N` is the name of the job.

