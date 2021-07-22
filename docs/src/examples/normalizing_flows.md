# Continuous Normalizing Flows with GalacticOptim.jl

Now, we study a single layer neural network that can estimate the density `p_x` of a variable of interest `x` by re-parameterizing a base variable `z` with known density `p_z` through the Neural Network model passed to the layer.

## Copy-Pasteable Code

Before getting to the explanation, here's some code to start with. We will
follow a full explanation of the definition and training process:

```julia
using DiffEqFlux, DifferentialEquations, Distributions, GalacticOptim

nn = Chain(Dense(1, 3, tanh), Dense(3, 1, tanh))
tspan = (0.0f0,10.0f0)
ffjord_test = FFJORD(nn,tspan, Tsit5())
data_train = Float32.(rand(Normal(6.0,0.7), 1, 100))

function loss_adjoint(θ)
    logpx = ffjord_test(data_train,θ)[1]
    loss = -mean(logpx)
end

adtype = GalacticOptim.AutoZygote()

optf = GalacticOptim.OptimizationFunction((x, p) -> loss_adjoint(x), adtype)
optfunc = GalacticOptim.instantiate_function(optf, ffjord_test.p, adtype, nothing)
optprob = GalacticOptim.OptimizationProblem(optfunc, ffjord_test.p)

res1 = GalacticOptim.solve(optprob,
                           ADAM(0.1),
                           maxiters = 100)

# Retrain using the LBFGS optimizer
optprob2 = remake(optprob,u0 = res1.u)

res2 = GalacticOptim.solve(optprob2,
                           LBFGS(),
                           allow_f_increases = false)
```

We can use DiffEqFlux.jl to define, train and output the densities computed by CNF layers. In the same way as a neural ODE, the layer takes a neural network that defines its derivative function (see [1] for a reference). A possible way to define a CNF layer, would be:

```julia
using DiffEqFlux, DifferentialEquations, Distributions, GalacticOptim

nn = Chain(Dense(1, 3, tanh), Dense(3, 1, tanh))
tspan = (0.0f0,10.0f0)
ffjord_test = FFJORD(nn,tspan, Tsit5())
```

where we also pass as an input the desired timespan for which the differential equation that defines `log p_x` and `z(t)` will be solved.

### Training a CNF layer

First, let's get an array from a normal distribution as the training data

```julia
data_train = Float32.(rand(Normal(6.0,0.7), 1, 100))
```

Now we define a loss function that we wish to minimize

```julia
function loss_adjoint(θ)
    logpx = ffjord_test(data_train,θ)[1]
    loss = -mean(logpx)
end
```

In this example, we wish to choose the parameters of the network such that the likelihood of the re-parameterized variable is maximized. Other loss functions may be used depending on the application. Furthermore, the CNF layer gives the log of the density of the variable x, as one may guess from the code above.

We then train the neural network to learn the distribution of `x`.

Here we showcase starting the optimization with `ADAM` to more quickly find a minimum, and then honing in on the minimum by using `LBFGS`.

```julia
# Train using the ADAM optimizer
res1 = DiffEqFlux.sciml_train(loss_adjoint, ffjord_test.p,
                              ADAM(0.1), cb = cb,
                              maxiters = 100)

# output
* Status: success

* Candidate solution
   u: [-1.88e+00, 2.44e+00, 2.01e-01,  ...]
   Minimum:   1.240627e+00

* Found with
   Algorithm:     ADAM
   Initial Point: [9.33e-01, 1.13e+00, 2.92e-01,  ...]

```

We then complete the training using a different optimizer starting from where `ADAM` stopped.

```julia
# Retrain using the LBFGS optimizer
res2 = DiffEqFlux.sciml_train(loss_adjoint, res1.u,
                              LBFGS(),
                              allow_f_increases = false)
# output
* Status: success

* Candidate solution
   u: [-1.06e+00, 2.24e+00, 8.77e-01,  ...]
   Minimum:   1.157672e+00

* Found with
   Algorithm:     L-BFGS
   Initial Point: [-1.88e+00, 2.44e+00, 2.01e-01,  ...]

* Convergence measures
   |x - x'|               = 0.00e+00 ≰ 0.0e+00
   |x - x'|/|x'|          = 0.00e+00 ≰ 0.0e+00
   |f(x) - f(x')|         = 0.00e+00 ≤ 0.0e+00
   |f(x) - f(x')|/|f(x')| = 0.00e+00 ≤ 0.0e+00
   |g(x)|                 = 4.09e-03 ≰ 1.0e-08

* Work counters
   Seconds run:   514  (vs limit Inf)
   Iterations:    44
   f(x) calls:    244
   ∇f(x) calls:   244
```

`References`:

[1] W. Grathwohl, R. T. Q. Chen, J. Bettencourt, I. Sutskever, D. Duvenaud. FFJORD: Free-Form Continuous Dynamic For Scalable Reversible Generative Models. arXiv preprint at arXiv1810.01367, 2018.
