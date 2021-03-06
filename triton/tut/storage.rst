============
Data storage
============

.. seealso::

   * :doc:`../usage/lustre`
   * :doc:`../usage/localstorage`
   * :doc:`../usage/quotas`
   * :doc:`../usage/smallfiles`
   * The storage advanced page: :doc:`../usage/storage`

In this tutorial, we go over places to store data on Triton and how to
access it remotely.

Basics
======

Triton has various ways to store data.  Each has a purpose, and when
you are dealing with the large data sets or intensive IO, efficiency
becomes important.

Roughly, we have small **home** directories (only for configuration
files), large Lustre (**scratch** and **work**, large, primary calculation
data), and special places for scratch during computations (**local
disks**).  At Aalto, there is **aalto home**, **project**, and
**archive** directories which, unlike Triton, are backed up but don't
scale to the size of Triton.

A file consists of its contents and metadata.  The metadata is things
like user, group, timestamps, permissions.  To view metadata, use ``ls
-l`` or ``stat``.

Filesystem performance can be measures by both IOPS (input-output
operations per second) and stream I/O speed.  ``/usr/bin/time -v`` can
give you some hints here.  You can see the :doc:`profiling
<../usage/profiling>` page for more info.

Think about I/O before you start! - General notes
=================================================

When people think of computer speed, they usually think of CPU speed.
But this is missing an important factor: how fast can data get to the
CPU?  In very many cases, input/output (IO) is the true bottleneck and
must be considered just as much as processor speed.  **In fact,
modern computers and especially GPUs are so fast, it is very easy for
a few GPUs with bad data access patterns to bring the cluster down for
everyone.**

The solution is similar to how you have to consider memory: there are
different types of filesystems with different tradeoffs between speed,
size, and performance, and you have to use the right one for the right
job.  Often times, you have to use several in tandem: for example,
store original data on **archive**, put your working copy on
**scratch**, and maybe even make a per-calculation copy on **local
disks**.

Consider:

-  How much IO in the first place?  Do you continually re-read the
   same data?
-  What's the pattern of it, and which filesystem is best for it?  If
   you read all at once, scratch is fine, but if there are many small
   files or random access, local disks may help.
-  Do you write log files / checkpoints more often than is needed?
-  Some programs use local disk as swap-space. Only turn on if you know
   it is reasonable.

There's a checklist in the :doc:`storage details page
<../usage/storage>`.

Avoid many small files! Use a few big ones instead. (we have a
:doc:`dedicated page <../usage/smallfiles>` on the matter)

Summary table
=============

.. include:: ../ref/storage.rst

Home directories
^^^^^^^^^^^^^^^^
The place you start when you log in.  For user init files, some small
config files, etc.  No calculation data. Daily backup.  Usually you
want to use scratch instead.

scratch and work: Lustre
^^^^^^^^^^^^^^^^^^^^^^^^
This is the big, high-performance, 2PB Triton storage.  The primary
place for calculations, data analyzes etc.  Not backed up but is
reliable against hardware failures (RAID6, redundant servers), but
*not* safe against human error..  It is
shared on all nodes, and has very fast access.  It is divided into two
parts, **scratch (by groups)** and **work (per-user)**.  In general, always
change to ``$WRKDIR`` or a group ``scratch`` directory when you first
log in and start doing work.

Lustre separates metadata and contents onto separate object and
metadata servers.  This allows fast access to large files, but a
larger overhead than normal filesystems.  See :doc:`our info on small
files <../usage/smallfiles>`.

See :doc:`../usage/lustre`

Local disks
^^^^^^^^^^^
Local disks are on each node separately.  For the fastest IOs with
single-node jobs. It is cleaned up after job is finished.  Since 2019,
things have gotten a bit more complicated since our newest (skl) nodes
don't have local disks.  If you want to ensure you have local storage,
submit your job with ``--gres=spindle``.

See the :doc:`Compute
node local drives <../usage/localstorage>` page for further details and script
examples.

ramfs - fast and highly temporary storage
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**On login nodes only**,
``$XDG_RUNTIME_DIR`` is a ramfs, which means that it looks like files
but is stored only in memory.  Because of this, it is extremely fast,
but has no persistence whatsoever.  Use it if you have to make small
temporary files that don't need to last long.  Note that this is no
different than just holding the data in memory, if you can hold in
memory that's better.


Quotas
======

All directories under ``/scratch`` (as well as ``/home``) have quotas. Two
quotas are set per-filesystem: disk space and files number.

Disk quota and current usage are printed with the command ``quota``.
'space' is for the disk space and 'files' for the total files number
limit. There is a separate quota for groups on which the user is a
member.

::

    $ quota
    User quotas for darstr1
         Filesystem   space   quota   limit   grace   files   quota   limit   grace
    /home              484M    977M   1075M           10264       0       0
    /scratch          3237G    200G    210G       -    158M      1M      1M       -

    Group quotas
    Filesystem   group                  space   quota   limit   grace   files   quota   limit   grace
    /scratch     domain users            132G     10M     10M       -    310M    5000    5000       -
    /scratch     some-group              534G    524G    524G       -    7534   1000M   1000M       -
    /scratch     other-group              16T     20T     20T       -   1088M      5M      5M       -

If you get a quota error, see :doc:`the quotas page <../usage/quotas>`
for the solution.


Accessing and transferring files remotely
=========================================

Transferring files to/from triton is exactly the same as any other
remote Linux server.

Remote mounting using SMB
^^^^^^^^^^^^^^^^^^^^^^^^^

By far, remote mounting of files is the easiest method to transfer files.  If you are
not on the Aalto networks (wired, ``eduroam``, or ``aalto`` with
Aalto-managed laptop), connect to the :doc:`Aalto VPN
</aalto/remoteaccess>` first.  Note that
this is automatically done on some department workstations (see
below) - if not, request it!

The scratch filesystem can be remote mounted using SMB inside secure
Aalto networks at the URLs

* scratch: ``smb://data.triton.aalto.fi/scratch/``.
* work: ``smb://data.triton.aalto.fi/work/$username/``.

On different operating systems:

* Linux (Ubuntu for example): File manager (Nautilus) → File →
  Connect to server.  Use the ``smb://`` URLs above.
* Windows: In the file manager, go to Computer (in menu bar on top, at
  least in Windows 10) → Map Network Drive) and "Map Network Drive".
  In Windows 10 → "This PC" → right click → "Add Network Location".
  (Note that this is different from right-click "Add network location"
  which just makes a folder link and has had some problems in the past.)
  Use the URLs above but replace ``smb://`` with ``\\`` and ``/`` with
  ``\``.  For example, ``\\data.triton.aalto.fi\scratch\``.
