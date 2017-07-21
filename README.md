# Making local ACS profiles in R

This is the repo for a presentation by Camille at [DataHaven](http://www.ctdatahaven.org/) for NNIP partners to learn about working with the `acs.R` package. The slides are online [here](https://ct-data-haven.github.io/acs_presentation). All the same material is also available as an [R notebook](https://ct-data-haven.github.io/acs_presentation/notebook.nb.html), so you can follow along and download the R Markdown document to mess around with the code. It's a walkthrough of my process in analyzing American Community Survey data on local towns and neighborhoods---but it doesn't have to be the way everyone writes their code!

## Recommended resources

This presentation is pretty technical, as it covers the details of working with a specific R package. For anyone who needs to brush up on some R skills, I recommend:

* DataCamp: http://datacamp.com Costs money but is totally worth it for interactive courses
* Tutorials from RStudio: http://rstudio.com/online-learning
* R for Data Science free online book: http://r4ds.had.co.nz/
* `swirl`, package for R tutorials inside RStudio: http://swirlstats.com/

This presentation makes heavy use of the `tidyverse` packages. If you aren't comfortable with those (`dplyr`, `tidyr`, `purrr`, etc), all the above resources should help you out.

I'd also recommend checking out the [acs.R handbook](http://dusp.mit.edu/sites/dusp.mit.edu/files/attachments/publications/working_with_acs_R_v_2.0.pdf)---34 pages of details about using the package, written by its author.

## API key

To use `acs.R`, you need a free [Census API key](https://www.census.gov/developers/). If you try to use functions from this package, you'll get a prompt saying you need to install your key, with instructions on how to do so. Install it once, and it's saved in your RStudio preferences.

## Questions?

Feel free to contact me at camille AT ctdatahaven DOT org.
