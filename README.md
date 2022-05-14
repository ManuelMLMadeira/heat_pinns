# Physics-Informed Neural Networks (PINNs) Implementation for the Heat Equation 
Solving the heat partial differential equation with Physics-Informed Neural Networks (PINNs).

## Information:

+ [Heat Equation (and a classical solution)](https://levelup.gitconnected.com/solving-2d-heat-equation-numerically-using-python-3334004aa01a);
+ [Physics-Informed Neural Networks original paper](https://arxiv.org/abs/1711.10561)

---
## Implementation:

The paper focuses on non-linear partial differential equations (PDE):

$$\frac{\partial u}{\partial t} + \mathcal{N}[u] = u_t + \mathcal{N}[u] = 0,$$ 
where the subscript denotes the partial derivative wrt to the variable that follows.

For the particular case of the Heat Equation: 

$$u_t - \alpha \nabla^2 \cdot u = u_t - \alpha (u_{x_1x_1} + u_{x_2x_2} + \ldots + u_{x_n x_n}).$$ 

Therefore, $u(t, x)$ denotes the latent/hidden solution that we want to find. This is the problem we have to solve. Moreover, for the Heat PDE, we have:

$$\mathcal{N}[u] = - \alpha \nabla^2 \cdot u.$$

The paper proposes two different methods of integrating Physics prior knowledge into a Neural Network learning task: **continuous time** and **discrete time** models. Each of the two directories contains the respective implementation of these models.


---
## Continuous Time PINN

Inside the 4 folders, you find continuous time PINN solutions using Pytorch. Each folder addresses a different problem (3 are $1$-dimensional - bar ones - and one $2$-dimensional - plate). Each problem was taken from a source link that you can find in the *"Results.....py"* file inside each folder. The suggested route is: 1) *bar_simple*; 2) *bar_temppeak*; 3) *bar_random_heat*; 4) *plate*.

### Approach

This method consists of sampling (*e.g.* via Latin Hypercube Sampling) both boundary/initial points (in this section simply called boundary points) and collocation points inside the domain considered and, then, impose a proper loss function for each of these two classes of points. Therefore, I started by building a training dataset:
* **Boundary points**: in these points, the targets considered were the conditions imposed by the boundary/initial conditions (in this section simply called boundary conditions). Therefore, for a boundary point, $(t_i, x_{1,i}, \ldots, x_{n,i})$, with target $y_i$, its contribution to the loss will merely be $MSE(u(t_i, x_{1,i}, \ldots, x_{n,i}), y_i)$, where $y_i$ is imposed by the boundary conditions.
* **Collocation points**: in these points, the target was considered to be $0$. This option is explained by the contribution of the collocation points to the loss: for a collocation point, $(t_i, x_{1,i}, \ldots, x_{n,i})$, with target $y_i$, we have $MSE(f(t_i, x_{1,i}, \ldots, x_{n,i}), yi) = MSE(f(t_i, x_{1,i}, \ldots, x_{n,i}), 0) = \| f(t_i, x_{1,i}, \ldots, x_{n,i}) \|^2$, as explained in the PINNs paper (summary: $f(t, x_{1}, \ldots, x_{n}) = u_t - \alpha \nabla^2 \cdot u = 0$. It has to hold, by definiton of the Heat PDE, on every collocation point).

After building this dataset for a given problem, we train a neural net to mimick the actual solution $u(x,t)$ by minimizing the loss in the generated dataset. Note that, therefore, $u(x,t)$ and $f(x, t)$ share the same parameters - the ones from the NN - which we'll optimize for that dataset.

### Implementation

Inside each folder you can find 4 different .py files:

* "PINN.py": 
  * the file where the class for the Physics Informed Neural Network is implemented (for arbitrary dimension $n$). It is the same for the 4 folders. It consists of the typical implementation of a NN using Pytorch, where we also add the methods necessary for the computation of $f(t,x)$.
* "..._dataset.py": 
  * the file where all the information about the problem setup is integrated, which results in the construction of the training (and test) dataset. Therefore, this file slightly varies across folders, since each folder consists of a different problem. 
  * **If your run this .py, you obtain a new dataset file**. 
  * Relevant technical details: The number of collocation and boundary points is defined by the user. Collocation points are generated using Latin Hypercube Sampling (inside the domain considered); Boundary points are generated by randomly (uniform distribution) picking one boundary and then randomly (uniform distribution) placing it on that boundary. The training dataset contains these two types of points with the respective target (explained in section "Approach"). We also build a test dataset in these files in the cases where a solution is available by other methods (*e.g.* closed-form solutions or standard numerical methods) to then compare them in the "Results...py" file. 
