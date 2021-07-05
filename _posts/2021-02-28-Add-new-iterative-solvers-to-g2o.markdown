---
layout: post
mathjax: true
comments: true
title:  "Add new iterative solvers to g2o"
author: John Z. Li
date:   2021-02-27 19:00:18 +0800
categories: robotics
tags: graph-optimization
---
The [g2o library](https://github.com/RainerKuemmerle/g2o)
is essentially a solver for nonlinear least square problems that
have equivalent representations in the form of hyper-graphs, where each vertex
is a state variable or a configuration parameter to be optimized on, and each edge
represents a constraints imposed on two vertexes it connects.

Generally, g2o uses  the Levenberg-Marquardt (LM) (or Gauss-Newton)
method to solve the nonlinear least
square problem.
At each iteration, the LM method solves a linear system to determine
Δx, the update of parameter x.
The coefficient matrix of the linear system
is sparse block-wise.
The diagonal blocks of it corresponds vertexes of the hyper-graph,
while a non-zero off-diagonal block corresponds to an edge of the hper-graph.

By default, g2o provides two types of  linear solvers, that is,  director solvers based on
sparse Cholesky factorization, from CSparse package or from the CHOLMOD package or from the Eigen package,
and an indirect solver that uses the Preconditioned Conjugate Gradient method.
(Of course, one
can use a direct solver based on Cholesky factorization for dense matrices.
Actually, g2o
provides such a solver based on dense Cholesky factorization from Eigen. Since
the performance of such a solver will be generally much slower, we don't consider it here.)
The actual implementation of the PCG method is dependent on the choice of the preconditioner.
What g2o uses is a block-Jacobi preconditioner, which simply uses the block diagonal matrix
consisting of the block diagonal sub-matrices of the coefficient matrix
as the preconditioner.

## Performance: how to correctly interpret benchmark results
The g2o [paper](http://ais.informatik.uni-freiburg.de/publications/papers/kuemmerle11icra.pdf)
includes a table which compares the performance of different linear solvers on some
typical workloads, which is as below: (time in seconds)

| Dataset        | CHOLMOD    | CSparse   | PCG                |
|----------------| :--------: | :-------: | :----------------: |
|Intel           |0.0028      |**0.0025** |0.0064 ± 0.0026     |
|MIT             |0.0086      |**0.0077** |0.3810 ± 0.3640     |
|Manhattan3500   |0.018       |0.018      |**0.011 ± 0.0009**  |
|Victoria        |0.026       |**0.023**  |1.559 ± 0.683       |
|Grid5000        |**0.178**   |0.484      |1.996 ± 1.185       |
|Sphere          |0.055       |0.398      |**0.022 ± 0.019**   |
|Garage          |0.019       |0.032      |**0.017 ± 0.016**   |
|New College     |6.19        |200.6      |**0.778 ± 0.201**   |
|Venice          |1.86        |39.1       |**0.287 ± 0.135**   |
|Scale Drift     |0.0034      |**0.0032** |0.005 ± 0.01        |

To evaluate the comparative performance of each linear solver, we measure
the performance of each solver with respect to the fastest speed (choosing the
average speed for the PCG solver). That is, using the fasted method for each dataset
as baseline. For example, for the Intel dataset, The maximal
speed is achieved by CSpasrse with 0.0025s, the relative score for CHOLMOD is thus
0.0028 ÷ 0.0025 = 1.12, and the relative score for PCG is 0.0064 ÷ 0.0025 = 2.56.
The means that CHOLMOD is 1.12 times slower than CSparse, and PCG is 2.56 times slower
than it in computing a specific problem.
We translate the above benchmark result to the following:

| Dataset        | CHOLMOD    | CSparse   | PCG                |
|----------------| :--------: | :-------: | :----------------: |
|Intel           |1.12        |     **1** |2.56                |
|MIT             |1.12        |**1**      |49.48               |
|Manhattan3500   |1.64        |1.64       |**1**               |
|Victoria        |1.13        |**1**      |67.78               |
|Grid5000        |**1**       |2.72       |11.21               |
|Sphere          |2.5         |1.81       |**1**               |
|Garage          |1.12        |1.88       |**1**               |
|New College     |7.96        |257.07     |**1**               |
|Venice          |6.48        |136.24     |**1**               |
|Scale Drift     |1.06        |**1**      |1.56                |

According to *Smith, James E. (1988). "Characterizing computer performance with
a single number". Communications of the ACM. 31 (10): 1202–1206.*, we could use
the geometric mean of each column to compare the average performance of each solver.
We have the following:

| Dataset        | CHOLMOD    | CSparse   | PCG                |
|----------------| :--------: | :-------: | :----------------: |
|Performance     |**1.80**    |  3.74     |3.29                |
|Normalized Performance|**1** | 2.07      |1.83                |

The conclusion is: performance-wise, CHOLMOD ≻ PCG ≻ CSparse.
There are several things worth to notice:

- CSparse is actually the slowest method among the three. Especially, if one looks
at the "New College" dataset and the "Venice" dataset, CSparse is slower by two orders
of magnitude. This means that the performance of CSparse is very hard to predict.
If you want a method to be used in a realtime context, CSparse should not be chosen.
- The g2o library only implements one flavor of PCG. It is not uncommon that in some cases,
by using a different preconditioner, such as one from incomplete Cholesky decomposition,
the PCG method can be an order of magnitude faster.
- Most importantly, **the sparse structure with a given optimization problem in g2o is
fixed during each iteration of the LM algorithm**. This means, for a given optimization
problem, we can find the optimal strategy to construct the preconditioner for iterative solvers.
Ffor example,
by finding an ordering of the vertexes of the hyper-graph that results in least fill-ins
in incomplete Cholesky decomposition.
Block Jacobi preconditioner is a one-size-fit-all
preconditioner.
With more understanding of the structure imposed by the problem, we should
be able to find preconditioner-constructing methods for a given problem that leads to performance
comparable, or faster than, direct methods.
- When using direct solvers to solve the linear system at each step of the LM method,
it is impossible to know a priori how many fill-ins will be produced during the execution of
the sparse solver. Fill-ins in sparse direct methods are not only about performance,
they are about memory usage too. Since one can not realistically estimate the maximal
memory needed in solving a sparse linear system, the method can not be used on memory constrained
platforms reliably.
But with the PCG approach, we can restrict the maximal allowable fill-ins beforehand,
thus the whole algorithm can be implemented on memory constrained platforms
(see,C-J. Lin and J. J. Moré, Incomplete Cholesky Factorizations with Limited memory, SIAM J. Sci. Comput. 21(1), pp. 24-45, 1999
).
- Another thing to notice is that there exist modified versions of the LM method called inexact or incomplete
LM methods, in which  at each iterative step, only an approximate solution of the linear system is required.
As the algorithm proceeds, the threshold that determines the precision of the solution is decreased, thus
the algorithm eventually converges to a LM method.
Under some conditions, for example, that the least square
problem to be solved is a small residual one, it can be proved that a inexact LM method can be superlinear
or even quadratic convergence.
Numerical experiments show that inexact LM algorithms on typical problems
greatly reduces the total number of CG iterations in the LM method
(S. J. Wright and J. N. Holt, An inexact
Levenberg-Marquardt method for nonlinear programming, J. Austral. Math. Soc. Ser. B 26 (1985), 387-403.)

- After getting the preconditioner, the execution of the PCG method mainly consists of matrix vector multiplication.
Matrix vector multiplication is known to be able to be accelerated in multiple ways, for example by using SIMD of the CPU,
using OpenMP, or even GPUs. For example,
Intel provides an efficient implementation of PCG in its MKL library,
see [this from Intel](http://www.hocomputer.de/IntelWeb/mkl_indepth.pdf).


## How to add  customized linear solvers to g2o.
### Preparation: source code reading
**The solver class**

The abstract class "solver" defined in "g2o/core/solver.h" is responsible to construct
the linear system form the Jocobian matrix and the error vector.

The solver class is orthogonal to the `SparseOptimizer` class,
which is forward declared in the "solver.h" header.
The solver class is two-step initialized,
that is, after calling its default constructor, one has to call the "init"
function on a Solver object, which passes an instance of
`SparseOptimizer` to the `Solver` instance.
The Solver class has a public virtual destructor and is movable-only.

The `Solver` has a protected variable to control
whether the solver object is called from a LM method ( which is defaulted to false)
and a corresponding getter and setter.
The user is supposed to specify whether the optimization method is LM method by using the setter.
The `Solver` class holds a pointer to the array of `x`,
the solution of the linear system, and `b`, the
right-hand side of the linear system.
**Notice that the solver will free the space occupied by these two
arrays in its destructor.
But no allocation happens when a Solver object is initialized, instead, there is
a protected "resizeVector" to allocate memory for the two.
If the memory is already allocated, and the asked
space is larger, new memory is allocated and data is copied to it before the old buffer is deleted.**

There is also a switch member variable and the corresponding setter to enable or disable
performing a Shur complement before solving the linear system. Another switch member
variable and the corresponding setter to enable/disable debug information output.

There are four member methods that need to be paid attention to:
1. `virtual bool buildStructure(bool zeroBlocks = false) = 0;` to build the hyper graph from an optimizer instance.
2. `virtual bool buildSystem()` to build the linear system to solve.
3. `virtual bool solve() = 0;` to actually solve the linear system.
4. `virtual bool setLambda(number_t lambda, bool backup = false) = 0` to update the lambda term in the
LM method, where lambda
can be negative to undo a previous positive update.

One could implement his own solver by providing an implementation class to this abstract class.
When the resulted linear system can be conveniently represented by a sparse block format, the g2o already
comes with an implementation as a class template.

**Sparse block matrices in g2o: the `SparseBlockMatrix` class**

This data type is defined in "g2o/core/sparse_block_matrix.{h, hpp}". A sparse block
matrix is represented as the below:

1. A protected "_hasStorage" member variable to indicate whether the sparse matrix
instance owns the underlying data buffers. Its value can be passed as a constructor
parameter.

2. A `std::vector<int> _rowBlockIndices;` and a `std::vector<int> _colBlockIndices;`.
These two vectors specifies how many rows and columns an element of the block matrix (that is, the submatrix that
consists of the block). The elements of each vector is increasing in values with index increasing with its last element
being the row number and collumn number, and the gap between two consecutive elements is the row/column size of the corresponding
block element. For example, with `A ={A₁₁, A₁₂; A₂₁, A₂₂}`, and A₁₁ is of 2-by-3 block, and A₂₂ is a 3-by-4 block, then
`_rowBlockIndices = {2, 5};` and `_colBlockIndices = {3, 7}`;

3. A `std::vector <IntBlockMap> _blockCols;` , where `IntBlockmap` is a `std::map` from
`int`s to sparse matrix blocks. When access the (i, j)-th block of a sparse block matrix,
first we access the j-th index of the vector (recall that it is a map), followed by accessing the
i-th element of of the map.

To access the (i, j)-th block element of a Sparse matrix, g2o provides two member methods.
The `block(int row, int col)` function which will return a pointer to the underlying matrix.
**Note: if the block method is called upon a non-const instance of a sparse block matrix,
and a third parameter is passed to member function block(), a new block might be allocated if
it does not already exist. This behavior will not be enabled if "_hasStorage" is false.**

Note that the size of a sparse block matrix could be zero.

**Sparse block matrices in g2o: the ` SparseBlockMatrixCCS` class**

The "CCS" in the name stands for "Column Compressed Storage".
A sparse block matrix of type `SparseBlockMatrixCCS` is represented by a vector of vectors of so-called
`RowBlock`, which is basically a data structure that wraps a pointer to a block and its block-wise row number.
The size of the outer vector is the block-wise total number of columns, and each inner vector
contains all the block elements of a generic column.

A `SparseBlockMatrixCCS` object is not supposed to own the blocks it points to via pointers.
It is supposed to be used in a "read-only" fashion. This format of sparse block matrix is
quick to access, but very slow to add or remove elements from.

Comment: This class should own its underlying buffers. Its copy and move constructors should
do the right thing. A corresponding `SparseBlockMatrixCCS_view` should be defined to explicitly
express the programmer's intention.

**The `SparseBlockMatrixHashMap` class**

As its name suggests, an object of type `SparseBlockMatrixHashMap` stores a sparse block matrix
in a vector of hash maps (unordered map in C++ terminology). Matrices of this type is supposed
to be used when they need to be dynamically updated, while it is not suitable to iterated over.

** The `BlockSolver` class**

The `BlockSolver` class inherits from the `Solver` class.
(Comment: in g2o, there is a class `BlockSolverBase` that inherit from the `Solver` class,
and the `BlockSolver` class is then inherit from the `BlockSolverBase` class. This seems
unnecessary to me.)

The `BlockSolver` class is a template class that implements the common functionalities needed any solver that
assumes that the Hessian matrix is a sparse block matrix.  The `BlockSolver` class
is responsible for
constructing the optimizable graph and constructing the linear system (and also optionally performing Shur
completion). This means that anyone who wants to provide a customized linear solver only needs to
override the `solve` function of the class.

The `BlockSolver` class template is instantiated against two int parameters,  the dimension of the pose representation
and the dimension of the landmark representation.  The values of the two are saved in two const static data members.
(Comment: this class could be a normal class, not a class template. In the same
optimization problem, only one instance of the class is created. This one instance of
class has a fixed sparsity pattern, including dimension of its blocks. The implementation of this class seems not having used
the information of the pose dimension. On other words, `BlockSolverX` is sufficient to use.)

If one calls `buildStructure` again on a `BlockSolver` object, it will call a protected member method `resize`
to throw away old data and allocate memory again. This `resize` member method calls another protected member
method `deallocate` to throw away the internal state.

This class holds a unique pointer to an instance of `LinearSolver`, which is the linear solver that is got
called in the `solve` member function.

It is worth to mention that g2o implements performance measurement by inserting calls before and
after delegating the work to an instance of the `LinearSolver` class.

### Some Mathematics
**The sparse marginal covariance  matrix**

In class `MarginalCovarianceCholesky`, the so-called marginal covariance matrix is computed if we already
know the Cholesky factor of the original matrix with permutation.
This matrix is the inverse matrix of the coefficient matrix.
it can be interpreted as  covariance matrix of the system scaled by Jacobians.
But inverse the sparse information matrix of the system.
in general leads to a dense matrix. Instead of doing this, the algorithm tries to only compute the required elements instead.
An example use case is to compute the covariance (uncertainty) of a landmark given all observations of the landmark along with odometry of the robot.
Hence, the cholesky solvers include the class to provide methods to obtain the covariance of vertices given the Cholesky factor.

Given the matrix
{% raw %}
$$
\begin{pmatrix}
1.0 & 0 & 0 & 2.0\\
0   & 0 & 0 & 3.0\\
4.0 & 0 & 5.0 & 0\\
0   & 6.0 & 0 & 0
\end{pmatrix}
$$
{% endraw %}
Using g2o's notation, a sparse matrix is represented with the following notation:
1. `_Ax` is  a pointer to an array of floating numbers [1.0, 4.0, 6.0, 5.0, 2.0, 3.0].
2. `_Ai` is a pointer to an array of integers that represents the row indices of each element (starting from 0) [0, 2, 3, 2, 0, 1].
We can see that `_Ai` is of the same size as `_Ax`.
3. `_Ap` is a pointer to an array of integers, each element of which stands for the starting index of the corresponding column, that is
[0, 2, 3, 4, 6] for this example.  Notice that the size of `_Ap` is larger than the number of columns of the matrix by one. That is `_Ai[_Ap[6]]`
pointing to illegal memory address.

In class `MarginalCovarianceCholesky`, only the upper triangle part of the marginal covariance matrix is computed,
while the Cholesky factor of the covariance matrix  is stored in the CCS format with only the lower triangular part  as explained above.
1. member variable `_n` means the matrix is a n-by-n matrix.
2. member variable `_map` is a hashtable from `int` to a floating number, the (i,j)-th element
of the matrix can be retrieved using the key `i *_n + j`. It is used to store the elements of the original matrix.
3. member variable `_perm` is a pointer to an array of integers, the array represents a permutation of the original matrix.
Permutation matrices have many equivalent formations. In g2o, it is arranged such that with `A=LU`
the (i, j)-th element of the original matrix can be computed by multiplying `_perm[i]`-th row of `L` by `_perm[j]`-th column of `U`.
For example [4, 2, 3, 1] means the pivot the 1st row of the matrix with the 4th row, followed by pivoting the 1st column of the vector 4th column of the matrix.
But notice that vector indices in g2o starts with 0, not 1.
4. member variable `_diag` is a standard vector of floating point numbers, which stores the reciprocal of the diagonal elements of the original matrix.
5. The member method `computeIndex` returns `n * row + col` for (row, col)-th elements.
6. The member method `computeEntry` needs some explanation.

**Permutation**

If $P$ is a permutation matrix and $PAP^T = L L^T$ is the Cholesky factorization of the permuted matrix $P.$
If
{% raw %}
$$
P =
\begin{pmatrix}
e_{\pi(1)}\\
e_{\pi(2)}\\
\cdots\\
e_{\pi(n)}
\end{pmatrix}
$$
{% endraw %}
where $pi$$ is a mapping from $\{0, 1, \ldots, n-1\}$ to $\{\pi( 0 ), \pi( 1 ), \ldots, \pi( n-1 )\}.$
g2o actually stores the vector corresponding to $\pi^{-1}$, that is the inverse mapping of $\pi$.
For example, if
{% raw %}
$$
\pi= \{0, 1, 2, 3\} \to \{ 3, 2, 0, 1\}
$$
{% endraw %}
we have
{% raw %}
$$
\pi^{-1}= \{0, 1, 2, 3\} \to \{2, 3, 1, 0\}
$$
{% endraw %}
And `_Perm = {2, 3, 1, 0};`.

**Compute maginal covariance matrix recursively**

To use the `MarginalCovarianceCholesky` class, the user has to first explicitly call
member function `setCholeskyFactor` to setup a CCS representation of the lower triangle matrix of
the Cholesky factor of the original matrix. Member variable `_diag` is computed inside this function.

Suppose that the information matrix is $H$, it has a Cholesky factorization of $H = L D L^T,$
where $L$ is a lower triangular matrix with diagonal elements being ones, and $D$ is a diagonal matrix.
Notice that
{% raw %}
$$
H = LDL^T\\
H^{-1} = (L^{-1})^TD^{-1}L^{-1}\\
L^TH^{-1}= D^{-1}L^{-1}
$$
{% endraw %}
Let
{% raw %} $H^{-1} = U^T + D_u + U$ {% endraw%}
where U is the strict upper triangular part of $H^{-1},$ and $D_u$ is its diagonal part.
Notice the righthand side of the above equation is a lower triangle matrix,  we have
{% raw %}
$$
L^T (U^T + D_u) = D^{-1} L^{-1} \\
L^T U = {0}
$$
{% endraw %}
From the first equation, we can compute the diagonal elements of $H^{-1}$.
From the second equation, we can compute the off-diagonal elements of $H^{-1}$.

The recursive function has the potential to blow up the stack while calculating the marginal covariance matrix.
Can be change this without affecting the performance of g2o too much?


#### The `LinearSolver` class

The `LinearSolver` class itself is an abstract class, This class is supposed to be
implemented by another class.

- The pure virtual member function `init` is used to setup the sparse block structure of the matrix "A".
- The pure virtual `solve` member function is used to actually solve a linear system.
- The pure virtual `solveBlocks` member function is used to get the inversion of diagonal blocks.
- The pure virtual `solvePattern` member function is used to compute the marginal covariance matrix of a sparse matrix.
- The member function `setWriteDebug` and `writeDebug` are setter and getter function to enalbe/disable dumping the coefficient matrix
to a disk file if the linear system is not PSD while computing. The `LinearSolver` class itself does not implement write debug functionality.
Any concrete class that implement `LinearSolver` directly or indirectly should treat this as an implementation requirement.
- The member function `allocateBlocks` and `deallocateBlocks` allocate and deallocate memory for a block diagonal matrix that
has compatible structure with the coefficient matrix.
- The static member function `blockToScalarPermutation` is used to transform a block permutation matrix to a scalar permutation matrix.

#### The `LinearSolverCCS` class that inherits from `LinearSolver`

The class has a protected instance variable `_blockOrdering` of type `bool`, which
determines whether the sparse block matrix is ordered by the AMD (short for Approximate
Minimum Degree) algorithm. (see P.  R.  Amestoy,  T.  A.  Davis,  and  I.  S.  Duff,An  approximate  minimum  degree  ordering algorithm, SIAM J. Matrix Anal. Appl., 17 (1996), pp. 886–905, https://doi.org/10.1137/S0895479894278952.)
And there are corresponding getter and setter for the property.

An instance of the class holds a pointer to an instance of `SparseBlockMatrix`,
This matrix is deleted when the destructor of `LinearSolverCCS` is called.

The class has a protected member function that allocates memory for the owned `SparseBlockMatrixCCS`
pointed by the owing pointer. If the pointer is not null, `delete` is called first.

This class implements the `solveBlock` and `solvePattern` methods of the `LinearSolver` class.
But inside these two functions, the actual computation is delegated to a protected virtual function
called `solveBlocks_impl` which does not have a default implementation, aka, it is pure virtual.
The latter takes a `std::function` of type `MarginalCovarianceCholesky -> void`. But there is no
way the user can pass an instance of the type `MarginalCovarianceCholesky` to the public interface
of this class. It is also not clear why an instance `MarginalCovarianceCholesky` is needed.
(**Is this an error the author of g2o made?**)


#### The linear solvers provided by g2o
g2o implements 5 linear solvers.
All of them locate in the subdirectory of `g2o/solvers`. There are 4 direct solvers:
1. The `cholmod` solver, which calls the `cholmod` library to do the actual work.
2. The `csparse` solver, which calls the 'CSparse' library that is part of "SuitSparse" to do the actual job.
3. The `dense` solver, which uses the Cholesky solver for dense matrices in Eigen to do the actual work.
4. The `eigen` solver, which uses the Cholesky solver for sparse matrices in Eigen to do the actual work.

All the 4 direct solvers copies the sparse block matrix to a corresponding non-block form, and
calls the solver on the copied data. **The block-ness of the sparse block matrix is actually not used.**
Since the dense solver is too slow, practically there are only 3 solvers.

Note that all the direct solvers, except the dense one, is implemented by inheriting the
`LinearSolverCCS` class template.
The dense solver inherits the `LinearSolver` class template.

g2o implements one indirect solver, which uses Conjugate Gradient algorithm with block Jacobi preconditioners.
The `LinearSolverPCG` class inherits from the `LinearSolver` class.

### The implementation of the `LinearSolverPCG` class
The class has a virtual destructor, though the destructor has an empty body.
An instance of this class does not own the underlying coefficient matrix, but it
maintains some meta data about the matrix for quicker access. This metadata is cleared
when the `init` member function is called.

1. The `_diag` member variable (protected) stores a `std::vector` of pointers to
the diagonal blocks of the coefficient matrix.
2. The `_J` member variable (protected) stores a `std::vector` of the inverse matrices
of the diagonal blocks of the coefficient matrix.
3. The `_indices` member variable (protected) is a `std::vector` of `std::pair`s,
which stores the base indices (row-and-column) of **non-diagonal** blocks
4. The `_sparseMat` member variable (protected) is a `std::vector` of constant pointers
to the block submatrices. **Notice that only the upper triangular part of blocks are stored**
since the matrix is symmetric.

There are three protected variables to configure the PCG algorithm:
1. The `_tolerance` member variable to determine when to terminate the iteration if
a solution is deemed good enough.
2. The `_residual` member variable which is the norm of $Ax-b$.
3. The `_maxIter` member variable which is the maximally allowable iteration steps.

There are two switch variable of type Boolean to adjust the behavior of the PCG method:
1. `_absoluteTolerance` (defaults to true)
and corresponding getter and setter to whether to compute the residual in absolute sense.
2. `_verbose` (defaults to false) and corresponding getter and setter to enable/disable
the debug information output.


#### Helper functions of the `LinearSolverPCG` class
1. The function `pcg_axpy`.
Given a matrix `A`, two vectors `x` and `y`, two offset `xoff` for `x` and `yoff` for `y`.
This function multiplies `A` to vector `x` restricted on indices from `xoff` up to the
number of columns of `A`,
and add the result to `y` restricted on indices from `yoff` to the number of rows of `A`.
If the dimension of `A, x, y` are compatible, it reduces to `y= Ax +y`.
2. The function `pcg_atxpy`. It is similar to the above function, but `A` is replaced by the
transpose of `A`. If the matrix and vectors have compatible dimension, it reduces to $y = A^Tx +y.$
3. There are two overloading member function, both of which are templated against `MatrixType`,
to multiply a diagonal matrix with a vector. One of them is supposed to multiply the diagonal blocks
of the coefficient matrix to a vector, and the other is used to multiply the inverse diagonal blocks
with a vector. Both are named `multDiag`.
4. The member function `mult` multiplies the coefficient matrix with a vector.


## The PCG algorithm.

### The CG algorithm.
Given a linear system $Ax=b$, The CG method computes $x$ by
1. randomly choosing an initial guess of $x_0$.
2. set the first conjugate vector as $p_0 = r_0 = b - Ax_0$.
3. for $k= 0, 1, \ldots,$ iterate until residual sufficiently small or the maximal iteration is reached:
{% raw %}
$$
\alpha_k = \frac{r_k^Tr_k}{p_k^TA P_k}\\
x_{k+1} = x_k + \alpha_k p_k\\
r_{k+1} = r_k - \alpha_k A p_k\\
p_{k+1} = r_{k+1} + \frac{r_{k+1}^T r_{k+1}}{r_k^T r_k} p_k
$$
{% endraw %}
Notice that $Ap_k$ only needs to be computed once per each iteration.

### The PCG method
With a given preconditioner $M$ which is also symmetric positive definite, instead
of solving the original linear system, we solve the following:
{% raw %}
$$
C^{-1}A C^{-T} (C^Tx) = C^{-1}b
$$
{% endraw %}
with $M = CC^T.$

We have the following PCG method:
1. randomly choosing an initial guess of $x_0$.
2. set $r_0 = b - Ax_0$, $p_0= z_0 = M^{-1} r_0$.
3. for $k = 0, 1, \ldots$, iterate until residual is sufficiently small or the maximal iteration is reached:
{% raw %}
$$
\alpha_k = \frac{r_k^T z_k}{p_k^TAp_k}\\
x_{k+1} = x_k + \alpha_k p_k\\
r_{k+1} = r_k - \alpha_k A p_k\\
z_{k+1} = M^{-1} r_{k+1}\\
p_{k+1} = z_{k+1} + \frac{r_{k+1}^Tz_{k+1}}{r_k^Tz_k}p_k
$$
{% endraw %}
Notice that one does not need to compute the inversion of $M$ explicitly, only
that $M^{-1}r$ is required.

One way to achieve better numerical stability is to use the so-called Polak–Ribière
formula to update $p_{k+1},$ which is defined as the following:
{% raw %}
$$
p_{k+1} = z_{k+1} + \frac{r_{k+1}^T(z_{k+1} - z_k)}{r_k^Tz_k}p_k
$$
{% endraw %}
The two formulas are equivalent in exact arithmetic, but they have different characteristic of
numerical stability. The Polak–Ribière
formula needs to store another vector, but is less sensitive to round-off errors.

## Example: a PCG method for g2o with SSOR preconditioner.
SSOR stands for [Symmetric Successive Over-Relaxation](https://en.wikipedia.org/wiki/Symmetric_successive_over-relaxation) .
### The class definition in header "linear_solver_pcg_ssor.h"

```cpp
#ifndef G2O_LINEAR_SOLVER_PCG_SSOR_H
#define G2O_LINEAR_SOLVER_PCG_SSOR_H

#include <g2o/config.h>

#include <Eigen/Core>
#include <utility>
#include <vector>

#include "Eigen/src/Core/GlobalFunctions.h"
#include "Eigen/src/Core/TriangularMatrix.h"
#include "g2o/core/batch_stats.h"
#include "g2o/core/eigen_types.h"
#include "g2o/core/linear_solver.h"
// According to "Preconditioning for modal discontinuous Galerkin methods for unsteady
// 3D Navier-Stokes equations" by Philipp Birken etc., the performance of SOR preconditioners
// is comparable to ILU(0) preconditioners.
// This implies that carefully tuned PCG solver with SSOR should has performance comparable to
// Incomplete Cholesky with zero fill-ins.
//
// the optimal relaxation factor for a given g2o problem can be determined by experiments.
// Intuitively, the relaxation factor means a weighting factor when information is propagated
// on the hyper graph. This has a clear physical interpretation in many robotics applications.
// This needs to be investigated further.
//
namespace g2o
{
/**
   * \brief linear solver using PCG, pre-conditioner is block Jacobi
   */
template <typename MatrixType>
class LinearSolverPCGSSOR : public LinearSolver<MatrixType>
{
public:
  LinearSolverPCGSSOR() : LinearSolver<MatrixType>()
  {
    _tolerance = cst(1e-6);
    _verbose = false;
    _absoluteTolerance = true;
    _maxIter = -1;
  }

  virtual ~LinearSolverPCGSSOR() {}

  virtual bool init() override
  {
    _firstCall = true;
    return true;
  }

  virtual bool isIterativeMethod() override { return true;}

  virtual bool solve(const SparseBlockMatrix<MatrixType> & A, number_t * x, number_t * b) override;

  //! return the tolerance for terminating PCG before convergence
  number_t tolerance() const { return _tolerance; }
  virtual bool setTolerance(double tol) override {
        _tolerance = tol;
        return true;
  }

  int maxIterations() const { return _maxIter; }
  void setMaxIterations(int maxIter) { _maxIter = maxIter; }

  bool absoluteTolerance() const { return _absoluteTolerance; }
  void setAbsoluteTolerance(bool absoluteTolerance) { _absoluteTolerance = absoluteTolerance; }

  bool verbose() const { return _verbose; }
  void setVerbose(bool verbose) { _verbose = verbose; }

  number_t relaxValue() const { return _omega; }
  void setRelaxValue(number_t rv) {
          _omega = rv; }

protected:
  typedef std::vector<MatrixType, Eigen::aligned_allocator<MatrixType>> MatrixVector;
  typedef std::vector<const MatrixType *> MatrixPtrVector;
  typedef Eigen::TriangularView<MatrixType, Eigen::Lower> TrigView;

  number_t _tolerance;
  number_t _residual_factor  = 2;
  bool _absoluteTolerance;
  bool _verbose;
  int _maxIter;
  number_t _omega = 1;  // in (0, 2)

  MatrixPtrVector _diagBlocks;
  g2o::VectorX _diag;  // the (non-block) diagonal part of A, scaled by (2-ω)/ω.
  // The lower triangular matrix of diagonal blocks scaled by ω, ω * inv(L), where L is the lower
  // triagular part of D, a diagonal block
  MatrixVector _J;

  std::vector<std::pair<int, int>> _indices;           // base index of non-diagonal blocks.
  std::vector<std::pair<int, int>> _indices_row_wise;  // base index non-diagonal block, row-wise.
  // pointers to blocks in upper triangular part, column-wise, no diagonal blocks.
  MatrixPtrVector _sparseMat;
  // pointers to block in upper triangular part row-wise, no diagonal blocks.
  MatrixPtrVector _sparseMat_row_wise;
  // number of blocks each block column, only upper triagular part, including diag
  std::vector<int> _noOfBlocksEachBlockCol;
  // number of rows each block row, only upper triagular part, including diag
  std::vector<int> _noOfBlockEachBlockRow;

  void multDiag(
    const std::vector<int> & colBlockIndices, MatrixVector & A, const VectorX & src,
    VectorX & dest);
  void multDiag(
    const std::vector<int> & colBlockIndices, MatrixPtrVector & A, const VectorX & src,
    VectorX & dest);
  void mult(const std::vector<int> & colBlockIndices, const VectorX & src, VectorX & dest);
  // solve (1/ω D + L)x=b;
  void forward_solve(const std::vector<int> & colBlockIndices, const VectorX & b, VectorX & x);
  // solve ((1/ω) D + L^T)x = b;
  void backward_solve(const std::vector<int> & colBlockIndices, const VectorX & b, VectorX & x);
  // solve M x =b
  void solve_precond(const std::vector<int> & colBlockIndices, const VectorX & b, VectorX & x);

private:
  bool _firstCall = true;
  g2o::VectorX temp;   // used in forward and backward solve
  g2o::VectorX bdiff;  //used in forward and backward solve
  g2o::VectorX tt;     //used in precond solve
};

}  // namespace g2o

#include "linear_solver_pcg_ssor.hpp"

#endif
```

### The implementation of the method in "linear_solver_pcg_ssor.hpp"
```cpp
#ifndef G2O_LINEAR_SOLVER_PCG_SSOR_HPP
#define G2O_LINEAR_SOLVER_PCG_SSOR_HPP
#include <g2o/config.h>

#include <cstddef>
#include <iterator>
#include <utility>

#include "Eigen/src/Core/util/Constants.h"
#include "g2o/core/eigen_types.h"
#include "linear_solver_pcg_ssor.h"

namespace g2o
{
namespace pcg_ssor_internal
{
#ifdef _MSC_VER
// MSVC does not like the template specialization, seems like MSVC applies type conversion
// which results in calling a fixed size method (segment<int>) on the dynamically sized matrices
template <typename MatrixType>
void pcg_axy(const MatrixType & A, const VectorX & x, int xoff, VectorX & y, int yoff)
{
  y.segment(yoff, A.rows()) = A * x.segment(xoff, A.cols());
}
#else
template <typename MatrixType>
inline void pcg_axy(const MatrixType & A, const VectorX & x, int xoff, VectorX & y, int yoff)
{
  y.segment<MatrixType::RowsAtCompileTime>(yoff) =
    A * x.segment<MatrixType::ColsAtCompileTime>(xoff);
}

template <>
inline void pcg_axy(const MatrixX & A, const VectorX & x, int xoff, VectorX & y, int yoff)
{
  y.segment(yoff, A.rows()) = A * x.segment(xoff, A.cols());
}
#endif

template <typename MatrixType>
inline void pcg_axpy(const MatrixType & A, const VectorX & x, int xoff, VectorX & y, int yoff)
{
  y.segment<MatrixType::RowsAtCompileTime>(yoff) +=
    A * x.segment<MatrixType::ColsAtCompileTime>(xoff);
}

template <>
inline void pcg_axpy(const MatrixX & A, const VectorX & x, int xoff, VectorX & y, int yoff)
{
  y.segment(yoff, A.rows()) += A * x.segment(xoff, A.cols());
}

template <typename MatrixType>
inline void pcg_atxpy(const MatrixType & A, const VectorX & x, int xoff, VectorX & y, int yoff)
{
  y.segment<MatrixType::ColsAtCompileTime>(yoff) +=
    A.transpose() * x.segment<MatrixType::RowsAtCompileTime>(xoff);
}

template <>
inline void pcg_atxpy(const MatrixX & A, const VectorX & x, int xoff, VectorX & y, int yoff)
{
  y.segment(yoff, A.cols()) += A.transpose() * x.segment(xoff, A.rows());
}
}  // namespace internal
// helpers end

template <typename MatrixType>
bool LinearSolverPCGSSOR<MatrixType>::solve(
  const SparseBlockMatrix<MatrixType> & A, number_t * x, number_t * b)
{
  const int diag_block_no = int(A.blockCols().size());  // how many blocks on diagonal
  const int block_no = A.nonZeroBlocks();
  const int non_diag_block_no = block_no - diag_block_no;

  if (_firstCall) {
    _indices.clear();
    _sparseMat.clear();
    _indices_row_wise.clear();
    _sparseMat_row_wise.clear();
    _noOfBlockEachBlockRow.clear();
    _noOfBlocksEachBlockCol.clear();
    _J.reserve(diag_block_no);
    _diagBlocks.reserve(diag_block_no);
    _indices.reserve(non_diag_block_no);
    _indices_row_wise.reserve(non_diag_block_no);
    _sparseMat.reserve(non_diag_block_no);
    _sparseMat_row_wise.reserve(non_diag_block_no);
    _noOfBlockEachBlockRow.reserve(diag_block_no);
    _noOfBlocksEachBlockCol.reserve(diag_block_no);
    temp = g2o::VectorX(A.cols());
    bdiff = g2o::VectorX(A.cols());
    tt = g2o::VectorX(A.cols());
    _diag = g2o::VectorX(A.cols());
    _diag.setZero();
    temp.setZero();
    bdiff.setZero();
    tt.setZero();
    _noOfBlockEachBlockRow.assign(diag_block_no, 1);
    _noOfBlocksEachBlockCol.assign(diag_block_no, 0);
    _residual_factor = std::sqrt(A.cols());
  }

  _J.clear();
  _diagBlocks.clear();
  const number_t sw = (2 - _omega) / _omega;

  // put the block matrix once in a linear structure, makes mult faster
  int colIdx = 0;
  for (size_t i = 0; i < size_t(diag_block_no); ++i) {
    //get a ref to the i-th block column
    const typename SparseBlockMatrix<MatrixType>::IntBlockMap & col = A.blockCols()[i];
    if (_firstCall) {
      _noOfBlocksEachBlockCol[i] = col.size();  // assuming only upper triag part is stored.
    }
    typename SparseBlockMatrix<MatrixType>::IntBlockMap::const_iterator it;
    for (it = col.begin(); it != col.end(); ++it) {
      if (it->first == (int)i) {  // only the upper triangular block is needed
        _diagBlocks.push_back(it->second);
        _diag.segment(A.colBaseOfBlock(i), A.colsOfBlock(i)) = it->second->diagonal() * sw;
        _J.push_back(
          it->second->template triangularView<Eigen::Lower>().solve(
            MatrixType::Identity(A.colsOfBlock(i), A.colsOfBlock(i))) *
          (MatrixType::Identity(A.colsOfBlock(i), A.colsOfBlock(i)) * _omega));
        break;
      }
      if (_firstCall) {
        _indices.push_back(
          // store the base index of the current block
          // std::make_pair(it->first > 0 ? A.rowBlockIndices()[it->first - 1] : 0, colIdx));
          std::make_pair(A.rowBaseOfBlock(it->first), colIdx));
        _sparseMat.push_back(it->second);  //store pointers to non-diagonal blocks.
        _noOfBlockEachBlockRow[it->first] += 1;
      }
    }
    colIdx = A.colBlockIndices()[i];  //base column index of the next column
  }

  if (_firstCall) {
    _indices_row_wise = _indices;
    _sparseMat_row_wise = _sparseMat;
    std::sort(_indices_row_wise.begin(), _indices_row_wise.end());  //sort row wise.
    // #pragma omp parallel for
    for (size_t j = 0; j < _indices_row_wise.size(); ++j) {
      auto idx = std::find(_indices.begin(), _indices.end(), _indices_row_wise[j]);
      auto pos = std::distance(_indices.begin(), idx);
      _sparseMat_row_wise[j] = _sparseMat[pos];
    }
    _firstCall = false;
  }

  int n = A.rows();
  assert(n > 0 && "Hessian has 0 rows/cols");
  Eigen::Map<VectorX> xvec(x, A.cols());
  const Eigen::Map<VectorX> bvec(b, n);
  // xvec.setZero();

  VectorX r, d, q, s;  //q = Ad
  d.setZero(n);        //conjugate direction
  q.setZero(n);        //q= A * d
  s.setZero(n);        //s = inv(M) * r

  xvec = bvec.array() / _diag.array() * sw;
  mult(A.colBlockIndices(), xvec, tt);
  r = bvec - tt;  // r = b- A *x_0

  solve_precond(A.colBlockIndices(), r, s);
  d = s;  // first conjugate direction

  number_t dn = r.dot(s);
  number_t d0 = _tolerance * dn * _residual_factor;

  number_t b_ninf = bvec.lpNorm<Eigen::Infinity>();
  number_t d_ninf = _diag.lpNorm<Eigen::Infinity>();

  number_t stop_criteria =
    (b_ninf + d_ninf * xvec.lpNorm<Eigen::Infinity>()) * _residual_factor * _tolerance;

  number_t residual = r.lpNorm<Eigen::Infinity>();

  number_t dold = dn;
  number_t a = 0;
  number_t ba = 0;

  int maxIter = _maxIter < 0 ? A.rows() : _maxIter;

  int iteration;
  for (iteration = 0; iteration < maxIter; ++iteration) {
    // if (_verbose) std::cerr << "residual[" << iteration << "]: " << dn << std::endl;

    if (residual < stop_criteria && dn < d0 ) break;  // done

    mult(A.colBlockIndices(), d, q);  //q=A * d
    a = dn / d.dot(q);

    xvec += a * d;
    // TODO: reset residual here every 50 iterations
    r -= a * q;

    solve_precond(A.colBlockIndices(), r, s);

    dold = dn;
    dn = r.dot(s);
    ba = dn / dold;
    d = s + ba * d;

    stop_criteria =
      (b_ninf + d_ninf * xvec.lpNorm<Eigen::Infinity>()) * _residual_factor * _tolerance;
    residual = r.lpNorm<Eigen::Infinity>();
  }

  // std::cerr << "residual[" << iteration << "]: " << residual << std::endl;
  G2OBatchStatistics * globalStats = G2OBatchStatistics::globalStats();
  if (globalStats) {
    globalStats->iterationsLinearSolver = iteration;
  }

  return true;
}

template <typename MatrixType>
void LinearSolverPCGSSOR<MatrixType>::multDiag(
  const std::vector<int> & colBlockIndices, MatrixVector & A, const VectorX & src, VectorX & dest)
{
  int row = 0;
  // #pragma omp parallel for
  for (size_t i = 0; i < A.size(); ++i) {
    pcg_ssor_internal::pcg_axy(A[i], src, row, dest, row);
    row = colBlockIndices[i];
  }
}

template <typename MatrixType>
void LinearSolverPCGSSOR<MatrixType>::multDiag(
  const std::vector<int> & colBlockIndices, MatrixPtrVector & A, const VectorX & src,
  VectorX & dest)
{
  int row = 0;
  // #pragma omp parallel for
  for (size_t i = 0; i < A.size(); ++i) {
    pcg_ssor_internal::pcg_axy(*A[i], src, row, dest, row);
    row = colBlockIndices[i];
  }
}

template <typename MatrixType>
void LinearSolverPCGSSOR<MatrixType>::mult(
  const std::vector<int> & colBlockIndices, const VectorX & src, VectorX & dest)
{
  // first multiply with the diagonal
  multDiag(colBlockIndices, _diagBlocks, src, dest);

  // now multiply with the upper triangular block
  for (size_t i = 0; i < _sparseMat.size(); ++i) {
    const int & srcOffset = _indices[i].second;
    const int & destOffsetT = srcOffset;
    const int & destOffset = _indices[i].first;
    const int & srcOffsetT = destOffset;

    const typename SparseBlockMatrix<MatrixType>::SparseMatrixBlock * a = _sparseMat[i];
    // destVec += *a * srcVec (according to the sub-vector parts)
    pcg_ssor_internal::pcg_axpy(*a, src, srcOffset, dest, destOffset);
    // destVec += *a.transpose() * srcVec (according to the sub-vector parts)
    pcg_ssor_internal::pcg_atxpy(*a, src, srcOffsetT, dest, destOffsetT);
  }
}

// solve (1/ω D + L)x=b=Wx;
template <typename MatrixType>
void LinearSolverPCGSSOR<MatrixType>::forward_solve(
  const std::vector<int> & colBlockIndices, const VectorX & b, VectorX & x)
{
  size_t off_diag_idx = 0;
  int rows_of_block;
  int srcOffset;  //base idx of i-th diagonal block
  int destOffset;
  int srcOffSetT;
  int destOffsetT;

  temp.setZero();

  for (size_t i = 0; i < _diagBlocks.size(); ++i) {
    rows_of_block = _J[i].rows();
    srcOffset = colBlockIndices[i - 1];  //base idx of i-th diagonal block
    destOffset = srcOffset;

    //how many blocks on i-th row except the diagonal block
    int no_of_off_diag_col = _noOfBlocksEachBlockCol[i] - 1;
    for (int j = 0; j < no_of_off_diag_col; ++j) {
      const typename SparseBlockMatrix<MatrixType>::SparseMatrixBlock * a =
        _sparseMat[off_diag_idx + j];
      srcOffSetT = _indices[off_diag_idx + j].first;
      destOffsetT = _indices[off_diag_idx + j].second;

      pcg_ssor_internal::pcg_atxpy(*a, x, srcOffSetT, temp, destOffsetT);
    }

    bdiff.segment(destOffset, rows_of_block) =
      b.segment(destOffset, rows_of_block) - temp.segment(destOffset, rows_of_block);
    pcg_ssor_internal::pcg_axy(_J[i], bdiff, srcOffset, x, destOffset);

    off_diag_idx += no_of_off_diag_col;  //exclude diagonal part
  }
}

// solve ((1/ω) D + L^T)x = b;
template <typename MatrixType>
void LinearSolverPCGSSOR<MatrixType>::backward_solve(
  const std::vector<int> & colBlockIndices, const VectorX & b, VectorX & x)
{
  const int nn = int(_diagBlocks.size());  //number of diagonal blocks.
  size_t off_diag_idx = 0;                 //start from the second last block-row.

  int srcOffset;  //base idx of i-th diagonal block
  int destOffset;
  int rows_of_block;
  int mm;
  int jj;
  int srcOff;
  int destOff;

  temp.setZero();

  for (int i = nn - 1; i >= 0; --i) {
    srcOffset = i ? colBlockIndices[i - 1] : 0;  //base idx of i-th diagonal block
    destOffset = srcOffset;
    rows_of_block = _J[i].rows();

    MatrixType mtx;
    int no_of_off_diag_row = _noOfBlockEachBlockRow[i] - 1;
    //how many block on i-th block row except the diagonal blcok
    for (int j = 0; j < no_of_off_diag_row; ++j) {
      mm = int(_sparseMat_row_wise.size());
      jj = mm - (off_diag_idx + j + 1);
      const typename SparseBlockMatrix<MatrixType>::SparseMatrixBlock * a = _sparseMat_row_wise[jj];
      srcOff = _indices_row_wise[jj].second;
      destOff = _indices_row_wise[jj].first;
      pcg_ssor_internal::pcg_axpy(*a, x, srcOff, temp, destOff);
    }

    bdiff.segment(destOffset, rows_of_block) =
      b.segment(destOffset, rows_of_block) - temp.segment(destOffset, rows_of_block);
    mtx = _J[i].transpose();
    pcg_ssor_internal::pcg_axy(mtx, bdiff, srcOffset, x, destOffset);

    off_diag_idx += no_of_off_diag_row;  //exclude diagonal part
  }
}

// solve M x =b
template <typename MatrixType>
void LinearSolverPCGSSOR<MatrixType>::solve_precond(
  const std::vector<int> & colBlockIndices, const VectorX & b, VectorX & x)
{
  forward_solve(colBlockIndices, b, x);  // lower triag-part
  tt = x.array() * _diag.array();
  backward_solve(colBlockIndices, tt, x);  //upper triag-part
}

}  //end namespace g2o

#endif
```

### Some boilerplate to register the newly defined method in "solver_pcg_ssor.cpp"
```cpp
#include "g2o/core/block_solver.h"
#include "g2o/core/optimization_algorithm_factory.h"
#include "g2o/core/optimization_algorithm_gauss_newton.h"
#include "g2o/core/optimization_algorithm_levenberg.h"
#include "g2o/core/solver.h"
#include "g2o/stuff/macros.h"
#include "linear_solver_pcg_ssor.h"

using namespace std;

namespace g2o
{
namespace
{
template <int p, int l>
std::unique_ptr<g2o::Solver> AllocateSolver()
{
  std::cerr << "# Using PCGSSOR poseDim " << p << " landMarkDim " << l << std::endl;

  return g2o::make_unique<BlockSolverPL<p, l>>(
    g2o::make_unique<LinearSolverPCGSSOR<typename BlockSolverPL<p, l>::PoseMatrixType>>());
}
}  // namespace

static OptimizationAlgorithm * createSolver(const std::string & fullSolverName)
{
  static const std::map<std::string, std::function<std::unique_ptr<g2o::Solver>()>>
    solver_factories{
      {"pcg_ssor", &AllocateSolver<-1, -1>},
      {"pcg_ssor3_2", &AllocateSolver<3, 2>},
      {"pcg_ssor6_3", &AllocateSolver<6, 3>},
      {"pcg_ssor7_3", &AllocateSolver<7, 3>},
    };

  string solverName = fullSolverName.substr(3);
  auto solverf = solver_factories.find(solverName);
  if (solverf == solver_factories.end()) return nullptr;

  string methodName = fullSolverName.substr(0, 2);

  if (methodName == "gn") {
    return new OptimizationAlgorithmGaussNewton(solverf->second());
  } else if (methodName == "lm") {
    return new OptimizationAlgorithmLevenberg(solverf->second());
  }

  return nullptr;
}

class PCGSSORSolverCreator : public AbstractOptimizationAlgorithmCreator
{
public:
  explicit PCGSSORSolverCreator(const OptimizationAlgorithmProperty & p)
  : AbstractOptimizationAlgorithmCreator(p)
  {
  }
  virtual OptimizationAlgorithm * construct() { return createSolver(property().name); }
};

G2O_REGISTER_OPTIMIZATION_LIBRARY(pcg_ssor);

G2O_REGISTER_OPTIMIZATION_ALGORITHM(
  gn_pcg_ssor,
  new PCGSSORSolverCreator(OptimizationAlgorithmProperty(
    "gn_pcg_ssor", "Gauss-Newton: PCG solver using SSOR pre-conditioner (variable blocksize)",
    "PCGSSOR", false, Eigen::Dynamic, Eigen::Dynamic)));
G2O_REGISTER_OPTIMIZATION_ALGORITHM(
  gn_pcg_ssor3_2,
  new PCGSSORSolverCreator(OptimizationAlgorithmProperty(
    "gn_pcg_ssor3_2", "Gauss-Newton: PCG solver using SSOR pre-conditioner (fixed blocksize)",
    "PCGSSOR", true, 3, 2)));
G2O_REGISTER_OPTIMIZATION_ALGORITHM(
  gn_pcg_ssor6_3,
  new PCGSSORSolverCreator(OptimizationAlgorithmProperty(
    "gn_pcg_ssor6_3", "Gauss-Newton: PCG solver using SSOR pre-conditioner (fixed blocksize)",
    "PCGSSOR", true, 6, 3)));
G2O_REGISTER_OPTIMIZATION_ALGORITHM(
  gn_pcg_ssor7_3,
  new PCGSSORSolverCreator(OptimizationAlgorithmProperty(
    "gn_pcg_ssor7_3", "Gauss-Newton: PCG solver using SSOR pre-conditioner (fixed blocksize)",
    "PCGSSOR", true, 7, 3)));
G2O_REGISTER_OPTIMIZATION_ALGORITHM(
  lm_pcg_ssor,
  new PCGSSORSolverCreator(OptimizationAlgorithmProperty(
    "lm_pcg_ssor", "Levenberg: PCG solver using SSOR pre-conditioner (variable blocksize)", "PCGSSOR",
    false, Eigen::Dynamic, Eigen::Dynamic)));
G2O_REGISTER_OPTIMIZATION_ALGORITHM(
  lm_pcg_ssor3_2,
  new PCGSSORSolverCreator(OptimizationAlgorithmProperty(
    "lm_pcg_ssor3_2", "Levenberg: PCG solver using SSOR pre-conditioner (fixed blocksize)", "PCGSSOR",
    true, 3, 2)));
G2O_REGISTER_OPTIMIZATION_ALGORITHM(
  lm_pcg_ssor6_3,
  new PCGSSORSolverCreator(OptimizationAlgorithmProperty(
    "lm_pcg_ssor6_3", "Levenberg: PCG solver using SSOR pre-conditioner (fixed blocksize)", "PCGSSOR",
    true, 6, 3)));
G2O_REGISTER_OPTIMIZATION_ALGORITHM(
  lm_pcg_ssor7_3,
  new PCGSSORSolverCreator(OptimizationAlgorithmProperty(
    "lm_pcg_ssor7_3", "Levenberg: PCG solver using SSOR pre-conditioner (fixed blocksize)", "PCGSSOR",
    true, 7, 3)));
}  // namespace g2o
```
