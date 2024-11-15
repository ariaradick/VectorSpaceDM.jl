# VSDM.jl

Vector Spaces for Dark Matter. This is the Julia implementation  of the wavelet-harmonic integration method, designed for the efficient calculation of dark matter direct detection scattering rates in anisotropic detectors, and for arbitrary dark matter velocity distributions. See the associated papers [[arXiv:2310.01483]](https://arxiv.org/abs/2310.01483) and [[arXiv:2310.01480]](https://arxiv.org/abs/2310.01480) for an in-depth discussion of the methods used in this package.

## Direct Detection Rates

For discrete final state energies $\Delta E$ the dark matter (DM) -- standard model (SM) scattering rate $R$ is given by

```math
R = N_T \frac{\rho_\chi}{m_\chi} \int d^3 q \: d^3 v \: g_\chi(\textbf{v}) \: f_s^2(\textbf{q}) \: \delta\left( \Delta E + \frac{q^2}{2 m_\chi} - \textbf{q} \cdot \textbf{v} \right) \frac{\bar{\sigma}_0 F_{\textrm{DM}}^2(\textbf{q})}{4 \pi \mu_\chi^2}
```

Our package factorizes this calculation by projecting $g_\chi(\textbf{v})$ and $f_s^2(\textbf{q})$ onto vector spaces such that

```math
f(\textbf{u}) \equiv |f\rangle = \sum_{n \ell m} \langle \phi_{n \ell m} | f \rangle | \phi_{n \ell m} \rangle
```

where the basis function $\phi_{n \ell m}$ is given by

```math
\phi_{n \ell m}(\textbf{u}) = r_n(u) Y_{\ell m}(\hat{u})
```

with $r_n(u)$ either a spherical tophat function or a spherical Haar wavelet and $Y_{\ell m}(\hat{u})$ the real spherical harmonics. We define the coefficients by

```math
\langle \phi_{n \ell m} | f \rangle \equiv f_{n \ell m} = \int \frac{d^3 u}{u_{\textrm{max}}^3} \: f(\textbf{u}) \: r_n(u) \: Y_{\ell m}(\hat{u})
```

where $u_\textrm{max}$ is the maximum value of $|\textbf{u}|$ supported by the radial basis functions. The rate can then be calculated by

```math
R = \frac{k_0}{T_\textrm{exp}} \sum_{\ell=0}^{\infty} \sum_{m,m'=-\ell}^\ell \sum_{n,n'=0}^{\infty} \langle v_{\textrm{max}}^3 g_\chi | n \ell m \rangle \mathcal{I}_{n,n'}^\ell \langle n' \ell m' | f_s^2 \rangle
```

where $k_0$ is a collection of constants, $T_\textrm{exp}$ is the exposure time, and $\mathcal{I}$ is referred to as the *kinematic scattering matrix*,

```math
\mathcal{I}_{n,n'}^\ell = \frac{q_{\textrm{max}}^3 / v_{\textrm{max}}^3}{2 m_\chi \mu_\chi^2} \int_0^\infty \frac{q dq}{q_{\textrm{max}}^2} r^{(q)}_{n'}(q) F_{\textrm{DM}}^2(q) \int_{v_\textrm{min}(q)}^\infty \frac{v dv}{v_{\textrm{max}}^2} P_\ell \left( \frac{v_{\textrm{min}}(q)}{v} \right) r^{(v)}_n(v)
```

and is analytically calculable for the tophat and Haar wavelet bases. Finally, we can also implement rotations of the detector (given generically by $\mathcal{R}$),

```math
R = \frac{k_0}{T_\textrm{exp}} \sum_{\ell=0}^{\infty} \sum_{m,m'=-\ell}^\ell \sum_{n,n'=0}^{\infty} \langle v_{\textrm{max}}^3 g_\chi | n \ell m \rangle \mathcal{I}_{n,n'}^\ell G^\ell_{m,m'}(\mathcal{R}) \langle n' \ell m' | f_s^2 \rangle
```

where $G$ is like the Wigner $D$-matrix, but for real spherical harmonics

```math
G_{m',m}^\ell(\mathcal{R}) \equiv \langle \ell m' | \mathcal{R} | \ell m \rangle
```

## Projecting $g_\chi$ and $f_s^2$

The first step in using this package to do a calculation is to project your velocity distribution $g_\chi(\textbf{v})$ and momentum form factor $f_s^2(\textbf{q})$ onto the vector space. First, decide your maximum $v$ and $q$ values. Make sure your $v$ and $q$ are in natural units. Then we can use the function `ProjectF`

```@meta
CurrentModule = VSDM
using Quaternionic
```

```
gx_nlm = ProjectF(gx, (n_max, ell_max), Wavelet(vmax))
fs2_nlm = ProjectF(fs2, (n_max, ell_max), Wavelet(qmax))
```

This will store your coefficients in a `ProjectedF` object, which has fields `fnlm` which stores the coefficient values, `lm` which stores the $(\ell,m)$ pairs for each column of `fnlm`, and `radial_basis` which stores the radial basis and with it, the maximum $v$ or $q$ value. You can save the coefficients with `writeFnlm`

```
writeFnlm("filename.csv", projected_f)
```

If you already have stored coefficients, you can load them with

```
readFnlm("filename.csv")
```

or

```
readFnlm("filename.csv", Wavelet(umax))
```

if you want to define the radial basis by hand. This is helpful because it can often take a while to calculate these coefficients, particularly for large `n_max` and `ell_max`.

## Calculating the Rate

To deal with rotations, we'll want to import the `Quaternionic.jl` package

```
using Quaternionic
```

you can see [their documentation](https://moble.github.io/Quaternionic.jl/stable/) to see how to convert from other rotation types to quaternions. We can define a quaternion by

```@setup abc
using Quaternionic
```

```@repl abc
q = quaternion(1.0, 2.0, 3.0, 4.0)
```

We also need to define the model, including the dark matter mass $m_\chi$, the dark matter form factor $F_{\textrm{DM}}$, and the transition energy $\Delta E$. To compactly store these values, we use a `ModelDMSM` object

```
dmsm_model = ModelDMSM(fdm_n, mX, mSM, deltaE)
```

Once you have a quaternion or rotor defining your desired rotation and have defined your model you can easily calculate the rate by

```
R = rate(q, dmsm_model, gx_nlm, fs2_nlm)
```

It's very quick to implement more rotations, so you can also pass a vector of quaternions (or rotors) to `rate`

```
R = rate(Q, dmsm_model, gx_nlm, fs2_nlm)
```