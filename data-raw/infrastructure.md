Package infrastrucure
================

This purpose of this document is to build the package infrastucture.

To upgrade the versions of **Vega-Lite** that we support:

1.  Modify the parameter `versions$value` in the yaml-header to this
    file.
2.  Render (knit) this document.
3.  Build-and-install this package on your local computer.
4.  Run the tests (`testthat::test()`); some snapshots will have
    changed.
5.  Rebuild the pkgdown website (`pkgdown::build_site()`), verify the
    visual-regression article (still to be built).
6.  Commit, push, and make PR.

These packages may not be listed in the `Suggests` section of the
`DESCRIPTION` file. It’s on you to make sure they are all up-to-date:

``` r
library("fs")
library("glue")
library("here")
```

    ## here() starts at /Users/ijlyttle/repos/public/vegawidget/vegawidget

``` r
library("purrr")
library("readr")
library("dplyr")
```

    ## 
    ## Attaching package: 'dplyr'

    ## The following objects are masked from 'package:stats':
    ## 
    ##     filter, lag

    ## The following objects are masked from 'package:base':
    ## 
    ##     intersect, setdiff, setequal, union

``` r
library("tibble")
library("stringr")
library("usethis")
library("conflicted")
library("vegawidget")

conflict_prefer("filter", "dplyr")
```

    ## [conflicted] Will prefer dplyr::filter over any other package

## Infrastructure

There are two source of “truth” for this process: this document, and the
contents of the directory `data-raw/templates`. (It may be useful to
note this in a “contributing” document for this package.) Each of the
infrastructure elements is created anew when this document is run; so,
between this document and `data-raw/templates`, we need to be able to
construct completely all the infrastructure.

Thus, we define this template directory and a function to delete and
create

``` r
dir_templates <- here("data-raw", "templates")

# create a create a clean directory, with a safety
create_clean <- function(path, path_safe = here::here()) {
  
  # tests that path is within path_safe
  is_path_safe <- function(path, path_safe) {
    identical(
      unclass(path_safe), 
      unclass(fs::path_common(c(path, path_safe)))
    )
  }
  
  if (!is_path_safe(path, path_safe)) {
    stop(
      "Nothing done because ",
      " path: ", shQuote(path), 
      " is not within path_safe: ", shQuote(path_safe)
    )
  }
  
  if (fs::dir_exists(path)) {
    fs::dir_delete(path)
  }
  
  fs::dir_create(path)
  
}
```

## Configure

We need to know which versions of the libraries (vega, vega-lite, and
vega-embed) to download. We do this by inspecting the manifest of a
specific version of the vega-lite library. This package has an internal
function, `get_vega_version()` to help us do this:

``` r
tbl_versions <- 
  enframe(unlist(params$versions), name = "widget", value = "vega_lite")
```

``` r
vega_version_all <- 
  tbl_versions |>
  left_join(
    map_dfr(tbl_versions$vega_lite, vegawidget:::get_vega_version),
    by = "vega_lite"
  )
   
vega_version_all
```

    ## # A tibble: 2 × 4
    ##   widget vega_lite vega   vega_embed
    ##   <chr>  <chr>     <chr>  <chr>     
    ## 1 vl5    5.5.0     5.22.0 6.20.8    
    ## 2 vl4    4.17.0    5.17.0 6.12.2

``` r
# we want to remove the "-rc.2" from the end of "4.0.0-rc.2"
# "-\\w.*$"   hyphen, followed by a letter, followed by anything, then end 
vega_version_short_all <- 
  vega_version_all |>
  mutate(across(starts_with("vega"), ~sub("-\\w.*$", "", .x)))
  
vega_version_short_all
```

    ## # A tibble: 2 × 4
    ##   widget vega_lite vega   vega_embed
    ##   <chr>  <chr>     <chr>  <chr>     
    ## 1 vl5    5.5.0     5.22.0 6.20.8    
    ## 2 vl4    4.17.0    5.17.0 6.12.2

``` r
if (!identical(vega_version_all, vega_version_short_all)) {
  warning("You may be using a release-candidate; please check.")
}
```