* Mac: Finder → Go → Connect to Server.  Use the ``smb://`` URLs above.

Depending on your OS, you may need to use either your username
directly or ``AALTO\username``.


Using sftp
^^^^^^^^^^

The *sftp* protocol uses ssh to transfer files.  On Linux and Mac, the
``sftp`` command line program are the must fundamental way to do this,
and are available everywhere.

A more user-friendly way of doing this (with a nice GUI) is the
`Filezilla program <https://filezilla-project.org/>`__.

Below is an example of the "raw" sftp usage::

    # Copying from HOME to local PC
    user@pc123 $ sftp user12@triton.aalto.fi:filename
    Connected to triton.aalto.fi.
    Fetching /home/user12/filename to filename
    # copying to HOME
    user@pc123 $ sftp -b - user12@triton <<< 'put testCluster.m'
    sftp> put foo
    # copying to WRKDIR
    user@pc123 $ sftp -b - user12@triton:/scratch/work/USERNAME/ <<< 'put testCluster.m'
    ...

With all modern OS it is also possible to just open your OS file manager (e.g. Nautilus on Linux) and just put as address in the bar::

    sftp://triton.aalto.fi

If you are connecting from remote and cannot use the VPN, you might connect instead to department machines like kosh, taltta, amor. 


Rsync
^^^^^

Rsync is similar to sftp, but is smarter at restarting files.  Use rsync
for large file transfers.  ``rsync`` actually uses ``ssh``, so
you can ``rsync`` from anywhere you can ``ssh`` from. ``Rsync`` is installed 
by default on Linux and Mac terminals. On Windows machines we recommend using `GIT-bash <https://gitforwindows.org/>`__.

While there are better places on the internet to read about rsync, it is good
to try it out to sychronise a local folder on your triton's scratch. Sometimes
the issue with copying files is related to group permissions. This command takes
care of permissions and makes sure that all your local files are identical (= same
MD5 fingerprint) to your remote files::

    rsync -avzc -e "ssh" --chmod=g+s,g+rw --group=GROUPNAME PATHTOLOCALFOLDER USERNAME@triton.aalto.fi:/scratch/DEPT/PROJECTNAME/REMOTEFOLDER/ 

Replease the bits in CAPS with your own case. Briefly, -a tries to preserve all attributes of the file, -v verbose to see what rsync is doing, -z use compression, -c skip files that have identical MD5 checksum, -e specify to use ssh (not necessary but needed for the commands coming after), --chmod sets the group permissions to shared (as common practice on scratch project folders), --group set the groupname to the group you belong to (note that GROUPNAME == PROJECTNAME on our scratch filesystem).

If you might want to just check if your local files are different from the remote ones. You can then run rsync in "dry run" so that you only see what the command would do, without actually doing anything.::

    rsync --dry-run -avzc ...

