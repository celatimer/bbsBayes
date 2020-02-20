# bbsBayes <img src="man/figures/logo.png" align="right" />

[![Build Status](https://travis-ci.org/BrandonEdwards/bbsBayes.svg?branch=master)](https://travis-ci.org/BrandonEdwards/bbsBayes)
[![CRAN\_Status\_Badge](http://www.r-pkg.org/badges/version/bbsBayes)](https://cran.r-project.org/package=bbsBayes)
[![lifecycle](https://img.shields.io/badge/lifecycle-maturing-blue.svg)](https://www.tidyverse.org/lifecycle/#maturing)

## Overview
bbsBayes is a package to perform hierarchical Bayesian analysis of North American Breeding Bird Survey (BBS) data. 'bbsBayes' will run a full model analysis for one or more species that you choose, or you can take more control and specify how the data should be stratified, prepared for JAGS, or modelled. 

<img src="man/figures/BARS_Continental_Trajectory.png" />
<img src="man/figures/BARS_trendmap.png" />


## Installation

We expect a CRAN release shortly, but for now you can install from Github using the following options:

Option 1: Most recent stable release (currently v2.0.0)
``` r
# To install v2.0.0 from Github:
install.packages("devtools")
library(devtools)
devtools::install_github("BrandonEdwards/bbsBayes", ref = "v2.0.0")
```

Option 2: Less-stable development version
``` r
# To install the development version from GitHub:
install.packages("devtools")
library(devtools)
devtools::install_github("BrandonEdwards/bbsBayes")
```

## Basic Status and Trend Analyses

bbsBayes provides functions for every stage of Breeding Bird Survey data analysis.

### Data Retrieval 
You can download BBS data by running `fetch_bbs_data`. This will save it to a
package-specific directory on your computer. You must agree to the terms and conditions of the data usage before downloading. You only need run this function once for each annual update of the BBS database.

``` r
fetch_bbs_data()
```

### Data Preparation
#### Stratification
Stratification plays an important role in trend analysis. Use the `stratify()` function for this job. Set the argument `by` to stratify by the following options:
* bbs_cws -- Political region X Bird Conservation region intersection (CWS method)
* bbs_usgs -- Political region X Bird Conservation region intersection (USGS method)
* bcr -- Bird Conservation Region only
* state -- Political Region only
* latlong -- Degree blocks (1 degree of latitude X 1 degree of longitude)

``` r
stratified_data <- stratify(by = "bbs_cws")
```

#### Jags Data
JAGS models require the data to be sent as a list depending on how the model is set up. `prepare_jags_data` subsets the stratified data based on species and wrangles relevent data to use for JAGS models.

``` r
jags_data <- prepare_jags_data(stratified_data, 
                               species_to_run = "Barn Swallow",
                                 min_max_route_years = 5,
                                 model = "gamye",
                                 heavy_tailed = T)
```
**Note:** This can take a very long time to run

### MCMC
Once the data has been prepared for JAGS, the model can be run. The following will run MCMC with default number of iterations. Note that this step usually takes a long time (e.g., 6-12 hours, or even days depending on the species, model). If multiple cores are available, the processing time is reduced with the argument `parallel = TRUE`.

``` r
mod <- run_model(jags_data = jags_data)
```

Alternatively, you can set how many iterations, burn-in steps, or adapt steps to use, and whether to run chains in parallel
``` r
jags_mod <- run_model(jags_data = jags_data,
                               n_saved_steps = 1000,
                               n_burnin = 10000,
                               n_chains = 3,
                               n_thin = 10,
                               parallel = FALSE,
                          parameters_to_save = c("n", "n3", "nu", "B.X", "beta.X", "strata", "sdbeta", "sdX"),
                          modules = NULL)

```
The `run_model` function generates a large list (object jagsUI) that includes the posterior draws, convergence information, data, etc.
It will send a warning if Gelman-Rubin Rhat cross-chain convergence criterion is > 1.1 for any of the monitored parameters. Re-running the model with a longer burn-in and/or more posterior iterations or greater thinning rates may improve convergence. The seriousness of these convergence failures is something the user must interpret for themselves. If all or the vast majority of the n parameters have converged (e.g., you're receiving this message for other monitored parameters), then inference on population trajectories and trends from the model are generally reliable. 
``` r
jags_mod$n.eff #shows the effective sample size for each monitored parameter
jags_mod$Rhat # shows the Rhat values for each monitored parameter
```
If important monitored parameters have not converged, we recommend inspecting the model diagnostics with the package ggmcmc. 
``` r 
install.packages("ggmcmc")
S <- ggmcmc::ggs(jags_mod$samples,family = "B.X") #samples object is an mcmc.list object
ggmcmc::ggmcmc(S,family = "B.X") ## this will output a pdf with a series of plots useful for assessing convergence. Be warned this function will be overwhelmed if trying to handle all of the n values from a BBS analysis of a broad-ranged species
```

### Model Predictions
There are a number of tools available to summarize and visualize the posterior predictions from the model. 
#### Annual Indices of Abundance and Population Trajectories
The main monitored parameters are the annual indices of relative abundance within a stratum (i.e., parameters "n[strata,year]"). The time-series of these annual indices form the estimated population trajectories.
``` r
indices <- generate_regional_indices(jags_mod)
```
By default, this function generates estimates for the continent (i.e., survey-wide) and for the individual strata. However, the user can also select summaries for composite regions (regions made up of collections of strata), such as countries, provinces/states, Bird Conservation Regions, etc.

``` r
indices <- generate_regional_indices(jags_mod = jags_mod,
                                     jags_data = jags_data,
                                     regions = c("continental",
                                     "national",
                                     "prov_state",
                                     "stratum"))
                                     #also "bcr", "bcr_by_country"
```

#### Population Trends
Population trends can be calculated from the series of annual indices of abundance. The trends are expressed as geometric mean rates of change (%/year) between two points in time. $Trend = (\frac {n[Minyear]}{n[Maxyear]})^{(1/(Maxyear-Minyear))}$
``` r
trends <- generate_regional_trends(indices = indices,
                                   Min_year = 1970,
                                   Max_year = 2018)
```
The `generate_regional_trends` function returns a dataframe with 1 row for each unit of the region-types requested in the `generate_regional_indices` function (i.e., 1 for each stratum, 1 continental, etc.). The dataframe has at least 27 columns that report useful information related to each trend, including the start and end year of the trend, lists of included strata, total number of routes, number of strata, mean observed counts, and estimates of the % change in the population between the start and end years. 

The `generate_regional_trends` function includes some other arguments that allow the user to adjust the quantiles used to summarize uncertainty (e.g., interquartile range of the trend estiamtes, or the 67% CIs), as well as include additional calculations, such as the probability a population has declined (or increased) by > X%. 
``` r
trends <- generate_regional_trends(indices = indices,
                                   Min_year = 1970,
                                   Max_year = 2018,
                                   prob_decrease = c(0,25,30,50),
                                   prob_increase = c(0,33,100))
```

### Visualizing Predictions

#### Population Trajectories
Generate plots of the population trajectories through time.This function produces a list of ggplot figures that can be combined into a single pdf file, or printed to individual devices.
``` r
tp = plot_indices(indices = indices,
                         species = "Barn Swallow",
                         add_observed_means = TRUE)
# pdf(file = "Barn Swallow Trajectories.pdf")
# print(tp)
# dev.off()
print(tp[[1]])
```
<img src="man/figures/BARS_Continental_Trajectory.png" />
`print(tp[[2]])`
<img src="man/figures/BARS_Canada_Trajectory.png" />
etc.

#### Trend Maps
The trends can be mapped to produce strata maps coloured by species population trends.
``` r
mp = generate_map(trends,
                  select = TRUE,
                  stratify_by = "bbs_cws",
                  species = "Barn Swallow")
print(mp)
```
<img src="man/figures/BARS_trendmap.png" />

#### Geofacet Trajectories

For stratifications that can be compiled by political regions (i.e., `bbs_cws`, `bbs_usgs`, or `state`), the function `geofacet_plot` will generate a ggplot object that plots the state and province level population trajectories in facets arranged in an approximately geographic arrangement. These plots offer a concise, range-wide summary of a species' population status and trends.

``` r
  gf <- geofacet_plot(indices_list = indices,
                     select = T,
                     stratify_by = "bbs_cws",
                     multiple = F,
                     trends = trends,
                     slope = F,
                     species = "Barn Swallow")
  
  #png("BARS_geofacet.png",width = 1500, height = 750,res = 150)
  print(gf)
  #dev.off()
  ```
<img src="man/figures/BARS_geofacet.png" />


There are numerous other functions available for analysis of the data.

### Advanced options and customized models

To be added soon...

#### Alternate extra-Poisson error distributions

#### Alternate Annual Indices

#### Alternate Measures of Trend

#### Custom regional summaries

#### Exporting the JAGS model

#### Modifying the JAGS model and data

#### Comparing Models

