@def title = "JuliaML Basics"
@def author = "Josh Day"
@def tags = ["Julia"]

# JuliaML Basics

~~~
<img src="https://avatars3.githubusercontent.com/u/15799035?v=3&s=200" style="width: 20%; margin:auto; border-radius: 5px">
~~~

[JuliaML](https://github.com/JuliaML) is a GitHub organization dedicated to building tools for machine learning in Julia.  This post serves as an introduction to some of the building blocks which JuliaML provides.  The two packages I'll discuss here are:

- [LossFunctions.jl](https://github.com/JuliaML/LossFunctions.jl)
- [PenaltyFunctions.jl](https://github.com/JuliaML/PenaltyFunctions.jl)

I'm assuming you have a little background in statistics/machine learning and an objective function such as

$$\frac{1}{n}\sum\_{i=1}^n f\_i(\beta)+\sum\_{j=1}^p \lambda\_j J(\beta\_j)$$

is recognizable to you as a "loss" + "penalty".



# LossFunctions.jl

An interface for dealing with loss functions. Internally, most losses are defined as distance-based (a function of $\hat y-y$) or margin-based (a function of $y * \hat y$) where $y$ is the target and $\hat y$ is a predicted value.

#### Distance-based losses

![](https://camo.githubusercontent.com/80ae72e5878d98aacb0eed6863958c02fd73b1b3/68747470733a2f2f7261776769746875622e636f6d2f4a756c69614d4c2f46696c6553746f726167652f6d61737465722f4c6f737346756e6374696f6e732f64697374616e63652e737667)
#### Margin-based losses

![](https://camo.githubusercontent.com/80dc81d308845dd34ff70a953aafffb30b2c5c3a/68747470733a2f2f7261776769746875622e636f6d2f4a756c69614d4c2f46696c6553746f726167652f6d61737465722f4c6f737346756e6374696f6e732f6d617267696e2e737667)

### Example Usage
```julia
using LossFunctions

l = L1DistLoss()
value(l, .1, .2)  # |.2 - .1|
deriv(l, .1, .2)  # sign(.2 - .1)

y = randn(100)
yhat = randn(100)

value(l, y, yhat)  # vector of value mapped to y[i], yhat[i]
value(l, y, yhat, AvgMode.Sum())  # sum(value(l, y, yhat)), but better
value(l, y, yhat, AvgModel.Mean()) # mean(value(l, y, yhat)), but better
```

#### Naive Gradient Descent Algorithm

Assume we want to use a linear transformation so that our prediction of a vector $y$ is $X\beta$.

Our loss function looks like
$$
    \frac{1}{n}\sum\_{i=1}^n f(y\_i, x\_i^T\beta),
$$
with gradient
$$
    X^T[\frac{1}{n}\sum_{i=1}^n f'(y\_i, x\_i^T\beta)].
$$

We can implement a naive, inefficient gradient descent algorithm as:

```julia
function f(l::Loss, x::Matrix, y::Vector; maxit=20, s=.5)
    n, p = size(x)
    β = zeros(p)
    for i in 1:maxit
        β -= (s / n) * x' * deriv(l, y, x * β)
    end
    β
end
```
This automatically works with whatever loss function I provide because multiple dispatch is amazing:
```julia
# make some fake data
x = randn(1000, 3)
y = x * [1.0, 2.0, 3.0] + randn(1000)
```

```julia
julia> f(L2DistLoss(), x, y)
3-element Array{Float64,1}:
 0.966367
 2.05046
 2.94227

julia> f(L1DistLoss(), x, y)
3-element Array{Float64,1}:
 1.00185
 2.00302
 2.95002

julia> f(HuberLoss(2.), x, y)
3-element Array{Float64,1}:
 0.967642
 2.0485
 2.94429
```

# PenaltyFunctions.jl
An interface for penalty/regularization functions.  It follows similar conventions to LossFunctions.

![](https://camo.githubusercontent.com/6e0b322991cdb606579e438d3868de0bdd06c7d0/68747470733a2f2f7261776769746875622e636f6d2f4a756c69614d4c2f46696c6553746f726167652f6d61737465722f50656e616c747946756e6374696f6e732f756e69766172696174652e737667)

```julia
θ = rand(5)  # parameter vector

p = L1Penalty()

value(p, θ)  # sum(abs, θ)
grad(p, θ)   # the gradient, sign.(θ)
prox(p, θ, .1)  # proximal mapping (soft-thresholding) with step size .1
```

Let's change our gradient descent algorithm to proximal gradient method:
```julia
function g(l::Loss, pen::Penalty, x::Matrix, y::Vector; maxit=20, s=.5, λ=.1)
    n, p = size(x)
    β = zeros(p)
    for i in 1:maxit
        β = prox(pen, β - (s / n) * x' * deriv(l, y, x * β), s * λ)
    end
    β
end
```

We can now do proximal gradient method on arbitrary loss/penalty combinations because multiple dispatch is amazing:
```julia
julia> g(L2DistLoss(), L1Penalty(), x, y; λ = .4)
3-element Array{Float64,1}:
 0.789894
 1.80736
 2.81209

julia> g(L1DistLoss(), L1Penalty(), x, y; λ = .4)
3-element Array{Float64,1}:
 0.0
 0.464488
 1.5998

julia> g(HuberLoss(2.), L1Penalty(), x, y; λ = .4)
3-element Array{Float64,1}:
 0.525633
 1.57115
 2.57907
```




# Takeaway

These two packages create a consistent "grammar" for losses and regularization.  Julia's multiple dispatch then allows us to write general algorithms like the proximal gradient method example.  This paves the way for machine learning experimentation that is unavailable in any other language that I know of.
