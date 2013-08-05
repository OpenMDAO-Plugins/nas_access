.. _`usage`:


===========
Usage Guide
===========

This package provides a mechanism to remotely submit jobs to the NASA High
End Computing (HEC) resources located at the Ames Research Center. Due to
security concerns, direct access to these systems requires either a manual
login process, or obtaining a time-limited `token`. The workaround implemented
here is to use the `DMZ` fileservers as an intermediary communication
channel between a local client and a special remote job entry (RJE) server.

.. figure:: dataflow.png
   :align: center

   Remote NAS Access Dataflow 

.. note::

    Due to the indirect nature of the communications between the local client
    and remote server, transactions are very slow (many seconds). Be patient
    when using this allocator and be advised that you probably don't want
    this allocator enabled unless you expect to require it.

To be able to remotely submit jobs you need to perform four configuration
operations:

#. "ssh" access from one or more HEC front-end machines (i.e., ``pfe1``) to one
   or both of the DMZ servers (``dmzfs1.nas.nasa.gov`` and ``dmzfs2.nas.nasa.gov``)
   must be set up so that no user interaction is required (see 'ssh' documentation
   for details).

#. "ssh" (Linux/Mac) or "plink"/"pscp" (Windows) access from your local machine
   to one or both of the DMZ servers must be set up so that no user interaction is
   required.

#. On the HEC front-end machine(s) your OpenMDAO environment must be configured
   to enable the :mod:`nas_access` and :mod:`pbs` packages, contained in the
   ``contrib`` area. This can be done by installing them or setting ``PYTHONPATH``
   appropriately. The ``PBS_Allocator`` should also be made available in your
   ``~/.openmdao/resources.cfg`` file
   
   ::

     [PBS]
     classname: pbs.PBS_Allocator
     accounting_id: no-default-set

   where ``no-default-set`` is replaced by your group account ID.

#. On your local machine your OpenMDAO environment must be configured to enable
   the :mod:`nas_access` package.

Use of the remote submission capability requires that the RJE server be
running on an HEC front-end machine and the simulation on your local machine
be set up to use the ``NAS_Allocator``.

First, start the RJE server on a front-end machine (i.e., ``pfe1``)::

    python -m nas_access.rje

note that the directory in which this command is run will be used for the
server log file as well as the parent directory for object server directories.

Next, set up your environment on the local machine to use the ``NAS_Allocator``.
This can be done by updating your ``~/.openmdao/resources.cfg`` file::

    [Pleiades]
    classname: nas_access.NAS_Allocator
    dmz_host: dmzfs1.nas.nasa.gov
    server_host: pfe1

Note that the ``dmz_host`` and ``server_host`` entries may be different
depending on which remote hosts you intend to use.

Alternatively, you can programmatically add the allocator to the resource
manager in your simulation code::

    from openmdao.main.resource import ResourceAllocationManager as RAM
    from nas_access import NAS_Allocator

    allocator = NAS_Allocator(name='Pleiades',
                              dmz_host='dmzfs1.nas.nasa.gov',
                              server_host='pfe1')
    RAM.add_allocator(allocator)

Now when you run your local simulation and it queries the resource allocation
manager for where to run an :class:`ExternalCode`, the ``Pleiades`` allocator
will participate in the process. Depending upon the resource description,
this allocator may be selected to be used for the run. This can easily be
forced by setting ``allocator='Pleiades'`` (or whatever you've named the
allocator).

A :ref:`tutorial` covering installation and testing step-by-step is available.

Consult the :ref:`nas_access_src_label` section for implementation details.


=====
Hints
=====


*Execution Directory*
_____________________

The default execution directory and file access directory is the
directory in which the OpenMDAO object server is running (a dynamically
generated subdirectory of the RJE server). You may set the execution directory
in the resource description, but at this time this has no effect on file
transfers. Consider having your job submission be a script which
can copy or link to other files on the remote host rather than setting the
execution directory to where those files reside.

Normally the execution directory is cleaned-up once the simulation has
completed. This can make for difficult debugging. To avoid having the directory
removed, set the environment variable ``OPENMDAO_KEEPDIRS`` to 1 before
starting the RJE server.


*Handling System Reboots*
_________________________

At NAS systems occasionally go down (typically for scheduled maintenance, but
sometimes without warning).  This can happen for any of the systems: front-ends,
DMZ servers, etc.  You can increase the reliability of accessing NAS by
defining multiple RJE servers using multiple front-end systems and multiple
DMZ servers.  Then, if one of the RJE servers is unavailable the Resource
Allocation Manager should be able to use one of the other RJE servers.
Note that for this to work your resource description should use 'generic'
allocation terms rather than ``allocator=Pleiades``.  Note that this will
only help for initial access.  If a server goes down during a job run there
currently is no way to recover.


*Execution Outside of PBS Job*
______________________________

Using NAS systems typically means submitting a job via PBS.  If your RJE server
is configured to use the PBS allocator, then your command will be executed
entirely within a PBS job.  If you have pre- or post-processing activities
which you don't want to count against your PBS allocation, you can instead
use the ``LocalHost`` allocator to run a command which does the preprocessing,
submits the PBS job, and then does the postprocessing.  This requires that
you perfom the ``qsub`` operation yourself.  Be sure to use the
``-W block=true`` option to ensure that ``qsub`` will not return until the
job has completed.  Also, when using a Local allocator for non-CPU intensive
operations, setting ``max_load`` to something higher than the default of 1
will ensure your submission isn't held up just because the local system
is busy.

To have your RJE server use a local allocator specify it on the command line::

    python -m nas_access.rje --allocator LocalHost


*Usage Within the GUI*
______________________

The only tricky part to using nas_access in the GUI is dealing with configuring
the resource manager.

Suppose you have a file that runs from the command line and in ``main()`` it
configures the resource manager to include a NAS_Allocator::

    def main():
        allocator = NAS_Allocator(name='Pleiades',
                                  dmz_host='dmzfs1.nas.nasa.gov',
                                  server_host='pfe22')
        RAM.add_allocator(allocator)

        # Configure and run a simulation.

If you 'execute' the file from within the GUI it's just like executing from the
command line, *but* when you reopen the project the 'execute' is replayed,
including running the simulation, which can be inconvenient.  An alternative is
to have a file that just configures the resource manager.  Execute that file.
Now add class files like normal and configure your simulation.  When you reopen
the project the resource manager configure script will be replayed, but the
rest of the model is just opened, not re-run.

