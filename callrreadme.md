# callr

This is an experimental project to support calling R from a D program. It's experimental because:

- I plan to make changes.
- The parts worth keeping will make it back into [embedrv2](https://github.com/bachmeil/embedrv2)

Much of this functionality has been publicly available for many years. I want to clean things up, and I might have to break a few things in the process. Some of the names reflect this (e.g., `RArrayInsideR`).

# Roadmap (December 2022)

1. Get basic things to work.
2. Introduce a struct `RData` that uses reference counting to let R know when it can delete variables that it's allocated.
3. Support working with four types of data allocated by R from D programs.
4. Provide convenient wrappers for `lm` output.

Once this is done, I'll have a better idea of how to proceed further.

# Example

I'm running Ubuntu 22.04. I have a program `test.d`:

```
import std.stdio;
import embedr.r;

void main() {
  startR();
  alias rvec = RData!RVector;
  alias rlist = RData!RList;
  auto x = rvec("rnorm(12)");
  writeln(x[0]);
  writeln(x[11]);
  printR(x.name ~ "[1]");
  printR(x.name ~ "[12]");
  evalRQ([`set.seed(200)`, `y <- rnorm(100)`, `x <- rnorm(100)`]);
  auto fit = rlist(`lm(y ~ x)`);
  printR(fit.data);
  writeln(fit.names);
  writeln(RVector(fit["coefficients"])[0..2]);
  closeR();
}
```

It's compiled with the command

```
ldmd2 test.d r.d lm.d -version=standalone -L/usr/lib/R/library/RInside/lib/libRInside.so -L/usr/lib/libR.so
```

Explanations of some lines:

```
import embedr.r;
```

This is the functionality needed to interoperate with R.

```
startR();
```

This initializes the R interpreter. If you don't call this, you'll see lots of stuff is `null` and get lots of segfaults. Always check that you have this line in your program if you can't figure out why you're getting strange results. Note that your program **will** compile and run without it. You just won't be getting usable output.

```
alias rvec = RData!RVector;
alias rlist = RData!RList;
```

Useful aliases so I don't have to type out those ugly templated type names. The `RData` struct is what makes all this work. It assigns a unique name to the R variable you want to reference from your D program. Since R will run out of memory if all variables are held forever, `RData` uses reference counting to tell R to `rm` the underlying variable when your program is done with it. By itself, the `RData` struct doesn't provide the best user experience, because it's just a pointer. The user of this library will want additional functionality. For instance, if you have a vector, you will at a minimum want to read the elements. That is why you have to specify the type.

```
auto x = rvec("rnorm(12)");
```

Run the code `rnorm(12)` inside R. Capture the result. A reference to the result, and the type information, is held inside `x`.

```
writeln(x[0]);
writeln(x[11]);
```

You can access the elements of `x` as a vector. Note that there is bounds checking on the D side, so `x[-4]` and `x[14]` would both fail. Since D indexing starts at 0, and `x` is a D variable, indexing starts at 0.

```
printR(x.name ~ "[1]");
printR(x.name ~ "[12]");
```

`printR` runs code inside R and prints the output to the screen. `x.name` is the unique name given to the variable inside R. The index values are 1 and 12, because this code is run inside R, and R indexing starts at 1.

```
evalRQ([`set.seed(200)`, `y <- rnorm(100)`, `x <- rnorm(100)`]);
```

`evalRQ` executes a string of code or sequentially executes an array of strings of code inside R. I set the seed for replicability and generate values of `y` and `x`.

```
auto fit = rlist(`lm(y ~ x)`);
```

Regress the generated `y` on `x`. Capture the result inside the D variable `fit`.

```
printR(fit.data);
```

Have R print the output to the screen.

```
writeln(fit.names);
```

The `RList` struct has element `names` that holds the names of all elements of the list. We see that it has the following elements:

```
["coefficients", "residuals", "effects", "rank", 
"fitted.values", "assign", "qr", "df.residual", 
"xlevels", "call", "terms", "model"]
```

```
writeln(RVector(fit["coefficients"])[0..2]);
```

`fit["coefficients"]` returns a pointer to the `coefficients` element of `fit`. If you want to work with it, you'll need to specify the type information (since R is a dynamic language). This prints coefficients 1 and 2.

```
closeR();
```

Do any cleanup.

# Speed

Won't this be terribly slow? 

The first answer is that you're doing this for the functionality it provides, not for speed. This probably won't be as fast as an optimized pure D program. It opens the door to the type of interactive data analysis you do in R. The nice features of D, including static types, will make that process run smoother.

The second answer is that there's not a lot going on. You're creating some structs and passing pointers between R and D. That's pretty cheap in the grand scheme of things. Now, if you're doing this on the inside of a loop that's running 10 billion times, you'll definitely pay a price. For the typical kinds of things you'll need this for, such as using an R package to do a long-running Bayesian simulation, the overhead will be so small that you can act as if it's zero. So while you can find cases where speed is a problem, it's probably not going to be your bottleneck. Keep in mind that R can sometimes be slow *if the program is written exclusively in R*. Many R packages are nothing but convenient interfaces for C, C++, and Fortran libraries. You spend a few milliseconds running a few characters of R code, then everything is passed off to a fast library.

# Design

This might change in the future; it describes the current (December 2022) design.

The current design involves four "low level" types that do the work to interoperate with R. The hope is that they will be easy to use, but that specialized higher-level wrappers will be written to provide commonly-used functionality. For example, OLS is a popular tool for estimating linear forecasting models. The low level types are `RData!RVector` and `RData!RMatrix` to hold the data and the output. A high level wrapper will contain RVector and RMatrix elements that can then be accessed to get the coefficients, residuals, degrees of freedom, etc.

The four low level types are:

- `RData!RVector`
- `RData!RMatrix`
- `RData!RList`
- `RData!RArray`

Others, such as S3 and S4 classes, can easily be added in the future.