* "..._train.py": 
  * the file where the PINN is trained on the given dataset. 
  * **If your run this .py, you obtain a new initial_model and final_model1**, which are the model obtained by mere initialization - no training at all - and after training on the dataset generated by "..._dataset.py", respectively. You can also run the file without saving those models by converting the "save_model" variable to False.
* "Results.....py": 
  * in this Jupyter Notebook (easier to observe figures...), you can observe the results from the previous files. In particular: 
    * the groundtruth solution (again, obtained via closed-form solution, when readily available, or standard numerical methods); 
    * the (test and training) datasets generated; 
    * the solution obtained by the initial_model; 
    * the solution obtained by the final_model1.

Computationally extenuous trainings were not performed in any problem, but in all of them we can obtain fair results with very simple models in a low number of epochs.

---
## Discrete Time PINN

### Approach

The approach suggested by the PINN paper consists of taking advantage from Implicit Runge-Kutta (IRK) methods. In particular, from the initial points, *i.e.*, $u(t_{n}, x)$ we predict $u(t_n + \Delta t, x)$ using a Neural Network with Physics information integrated. Therefore, time is frozen at a given time step $t_n$ and we intend to train a NN to output $o_1(x) = [u^{n+c_1}(x), u^{n+c_2}(x), \ldots, u^{n+c_q}(x), u^{n+1}(x)]$, where $u^{n+c_1} = u(t_n + c_j \Delta t, x)$, $c_j$ are the coefficients from the $c$ array from Butcher tableau (a general representation for all the RK methods) and $q$ denotes the number of stages of the IRK method we picked. Note that the output of the NN only depends on $x$. Moreover, we add a Physics informed layer, that imposes the IRK method equations on the output of the NN. In particular, we have:

$$ o_2(x) = o_1(x) + \Delta t M \mathcal{N},$$

where $o_2(x)$ denotes the output of the physics informed layer; $M$ consists of the stacking matrix of $A$ and $b$ (again, coefficients from the Butcher tableau); and $\mathcal{N} = \left[\mathcal{N}[u^{n+c_1}(x)], \mathcal{N}[u^{n+c_2}(x)], \ldots, \mathcal{N}[u^{n+c_q}(x))] \right] = - \alpha [\nabla^2 \cdot u^{n+c_1}(x), \nabla^2 \cdot u^{n+c_2}(x), \ldots, \nabla^2 \cdot u^{n+c_q}(x)]$. An important point from this approach is that it is possible to generate $A$, $b$ and $c$ coefficients for an arbitrary number of stages of the IRK using Gauss-Legendre methods. Therefore, it is possible to confer to a discrete time PINN an arbitrary representation power (*i.e.*, number of stages) of the method at a much lower cost than in typical approaches.

Considering this approach, we again define two different losses for the two different point in our dataset:
* **Boundary points**: $MSE(o_1(x_b), \mathbf{0})$, where $\textbf{0}$ denotes a vector of zeros of the same dimension as $o_1(x)$. This target is imposed by the definition of boundary conditions used, where $u(t,x_b) = 0, \forall t$. 
* **Collocation points**: $MSE(o_2(x_c), \mathbf{u(t_n, x_c)})$, where $\mathbf{u(t_n, x_c)}$ denotes a vector repeatedly filled with $u(t_n, x_c)$ of the same dimension as $o_2(x)$. This target results naturally by the manipulation of RK methods equations (by definition, each entry of $o_2(x)$ is a prediction of $u(t_n, x)$).

After building this dataset for a given problem, we train a neural net to mimick the actual solution $u(x,t)$ by minimizing the loss in the generated dataset. Note that, therefore, $o_1(x)$ and $o_2(x)$ share the same parameters - the ones from the NN - which we'll optimize for the generated dataset.

### Disclaimer

Given the approach above, the implementation followed the same structure as the continuous time models. However, this implementation seems not to be correct (even though the training loss decreases over epochs of training). 

Possible reasons of failure: wrong implementation; excessively high $\Delta t$ (not likely); excessively simple NN; lack of training. Please feel free to contact me for further discussion :) 