## htmlwidgets

First, let’s create a clean directory for the htmlwidget

``` r
dir_htmlwidgets <- here("inst", "htmlwidgets")
dir_lib <- path(dir_htmlwidgets, "lib")
dir_vegaembed <- path(dir_lib, "vega-embed")

create_clean(dir_htmlwidgets)
dir_create(dir_lib)
dir_create(dir_vegaembed)
```

### vegawidget

First, copy some files from our templates directory.

``` r
write_js <- function(x) {
  path(dir_templates, "vegawidget-all.js") %>%
    readr::read_lines() %>%
    purrr::map_chr(
      ~glue::glue_data(list(widget = x), .x, .open = "${")
    ) %>%
    readr::write_lines(path(dir_htmlwidgets, glue::glue("vegawidget-{x}.js")))  
}

walk(tbl_versions$widget, write_js)
```

The file `vegawidget-all.yaml` requires the versions the JavaScript
libraries; we interpolate these from `vega_version_short`.

``` r
write_yaml <- function(x) {
  path(dir_templates, "vegawidget-all.yaml") %>%
    readr::read_lines() %>%
    purrr::map_chr(
      ~glue::glue_data(vega_version_all %>% filter(widget == !!x), .x)
    ) %>%
    readr::write_lines(path(dir_htmlwidgets, glue::glue("vegawidget-{x}.yaml")))  
}

walk(vega_version_all$widget, write_yaml)
```

The file `vega-embed.css` adds some css for the (old-style) links that
appeared below a rendered spec:

``` r
fs::file_copy(
  fs::path(dir_templates, "vega-embed.css"), 
  fs::path(dir_vegaembed, "vega-embed.css")
)
```

Here’s where we download the libraries themselves, along with the
licences.

``` r
license_downloads <- 
    tribble(
    ~path_local,                         ~path_remote,
    "vega-lite/LICENSE",                 "https://raw.githubusercontent.com/vega/vega-lite/next/LICENSE",
    "vega/LICENSE",                      "https://raw.githubusercontent.com/vega/vega/master/LICENSE",
    "vega-embed/LICENSE",                "https://raw.githubusercontent.com/vega/vega-embed/next/LICENSE"
  )

license_downloads
```

    ## # A tibble: 3 × 2
    ##   path_local         path_remote                                                
    ##   <chr>              <chr>                                                      
    ## 1 vega-lite/LICENSE  https://raw.githubusercontent.com/vega/vega-lite/next/LICE…
    ## 2 vega/LICENSE       https://raw.githubusercontent.com/vega/vega/master/LICENSE 
    ## 3 vega-embed/LICENSE https://raw.githubusercontent.com/vega/vega-embed/next/LIC…

``` r
downloads_template <- 
  tribble(
    ~path_local,                                 ~path_remote,
    "vega-lite/vega-lite@{vega_lite}.min.js",    "https://cdn.jsdelivr.net/npm/vega-lite@{vega_lite}",
    "vega/vega@{vega}.min.js",                   "https://cdn.jsdelivr.net/npm/vega@{vega}",
    "vega-embed/vega-embed@{vega_embed}.min.js", "https://cdn.jsdelivr.net/npm/vega-embed@{vega_embed}",
  )

downloads_local <- function(widget) {
  
  df_local <- dplyr::filter(vega_version_all, widget == !!widget)
  
  downloads_template |>
  dplyr::mutate(
    dplyr::across(
      dplyr::starts_with("path"), 
      ~map_chr(.x, ~glue::glue_data(df_local, .x))
    )
  )  
}

htmlwidgets_downloads_all <- map_dfr(vega_version_all$widget, downloads_local)

htmlwidgets_downloads_all
```

    ## # A tibble: 6 × 2
    ##   path_local                          path_remote                               
    ##   <chr>                               <chr>                                     
    ## 1 vega-lite/vega-lite@5.5.0.min.js    https://cdn.jsdelivr.net/npm/vega-lite@5.…
    ## 2 vega/vega@5.22.0.min.js             https://cdn.jsdelivr.net/npm/vega@5.22.0  
    ## 3 vega-embed/vega-embed@6.20.8.min.js https://cdn.jsdelivr.net/npm/vega-embed@6…
    ## 4 vega-lite/vega-lite@4.17.0.min.js   https://cdn.jsdelivr.net/npm/vega-lite@4.…
    ## 5 vega/vega@5.17.0.min.js             https://cdn.jsdelivr.net/npm/vega@5.17.0  
    ## 6 vega-embed/vega-embed@6.12.2.min.js https://cdn.jsdelivr.net/npm/vega-embed@6…

