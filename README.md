# Basic Instructions for Using PFIO


**MpiServer Class**
- The clients sends the data to `Oserver`.
- All processors in `Oserver` would coordinate to create different shared memory windows for different collections.
- The processors use one-sided `MPI_PUT` to send the data to the shared memory.
- Different collections are written by different processors. Those writing processors are distributed among nodes as eveny as possible.
- All the other processors have to wait for the wrting processors to  finish jobs before responding to Clients’ next round of requests.



**MultiGroupServer Class**
- The oserver is devided into frontend and backend.
- When the frontend receive the data,  its root process asks backend‘s root (or head) for an idle process for each collection. Then it broadcasts the info to the other frontend processes.
- When the frontend processors  forward (`MPI_SEND`)  the data to the backend ( different collections to different backend processors), they get back the the clients without waiting for the actual writing.


## Command Line

There are two options to submit an executable through the `mpi_run` or `mpi_exec` command.

### Regular MPI Command
If the regular `mpi_run` or `mpi_exec` command is used:

    mpi_run -np npes ExeccuTable

the `MpiServer` is used as `oserver`.
The `client` processes are overlapping with `oserver` processes.
The `client` and `oserver` are sequetically working together.
When ``client`` sends data, it actually makes a copy, then the `oserver` takes over the work,
i.e., shuffling data and writing data to the disk. After `MpiServer` is done, the `client` moves on.

### Command with IOserver Options
**If you want to use in total `npes` processes among which `n1` are reserved for the model 
(doing calculations) and `n2` for the `MpiServer`, use the command:**

    mpi_run -np npes ExeccuTable –npes_model n1  --npes_output_server n2

- Note that `npes` is not equal to `n1+n2`.
- The `client` (model) will use the minimum number of nodes that contain `n1` cores. For example, if each node has `n` processors, the `npes = ceiling(n1/n)*n + n2`.
- If  `--isolate_nodes` is set to false ( by default, it is true), the `oserver` and `client` can co-exist in the same node, and `npes = n1 + n2`.
- `--npes_output_server n2` can be replaced by  `--nodes_output_server n2`. Then the `npes = ceiling(n1/n)*n + n2*n`.

**If you want to use in total `npes` processes among which `n1` are reserved for the model 
(doing calculations) and `n2` for the `MultiGroupServer`, use the command:**

    mpi_run -np npes ExeccuTable –npes_model n1 --npes_output_server n2 --oserver_type multigroup --npes_backend_pernode n3

- For each node of oserver, `n3` processes are used as backend.
- For example, if each node has `n` cores, then `npes = ceiling(n1/n)*n + n2*n`.
- The frontend has `n2*(n-n3)` processes and the backend has `n3*n` processes.
- The frontend has `ceiling(n2/n)*(n-n3)` processes and the backend has `n3*n` processes.

**If you want to pass a vector of `oservers`, use:**

    mpi_run -np npes ExeccuTable –npes_model n1  --npes_output_server n2 n3 n4

- The command creates `n2`-node, `n3`-nodes and `n4`-nodes `MpiServer`.
- The `oservers` are independent. The client would take turns to send data to different `oservers`.
- If each node has `n` processors, then `npes = ceiling(n1/n)*n + (n2+n3+n4)*n`.
- **Advantage**: Since the `oservers` are independent, the `client` has choice to send data to idle `oserver`.
- **Disavantage**: Finding an idle `oserver` is not easy.

**If you want to pass a vector of `oservers` and the `MultiGroupServer`, use the command:**

    mpi_run -np npes ExeccuTable –npes_model n1  --npes_output_server n2 n3 n4 --oserver_type multigroup --npes_backend_pernode n5

- The command creates `n2`-node, `n3`-nodes and `n4`-nodes `MultiGroupServer`.
- The `oservers` are independent. The `client` would take turns to send data to different `oservers`.
- If each node has `n` processors, then `npes = ceiling(n1/n)*n + (n2+n3+n4)*n`.
- Each `oserver` has `n2*n5`, `n3*n5`, and `n4*n5` backend processes respectively.


**If you want `MpiServer` to use one-sided `mpi_put` and shared memory:**

   mpi_run -np npes ExeccuTable –npes_model n1 --npes_output_server n2 --one_node_output true

- The option `--one_node_output true` makes it easy to create `n2` oservers and each is one-node oserver.
- It is equivalent to `--nodes_output_server 1 1 1 1 1 ...' with `n2` “1”s.

**Additional Options**

`--fast_oclient true`

- After the client sends history data to the oserver, by default it waits and makes sure all the data is sent even it uses non-blocking isend. If this option is set to true, the client copies the data before non-blocking isend. It waits and cleans up the copies next time when it re-uses the oserver.

## Example of an Application

The file `pfio_MAPL_demo.F90" is a standalone program that implement the use of PFIO.
It writes several time records of 2D and 3D arrays.
The compilation of the program generates the executable, `pfio_MAPL_demo.x`.
If we reserve 2 `haswell` nodes (28 cores in each), want to run the model on 28 cores and use 1 `MultiGroup` with 5 backend processes, then the execution command is:

    mpiexec -np 56 pfio_MAPL_demo.x --npes_model 28 --oserver_type multigroup --nodes_output_server 1 --npes_backend_pernode 5

- The frontend has `28-5=23` processes and the backend has `5` processes.

