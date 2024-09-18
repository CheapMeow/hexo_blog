---
title: 'Two kinds of poisson equation'
date: 2024-09-18 09:45:00
tags:
  - CFD
  - PDE
  - Poisson Equation
---

## Two kinds of poisson equation

### Taking the Divergence Directly from the Navier-Stokes Equation

Incompressibility condition:

$$
\dfrac{\partial{u}}{\partial{x}} + \dfrac{\partial{v}}{\partial{y}} + \dfrac{\partial{w}}{\partial{z}} = 0
$$

Momentum equations without source terms:

$$
\begin{align*}
    \dfrac{\partial{u}}{\partial{t}} + u\dfrac{\partial{u}}{\partial{x}} + v\dfrac{\partial{u}}{\partial{y}} + w\dfrac{\partial{u}}{\partial{z}} &= -\dfrac{1}{\rho}\dfrac{\partial{p}}{\partial{x}} + \nu \left(\dfrac{\partial^2{u}}{\partial{x^2}}+\dfrac{\partial^2{u}}{\partial{y^2}}+\dfrac{\partial^2{u}}{\partial{z^2}}\right), \\
    \dfrac{\partial{v}}{\partial{t}} + u\dfrac{\partial{v}}{\partial{x}} + v\dfrac{\partial{v}}{\partial{y}} + w\dfrac{\partial{v}}{\partial{z}} &= -\dfrac{1}{\rho}\dfrac{\partial{p}}{\partial{y}} + \nu \left(\dfrac{\partial^2{v}}{\partial{x^2}}+\dfrac{\partial^2{v}}{\partial{y^2}}+\dfrac{\partial^2{v}}{\partial{z^2}}\right), \\
    \dfrac{\partial{w}}{\partial{t}} + u\dfrac{\partial{w}}{\partial{x}} + v\dfrac{\partial{w}}{\partial{y}} + w\dfrac{\partial{w}}{\partial{z}} &= -\dfrac{1}{\rho}\dfrac{\partial{p}}{\partial{z}} + \nu \left(\dfrac{\partial^2{w}}{\partial{x^2}}+\dfrac{\partial^2{w}}{\partial{y^2}}+\dfrac{\partial^2{w}}{\partial{z^2}}\right)
\end{align*}
$$

Taking the divergence of the momentum equations:

Differentiate each component of the momentum equation with respect to \(x\), \(y\), and \(z\):

$$
\begin{align*}
    \dfrac{\partial{}}{\partial{t}}\dfrac{\partial{u}}{\partial{x}} + \left(\dfrac{\partial{u}}{\partial{x}}\right)^2 + u\dfrac{\partial^2{u}}{\partial{x^2}} + \dfrac{\partial{v}}{\partial{x}}\dfrac{\partial{u}}{\partial{y}} + v\dfrac{\partial^2{u}}{\partial{x}\partial{y}} + \dfrac{\partial{w}}{\partial{x}}\dfrac{\partial{u}}{\partial{z}} + w\dfrac{\partial^2{u}}{\partial{x}\partial{z}} &= -\dfrac{1}{\rho}\dfrac{\partial^2{p}}{\partial{x^2}} + \nu \dfrac{\partial{}}{\partial{x}}\left(\dfrac{\partial^2{u}}{\partial{x^2}}+\dfrac{\partial^2{u}}{\partial{y^2}}+\dfrac{\partial^2{u}}{\partial{z^2}}\right), \\
    \dfrac{\partial{}}{\partial{t}}\dfrac{\partial{v}}{\partial{y}} + \dfrac{\partial{u}}{\partial{y}}\dfrac{\partial{v}}{\partial{x}} + u\dfrac{\partial^2{v}}{\partial{x}\partial{y}} + \left(\dfrac{\partial{v}}{\partial{y}}\right)^2 + v\dfrac{\partial^2{v}}{\partial{y^2}} + \dfrac{\partial{w}}{\partial{y}}\dfrac{\partial{v}}{\partial{z}} + w\dfrac{\partial^2{v}}{\partial{y}\partial{z}} &= -\dfrac{1}{\rho}\dfrac{\partial^2{p}}{\partial{y^2}} + \nu \dfrac{\partial{}}{\partial{y}}\left(\dfrac{\partial^2{v}}{\partial{x^2}}+\dfrac{\partial^2{v}}{\partial{y^2}}+\dfrac{\partial^2{v}}{\partial{z^2}}\right), \\
    \dfrac{\partial{}}{\partial{t}}\dfrac{\partial{w}}{\partial{z}} + \dfrac{\partial{u}}{\partial{z}}\dfrac{\partial{w}}{\partial{x}} + u\dfrac{\partial^2{w}}{\partial{x}\partial{z}} + \dfrac{\partial{v}}{\partial{z}}\dfrac{\partial{w}}{\partial{y}} + v\dfrac{\partial^2{w}}{\partial{y}\partial{z}} + \left(\dfrac{\partial{w}}{\partial{z}}\right)^2 + \dfrac{\partial^2{w}}{\partial{z^2}} &= -\dfrac{1}{\rho}\dfrac{\partial^2{p}}{\partial{z^2}} + \nu \dfrac{\partial{}}{\partial{z}}\left(\dfrac{\partial^2{w}}{\partial{x^2}}+\dfrac{\partial^2{w}}{\partial{y^2}}+\dfrac{\partial^2{w}}{\partial{z^2}}\right)
