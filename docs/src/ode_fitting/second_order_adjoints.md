# Newton and Hessian-Free Newton-Krylov with Second Order Adjoint Sensitivity Analysis

In many cases it may be more optimal or more stable to fit using second order
Newton-based optimization techniques. Since DiffEqSensitivity.jl provides
second order sensitivity analysis for fast Hessians and Hessian-vector
products (via forward-over-reverse), we can utilize these in our neural/universal
differential equation training processes.

`sciml_train` is setup to automatically use second order sensitivity analysis
methods if a second order optimizer is requested via Optim.jl. Thus `Newton`
and `NewtonTrustRegion` optimizers will use a second order Hessian-based
optimization, while `KrylovTrustRegion` will utilize a Krylov-based method
with Hessian-vector products (never forming the Hessian) for large parameter
optimizations.

```julia
using Lux, DiffEqFlux, Optimization, OptimizationFlux, DifferentialEquations, Plots, Random

rng = Random.default_rng()
u0 = Float32[2.0; 0.0]
datasize = 30
tspan = (0.0f0, 1.5f0)
tsteps = range(tspan[1], tspan[2], length = datasize)

function trueODEfunc(du, u, p, t)
    true_A = [-0.1 2.0; -2.0 -0.1]
    du .= ((u.^3)'true_A)'
end

prob_trueode = ODEProblem(trueODEfunc, u0, tspan)
ode_data = Array(solve(prob_trueode, Tsit5(), saveat = tsteps))

dudt2 = Lux.Chain(ActivationFunction(x -> x.^3),
                  Lux.Dense(2, 50, tanh),
                  Lux.Dense(50, 2))
prob_neuralode = NeuralODE(dudt2, tspan, Tsit5(), saveat = tsteps)
p,st = Lux.setup(rng, dudt2)

function predict_neuralode(p)
  Array(prob_neuralode(u0, p, st)[1])
end

function loss_neuralode(p)
    pred = predict_neuralode(p)
    loss = sum(abs2, ode_data .- pred)
    return loss, pred
end

# Callback function to observe training
list_plots = []
iter = 0
cb = function (p, l, pred; doplot = false)
  global list_plots, iter

  if iter == 0
    list_plots = []
  end
  iter += 1

  display(l)

  # plot current prediction against data
  plt = scatter(tsteps, ode_data[1,:], label = "data")
  scatter!(plt, tsteps, pred[1,:], label = "prediction")
  push!(list_plots, plt)
  if doplot
    display(plot(plt))
  end

  return l < 0.01
end

adtype = Optimization.AutoZygote()
optf = Optimization.OptimizationFunction((x,p)->loss_neuralode(x), adtype)

optprob1 = Optimization.OptimizationProblem(optf, Lux.ComponentArray(p))
pstart = Optimization.solve(optprob1, ADAM(0.01), cb=cb, maxiters = 100).u

optprob2 = Optimization.OptimizationProblem(optf, pstart)
pmin = Optimization.solve(optprob2, NewtonTrustRegion(), cb=cb, maxiters = 200)
pmin = Optimization.solve(optprob2, Optim.KrylovTrustRegion(), cb=cb, maxiters = 200)
```

Note that we do not demonstrate `Newton()` because we have not found a single
case where it is competitive with the other two methods. `KrylovTrustRegion()`
is generally the fastest due to its use of Hessian-vector products.