``` r
get_file <- function(path_local, path_remote, path_local_root) {
  
  path_local <- fs::path(path_local_root, path_local)
  
  # if directory does not yet exist, create it
  # TODO: swap out with create_clean()
  dir_local <- fs::path_dir(path_local)
  
  if (!fs::dir_exists(dir_local)) {
    dir_create(dir_local)
  }
  
  download.file(path_remote, destfile = path_local)

  invisible(NULL)
}
```

Here, we create the `lib` directory, then “walk” through each row of the
`downloads` data frame to get each of the files and put it into place.

``` r
pwalk(license_downloads, get_file, path_local_root = dir_lib)
```

``` r
pwalk(htmlwidgets_downloads_all, get_file, path_local_root = dir_lib)
```

### Versions

We use this to support the `vega_version()` function.

``` r
.vega_version_all <- as.data.frame(vega_version_all)
.widget_default <- params$widget_default
```

### Vegawidget handlers

``` r
.vw_handler_library <-
  list(
    event = .vw_handler_def(
      args = c("event", "item"),
      bodies = list(
        item = .vw_handler_body(
          params = list(),
          text = c(
            "// returns the item",
            "return item;"
          )
        ),
        datum = .vw_handler_body(
          params = list(),
          text = c(
            "// if there is no item or no datum, return null",
            "if (item === null || item === undefined || item.datum === undefined) {",
            "  return null;",
            "}",
            "",
            "// returns the datum behind the mark associated with the event",
            "return item.datum;"
          )
        )
      )
    ),
    signal = .vw_handler_def(
      args = c("name", "value"),
      bodies = list(
        value = .vw_handler_body(
          params = list(),
          text = c(
            "// returns the value of the signal",
            "return value;"
          )
        )
      )
    ),
    data = .vw_handler_def(
      args = c("name", "value"),
      bodies = list(
        value = .vw_handler_body(
          params = list(),
          text = c(
            "// returns a copy of the data",
            "return value.slice();"
          )
        )
      )
    ),
    effect = .vw_handler_def(
      args = "x",
      bodies = list(
        shiny_input = .vw_handler_body(
          params = list(inputId = NULL, expr = "x"),
          text = c(
            "// sets the Shiny-input named `inputId` to `expr` (default \"x\")",
            "Shiny.setInputValue('${inputId}', ${expr});"
          )
        ),
        console = .vw_handler_body(
          params = list(label = "", expr = "x"),
          text = c(
            "// if `label` is non-trivial, prints it to the JS console",
            "'${label}'.length > 0 ? console.log('${label}') : null;",
            "// prints `expr` (default \"x\") to the JS console",
            "console.log(${expr});"
          )
        ),        
        element_text = .vw_handler_body(
          params = list(selector = NULL, expr = "x"),
          text = c(
            "// to element `selector`, adds text `expr` (default \"x\")",
            "document.querySelector('${selector}').innerText = ${expr}"
          )
        )
      )
    )
  )
```

### Write this out

``` r
usethis::use_data(
  .vega_version_all,
  .widget_default,
  .vw_handler_library,
  internal = TRUE, 
  overwrite = TRUE
)
```

    ## ✔ Setting active project to '/Users/ijlyttle/repos/public/vegawidget/vegawidget'
    ## ✔ Saving '.vega_version_all', '.widget_default', '.vw_handler_library' to 'R/sysdata.rda'