\end{align*}
$$

Summing the three components gives the divergence. Using the incompressibility condition, we can factor out the divergence condition as a common factor:

$$
\begin{align*}
\dfrac{\partial{}}{\partial{t}}\left(\dfrac{\partial{u}}{\partial{x}} + \dfrac{\partial{v}}{\partial{y}} + \dfrac{\partial{w}}{\partial{z}}\right) + \left[\left(\dfrac{\partial{u}}{\partial{x}}\right)^2 + \left(\dfrac{\partial{v}}{\partial{y}}\right)^2 + \left(\dfrac{\partial{w}}{\partial{z}}\right)^2 + 2\dfrac{\partial{v}}{\partial{x}}\dfrac{\partial{u}}{\partial{y}} + 2\dfrac{\partial{v}}{\partial{x}}\dfrac{\partial{u}}{\partial{z}} + 2\dfrac{\partial{w}}{\partial{y}}\dfrac{\partial{v}}{\partial{z}}\right] + \\
u\dfrac{\partial{}}{\partial{x}}\left(\dfrac{\partial{u}}{\partial{x}} + \dfrac{\partial{v}}{\partial{y}} + \dfrac{\partial{w}}{\partial{z}}\right) + v\dfrac{\partial{}}{\partial{y}}\left(\dfrac{\partial{u}}{\partial{x}} + \dfrac{\partial{v}}{\partial{y}} + \dfrac{\partial{w}}{\partial{z}}\right) + w\dfrac{\partial{}}{\partial{z}}\left(\dfrac{\partial{u}}{\partial{x}} + \dfrac{\partial{v}}{\partial{y}} + \dfrac{\partial{w}}{\partial{z}}\right) = \\
-\dfrac{1}{\rho}\left(\dfrac{\partial^2{p}}{\partial{x^2}} + \dfrac{\partial^2{p}}{\partial{y^2}} + \dfrac{\partial^2{p}}{\partial{z^2}}\right) + \nu \left[\dfrac{\partial^2{}}{\partial{x^2}}\left(\dfrac{\partial{u}}{\partial{x}} + \dfrac{\partial{v}}{\partial{y}} + \dfrac{\partial{w}}{\partial{z}}\right) + \dfrac{\partial^2{}}{\partial{y^2}}\left(\dfrac{\partial{u}}{\partial{x}} + \dfrac{\partial{v}}{\partial{y}} + \dfrac{\partial{w}}{\partial{z}}\right) + \dfrac{\partial^2{}}{\partial{z^2}}\left(\dfrac{\partial{u}}{\partial{x}} + \dfrac{\partial{v}}{\partial{y}} + \dfrac{\partial{w}}{\partial{z}}\right)\right]
\end{align*}
$$

Substituting the incompressibility condition, we get:

$$
-\dfrac{1}{\rho}\left(\dfrac{\partial^2{p}}{\partial{x^2}} + \dfrac{\partial^2{p}}{\partial{y^2}} + \dfrac{\partial^2{p}}{\partial{z^2}}\right) = \left(\dfrac{\partial{u}}{\partial{x}}\right)^2 + \left(\dfrac{\partial{v}}{\partial{y}}\right)^2 + \left(\dfrac{\partial{w}}{\partial{z}}\right)^2 + 2\dfrac{\partial{v}}{\partial{x}}\dfrac{\partial{u}}{\partial{y}} + 2\dfrac{\partial{w}}{\partial{x}}\dfrac{\partial{u}}{\partial{z}} + 2\dfrac{\partial{w}}{\partial{y}}\dfrac{\partial{v}}{\partial{z}}
$$

Similarly in 2d case:

