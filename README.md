
[![Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/schrodingersket/hedges/HEAD?urlpath=notebooks%2Fmain.ipynb)

# Hyperbolic Educational Discontinuous Galerkin Equation Solver

This repository contains a Discontinuous Galerkin Python solver for 1D Hyperbolic PDEs. It is 
intended to be used primarily for education and as such prioritizes clarity in the codebase over
computational efficiency or cleverness. It is designed to be extensible and modular so that support
for new hyperbolic systems is easy to implement, and comes out of the box with an example for
the 1D shallow water equations (SWE) with variable bathymetry, prescribed inflow, and free outflow.

## Usage

Once you've installed the dependencies listed in `requirements.txt` 
(`pip install -r requirements.txt`), running `main.py` will generate a full 
animation of the SWE for several different variations on Gaussian inflows:

```python main.py```

Each simulation result is saved in a sub-directory of the `out` folder according to the format
`<epoch timestamp>_<six-digit random hex>`; the simulation data itself is saved in `solution.csv`,
the parameters for the simulation in `parameters.csv`, and a GIF of the solution animation is
saved at `swe_1d.gif`.

The main script (`main.py`) instantiates several instances of `simulation.SWEFlowRunner`, which is
configured specifically for running simultaneous simulations for prescribed inflow and transmissive 
outflow for the Shallow Water Equations. While that class may be useful as a reference 
implementation for running several simulations, the following is a full example of 
running a single simulation for the purpose of experimentation/modification:

```python
#!/usr/bin/env python3
# coding: utf-8

# # 1D Discontinuous Galerkin Shallow Water Solver
# 
# We solve the 1D Shallow Water Equations in conservative form:
# 
# \begin{align*}
#     h_t + q_x &= 0 \\
#     q_t + \left[ \frac{q^2}{h} + \frac{1}{2} g h^2 \right]_x &= -g h b_x - C_f \left(\frac{q}{h}\right)^2
# \end{align*}
# 
import numpy as np
import matplotlib.pyplot as plt


import hedges.bc as bc
import hedges.fluxes as fluxes
import hedges.quadrature as quadrature
import hedges.rk as rk
import hedges.swe_1d as swe_1d


# Physical parameters
#
g = 1.0  # gravity

# Domain
#
tspan = (0.0, 4.0)
xspan = (-1, 1)

# Bathymetry parameters
#
b_smoothness = 0.1
b_amplitude = 0.02
b_slope = -0.05
assert(b_smoothness > 0)

# Inflow parameters
#
inflow_amplitude = 0.05
inflow_smoothness = 1.0
inflow_peak_time = 2.0
assert(inflow_amplitude > 0)

# Initial waveheight
#
h0 = 0.2
assert(h0 > 0)


def swe_bathymetry(x):
    """
    Describes bathymetry with an upslope which is perturbed by a hyperbolic tangent function.
    """
    return b_slope * x + b_amplitude * np.arctan(x/b_smoothness)


def swe_bathymetry_derivative(x):
    """
    Derivative of swe_bathymetry
    """
    return b_slope * np.ones(x.shape) + b_amplitude * b_smoothness/(1 + np.square(b_smoothness*x)) * np.exp(-np.square(b_smoothness*x))


def q_bc(t):
    """
    Describes a Gaussian inflow, where the function transitions to a constant value upon attaining
    its maximum value.

    :param t: Time
    :return:
    """
    tt = np.array(t)
    return inflow_amplitude * np.exp( -np.square(np.minimum(tt, inflow_peak_time * np.ones(tt.shape)) - inflow_peak_time) / (2 * np.square(inflow_smoothness)) )


def initial_condition(x):
    """
    Creates initial conditions for (h, uh).

    :param x: Computational domain
    :return:
    """
    initial_height = h0 * np.ones(x.shape) - swe_bathymetry(x)  # horizontal water surface
    initial_flow = q_bc(0) * np.ones(x.shape)  # Start with whatever flow is prescribed by our inflow BC

    ic = np.array((
        initial_height,
        initial_flow,
    ))

    # Verify consistency of initial condition
    #
    if not np.allclose(ic[1][0], q_bc(0)):
        raise ValueError('Initial flow condition must match prescribed inflow.')

    return ic

# Plot bathymetry and ICs
#
xl, xr = xspan
t0, tf = tspan

xx = np.linspace(xl, xr, num=100)
tt = np.linspace(t0, tf, num=100)

fig, (h_ax, hv_ax, q_bc_ax) = plt.subplots(3, 1)

ic = initial_condition(xx)
bb = swe_bathymetry(xx)
qq_bc = q_bc(tt)

# Plot initial wave height and bathymetry
#
h_ax.plot(xx, ic[0] + bb)
h_ax.plot(xx, bb)
h_ax.set_title('Initial wave height $h(x, 0)$')

# Plot initial flow rate
#
hv_ax.plot(xx, ic[1])
hv_ax.set_title('Initial flow rate $q(x, 0)$')

# Plot flow rate at left boundary over simulation time
#
q_bc_ax.plot(tt, qq_bc)
q_bc_ax.set_title('Boundary flow rate $q({}, t)$'.format(xl))

plt.tight_layout()
plt.show()

# Instantiate solver with bathymetry
#
solver = swe_1d.ShallowWater1D(
    b=swe_bathymetry,
    b_x=swe_bathymetry_derivative,
    gravity=g
)


t_interval_ms = 20
dt = t_interval_ms / 1000
surface_flux = fluxes.lax_friedrichs_flux
print('Integrating ODE system...')
solution = solver.solve(
    tspan=tspan,
    xspan=xspan,
    cell_count=16,
    polydeg=4,
    initial_condition=initial_condition,
    intercell_flux=surface_flux,
    left_boundary_flux=swe_1d.ShallowWater1D.bc_prescribed_inflow(q_bc, gravity=g, surface_flux=surface_flux),
    right_boundary_flux=bc.transmissive_outflow(surface_flux=surface_flux),
    quad_rule=quadrature.gll,
    **{
        'method': rk.SSPRK33,
        't_eval': np.arange(tspan[0], tspan[1], dt),
        'max_step': dt,  # max time step for ODE solver
        'rtol': 1.0e-6,
        'atol': 1.0e-6,
    }
)

# Plot solution animation
#
ani, plt = solver.plot_animation(solution, frame_interval=t_interval_ms)

# Save animation to file
#
movie_name = 'swe_1d.gif'
print('Writing movie to {}...'.format(movie_name))

ani.save(movie_name, progress_callback=lambda i, n: print(
    f'Saving animation frame {i + 1}/{n}'
) if i % 50 == 0 else None)
print('Animation written to {}.'.format(movie_name))

plt.show()
```

The `plot_animation` function of the `hedges.hyperbolic_solver_1d.Hyperbolic1DSolver` base class returns 
a tuple of the form 
([FuncAnimation](https://matplotlib.org/3.5.1/api/_as_gen/matplotlib.animation.FuncAnimation.html), 
[matplotlib](https://matplotlib.org/3.5.1/api/matplotlib_configuration_api.html#matplotlib)) which 
may be used to save individual frames or modify plots as desired. Subclass implementations may 
optionally override the default plotting behavior, with a reference implementation provided in 
`hedges.swe_1d.ShallowWater1D`.

A Jupyter notebook is provided in this repository at `hedges/main.ipynb`, and may be viewed (and
interacted with) at 
[BinderHub](https://mybinder.org/v2/gh/schrodingersket/hedges/HEAD?urlpath=notebooks%2Fmain.ipynb).

## Extending the Solver

Currently, only the 1D Shallow Water equations are implemented. However, support can easily be added
for other 1D hyperbolic PDE systems by simply subclassing `hedges.hyperbolic_solver_1d.Hyperbolic1DSolver`
and implementing the flux `F(u)`, source `S(u)`, and flux Jacobian `F'(u)` terms for the desired 
system in conservative form. See `hedges.swe_1d.ShallowWater1D` for a reference implementation.

A local Lax-Friedrichs flux is used as the approximate Riemann solver between cell 
interfaces, but the `solve` method of the `hedges.hyperbolic_solver_1d.Hyperbolic1DSolver` class accepts
a function reference which can be used to implement e.g. HLL flux (or exact Riemann solvers).
