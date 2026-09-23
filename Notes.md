# Notes for implementing a CFD Solver

## Chapter 1: Setting up the Environment

## Chapter 2: Structure of a CFD Solver

### CFD Solver Framework
* Every CFD solver can be broken down into three sections:
1. Pre-processing: We prepare the simulation and all the steps here are performed once, at the beginning.
2. Solving: This is where we solve the equations, thus it is naturally an iterative step where we keep solving until we reach a converged solution.
3. Post-processing: Here we make sense of the results (colorful plots and graphs) and this is also done only once.

![alt text](image.png)

In the image above, we can see the steps involved in each section. 

1. Pre-processing (only once)
* Read params: Every simulation has some solver settings. These can come from a text file or some GUI.
* Allocate memory: We allocate memory for the arrays involved in the solving. These arrays can store the pressure/velocity/temperature etc.
* Initialise solution: Give a starting value
* Create/read mesh: Lastly, we create or read a mesh on which the equations will be solved.

2. Solving (iterative)
* Preparing solution update: Before we continue with current timestep, we need to calculate a few things like store previous values to calculate residuals or calculate a stable timestep in transient flows.
* Solve equations: We solve the governing equations along with additional phenomenons (turbulence, compressibility, etc.). We also keep track of the residuals which are used as a stopping criteria.
* Update boundary conditions: Once the solution updates, we need to update the BCs as well to reflect the changes
* Custom post-processing: If we want to keep track of certain quantities of interests, and to understand their evolution across iterations, we perform these calculations at the end of each iteration.
* Check convergence: Once the residuals are below a certain threshold, we stop the simulation and say that the results have converged.

3. Post-processing (only once)
* Write out simulation data: This output could be a simple .csv file or a format that is readable by ParaView etc.
* Deallocate memory: Languages lacking automatic garbage collection, we should de-allocate and free the memory after use to prevent memory leaking.

### The Governing Equations
We are going to work with Euler equations, which is a special inviscid case of Navier-Stokes (NS) equations. Euler and NS both retain the non-linear/convective term which is responsible for creating shocks and turbulence. High speed flows (aerospace applications) are generally inviscid with only a small viscous region.

We start with the compressible version of momentum equation of the NS equations.

$$
\frac{\partial (\rho \mathbf{u})}{\partial t}
+ \nabla \cdot (\rho \mathbf{u} \otimes \mathbf{u})
=
-\nabla p
+
\nabla \cdot
\left[
\mu
\left(
\nabla \mathbf{u}
+
(\nabla \mathbf{u})^T
-
\frac{2}{3}(\nabla \cdot \mathbf{u})\mathbf{I}
\right)
\right]
+
\nabla\left[\xi(\nabla \cdot \mathbf{u})\right]
$$

* $\frac{\partial (\rho \mathbf{u})}{\partial t}$ : This is the time-dependence of NS, which allows phenomenons to develop in time.
* $\nabla \cdot (\rho \mathbf{u} \otimes \mathbf{u})$ : This is the non-linear, convective term, which is why NS equations are so difficult to solve (OpenAI gave a paticular solution for the Clay Institute's Millenium problem). This pesky term produces shock-waves, generates and sustains turbulence.
* $ \nabla p$ : This is the pressure gradient and it causes flow between regions with differing pressures.
* $\nabla \cdot \left[\mu\left(\nabla \mathbf{u}+(\nabla \mathbf{u})^T-\frac{2}{3}(\nabla \cdot \mathbf{u})\mathbf{I}\right)\right]$ : This governs the diffusive and mixing processes.
* $\nabla\left[\xi(\nabla \cdot \mathbf{u})\right]$ : This is the second viscosity parameter $\xi$ and enhances mixing. For monoatomic gases $\xi=0$ and for non-monoatomic gases $\xi \neq 0$.

So, if we want to move from NS to Euler, we have to make the flow inviscid, which means, all viscosity terms are set to $0$, thus, $\rho = 0$ and $\xi = 0$. This simplifies the NS equations and we arrive at the Euler equations:

$$
\frac{\partial (\rho \mathbf{u})}{\partial t}
+ \nabla \cdot (\rho \mathbf{u} \otimes \mathbf{u})
=
-\nabla p
$$

Bringing the pressure gradient into the dot product, we get the final form of Euler equations:

$$
\frac{\partial (\rho \mathbf{u})}{\partial t}
+ \nabla \cdot (\rho \mathbf{u} \otimes \mathbf{u} - p \mathbf{I}) = 0
$$



