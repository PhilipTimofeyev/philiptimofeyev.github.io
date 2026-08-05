---
title: Linear Regression for Handwritten Digit Classification - Part I
layout: post
date: 2026-08-04 19:16:00 +0800
author: philip
categories: [Blogging, Tutorial]
tags: [machine]
render_with_liquid: false
media_subpath: /assets/img/posts/linear_regression_part_i
math: true
---

---



Linear regression is a fundamental tool and first step into supervised learning. Supervised learning is when a model is trained on labeled data sets containing inputs and corresponding outputs.

#### Why Linear Regression Is *Not* Good for Classification

Linear regression is good at predicting a *quantitative* result given inputs instead of a *qualitative* one. For example, predicting the cost of a chocolate bar based on weight would be a quantitive prediction, while predicting who a person's favorite composer is based on age, would be qualitative. 

Encoding the composers into a numerical output variable, $Y$, could look something like this:

The issue here is that the encodings, $\Set{1, 2 ,3}$, do not represent any mathematical meaning between the composers. Bach isn't half of Mendelssohn and Saint-Saëns isn't three times Bach. If we compare this to an example of chocolate bars:


$$
Y = \Set{ 
    \begin{aligned} 
        &$1 \quad\text{Chocolate A}\\ 
        &$2 \quad\text{Chocolate B}\\ 
        &$3 \quad\text{Chocolate C}
    \end{aligned} 
}
$$

Here the differences between chocolate bar prices makes mathematical sense and are meaningful. Linear regression assumes that increments represent proportional increases, ie. as the weight of a chocolate bar doubles, the cost doubles. This does not quite work for qualitative data.

<div style="display: flex; flex-direction: row; gap: 20px; margin-bottom: 30px;">   <div style="flex: 1;">     <img src="chocolate_graph.png" alt="Chocolate graph" style="width: 100%;">   </div>    <div style="flex: 1;">     <img src="composer_graph.png" alt="Composer graph" style="width: 100%;">   </div> </div>



In the left graph, we have graphed our composer qualitative data set, and in the right graph, we have graphed the chocolate quantitative data set, both with a best-fit line. When looking at the chocolate graph, it's fair to assume that as the weight of the chocolate increases, the price also increases. We can see there's a positive linear relationship between the two. This works because there is a proportional mathematical significance between the Y variables, $2 is twice $1. 

Looking at the composer graph, you may want to assume that as a person's age increases, they start to prefer Mendelssohn. But this does not work. Even though we've assigned numerical values to the composers, they don't carry any mathematical meaning. Mendelssohn is not twice of Bach.

If we use the best-fit line for the cost of a chocolate bar, a 12 oz bar should cost $\approx\text{}2.57$ which is between a three dollar 15oz bar, and a two dollar 7oz bar. If we try the same for the composer data set, a 30 year old person would give a composer preference of $1.75$, but what does a preference between Mendelssohn and Bach mean? You can't have a preference of Bach with a $.75$ preference of Mendelssohn, that doesn't translate to any useful meaning.

#### Applying To Digit Classification

The data used is the MNIST digit dataset which is composed of 70,000 images, 60,000 for training and 10,000 for validation. Each image is a 28x28 pixel handwritten digit, 0 through 9.

What our instinct may tell us to initially do is to set up the basic $Y=\beta_0 + \beta_1 X$ equation. If we flatten the 28x28 image into a 1x784 row of pixels, each pixel can be considered a predictor in the equation. The equation would look like this:

$$
Y=\beta_0+\beta_1 \text{Pixel}_1 + \beta_2 \text{Pixel}_2+...+\beta_{784}\text{Pixel}_{784}
$$

Every pixel from the dataset has a value ranging between `0-255` that measures the intensity.

<div style="display: flex; gap: 20px; justify-content: center;">   <img src="mnist_digits.png" alt="MNIST digit" width="300">  </div>



Now what about $Y$? Our digits are 0 through 9, so we might lean towards encoding the digits like so:

$$
Y = \Set{ 
    \begin{aligned} 
        &0\\     
        &1\\ 
        &2\\ 
        &3\\ 
        &4\\ 
        &5\\ 
        &6\\ 
        &7\\
        &8\\ 
        &9\\         
    \end{aligned} 
} 
$$


But we run into a problem similar to our composers issue. Let's say we start training and our first image is the digit 0. We apply it to the equation and we get 784 data points that all represent "0". Then let's say the next digit is 1. Now we're trying to create a best-fit line that goes through 0 and 1, much like the composers Bach and Saent-Saëns. But even though the encodings are numbers, as are the drawn digits, they have no mathematical relation to each other. If instead of handwritten digits we used hand-written animals, a dog would have no mathematical relation to a walrus.

So using linear regression for classification when there are three or more classes doesn't work. But what about two classes?

#### Classification with a Binary Qualification Response

If we only allow two qualitative response variables, we can encode them into a binary format which can be interpreted as either 0 or 1. In our digit classification case, we could train each digit to either be the digit trained, or not. For example, we can train the digit 0 by creating a large matrix of the 60,000 digits with each row being a digit, each column representing the pixel along with its predictor, and the $Y$ value being either $0$ for when the digit is not $0$, or $1$ when the digit is $0$. Once this matrix equation is solved, we end up with the weights for the pixels for that digit. We can save the weights and repeat the process for all of the digits.

