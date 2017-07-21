# Making Local ACS Profiles in R
Camille Seaberry, DataHaven  
July 4, 2017  




## Introducing `acs.R`

* `acs.R` package provides easier interface for working with Census API
* Focused on ACS, but also can access SF1, SF3 decennial tables
* Comes with its own weird objects---something to get used to
* Developed by Ezra Habel Glenn at MIT
* Very good user guide! http://bit.ly/acshandbook

See the repo for this presentation at https://github.com/CT-Data-Haven/acs_presentation


## Goal & what we're working on

**Goal:** make a profile of several indicators for local geographies in R

**Input:** the `acs.R` package & API key from the Census

**Output:** CSV file ready to share with clients, public, etc


## The plan

* Make combined geographies
* Pull several ACS tables using the `acs` package
* Aggregate variables as needed
* Calculate stuff---rates, etc
* Get everything into a single dataframe & write to csv


## Brushing up on R

This will be pretty technical---sorry in advance!

Some resources for practicing R:

* DataCamp: http://datacamp.com Costs money but is totally worth it for interactive courses
* Tutorials from RStudio: http://rstudio.com/online-learning
* R for Data Science free online book: http://r4ds.had.co.nz/
* swirl, package for R tutorials inside RStudio: http://swirlstats.com/


## #TeamTidyverse

* Making heavy use of the tidyverse packages
* `purrr` lets us use `map` functions to work with lists
* Learn more: 
    + http://tidyverse.org/
    + http://r4ds.had.co.nz/


```r
library(acs)
library(tidyverse)
library(stringr)
```

There's a very new (May 2017) package called `tidycensus` that I haven't worked with yet, but recommend people also check out: https://github.com/walkerke/tidycensus


## ACS helper functions

`acs` package comes with many helper functions:

* `geo.lookup` to help with looking up geography details---useful for getting FIPS codes, or when you forgot what county a town is in
* `acs.lookup` for variables and table numbers, but the output is annoying to work with
* `currency.convert` for inflation adjustments
* Plus dataframes of FIPS codes

### Getting ACS table numbers

In addition to `acs.lookup` (if you can sort through the output okay), you can get table numbers from ACS technical docs (look up "table shells").

