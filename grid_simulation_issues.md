## Problem ##
In our cluster architecture, one of our primary goals is to allow for the graceful removal and addition of nodes to the cluster. We would also like to accomodate both cloud-style workloads as well as traditional HPC-style workloads. This sort of resilience to sudden node availability changes is inherent in many cloud software stacks, so cloud workloads should tolerate it well without much adjustment. However, traditional HPC-style programming frameworks (such as MPI)  and schedulers (such as PBS) assume that the underlying hardware present at the beginning of the job will remain, unchanged, until the end of the job.

## Electromagnetics example ##
In the finite-difference time-domain (FDTD) method of simulating electromagnetics problems, a 3D space is discretized into some number of cells along each axis. Given some intial conditions and boundary conditions, Maxwell's curl equations are then solved using finite-difference approximations of the spatial and time derivatives. In the explicit FDTD method, the influence of each node on its neighbors must be directly computed at each time step.

The explicit FDTD method can be implemented within MPI, with the first example in the literature appearing in 2001 in an article by Guiffaut and Mahdjoubi [1]. They describe a scheme for decomposing the 3D spatial grid into several subdomains along two axes. Each worker in the MPI pool receives one of these subdomains to work on. For nodes in the interior of each subdomain, calculations can be carried out node-to-node with no inter-subdomain communication. However, each node that lies on a boundary between subdomains must use the field values computed at its neighbor to fully compute its own field values. Hence, significant communication must occur at each time step among these boundary nodes

## Fluctuating hardware environment ##
In a standard HPC environment, the programmer would know ahead of time how many (computing) nodes they could reasonably expect to use for a simulation. They would write their scheduling script to request a certain number of nodes, and the scheduler would allocate their job resources accordingly. Within the program, MPI would then be used to split the simulation grid into subdomains, allocate each subdomain to a computation node, and manage communication between the subdomains. Throughout the simulation, the underlying hardware remains invariant, and in the case that it does not, the program fails.

However, in our proposed cloud architecture, the underlying hardware is not guaranteed to remain invariant.
At any point, computational nodes may be

1. removed with some warning (e.g., the scheduler decided it doesn't like you as much anymore, so it's giving away some of your resources), 
2. removed with no warning (e.g., hardware fails),
3. added with some warning (e.g. the scheduler likes you again, gives you more resources),
4. or added with no warning (e.g., a failed node comes back online).

No mater which of these occurs, the running program must tolerate the change.

This issue can be addressed one of two ways. Either the underlying software architecture or scheduler can handle it on the fly, presenting no apparent changes to the running program, or the program can handle it. Here I propose a method for a simulation handling these sorts of change gracefully.

## Handling hardware change within the simulation ##
Here is the basic procedure:

* Removal
    1. At certain intervals, the full state of the simulation is saved to disk.
    2. If a warning comes that a node is going to be removed, the full state of the simulation is immediately written to disk.
    3. Upon removal of the node, the simulation is paused, and a new subdomain breakdown is computed.
    4. Results are reloaded from the last save state and the simulation resumes with new settings.
* Addition
    1. Note that this procedure does not change for sudden or planned addition.
    2. When a node is added, the simulation is paused to make a decision.
    3. If the master process decides that incorporating the new node(s) would be worth the overhead involved in reconfiguring the subdomains, the current state of the simulation is saved to disk, the subdomains are reconfigured, and the simulation is resumed using the new configuration.

This method requires some significant additional programming when developing the simulation, which may deter users. However, use of this method could be incentivized by providing discounts for users who incorporate this method and indicate that their program can tolerate architecture changes.

## Handling hardware change in the cluster software stack ##
TKTK


***


[1]: Guiffaut, C., and K. Mahdjoubi. "A parallel FDTD algorithm using the MPI library." IEEE Antennas and Propagation Magazine 43.2 (2001): 94-103.