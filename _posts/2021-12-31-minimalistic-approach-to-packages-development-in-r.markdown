The package concept is a basic one to ensure reproducibility of your R code. The R packaging philosophy makes wonder in dependence management and code documentation culture. Actually, that is one of the main reasons why R is great for a field data analysis. The downside of this approach is that things can easily go over-complicated.

![creativity_v0](/assets/creativity_v0.JPG){:class="img-responsive"}

There are a lot of brilliant tutorials explaining how to develop an R package. But usually they suppose that a package developer should rely on some additionally libraries from the very beginning. That may be perfectly reasonable but my personal preference is to start as simple as I can. Implementation of my KISS approach towards R package development is presented bellow. Apart of creation, building and installing the package, we will also look on a way to remove its' testing versions from your machine. From my experience, that feels very encouraging when experimenting if you know in advance how to clean!

## How to create an R package

Let's create a package calculating the barometric pressure distribution. That is a dependence of the atmospheric air pressure on the altitude and air temperature. It's exponential reflecting a simple fact that the air is quickly getting thinner when you are climbing.

A function to assess this effect is the barometric distribution which may be defined as follows:

```R
p_barometric <- function(T, h) {
	
	M <- 29.0e-3 # [kg/mol]
	g <- 9.81    # [m/s^2]
	R <- 8.31    # [J/(mol * K)]
	p0 <- 1.01e5 # [Pa]

	p <- p0 * exp(-((M * g * h)/(R * T)))
}
```

Let's trace the whole workflow needed to put this function into an R package. The way is actually quite short. 

First, create a package from within an R session

```R
package.skeleton(name = "./barometricR", list = c("p_barometric"))
```

The `package.skeleton()` function creates on your disc a minimal file structure to make you package work. There are namely the `"man"` and the `"R"` directories with documentation and R source files, respectively. Besides, the files `"DESCRIPTION"`, `"NAMESPACE"` and, a very nice point, `"Read-and-delete-me"` are being created.


Then you need to use the terminal and call the following bunch of commands that will fulfill all the magic:

```bash
R CMD build barometricR
R CMD check barometricR
R CMD install barometricR
```

Finally, you can return to the R session and load a freshly created local library

```R
library(barometricR)
```

Now let's look on each step in more details.

### Create

First of all, a certain file structure should be created to build and install the package. Theoretically, that is perfectly possibly by hand but the R build-in `package.skeleton()` function automates this process quite conveniently. To use it you have to load the functions intended for packaging into R session. Apart of the commonly used `ls()`, there is the `lsf.str()` function to check which functions you have in the R environment `lsf.str()` and list their arguments, like this

```R
> p_barometric : function (T, h)
```

### Build

The `R CMD` `build` tool should be run in the folder containing the package folder not inside the package folder itself. That is not quite obvious but very convenient to build a number of packages from a single location. This command builds a package zip-archive.

Apart of the package building, the `R CMD build` command run checking of the `.Rd` documentation files. Which means that package in the R-world  should documented *before* the package itself will even built. In my opinion, that is exactly what makes R an excellent data science tool as it gives a proper info page each package function. 

The R info pages use LaTeX-like syntax which in our case may look for example like this:

```LaTeX
\name{p_barometric}
\alias{p_barometric}
\title{
Calculate the barometric distribution
}
\description{
  The function calculates how the ambient pressure changes with increasing the altitude.
}
\usage{
p_barometric(T, h)
}
\arguments{
  \item{T}{
   The ambient air temperature, K
}
  \item{h}{
   The height above the sea level, m
}
}
\value{
  The pressure value, Pa.
}
\examples{
p_barometric(T = 300, h = 1000)
}
```

By the way, the title of .Rd files can't be empty while everything beyond is... well, desirable. Not a proper way, of course, but a viable quick-and-dirty approach to test your package quickly.

As you can guess from our quite a basic case, automation of R code documentation is a life-saving thing. In one of the next posts we'll look on `roxygene` which is the *standard de facto* of R packages documenting.


### Check

Apart of checking, `R CMD check` builds pdf documentation files. The package checking results are avilable in the `*.log` and `*.out` files. The checking can be done on a specified package archive version or on a package which has not been built yet.

### Install

Installation of the R package inside your system is quite straightforward with `R CMD install  barometricR` command. 

### Use

The usual `library()` or `require()` functions are needed to load the created package into an R session. In the course of package development, reloading of the package itself or its' functions is essential. The package can be unloaded with `detach()` using an `unload` argument:

`detach("package:barometricR", unload=TRUE)`

Useful commands to check the results of loads/unloads on are `sessionInfo()` to check the big picture and `(.packages())` for more specific information on the loaded packages.

Quite a typical issue during a package-testing R session looks like:

```R
> The following object is masked _by_ '.GlobalEnv':
>    p_barometric
```

That means a conflict due two functions with the same name `p_barometric`. It can happen easiliy if the `p_barometric` was defined in the R session to be imported with a `package.skeleton()` call. The solution is simple
`rm(p_barometric)` or `rm(list = "p_barometric")`. The conflict can also happen if functions with the same name are defined by different loaded packages. There is a great [wiki-post](https://stackoverflow.com/a/39137111/8465924) of the StackOverflow on that issue. 


## How to remove an R package

Probably, that is the most important point ever when experimenting with package building under your main operation system... Fortunately, the solution is rather simple. You can remove the installed R package either from inside an R session with `remove.packages("barometricR")` or from the command line with `R CMD REMOVE [options] [-l lib] pkgs`. Both commands remove the package from your operation system but leave untouched your local library as a zip archive and the package source files.

You may ensure that the package was in fact removed by looking on the folder where are R packages stored. Path to the folder is given by `.libPaths()` from inside an R session.

## Further reading

An introdactory to R packaging wouldn't be complete without mentioning the detailed [**Writing R Extensions**](https://cran.r-project.org/doc/manuals/r-release/R-exts.html) by R core team and the [**R Packages**](https://r-pkgs.org/) book by H.Wickham and J.Bryan. Both are detailed, quite extensive and must-reads for everyone wanting to dive into the R development world. However, you don't need still to be an expert to start. A good moment to remember...

![creativity](/assets/creativity_v2.JPG){:class="img-responsive"}

