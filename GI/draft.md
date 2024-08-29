


$$
\begin{aligned}
\int_a^b f(x) \mathrm dx &= \lim_{n \rarr \infty} \frac{1}{n} \sum_{i=1}^{n} f(x_i)(b-a) \\
&= \lim_{n \rarr \infty} \frac{1}{n} \sum_{i=1}^{n} \frac{f(x_i)}{\textcolor{red}{\frac{1}{b-a}}}
\end{aligned}
$$

$$
\int f(x) \mathrm d x \sim F_n(x) = \frac{1}{n} \sum_{k=1}^{n} \frac{f(X_k)}{PDF(X_k)}
$$

$$
L_o(p, \omega_o) = \int_{\Omega_+}L_i(p, \omega_i)f_r(p, \omega_i, \omega_o) (n \cdot \omega_i) \ \mathrm d\omega_i
$$


$$
L_o(p, \omega_o) \approx \frac{1}{N} \sum_{i=1}^{N} \frac{L_i(p, \omega_i) f_r(p, \omega_i, \omega_o) (n \cdot \omega_i)}{p(\omega_i)}
$$


$$
\begin{aligned}
L_o(p, \omega_o) &= \int_{\Omega}f_r(p, \omega_i, \omega_o)L_i(p, \omega_i) \cos\theta \mathrm d\omega_i \\ 
&= \int_\Omega (k_d\frac{c}{\pi} + k_s \frac{DFG}{4\cos\theta_i \cos\theta_o})L_i(p, \omega_i)  \cos\theta_i \mathrm d\omega_i
\end{aligned}
$$


