Sometimes you want to copy only certain files. E.g. go through all folders, consider only files ending with ``py``::
    
    rsync -avzc --include '*/' --include '*.py' --exclude '*' ...

Sometimes you want to copy only files under a certain size (e.g. 100MB)::

   rsync -avzc --max-size=100m ...

Rsync does NOT delete files by default, i.e. if you delete a file from the local folder, the remote file will not be deleted automatically, unless you specify the `--delete` option.

Please note that when working with files containing code or simple text, git is a better option to synchronise your local folder with your remote one, because not only it will keep the two folders in sycn, but you will also gain version controlling so that you can revert to previous version of your code, or txt/csv files.


Remote mounting using sshfs
^^^^^^^^^^^^^^^^^^^^^^^^^^^

``sshfs`` is a neat program that lets you mount remote filesystems via
ssh only.  It is well-supported in Linux, and somewhat on other
operating systems.  It's true advantage is that you can mount any
remote ssh server - it doesn't have to be set up specially for SMB or
any other type of mounting.  On Ubuntu, you can mount by "File → Connect to
server" and using ``sftp://triton.aalto.fi/scratch/work/USERNAME``.

The below uses command line programs to do the same, and makes the
``triton_work`` on your local computer access all files in
``/scratch/work/USERNAME``.  Can be done with other folders.::

    mkdir triton_work
    sshfs USERNAME@triton.aalto.fi:/scratch/work/USERNAME triton_work

Note that ``ssh`` binds together many ways of accessing Triton, with a
similar syntax and options.  ``ssh`` is a very important program and
binds together all types of remote access, and learning to use it well
will help you for a long time.


Exercises
^^^^^^^^^
1. Mount your work directory by SMB and transfer a file to Triton.
   Note that you must be on ``eduroam``, the ``aalto`` *with Aalto
   laptop*, or connected to the Aalto VPN.

2. Or, use rsync, sftp, or sshfs to transfer a file.

3. (Advanced) If you have a Linux on Mac computer, study the ``rsync``
   manual page and try to transfer a file.


Accessing files from Department workstations
============================================

This varies per department, with some strategies that work from
everywhere.

These mounts that are already on workstations require a valid Kerberos
ticket (usually generated when you log in). On long sessions these might
expire, and you have to renew them with ``kinit`` to keep going.

Generic
^^^^^^^

The staff shell server ``taltta.aalto.fi`` has scratch and work mounted
at ``/m/triton``, and department directories are also in the standard
paths ``/m/{cs,nbe}/{scratch,work}/``.

NBE
^^^

Work directories are available at ``/m/nbe/work`` and group scratch
directories at ``/m/nbe/scratch/$project/``.

PHYS
^^^^

Directories available on demand through SSHFS. See the `Data
transferring <https://wiki.aalto.fi/display/TFYintra/Data+transferring>`__ page
at PHYS Intranet (accessible by PHYS users only).

CS
^^

Work directories are available at ``/m/cs/work/``, and group scratch
directories at ``/m/cs/scratch/$project/``.


Exercises
=========

``strace`` is a command which tracks **system calls**, basically the
number of times the operating system has to do something.  It can be
used as a rudimentary way to see how much I/O load there is.

1. Use ``strace -c`` to compare the number of system calls in ``ls``,
   ``ls -l``, ``ls --no-color``, and ``ls --color``.  You can use the directory
   ``/scratch/scip/lustre_2017/many-files/`` as a place with many
   files in it.  How many system calls per file were there for each
   option?

2. Using ``strace -c``, compare the times of ``find`` and ``lfs find``
   on the directory mentioned above.  Why is it different?

3. (Advanced, requires slurm knowledge from future tutorials)  You
   will find some sample files in ``/scratch/scip/examples/io``.
   Create a temporary directory and...

   a) Run ``create_iodata.sh`` to make some data files in ``data/``

   b) Compare the IO operations of ``find`` and ``lfs find`` on this
      directory.

   c) use the ``iotest.sh`` script to do some basic analysis.  How
      long does it take?  Submit it as a slurm batch job.

   d) Modify the iotest.sh script to copy the ``data/`` directory to
      local storage, do the operations, then remove the data.  Compare
      to previous strategy.

   e) Use ``tar`` to compress the data while it is on lustre.  Unpack
      this tar archive to local storage, do the operations, then
      remove.  Compare to previous strategies.



Next steps
==========

The next tutorial is about :doc:`interactive jobs <interactive>`.

If you are doing anything IO heavy, you might want to read the
:doc:`advanced storage page <../usage/storage>`.

Optimizing data storage isn't very glamorous, but it's an important
part of high-performance computing.  You can find some hints on the
:doc:`profiling <../usage/profiling>` page.

We have these related pages:

* :doc:`../usage/lustre`
* :doc:`../usage/localstorage`
* :doc:`../usage/quotas`
* :doc:`../usage/smallfiles`
