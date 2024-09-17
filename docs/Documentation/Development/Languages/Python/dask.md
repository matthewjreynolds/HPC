---
layout: default
title: Dask
parent: Python
nav_order: 2
has_children: true
grand_parent: Programming Languages
---


# Dask

Dask is a framework for parallelizing Python code. The most common use case is to enable Python programmers to scale scientific and machine learning analyses to run on distributed hardware. Dask has similarities to Apache Spark (see [FAQ](https://docs.dask.org/en/stable/faq.html#how-does-dask-compare-with-apache-spark) for comparison), but Dask is more Python native and interfaces with common scientific libraries such as NumPy and Pandas.

## Installation

!!! Warning
    Conda environments should be *always* be installed outside of your home directory for storage and performance reasons. **This is especially important for frameworks like Dask**, whose parallel processes can particularly strain the `/home` filesystem. Please refer to our dedicated [conda documentation](../../../Environment/Customization/conda.md#reduce-home-directory-usage) for more information on how to setup your conda environments to redirect the installation outside of `/home` by default.

Dask can be installed via Conda/Mamba. For example, to install Dask into a new environment from `conda-forge` into your `/projects` allocation folder, first load the appropriate conda (or mamba) module (e.g., `module load mamba` on Kestrel), and then run the following on a compute node.


```
# Be sure to replace "<allocation_handle>" with your HPC project.

# interactive job
salloc -A <allocation_handle> -p debug -t 01:00:00

# load mamba module
ml mamba

# create and activate `dask-env` environment with Python 3.12
mamba create --prefix=/projects/<allocation_handle>/dask-env conda-forge::python=3.12 conda-forge::dask
conda activate /projects/<allocation_handle>/dask-env
```

This installs Dask along with common dependencies such as NumPy. Additionally, the `dask-jobqueue` package (discussed below), can be installed via:

```
mamba install conda-forge::dask-jobqueue
```

Further, there is the `dask-mpi` package (also discussed below). To ensure compatibility with the system MPI libraries, it is recommended to install `dask-mpi` using pip. As such, we recommending installing any conda packages first. `dask-mpi` depends on `mpi4py`, although we have found that the pip install command does not automatically install `mpi4py`, so we install it explicitly. Also, installation of `mpi4py` will link against the system libraries, so the desired MPI library should be loaded first. In addition, it may be necessary to explicitly specify the MPI compiler driver. For example, to install mpi4py on Kestrel using the Intel programming environment and its associated MPI (`PrgEnv-intel`), you would do the following:

```
module load PrgEnv-intel
MPICC=`which mpicc` pip install dask-mpi mpi4py
```

## Dask single node

Dask can be used locally on your laptop or an individual node. Additionally, it provides wrappers for multiprocessing and threadpools. One advantage of using `LocalCluster` is that you can easily drop in another cluster configuration to further parallelize, with minimal modification of the code.

The following is a simple example that uses a local cluster with the `dask.delayed` interface, which can be used when the problem doesn't fit into one of the built-in collection types such as `dask.array` or `dask.dataframe`:

??? example "Dask local cluster"

    ```python
    from distributed import Client, LocalCluster
    import dask
    import time
    import random 
    
    @dask.delayed
    def inc(x):
        time.sleep(random.random())
        return x + 1
    
    @dask.delayed
    def dec(x):
        time.sleep(random.random())
        return x - 1
    
    @dask.delayed
    def add(x, y):
        time.sleep(random.random())
        return x + y
    
    def main ():
       cluster = LocalCluster(n_workers=2)
       client = Client(cluster)
       zs = []
       for i in range(256):
          x = inc(i)
          y = dec(x)
          z = add(x, y)
          zs.append(z)
       
       result = dask.compute(*zs)
       print (result)
    
    
    if __name__ == "__main__":
       main()
    ```

## Dask Jobqueue

The [`dask-jobqueue`](https://jobqueue.dask.org/en/latest/index.html#) library makes it easy to deploy Dask to a distributed cluster using Slurm (via [SLURMCluster](https://jobqueue.dask.org/en/latest/generated/dask_jobqueue.SLURMCluster.html)).  This is particularly useful when running an interactive notebook, where the workers can be scaled dynamically. 

For the following example, first make sure that both `dask` and `dask-jobqueue` have been installed.  Create a file named `dask_slurm_example.py` with the following contents, and replace `<project>` with your project allocation.

Assuming you are on Kestrel, this example will request two jobs from the `shared` partition.

??? example "`dask_slurm_example.py`"

    ```python
    from dask_jobqueue import SLURMCluster
    import socket
    from dask.distributed import Client
    from collections import Counter
    
    cluster = SLURMCluster(
       cores=18,
       memory='24GB',
       account='<allocation_handle>',
       walltime='00:30:00',
       processes=17,
       queue='shared'
    )
    
    client = Client(cluster)
    
    def test():
       return socket.gethostname()
    
    result = []
    cluster.scale(jobs=2)
    
    for i in range(2000):
       result.append(client.submit(test).result())
       
    print(Counter(result))
    print(cluster.job_script())
    ```
    
Then the script can simply be executed directly from a login node:

```bash
python dask_slurm_example.py
```

Note that although 2 jobs are requested, Dask launches the jobs dynamically, so depending on the status of the job queue, your results may indicate that only a single node was used.


## Dask MPI

Dask also provides a package called [`dask-mpi`](http://mpi.dask.org/en/latest/index.html) that uses MPI to create the cluster.  Note that `dask-mpi` only uses MPI to start the cluster, not for inter-node communication.

Dask-MPI provides two interfaces to launch Dask, either from a batch script using the Python API, or from the command line.

Here we show a simple example that uses Dask-MPI with a batch script.  Make sure that you have installed `dask-mpi` following the [Installation Instructions](#installation).  Create `dask_mpi_example.py` and `dask_mpi_launcher.sh` with the contents below.  In `dask_mpi_launcher.sh`, replace `<project>` with your allocation, and `/path/to/dask-env` with the full conda prefix path into which you [installed dask](#installation).

??? example "`dask_mpi_example.py`"

    ```python
    from dask_mpi import initialize
    from dask.distributed import Client
    import socket
    import time
    from collections import Counter
    
    def test():
       return socket.gethostname()
       
    def main():
       initialize(nthreads=5)
       client = Client()
       time.sleep(15)
    
       result = []
    
       for i in range (0,100):
          result.append(client.submit(test).result())
          time.sleep(1)
          
       out = str(Counter(result))
       print(f'nodes: {out}')
    
    main()
    ```
    
??? example "`dask_mpi_launcher.sh`"

    ```bash
    #!/bin/bash 
    #SBATCH --nodes=2
    #SBATCH --ntasks=4
    #SBATCH --time=10
    #SBATCH --account=<project>
    
    ml mamba
    conda activate /path/to/dask-env
    srun -n 4 python dask_mpi_example.py
    ```
    
The job is then launched as:

```bash
sbatch dask_mpi_launcher.sh
```

!!! warning

    We have observed errors such as `distributed.comm.core.CommClosedError` when using `dask-mpi`.  These errors may be related to known issues such as [GitHub Issue #94](https://github.com/dask/dask-mpi/issues/94).  Users that experience issues with `dask-mpi` are encouraged to use `dask-jobqueue` instead.

## References
[Dask documentation](https://docs.dask.org/en/latest/)

[Dask Jobqueue](https://jobqueue.dask.org/en/latest/)

[Dask MPI](http://mpi.dask.org/en/latest/)