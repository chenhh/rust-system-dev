# 線性規劃(linear programming)

## good\_lp crate

\[[web](https://crates.io/crates/good\_lp)]

## Farmer's problem

$$
\begin{array}{rl}
\min & 150 x_1 + 230 x_2 +260 x_3 + 238 y_1 - 170 w_1 + 210 y_2 - 150 w_2 - 36 w_3 - 10 w_4 \\
s.t. & x_1 + x_2 + x_3 \leq 500, \\
     & 2.5 x_1 + y_1 - w_1 \geq 200, \\
     & 3 x_2 + y_2 - w_2 \geq 240, \\
     & w_3 + w_4 \leq 20 x_3, \\
     & w_3 \leq 6000, \\
     & x_1, x_2, x_3, y_1, y_2, w_1, w_2, w_3, w_4 \geq 0
\end{array}
$$

## 參考資料

* [COIN-OR](https://www.coin-or.org/)
* [\[github\] Cbc](https://github.com/coin-or/Cbc)
* [\[github\] good\_lp](https://github.com/rust-or/good\_lp)
* [\[By Hans Mittelmann\] BENCHMARKS FOR OPTIMIZATION SOFTWARE](http://plato.asu.edu/bench.html)
* [\[GNU\] GSL - GNU Scientific Library](https://www.gnu.org/software/gsl/)
* [https://lib.rs/crates/varpro](https://lib.rs/crates/varpro)