##### Preparing the Data

To prepare our MNIST data set for training, we need to do a few things: 

1. Flatten the 28x28 pixel matrix to a single row of 784 elements, 
2. Convert each pixel into either being on or off using binary values of `0` or `1`.
3. Convert the label set to binary.

Here is what the Rust implementation may look like:

```rust
fn prepare_train_data(
    trn_img: &[u8],
    trn_lbl: &[u8],
    digit_to_train: u8,
) -> Result<(DMatrix<f64>, DVector<f64>)> {
    let train_data = DMatrix::from_row_slice(N_TRAINING_SET as usize, 784, trn_img)
        .map(|pixel| if pixel as f64 > 0.0 { 1.0 } else { 0.0 });

    // Add bias term in the form of a column of 1's
    let train_data = train_data.insert_column(0, 1.0);

    let train_label = DVector::from_row_slice(trn_lbl)
        .map(|digit| if digit == digit_to_train { 1.0 } else { 0.0 });

    Ok((train_data, train_label))
}

```



### Solving Least Squares

When solving the standard least squares problem with a single slope (predictor) and bias, the equation is:

$$
y_i\approx \hat{\beta}_0 +\hat{\beta}_1x_i
$$


To find the least squares the solution, the goal is to minimize the *residual sum of squares*:

$$
RSS=e^2_1+e^2_2+...+e^2_n
\ \text{or }

(y_1-\hat{\beta}_0-\hat{\beta}_1x_1)^2+(y_2-\hat{\beta}_0-\hat{\beta}_1x_2)^2+...+(y_n-\hat{\beta}_0-\hat{\beta}_1x_n)^2
\\
\\
\text{The total sum of squared errors is represented as:}
\\
S=\sum_{i=1}^n(y_i-(\hat{\beta}_0+\hat{\beta}_1x_i))^2
\\
\small\text{$S$ is what needs to be minimized }
$$


To minimize the function, we must find where the slope of $S=0$. Taking the partial derivatives with respect to $\beta_0$ and $\beta_1$, where $\beta_0$ represents the bias and $\beta_1$ represents the predictor:

$$
\frac{\partial S}{\partial {\beta}_0}=-2\sum_{i=1}^n(y_i-(\hat{\beta}_0+\hat{\beta}_1x_i))
\\\\
\frac{\partial S}{\partial {\beta}_1}=-2\sum_{i=1}^nx_i(y_i-(\hat{\beta}_0+\hat{\beta}_1x_i))
\\\\
\text{Setting the first equation to 0 to find the minimum and solving:}
\\\\
\sum_{i=1}^ny_i-\hat{\beta}_1\sum_{i=1}^nx_i=n\hat{\beta}_0
\\\\
\text{When we divide by $n$, this gives us the samples means:}
\\\\
\frac{1}{n}{\sum_{i=1}^ny_i}=\bar{y} \qquad  \hat{\beta}_1(\frac{1}{n}{\sum_{i=1}^nx_i})=\hat{\beta}_1\bar{x}
\\\\\text{Substituting back into the first equation:}
\\\\
\hat{\beta}_0=\bar{y}-\hat{\beta}_1\bar{x}
\\\\\text{Now setting the second equation to 0 and solving:}
\\\\
\sum{x_iy_i}-\hat{\beta}_1\sum{x_i^2}-\hat{\beta}_0\sum{x_i}=0
\\\\
\text{If we substitute what we previously found for $\hat{\beta}_0$}:
\\\\
\sum{y_i}-\bar{y}\sum{x_i}=\hat{\beta}_1(\sum{x_i^2}-\bar{x}\sum{x_i})
\\\\
\
\text{Finally solving for $\hat{\beta}_1$:}
\\\\
\hat{\beta}_1=\frac{\sum_{i=1}^n(x_i-\bar{x})(y_i-\bar{y})}{\sum_{i=1}^n(x_i-\bar{x})^2}=\frac{\text{Cov}(x,y)}{\text{Var}(x)}=\frac{S_{xy}}{S_{xx}}
$$


These two formulas give us the solution to the bias and *one* predictor. But for the digit classifier, there are 784 predictors! It may seem that we can just apply the $\hat{\beta}_1$ for all values of $n$ to get the multiple predictors. But this does not work because of **omitted variable bias**. The current formula is saying "$\hat{\beta}_1$ represents the change in $Y$ per unit change in $x_1$", but ignores $x_2$, $x_3$ and so on. The correct approach is "the change in $Y$ per unit change in $x_1$ *while holding all other x's constant*". 

Because predictors are generally related to each other, the $\hat{\beta}_1$ minimizer formula ignores the effect of other predictors and this creates the omitted variable bias. For this reason, a simple linear regression could have different results than a multilinear regression. 

To account for the other predictors, we enlist linear algebra for the job!



<div align="center">   <h3>Part II (coming soon)</h3> </div>



## References

James, G., Witten, D., Hastie, T., Tibshirani, R., & Taylor, J. (2023). *An Introduction to Statistical Learning*. Springer.

Strang, G. (2023). *Introduction to Linear Algebra*. Wellesley-Cambridge Press.
