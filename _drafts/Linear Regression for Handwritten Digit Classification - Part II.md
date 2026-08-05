# Linear Regression for Handwritten Digit Classification - Part 2

Here we explore the use of linear algebra to solve the least squares problem.

#### Multilinear Least Squares

By representing $y$, the predictors, and the bias in a matrix, the equation can be represented like so:

```math
A\vec{x}=\vec{b}
\\\\
\left[
  \begin{array}{cc}
    1_1 & 0_2 ..&0_{785}  \\
    ...&...&...\\
    1_1 & 1_2 ..&1_{785}   \\ 
  \end{array}
\right]
\left[
\begin{array}{c}
x_1\\
...\\
x_{785}


\end{array}\right]
=
\left[
\begin{array}{c}
0_1\\
...\\
0_{60,000}

\end{array}\right]
```



We are looking for an $A\vec{x}$ (where $\vec{x}$ represents the predictors $\hat{\beta}_{1..n}$ and the bias $\hat{\beta}_0$) that is closest to $\vec{b}$, but still in the column space of matrix $A$. 

Similar to the single linear least squares approach, we are trying to minimize the residual error, in this case, vector $\vec{r}$, which is the difference between our predicted $\hat{b}$ and the actual $\vec{b}$:

```math
\vec{r} = \vec{b}-A\hat{x}
```

Geometrically speaking, the closer $A\hat{x}$ comes to $\vec{b}$ , the smaller the residual error. Similar to how we were minimizing the RSS earlier, in linear algebra, we are minimizign the squared residual: $\|\vec{r}\|^2$. The shortest distance from a point ($\vec{b}$) to a subspace ($A\hat{x}$) is a perpendicular (orthogonal) line, so we are projecting $\vec{b}$ onto the column space of $A$, and want each column to be orthogonal to the residual vector to minimize the residual error. When the dot product of two vectors is $0$, then those vectors are orthogonal. 

So the deriving the equation for $\hat{x}$:

```math
A^T\vec{r}=0\\\qquad\small\text{($A$ is transposed so that each column vector of $A$ becomes a row vector to permit multiplication of the \\residual column vector)}
\\\\
\text{Substituting $\vec{b}-A\hat{x}$ for $\vec{r}$: }
\\A^T(\vec{b}-A\hat{x})=0
\\\\\text{This gives us the Normal Equation:}
\\A^TA\hat{x} = A^T\vec{b}\
\\\\\text{To solve for $\hat{x}$, we multiply both sides by the inverse of ${(A^TA)}$:}
\\\hat{x} = (A^TA)^{-1}A^T\vec{b}
```

Because MNIST training dataset matrix will have more rows than columns, also known as a skinny matrix, it is overdetermined. 

#### Undetermined vs. Overdetermined Matrix

A matrix is overdetermined when $m>n$, $m$ representing rows and $n$ representing columns. Geometrically, since there are more equations than dimensions, there is usually no single point where the lines or planes intersect. 

```math
\left[
  \begin{array}{cc}
    1 & 2   \\
    2 & 1  \\ 
  \end{array}
\right]
\left[
\begin{array}{c}
x_1
\\x_2
\end{array}\right]
=
\left[
\begin{array}{c}
3
\\3
\end{array}\right]
\\\qquad\small\text{Determined system - One unique solution}
```

```math
\left[
  \begin{array}{ccc}
    1 & 1 & 2   \\
    2 & 1 & 1  \\ 
  \end{array}
\right]
\left[
\begin{array}{c}
x_1
\\x_2
\\x_3
\end{array}\right]
=
\left[
\begin{array}{c}
4
\\4

\end{array}\right]
\\\small\text{Undertermined system - Generally infinitely many solutions}
```

```math
\left[
  \begin{array}{cc}
    1 & 2    \\
    1 & 1   \\ 
    2 & 1
  \end{array}
\right]
\left[
\begin{array}{c}
x_1
\\x_2

\end{array}\right]
=
\left[
\begin{array}{c}
4
\\4
\\4

\end{array}\right]
\\\small\text{Overdetermined system - Generally no solutions}
```

Since the matrix is overdetermined and it is rather unlikely that there is an exact solution (imagine each of the pixels is a point in a hyperspace, what are the chances that a straight line passes through them all perfectly?), we need to use the least squares method to find the line that minimizes the distance between the points.

For us to solve the equation ${x} = (A^TA)^{-1}A^T\vec{b}$, we need to find the inverse of $A^TA$, but because our digit matrix is overdetermined, this is not possible, so we must find the pseudo-inverse using singular value decomposition or QR decomposition. What will help us decide which one is best is understanding the rank of our matrix. 

#### Matrix Rank

A Matrix's rank is determined by how many pivot columns it has. The rank can be found in several different ways such as changing it into row echelon form, counting the number of non-zero singular values in the singular value matrix $\Sigma$, or counting the number of non-zero values in the right triangle matrix in $QR$ decomposition.  

The rank of our digit matrix will be *rank deficient*, meaning the number of non-zero singular values will be less than the $min(n|m)$, where $n$ and $m$ represent the number of rows and columns, respectively. This is most likely due to the low variation of the pixels in the data set. A majority of them are black pixels, and because we converted the greyscale values of 0 - 255 to a binary 0 or 1, we have further reduced the subtle variations that may keep the columns independent. These factors play a role in the rank of a matrix, which ultimately decides what methods we can best use to find the least squares solution.

#### QR Decomposition