Or, the easiest way to get table numbers might just be FactFinder :(


## Making ACS geographies

* `acs` package has several of its own object types, including `geo.set` for geographies
* Make `geo.set` objects based on FIPS codes and/or names
* Can combine multiple geographies in a few ways:
    + as a list of geographies (`combine` = `FALSE`)
    + a merged geography (`combine` = `TRUE`)
  
  
## Local geographies

See documentation on `geo.make` to see lots of different ways to make geographies, given FIPS codes and names. Ones I use commonly involve:

Single state


```r
ct <- geo.make(state = 09)
```

Single town


```r
nhv <- geo.make(state = 09, county = 09, county.subdivision = "New Haven")
```

Multiple towns merged into single geography with `combine = T`


```r
inner_ring <- geo.make(state = 09, county = 09, 
                       county.subdivision = c("Hamden", "West Haven", "East Haven"), 
                       combine = T, combine.term = "Inner Ring towns")
```


## Local geographies con't

#### Neighborhoods get a little more tricky

Single Census tract


```r
dixwell <- geo.make(state = 09, county = 09, tract = 141600)
```

Single tract, multiple block groups merged


```r
west_rock <- geo.make(state = 09, county = 09, 
                      tract = 141300, block.group = c(1, 4), 
                      combine = T, combine.term = "West Rock")
```

Mashup of block groups from multiple tracts (using wildcard `"*"`)


```r
beaver_hills <- geo.make(state = 09, county = 09, 
                         tract = c(141400, 141300), block.group = c("*", 2), 
                         combine = T, combine.term = "Beaver Hills")
```

Then make `geo.set` using `c`


```r
geos <- c(ct, nhv, inner_ring, beaver_hills, dixwell, west_rock)
```


## Pulling an ACS table

Total population: table number B01003

`acs.fetch` gets an ACS table for a geography & year---yields an `acs` object


```r
pop <- acs.fetch(geography = geos, endyear = 2015, 
                 table.number = "B01003", col.names = "pretty")
pop
```

```
## ACS DATA: 
##  2011 -- 2015 ;
##   Estimates w/90% confidence intervals;
##   for different intervals, see confint()
##                                                  Total Population: Total    
## Connecticut                                      3593222 +/- 0              
## New Haven town, New Haven County, Connecticut    130612 +/- 50              
## Inner Ring towns                                 145816 +/- 96.4676111448812
## Beaver Hills                                     5521 +/- 805.957194893128  
## Census Tract 1416, New Haven County, Connecticut 4898 +/- 503               
## West Rock                                        4132 +/- 463.159799637231
```


## Pulling several ACS tables

My favorite method: using `purrr` functions, map over a named list of table numbers---yields a named list of `acs` objects. Super convenient when you're working with 20+ tables.


```r
table_nums <- list( 
  total_pop = "B01003", 
  tenure = "B25003",
  poverty = "C17002"
)

fetch <- table_nums %>% 
  map(~acs.fetch(geography = geos, endyear = 2015, 
                 table.number = ., col.names = "pretty"))
```


## Analysis

`acs.R` has several functions for analysis, and allows many standard functions to work on `acs` objects---check the docs for `acs-class`.

Margins of error are handled for you! Add, divide, etc. columns in your tables without worrying about how to deal with MOEs.

(I got tired of repeating some of these operations, and wrote an entire package to streamline this: https://github.com/CT-Data-Haven/acsprofiles)

I'll have several `acs` objects after analysis---`total_pop`, `tenure`, `poverty`---then `cbind` them all together into a single `acs` object.

### Total population

Total population is ready to go, but it helps to shorten the name (also helps with a rounding trick later)


```r
total_pop <- fetch$total_pop[, 1]
acs.colnames(total_pop) <- "num_total_pop"
```


## Analysis con't

### Homeownership rate

Step by step:

* Get denominator: total households (column 1)
* Get number of owner-occupied households (column 2)
* Divide to get rate

Divide using `divide.acs` from `acs.R`


```r
households <- fetch$tenure[, 1]
owned <- fetch$tenure[, 2]
owned_rate <- divide.acs(owned, households)

tenure <- list(households, owned, owned_rate) %>% reduce(cbind)

# names come out ugly after division
acs.colnames(tenure) <- c("num_households", "num_owned_hh", "percent_owned_hh")
```


## Analysis con't

### Poverty & low-income rates

Step by step:

* Get denominator: total population for which poverty status is determined (column 1)
* Get population in poverty, i.e. below 1.0 x FPL (columns 2 + 3)
* Get low-income population, i.e. below 2.0 x FPL (columns 2 through 7)
* Divide to get rates


## Poverty & low-income con't

Add columns as necessary using `apply`, divide using `divide.acs` from `acs.R`. Really not as ugly as this looks!


```r
deter <- fetch$poverty[, 1]
poverty <- apply(X = fetch$poverty[, 2:3], FUN = sum, MARGIN = 2, agg.term = "poverty")
pov_rate <- divide.acs(poverty, deter)

low_income <- apply(X = fetch$poverty[, 2:7], FUN = sum, MARGIN = 2, agg.term = "low inc")
low_inc_rate <- divide.acs(low_income, deter)

poverty <- list(deter, poverty, pov_rate, low_income, low_inc_rate) %>%
  reduce(cbind)

# names come out ugly after division
acs.colnames(poverty) <- c("num_poverty_determined", "num_in_poverty", 
                      "percent_in_poverty", "num_low_income", "percent_low_income")
```


## Finish with a dataframe

`acs` objects have several slots, including `@geography`, `@estimate`, `@standard.error`

A simple dataframe here will contain the name of the geography, then columns for all estimates. Using `@standard.error`, you can include margins of error calculations.


```r
all_tables <- list(total_pop, tenure, poverty) %>% reduce(cbind)

# multiplying standard.error by qnorm(0.95) to get 90% MOE as used in ACS tables online
profile <- data.frame(name = all_tables@geography$NAME,
                      all_tables@estimate,
                      all_tables@standard.error * qnorm(0.95)) %>%
  tbl_df() %>%
  mutate(name = str_replace(name, " town,.+", "")) %>%
  mutate_at(vars(starts_with("percent")), funs(round(., digits = 3))) %>%
  mutate_at(vars(starts_with("num")), funs(round(.))) %>%
  setNames(str_replace(names(.), ".1", "_moe"))

# manually changing name for Dixwell---could use workaround if there were more to redo
profile$name[str_detect(profile$name, "Census Tract 1416")] <- "Dixwell"
```


## Our profile is ready!


```r
rmarkdown::paged_table(profile)
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["name"],"name":[1],"type":["chr"],"align":["left"]},{"label":["num_total_pop"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["num_households"],"name":[3],"type":["dbl"],"align":["right"]},{"label":["num_owned_hh"],"name":[4],"type":["dbl"],"align":["right"]},{"label":["percent_owned_hh"],"name":[5],"type":["dbl"],"align":["right"]},{"label":["num_poverty_determined"],"name":[6],"type":["dbl"],"align":["right"]},{"label":["num_in_poverty"],"name":[7],"type":["dbl"],"align":["right"]},{"label":["percent_in_poverty"],"name":[8],"type":["dbl"],"align":["right"]},{"label":["num_low_income"],"name":[9],"type":["dbl"],"align":["right"]},{"label":["percent_low_income"],"name":[10],"type":["dbl"],"align":["right"]},{"label":["num_total_pop_moe"],"name":[11],"type":["dbl"],"align":["right"]},{"label":["num_households_moe"],"name":[12],"type":["dbl"],"align":["right"]},{"label":["num_owned_hh_moe"],"name":[13],"type":["dbl"],"align":["right"]},{"label":["percent_owned_hh_moe"],"name":[14],"type":["dbl"],"align":["right"]},{"label":["num_poverty_determined_moe"],"name":[15],"type":["dbl"],"align":["right"]},{"label":["num_in_poverty_moe"],"name":[16],"type":["dbl"],"align":["right"]},{"label":["percent_in_poverty_moe"],"name":[17],"type":["dbl"],"align":["right"]},{"label":["num_low_income_moe"],"name":[18],"type":["dbl"],"align":["right"]},{"label":["percent_low_income_moe"],"name":[19],"type":["dbl"],"align":["right"]}],"data":[{"1":"Connecticut","2":"3593222","3":"1352583","4":"906227","5":"0.670","6":"3483303","7":"366351","8":"0.105","9":"822732","10":"0.236","11":"0","12":"3661","13":"5290","14":"0.004","15":"824","16":"7025","17":"0.002","18":"10695","19":"0.003"},{"1":"New Haven","2":"130612","3":"49771","4":"14374","5":"0.289","6":"121961","7":"32480","8":"0.266","9":"59530","10":"0.488","11":"50","12":"926","13":"663","14":"0.014","15":"552","16":"2312","17":"0.019","18":"3006","19":"0.025"},{"1":"Inner Ring towns","2":"145816","3":"54537","4":"34404","5":"0.631","6":"137192","7":"15007","8":"0.109","9":"35310","10":"0.257","11":"96","12":"818","13":"781","14":"0.017","15":"599","16":"1508","17":"0.011","18":"2331","19":"0.017"},{"1":"Beaver Hills","2":"5521","3":"2065","4":"906","5":"0.439","6":"5521","7":"1401","8":"0.254","9":"2608","10":"0.472","11":"806","12":"240","13":"163","14":"0.094","15":"806","16":"574","17":"0.110","18":"755","19":"0.153"},{"1":"Dixwell","2":"4898","3":"1832","4":"262","5":"0.143","6":"4099","7":"1344","8":"0.328","9":"2213","10":"0.540","11":"503","12":"146","13":"101","14":"0.056","15":"498","16":"451","17":"0.117","18":"551","19":"0.150"},{"1":"West Rock","2":"4132","3":"843","4":"123","5":"0.146","6":"2066","7":"802","8":"0.388","9":"1154","10":"0.559","11":"463","12":"126","13":"47","14":"0.059","15":"449","16":"333","17":"0.182","18":"372","19":"0.217"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

```r
write_csv(profile, "mini ACS profile.csv")
```


## Same limitations as always

Having a great workflow doesn't get us over the problem of large margins of error with small geographies. MOEs for low-income rates aren't bad for towns & bigger, but ugly for neighborhoods



![](index_files/figure-html/unnamed-chunk-17-1.png)<!-- -->


## Have fun with the ACS!

https://github.com/CT-Data-Haven

camille@ctdatahaven.org




