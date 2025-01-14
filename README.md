Coverage.jl
===========

[![Build Status](https://github.com/JuliaCI/Coverage.jl/workflows/CI/badge.svg)](https://github.com/JuliaCI/Coverage.jl/actions/workflows/CI.yml?query=branch%3Amaster)
[![coveralls](https://coveralls.io/repos/github/JuliaCI/Coverage.jl/badge.svg?branch=master)](https://coveralls.io/github/JuliaCI/Coverage.jl?branch=master)
[![codecov](https://codecov.io/gh/JuliaCI/Coverage.jl/branch/master/graph/badge.svg?label=codecov)](https://codecov.io/gh/JuliaCI/Coverage.jl)

**"Take Julia code coverage and memory allocation results, do useful things with them"**

**Code coverage**: Julia can track how many times, if any, each line of your code is run. This is useful for measuring how much of your code base your tests actually test, and can reveal the parts of your code that are not tested and might be hiding a bug. You can use Coverage.jl to summarize the results of this tracking, or to send them to a service like [Coveralls.io](https://coveralls.io) or [Codecov.io](https://codecov.io/github/JuliaCI).

**Memory allocation**: Julia can track how much memory is allocated by each line of your code. This can reveal problems like type instability, or operations that you might have thought were cheap (in terms of memory allocated) but aren't (i.e. accidental copying).

## Comparison of coverage packages

- **[Coverage.jl](https://github.com/JuliaCI/Coverage.jl) (this package): allows you to take coverage results and submit them to online web services such as Codecov.io and Coveralls.io**
- [CoverageTools.jl](https://github.com/JuliaCI/CoverageTools.jl): core functionality for processing code coverage and memory allocation results

Most users will want to use [Coverage.jl](https://github.com/JuliaCI/Coverage.jl).

## Working locally

### Code coverage

*Step 1: collect coverage data.* If you are using your default test suite, you can collect coverage data with `Pkg.test("MyPkg"; coverage=true)`. Alternatively, you can collect coverage data manually: in the terminal, navigate to whatever directory you want to start from as the working directory, and run julia with the `--code-coverage` option:

```sh
julia --code-coverage=user
```
or more comprehensively (if you're interested in getting coverage for Julia's standard libraries)
```sh
julia --code-coverage=tracefile-%p.info --code-coverage=user  # available in Julia v1.1+
```
You can add other options (e.g., `--project`) as needed. After the REPL starts, execute whatever commands you wish, and then quit Julia. Coverage data are written to files when Julia exits.

*Step 2: collect summary statistics (optional).* Navigate to the top-level directory of your package, restart Julia (with no special flags) and analyze your code coverage:

```julia
using Coverage
# process '*.cov' files
coverage = process_folder() # defaults to src/; alternatively, supply the folder name as argument
coverage = append!(coverage, process_folder("deps"))  # useful if you want to analyze more than just src/
# process '*.info' files, if you collected them
coverage = merge_coverage_counts(coverage, filter!(
    let prefixes = (joinpath(pwd(), "src", ""),
                    joinpath(pwd(), "deps", ""))
        c -> any(p -> startswith(c.filename, p), prefixes)
    end,
    LCOV.readfolder("test")))
# Get total coverage for all Julia files
covered_lines, total_lines = get_summary(coverage)
# Or process a single file
@show get_summary(process_file(joinpath("src", "MyPkg.jl")))
```
The fraction of total coverage is equal to `covered_lines/total_lines`.

*Step 3: identify uncovered lines (optional).* To discover which functions lack testing, browse through the `*.cov` files in your `src/`
directory and look for lines starting with `-` or `0`, which mark lines that were never executed.
Numbers larger than 0 are counts of the number of times the respective line was executed.
Note that blank lines, comments, lines with `end` statements, etc. are marked with `-` but do not count against your coverage.

Be aware of a few limitations:

- a line that can take one of two branches gets marked as covered even if only one branch is tested
- currently, code run by Julia's internal interpreter [is not marked as covered](https://github.com/JuliaLang/julia/issues/37059).

### Exclude specific lines or sections from coverage

To exclude specific code blocks, surround the section with `COV_EXCL_START` and `COV_EXCL_STOP` comments:
```julia
# COV_EXCL_START
foo() = nothing
# COV_EXCL_STOP
```

To exclude a single line, add a comment with `COV_EXCL_LINE`:
```julia
const a = 1  # COV_EXCL_LINE
```

### Memory allocation

Start julia with
```sh
julia --track-allocation=user
```
Then:
- Run whatever commands you wish to test. This first run is to ensure that everything is compiled (because compilation allocates memory).
- Call `Profile.clear_malloc_data()`
- Run your commands again
- Quit julia

Finally, navigate to the directory holding your source code. Start julia (without command-line flags), and analyze the results using
```julia
using Coverage
analyze_malloc(dirnames)  # could be "." for the current directory, or "src", etc.
```
This will return a vector of `MallocInfo` objects, specifying the number of bytes allocated, the file name, and the line number.
These are sorted in increasing order of allocation size.

### LCOV export

There are many tools to work with LCOV info-format files as generated by the `geninfo` tool. Coverage.jl can generate these files:

```julia
coverage = process_folder()
LCOV.writefile("coverage-lcov.info", coverage)
```

### Cleaning up .cov files

When using Coverage.jl locally, over time a lot of `.cov` files can accumulate. Coverage.jl provides the `clean_folder` and `clean_file` methods to either clean up all `.cov` files in a directory (and subdirectories) or only clean the `.cov` files associated with a specific source file.

## Tracking Coverage with [Codecov.io](https://codecov.io)

[Codecov.io](https://codecov.io) is a test coverage tracking tool that integrates with your continuous integration servers (e.g. [TravisCI](https://travis-ci.com/)) or with HTTP POSTs from your very own computer at home.

1. Enable [Codecov.io](https://codecov.io) for your repository.
   - If it is public on GitHub and you are using using Travis, CircleCI or
     Appveyor, this is all you need to do. You can sign into Codecov using your
     Github identity.
   - Otherwise you will need to define a `CODECOV_TOKEN` environment variable
     with the Repository Upload Token available under the Codecov settings.

2. Use the command line option when you run your tests:
   - Either with something like `julia --code-coverage test/runtests.jl`, or
   - with something like  `julia -e 'Pkg.test("MyPkg", coverage=true)'`

3. Configure your CI service to upload coverage data:

   - If you are using Travis with `language: julia`, simply add `codecov: true`
     to your `.travis.yml`.

   - You can also add the following to the end of your `.travis.yml`. This
     line downloads this package, collects the per-file coverage data, then
     bundles it up and submits to Codecov. Coverage.jl assumes that the
     working directory is the package directory, so it changes to that first
     (so don't forget to replace `MyPkg` with your package's name!

   - On Travis CI:

       ```yml
       after_success:
       - julia -e 'using Pkg; Pkg.add("Coverage"); using Coverage; Codecov.submit(process_folder())'
       ```

   - On AppVeyor:

       ```yml
       after_test:
       - C:\projects\julia\bin\julia -e "using Pkg; Pkg.add(\"Coverage\"); using Coverage; Codecov.submit(process_folder())"
       ```

   - If you're running coverage on your own machine and want to upload results
     to Codecov, make a bash script like the following:

       ```bash
       #!/bin/bash
       CODECOV_TOKEN=$YOUR_TOKEN_HERE julia -e 'using Pkg; using Coverage; Codecov.submit_local(process_folder())'
       ```

## Tracking Coverage with [Coveralls.io](https://coveralls.io)

[Coveralls.io](https://coveralls.io) is a test coverage tracking tool that integrates with your continuous integration solution (e.g. [TravisCI](https://travis-ci.com/)).

1. Enable [Coveralls.io](https://coveralls.io) for your repository. If it is
   public on GitHub and you are using TravisCI, this is all you need to do. If
   you are using AppVeyor, you need to add a secure environment variable
   called `COVERALLS_TOKEN` to your `.appveyor.yml` (see
   [here](https://www.appveyor.com/docs/build-configuration/#secure-variables)).
   Your repo token can be found in your Coveralls repo settings. If neither of
   these are true, please submit an issue, and we can work on adding
   additional functionality for your use case.

2. Activate the `--code-coverage` command line option when you run your tests
   - Either with something like `julia --code-coverage test/runtests.jl`, or
   - with something like  `julia -e 'Pkg.test("MyPkg", coverage=true)'`

3. Configure your CI service to upload coverage data:

   - If you are using Travis with `language: julia`, simply add `coveralls: true`
     to your `.travis.yml`.

   - You can also add the following to the end of your `.travis.yml`. This
     line downloads this package, collects the per-file coverage data, then
     bundles it up and submits to Coveralls. Coverage.jl assumes that the
     working directory is the package directory, so it changes to that first
     (so don't forget to replace `MyPkg` with your package's name!

   - On Travis CI:

       ```yml
       after_success:
       - julia -e 'using Pkg; Pkg.add("Coverage"); using Coverage; Coveralls.submit(process_folder())'
       ```

   - On AppVeyor:

       ```yml
       after_test:
       - C:\julia\bin\julia -e "using Pkg; Pkg.add(\"Coverage\"); using Coverage; Coveralls.submit(process_folder())"
       ```

## A note for advanced users

Coverage tracking in Julia is not yet quite perfect. One problem is that (at
least in certain scenarios), the coverage data emitted by Julia does not mark
functions which are never called (and thus are not covered) as code. Thus,
they end up being treated like comments, and are *not* counted as uncovered
code, even though they clearly are. This can arbitrarily inflate coverage
scores, and in the extreme case could even result in a project showing 100%
coverage even though it contains not a single test.

To overcome this, Coverage.jl applies a workaround which ensures that all
lines of code in all functions of your project are properly marked as "this is
code". This resolves the problem of over reporting coverage.

Unfortunately, this workaround itself can have negative consequences, and lead
to under reporting coverage, for the following reason: when Julia compiles
code with inlining and optimizations, it can happen that some lines of Julia
code do not correspond to any generated machine code; in that case, Julia's
code coverage tracking will never mark these lines as executed, and also won't
mark them as code. One may now argue whether this is a bug in itself or not,
but that's how it is, and normally would be fine -- except that our workaround
now does mark these lines as code, but code which now never has a chance as
being marked as executed.

We may be able to improve our workaround to deal with this better in the
future (see also <https://github.com/JuliaCI/Coverage.jl/pull/188>), but this
has not yet been done and it is unclear whether it will take care of all
instances. Even better would be if Julia improved the coverage information it
produces to be on par with what e.g. C compilers like GCC and clang produce.
Since it is unclear when or if any of these will happen, we have added an
expert option which allows Julia module owners to disable our workaround code,
by setting the environment variable `DISABLE_AMEND_COVERAGE_FROM_SRC` to
`yes`.

For Travis, this can be achieved by adding the following to `.travis.yml`:

    env:
      global:
        - DISABLE_AMEND_COVERAGE_FROM_SRC=yes

For AppVeyor, add this to `.appveyor.yml`:

    environment:
      DISABLE_AMEND_COVERAGE_FROM_SRC: yes

## Some Julia packages using Coverage.jl

*Pull requests to add your package welcome (or open an issue)*

* [ArgParse.jl](https://github.com/carlobaldassi/ArgParse.jl)
* [AstroLib.jl](https://github.com/giordano/AstroLib.jl)
* [AudioIO.jl](https://github.com/ssfrr/AudioIO.jl)
* [Augur.jl](https://github.com/AugurProject/Augur.jl)
* [Bootstrap.jl](https://github.com/julian-gehring/Bootstrap.jl)
* [CAIRS.jl](https://github.com/scheidan/CAIRS.jl)
* [ClimateTools.jl](https://github.com/Balinus/ClimateTools.jl)
* [DASSL.jl](https://github.com/pwl/DASSL.jl)
* [DataFrames.jl](https://github.com/JuliaStats/DataFrames.jl)
* [Decimals.jl](https://github.com/tensorjack/Decimals.jl)
* [Distributions.jl](https://github.com/JuliaStats/Distributions.jl)
* [DSP.jl](https://github.com/JuliaDSP/DSP.jl)
* [ExtractMacro.jl](https://github.com/carlobaldassi/ExtractMacro.jl)
* [FastaIO.jl](https://github.com/carlobaldassi/FastaIO.jl)
* [FiniteStateMachine.jl](https://github.com/tensorjack/FiniteStateMachine.jl)
* [FourierFlows.jl](https://github.com/FourierFlows/FourierFlows.jl)
* [Gadfly.jl](https://github.com/dcjones/Gadfly.jl)
* [GeometricalPredicates.jl](https://github.com/skariel/GeometricalPredicates.jl)
* [Glob.jl](https://github.com/vtjnash/Glob.jl)
* [GradientBoost.jl](https://github.com/svs14/GradientBoost.jl)
* [GraphCentrality.jl](https://github.com/sbromberger/GraphCentrality.jl)
* [GraphLayout.jl](https://github.com/IainNZ/GraphLayout.jl)
* [Homebrew.jl](https://github.com/JuliaLang/Homebrew.jl)
* [HttpParser.jl](https://github.com/JuliaLang/HttpParser.jl)
* [IntervalTrees.jl](https://github.com/BioJulia/IntervalTrees.jl)
* [IPNets.jl](https://github.com/sbromberger/IPNets.jl)
* [JointMoments.jl](https://github.com/tensorjack/JointMoments.jl)
* [JuMP.jl](https://github.com/JuliaOpt/JuMP.jl)
* [LibGit2.jl](https://github.com/jakebolewski/LibGit2.jl)
* [LightGraphs.jl](https://github.com/JuliaGraphs/LightGraphs.jl)
* [LinearExpressions.jl](https://github.com/cdsousa/LinearExpressions.jl)
* [Orchestra.jl](https://github.com/svs14/Orchestra.jl)
* [ODE.jl](https://github.com/JuliaLang/ODE.jl)
* [OnlineStats.jl](https://github.com/joshday/OnlineStats.jl)
* [OpenCL.jl](https://github.com/JuliaGPU/OpenCL.jl)
* [OpenStreetMap.jl](https://github.com/tedsteiner/OpenStreetMap.jl)
* [PValueAdjust.jl](https://github.com/dirkschumacher/PValueAdjust.jl)
* [QuantEcon.jl](https://github.com/spencerlyon2/QuantEcon.jl)
* [QuantileRegression.jl](https://github.com/vincentarelbundock/QuantileRegression.jl)
* [RationalSimplex.jl](https://github.com/IainNZ/RationalSimplex.jl)
* [RDF.jl](https://github.com/joejimbo/RDF.jl)
* [Requests.jl](https://github.com/loladiro/Requests.jl)
* [Restful.jl](https://github.com/ylxdzsw/Restful.jl)
* [Robotics.jl](https://github.com/cdsousa/Robotics.jl)
* [RouletteWheels.jl](https://github.com/jbn/RouletteWheels.jl)
* [SASLib.jl](https://github.com/tk3369/SASLib.jl)
* [SimJulia.jl](https://github.com/BenLauwens/SimJulia.jl)
* [SIUnits.jl](https://github.com/loladiro/SIUnits.jl)
* [StatsBase.jl](https://github.com/JuliaStats/StatsBase.jl)
* [TaylorIntegration.jl](https://github.com/PerezHz/TaylorIntegration.jl)
* [TaylorSeries.jl](https://github.com/JuliaDiff/TaylorSeries.jl)
* [TextWrap.jl](https://github.com/carlobaldassi/TextWrap.jl)
* [Tidier.jl](https://github.com/TidierOrg/Tidier.jl)
* [TimeData.jl](https://github.com/cgroll/TimeData.jl)
* [TypeCheck.jl](https://github.com/astrieanna/TypeCheck.jl)
* [Unitful.jl](https://github.com/ajkeller34/Unitful.jl)
* [URIParser.jl](https://github.com/loladiro/URIParser.jl)
* [URITemplate.jl](https://github.com/loladiro/URITemplate.jl)
* [Voting.jl](https://github.com/tchajed/Voting.jl)
* [WAV.jl](https://github.com/dancasimiro/WAV.jl)
* [Weave.jl](https://github.com/mpastell/Weave.jl)
* [WeightedStats.jl](https://github.com/tensorjack/WeightedStats.jl)
* [YAML.jl](https://github.com/dcjones/YAML.jl)
