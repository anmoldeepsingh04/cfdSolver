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

Assuming that the velocity vector $\mathbf{u} = [u,v,w]^T$, the non-linear term with the dyadic product (similar to outer product of two vectors) can be written as:

$$
\rho \mathbf{u} \otimes \mathbf{u} = \rho

\begin{pmatrix}
u \\ v \\ w
\end{pmatrix}

\begin{pmatrix}
u & v & w
\end{pmatrix}

=

\begin{pmatrix}
\rho u^2 & \rho uv & \rho uw \\
\rho uv & \rho v^2 & \rho vw \\
\rho uw & \rho vw & \rho w^2
\end{pmatrix}
$$
which is a second order tensor (or a 2D matrix). 

We have three equations for the three velocity components, but we have a total of five unknowns $\begin{pmatrix} u,v,w,p, \rho \end{pmatrix}$, this, we need two additional equations to solve this system.

The first equation comes from the mass conservation, which gives us the density, $ \rho $:

$$
\frac{\partial \rho}{\partial t} + \nabla \cdot (\rho \textbf u) = 0
$$

Now, we only need one more equation to solve for the pressure. But, we don't have a conservation law for pressure, so we use some thermodynamic relations to get the pressure.

We know that the total energy is the sum of kinetic and potential energy,

$$
\implies E = \text{Kinetic Energy} + \text{Potential Energy} = \rho e + \frac{1}{2}\rho \textbf{u}^2
$$

and the fact that the total energy of any isolated system is always conserved gives us an equation for pressure:

$$
\frac{\partial E}{\partial t} + \nabla \cdot (E + p) = 0
$$
This equation is valid only for an inviscid flow.

But in the process of writing this relation, we introduced another unknown, $ E $, for which, we use some thermodynamic relations to get another equation. Thus, we have $6$ equations and $6$ unknowns, which will close the system.