$$
-{1 \over \rho} \left( {{\partial^2 p} \over {\partial x^2}} + {{\partial^2 p} \over {\partial y^2}} \right) =
 \left( {{\partial u} \over {\partial x}} \right)^2 +
2 {{\partial u} \over {\partial y}} {{\partial v} \over {\partial x}}+
\left( {{\partial v} \over {\partial y}} \right)^2
$$

That is pressure poisson equation.

Taking two dimensions as an example, the pressure Poisson equation is discretized

$$
\begin{split}
& \frac{p_{i+1,j}^{n}-2p_{i,j}^{n+1}+p_{i-1,j}^{n}}{\Delta x^2} + \frac{p_{i,j+1}^{n}-2p_{i,j}^{n+1}+p_{i,j-1}^{n}}{\Delta y^2} = \\
& \qquad \rho\left[\frac{1}{\Delta t}\left(\frac{u_{i+1,j}-u_{i-1,j}}{2\Delta x}+\frac{v_{i,j+1}-v_{i,j-1}}{2\Delta y}\right) - \frac{u_{i+1,j}-u_{i-1,j}}{2\Delta x}\frac{u_{i+1,j}-u_{i-1,j}}{2\Delta x} - 2\frac{u_{i,j+1}-u_{i,j-1}}{2\Delta y}\frac{v_{i+1,j}-v_{i-1,j}}{2\Delta x} - \frac{v_{i,j+1}-v_{i,j-1}}{2\Delta y}\frac{v_{i,j+1}-v_{i,j-1}}{2\Delta y}\right]
\end{split}
$$

Arranged:

$$
\begin{split}
p_{i,j}^{n} = & \frac{\left(p_{i+1,j}^{n}+p_{i-1,j}^{n}\right) \Delta y^2 + \left(p_{i,j+1}^{n}+p_{i,j-1}^{n}\right) \Delta x^2}{2(\Delta x^2+\Delta y^2)} \\
& -\frac{\rho\Delta x^2\Delta y^2}{2\left(\Delta x^2+\Delta y^2\right)} \\
& \times \left[\frac{1}{\Delta t} \left(\frac{u_{i+1,j}-u_{i-1,j}}{2\Delta x} + \frac{v_{i,j+1}-v_{i,j-1}}{2\Delta y}\right) - \frac{u_{i+1,j}-u_{i-1,j}}{2\Delta x}\frac{u_{i+1,j}-u_{i-1,j}}{2\Delta x} - 2\frac{u_{i,j+1}-u_{i,j-1}}{2\Delta y}\frac{v_{i+1,j}-v_{i-1,j}}{2\Delta x} - \frac{v_{i,j+1}-v_{i,j-1}}{2\Delta y}\frac{v_{i,j+1}-v_{i,j-1}}{2\Delta y}\right]
\end{split}
$$

Because it generally chooses explicit discrete pressure, it directly uses this discrete format to iterate the pressure without solving a set of linear algebra equations.

### Find the divergence of Helmholtz decomposition

Helmholtz decomposition: In fluid mechanics, a twice differentiable three-dimensional flow field (velocity field) can be decomposed into the sum of an irrotational velocity field and a divergence-free velocity field

Since the curl of the gradient is 0, it can be written as

$$
\mathbf{w} = \mathbf{u} + \nabla p
$$

Where $\mathbf{w}$ is the original flow field, $\mathbf{u}$ is the desired divergence-free velocity field

Calculate the divergence of the left and right sides of the above equation, and we get

$$
\nabla \cdot \mathbf{w} = \nabla \cdot (\mathbf{u} + \nabla p) = \nabla \cdot \nabla p = \nabla^2 p
$$

This is also the pressure Poisson equation

Discretized as

$$
\frac{u_{i+1,j}-u_{i-1,j}}{2\Delta x} + \frac{v_{i,j+1}-v_{i,j-1}}{2\Delta y} = \frac{p_{i+1,j}^{n}-2p_{i,j}^{n}+p_{i-1,j}^{n}}{\Delta x^2} + \frac{p_{i,j+1}^{n}-2p_{i,j}^{n}+p_{i,j-1}^{n}}{\Delta y^2}
$$

Generally, implicit discretization of pressure is chosen, so it is necessary to solve a set of linear algebra equations, such as using Jacobi iteration. After solving $p$, the pressure gradient is applied to the original flow field to obtain the divergence-free velocity field

$$
\mathbf{u} = \mathbf{w} - \nabla p
$$

<script src="https://utteranc.es/client.js"
        repo="CheapMeow/cheapmeow.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>