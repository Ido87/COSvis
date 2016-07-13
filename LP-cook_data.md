---
title: "Code for Cooking Data"
author: "Daniel Klotz"
date: "6 Juli 2016"
output: 
  html_document:
    number_sections: yes
---



# Introduction 
This section defines code of the **visCOS** functions used for "cooking data".
Within the scope of **visCOS** "cooking data" is defined as the process of 
transforming *raw data* into *cooked*, which can be used for the internal
visualisations.

# Code
The following functions are defined below 

function | exported
------------- | -------------
`get_runoff_example` | yes
`remove_chunk` | yes
`only_observed_basins` | no
`prepare_complete_date` | yes
`implode_cosdate` | no
`remove_leading_zeros` | no
`mark_periods` | yes
`runoff_as_xts` | no
`extract_objective_functions` | yes

## `get_runoff_example`
visCos porvides some exemplary data. All the available functions are tested 
with those. `get_runoff_example` is a small wrapper function to get the data from 
within the package via `read.csv`

```r
  #' Get runoff example
  #' 
  #' Get exemplary runoff data to test the different functions of visCOS
  #' @export
  get_runoff_example <- function() {
    file_path <- system.file("extdata", 
                             "runoff_example.csv", 
                             package = "visCOS")
    runoff_example <- read.csv(file_path)
    return(runoff_example)
  }
```


## `remove_chunk` 
This function removes all columns not specified in the `viscos_options` string
`data_regex`, aswell basins where no observations are available.
The first part is done by identifying the names of the data_frame and 
searching for the idx of the names defined in the regular expression string. 
The second part is done via the function prepare.only_observed (see next subchapter).
It is actually so pretty simple, but realy usefull. Since the solution is 
quite specific the function tests if the input is a data.frame and returns an 
error if not. 

**Note** that the routine is **not** case sensitive. It does not distinguish 
inbetween small and capital letters!

The function header is: 

```r
  #' removes chunk in runoff_data
  #' 
  #' Remove all collumns which are not foreseen (see: viscos_options) from 
  #' runoff data 
  #' 
  #' @param runoff_data data.frame object containing at lesast COSdate, 
  #' Qsim and Qobs (see: xxx)
  #' @return data.frame object withouth the chunk
  #' @export
  remove_chunk <- function(runoff_data) {
```

The first part of the code loads the dependencies and makes sure that the 
runoff_data varibale is a data.frame (see: chapter about *defensive coding*):

```r
  require("magrittr", quietly = TRUE)
  assert_dataframe(runoff_data)
```

The body of `remove_chunk` works as following: First, the names of the names of 
the runoff_data are determines via `names` and set to all lower letters with 
`tolower`. This is the reason why the function is not case sensitive!
Then helper function `get_regex_for_runoff_data` is called to get regular 
expressions that identify the wanted columns. Later, `grep` is used to get 
the index `idx` of the columns with the given names (`lowercase_names_in_data`). 
Finally, `idx`is then used to select the wanted columns.

```r
  lowercase_names_in_data <- runoff_data %>% names %>% tolower 
  # 
  regex_columns <- get_regex_for_runoff_data() # see: helpers

  idx <- regex_columns %>% 
    grep(.,lowercase_names_in_data)
    no_chunk_runoff_data <- runoff_data[ , idx]
    return( only_observed_basins(no_chunk_runoff_data) )
  }
```

### `only_observed_basins`
This function removes basins for which no observations are available. No 
observation are interpreted as columns in which the all entries are either -999
or `NA's`. The function header looks like this: 

```r
# remove basins withouth observations
#
# Removes basins withouth observation (-999/NA values) from the provided dataframe
#
# @param runoff_data A raw runoff_data data.frame, which may containts basins 
# withouth observations.
# \strong{Note:} It is assumed that all available basins are simulated!
# @return data.frame without the observation-free basins
# @export
only_observed_basins <- function(runoff_data) {
```

As so often it is first checked if `magrittr` is available. 
Afterwards, it is asserted that `runoff_data` is indeed a data frame 
(see the chapter about defensive code). With that the preperations are finished, 
which should be fine as the function is supposedly only called from within 
`remove_chunk`. 

```r
  require("magrittr", quietly = TRUE)
  assert_dataframe(runoff_data)
```

In order to handle the `NA's` and -999 in the same way all `NA's` are replaced 
with -999. Then the `lapply` function is used to find the max of each column. 

```r
  runoff_data[is.na(runoff_data)] <- -999 
  colmax <- lapply(X = runoff_data, FUN = max) # get max of any column
```

If any column has a max of -999 it is worth futher investigation.
Unfortunately the -999 columns could also mean, that something went wrong 
with the simulation, thus one has to make sure that only obsevation columns 
are selected. To do so we first identify the idx of the max == -999 cols and 
build a regular expression with the help of the `viscos_options`.
We combine both infromation via `grepl` to check if only observed columns have
been selected. If so everything is fine, but if `any` of the selected cols came
from an observation col the function stops the execution and gives a warning 
message. 
The next step involves a little trick in which +1 is added to the `idx` to 
obtain the indexes of the corresponding simulation. Here we assume that for each
observation there is a corresponding simulation **in next the next colum**!
This allows us then to select only the observed basins. But now all NA values 
are actually -999! Thus, we replace them again back to NA, via `lapply(...,min)`
assuming that all negative values are NAs. This is not a strong assumation, 
because there should never be negative flows! In the end the resulting
data.frame is returned.

```r
  if ( any(colmax < 0.0) ){
    idx_temp <- which(colmax < 0.0) 
    obs_regex <- paste(viscos_options()$name_data1,".*", sep ="")
    OnlyQobsSelected <- idx_temp %>%
      names %>% 
      tolower %>%
      grepl(obs_regex,.) %>%
      any
    if (!OnlyQobsSelected){
      stop("There are Qsim withouth simulation (i.e. only values smaller 0). Pls remove them first")
    }
```


```r
    idx_slct <- c(idx_temp,idx_temp + 1) %>% sort() 
    d_onlyObserved <- runoff_data[-idx_slct]
    # set remaining negative Qobs to NA, so that HydroGOF can be used correctly, also ignoring NAs
    colmin <- lapply(X = d_onlyObserved, FUN = min)
    idx_temp <- which(colmin < 0.0) 
    d_onlyObserved[d_onlyObserved[idx_temp]<0, idx_temp] <- NA
    return(d_onlyObserved)
  } else {
    return(runoff_data)
  }
}
```


## `prepare_complete_date` 
This function is a wrapper. It is however not finished yet! 
The idea is the following. The idea is to provide a handy function to complete 
the date-columns. Within the data.frame the user has to provide either 5 columns 
with the *COSdate* format: `yyyy-mm-dd-hh-min` i.e.: year-month-day-hour-minute or 
an equivalent column in `POSIXct` format 
(see: [link](https://stat.ethz.ch/R-manual/R-devel/library/base/html/as.POSIXlt.html)).

*However*, it is currently only possible to convert *COSdate* into `POSIXct` dates
via the `implode_cosdate` function. 

The function itself is basically a switch which calls different functions, 
if `name_cosyear` and/or `name_posix` is found and returns an error if none of 
the two are found. If the user has not named his columnds according to COSERO 
convention he can provide the names of the *COSdate* year via `name_cosyear` and
the names of the POIXct column via `name_posix`. The function returns an error 
if `runoff_data` is not a data frame and if `OK_COSdate` or `OK_POSIXdates` are
not logical due to some problems with the dates/time-formats. 

A *caveat* with this method is that only the `name_cosyear` is checked to infer 
if the 5 *COSdate* columns are available. This can cause errors later if some 
other columns are missing. Sadly it is not apparent how to avoid that problem,
withouth defining the names of all columns. 

Furthermore, please we need to be aware that a fixed that the time-zone is 
assumed to be *UTC* to avoid problems with leaps in time (summer/winter time).

To function `implode_cosdate` is used to obtain the POSIXct date from the 
*COSdate* columns. The following chapter will show how this is done. 

```r
#' Complete the date-formats with POSIXct or COSdate
#' 
#' Complete the data-formats of your data.frame `POSIXct` and/or `COSdate`
#' 
#' @param runoff_data The data.frame, which contains the runoff information
#' @param name_cosyear string with the name of the `COSdate` year column
#' @param name_posi string with the name of the POSIXct column 
#' @return The new runoff data.frame with the added data-format. 
#' @export
prepare_complete_date <- function(runoff_data = NULL, 
                                  name_cosyear = "yyyy",
                                  name_posix = "POSIXdate") {
```


```r
  # make sure that magrittr is loaded: 
  require("magrittr", quietly = TRUE)
  assert_dataframe(runoff_data)
  # check for COSdates and stop if non-logical expression are obtained
  OK_COSdate <- any(names(runoff_data)== viscos_options()$name_COSyear)
  OK_POSIXdates <- any(names(runoff_data)== viscos_options()$name_COSposix)
  if ( !is.logical(OK_COSdate) | !is.logical(OK_POSIXdates) ) {
    stop("Something seems to be wrong with the date / time formats :(")
  }
  # choose function depending on which formats are available!
  if (!OK_COSdate & !OK_POSIXdates) {
    stop("No COSdates and no POSIXct-dates in the data!")
  } else if (OK_COSdate & !OK_POSIXdates) { 
    runoff_data <- implode_cosdate(runoff_data) # see following chapter
  } else if (!OK_COSdate & OK_POSIXdates) {
    stop("POSIXct to COSdates not yet supported :(")
  }
  return(runoff_data)
}
```

### `implode_cosdate`
This function is used to transform the old-school *COSdate* format into the 
widely spread POSIXct format, used within R and in many packages. The function 
is called by the function above and not provided to the user! It takes a 
data frame (`runoff_data`) and uses `viscos_options` to transfrom the different
*COSdate* columns into POSIXct dates via `paste`. 
The function header is: 

```r
# transform COSdate into the nicer POSIXct-date format
#
# Takes a data.frame, which contains the COSdate format (see: xxx) and 
# transforms it into a POSIXct series. Note that time is assumed to be in UTC
implode_cosdate <- function(runoff_data) {
```

The following sanity checks are applied: 
(I) A check is made if magrittr is loaded or can be loaded. 
(II) A test is made to check if `runoff_data` is a `data.frame`. 
Afterwards the paste function is used to concatenate the different *COSdate* 
columns (i.e. year, month, day, hour and minute) to a POSIXct date format.
The obtaine series of strings is saved into the variable `POSIXdate` and 
added as the column `viscos_options()$name_COSposix` (see section about 
`viscos_options`) to the `runoff_data.

```r
  require("magrittr", quietly = TRUE)
  assert_dataframe(runoff_data)
  name_string <-  runoff_data %>% names %>% tolower
  #
  POSIXdate <- paste(runoff_data[[viscos_options()$name_COSyear]],
                     sprintf("%02d",runoff_data[[viscos_options()$name_COSmonth]]),
                     sprintf("%02d",runoff_data[[viscos_options()$name_COSday]]),
                     sprintf("%02d",runoff_data[[viscos_options()$name_COShour]]),
                     sprintf("%02d",runoff_data[[viscos_options()$name_COSmin]]),
                     sep= "" ) %>%
    as.POSIXct(format = "%Y%m%d%H%M",tz = "UTC")
  runoff_data[[viscos_options()$name_COSposix]] <- POSIXdate
  return(runoff_data)
}
```


## `remove_leading_zeros`
This function removes leading zeros from column names of the data.frame 
`runoff_data`.The function has no defensive codidng, appart from the execution 
of `remove_chunk` (see corresponding chapter) to be sure that no non-needed 
columns are within the data.frame. 


```r
remove_leading_zeros <- function(runoff_data) {
  require("magrittr", quietly = TRUE)
  runoff_data %<>% remove_chunk
```

The body of the function works in the following way:
The variables `runoff_names` ist computed by obtaining the names of the columns 
of `runoff_data` are obtained with `names` and removing all teh digits with 
`gsub("\\d",...)`. The basin digits, `runoff_nums` are computed by doing the 
oposite, i.e.: removing all the non digits via `gsub("\\D",...)`. Afterwards, 
the empty strings ("") are replaced with a small trick: The characters are 
transformed into a numerics (induces `NA's`) and back again. 
In the last step the variables `runoff_names` and `runoff_nums` are simply put
togehter again via the paste command. The result has no leading zeros as in the 
variable numeration. 

```r
  runoff_names <- runoff_data %>% names %>% gsub("\\d","",.) 
  # get numbers and remove leading zeros
  runoff_nums <- runoff_data %>% 
    names %>% 
    gsub("\\D","",.) %>% 
    as.numeric %>%  
    as.character
  runoff_nums[is.na(runoff_nums)] = ""
  # paste new nums as new data_names 
  names(runoff_data) <- paste(runoff_names, runoff_nums, sep = "")
  return(runoff_data)
}
```


## `mark_periods`
This function can be used to create a periods colum in your data.frame. The
period colum is a integer vector, which is 1 if the row is within the period 
and 0 if it ous of the period. 

Building it was acutally quite tricky. The author himself (Klotz) is not 
sure if it is realy a nice solution. But it is the best solution, in terms of 
speed and clearness, that we could come up so far. 

As shown below the function takes the `runoff_data` data.frame and the two 
integers `start_month`and `end_month`. They allow the user to define the periods
within the `runoff_data` on a monthly resolution. As the name suggests 
`start_month` defines the beginning of the period and `end_month` the end of 
the period. The function requires `dplyr` and `magrittr` and tests if 
`runoff_data` is a data.frame. An error is returned if it is not. 
Furthermore, if there is *chunk* in the data.frame it is removed by 
`remove_chunk`(see above) and `prepare_complete_date` is used to ensure that 
the full time-information, i.e. both date formats, is available.

```r
#' calculate periods
#' 
#' Mark the periods within runoff_data. 
# The makring uses a monthly resolution, which are defined by the integers 
#' `start_month` and `end_month`.  
#'
#' @param runoff_data The data.frame, which contains the runoff information
#' @return The runoff data.frame reduced and ordered according to the 
#' hydrological years within the data. 
#' \strong{Note:} The periods columns are formatted as characters!
#' @export
mark_periods <- function(runoff_data, start_month = 10, end_month = 9) {
  require("dplyr", quietly = TRUE, warn.conflicts = FALSE)
  require("magrittr", quietly = TRUE)
  assert_dataframe(runoff_data)
  runoff_data %<>% remove_chunk %>% prepare_complete_date()
```

The body of the function is splitted in two parts. 
In the *first part* the variable `period_range` is defined as a integer sequence 
from the `start_month` to the `end_month`. 
The solution distinguishes between two different cases: 
`start_month` <= `end_month` and `start_month` > `end_month`. The solution 
for the former is trivial, as `seq` can be used. The solution for the latter is 
in priciple also simple as two ranges (`range_1` and `range_2`) can be 
constructed and concatenated. 
Note that both solution addinitally construct the integer-vector 
`out_of_period`. The vector contains inforamtion about the moths which are not 
within the the period_range. It would not be necessary for the first case. But, 
since the current solution still seems to be suboptimal, it is needed for the 
second case, in order to mark the months which are out of the period later. 

```r
  # (I) get labels for the monts
  if (start_month <= end_month ) {
    period_range <- seq(start_month,end_month)
    out_of_period <- seq(1,12) %>% extract( !(seq(1,12) %in% period_range) )
  } else if (start_month > end_month) {
    range_1 <- seq(start_month,12)
    range_2 <- seq(1,end_month)
    period_range <- c(range_1,range_2)
    out_of_period <- seq(1,12) %>% extract( !(seq(1,12) %in% period_range) )
  }
```

The *second part* is a bit trickier. 

First, the function `eval_diff` is defined. It takes a vector `a` at calculates 
the differences for it, where the first entry of `a` is kept, 
e.g. `eval_diff( c(3,2,5,7,8) )` is `3 -1  3  2  1`.
Then, `eval_diff` is used to to mark all the starms month of the runoff data and 
save them into an `period` column. The ` pmax(.,0)` operation guaratness that 
negative diferrencing (as above in 3 -> 2) is not included and the cumulative 
sum is used to count the periods are within the data.frame.  

Secondly, the `out_of_period` of all years set back to zero again, by simply 
checking which months of our data.frame are equal to the `out_of_period` entries.
There are two problem with that solution. One is that the last year is not
extracted proberly if `start_month` > `end_month`. For this case `dplyr::mutate` 
is used to set the out-of-period-dates from the last year to 0,
if they are bigger then `end_month`.
The second problem is that the first and last period are also included in the 
solution even if they are **not** complete! *Thus, if an user does not want that 
these solutions to be "marked" he has to remove it himself!*

```r
  # (II) mark periods:
    eval_diff <- function(a) {c(a[1],diff(a))}
    runoff_data[[viscos_options()$name_COSperiod]] <- runoff_data[[viscos_options()$name_COSmonth]] %in% c(start_month) %>% 
      eval_diff %>% 
      pmax(.,0) %>% 
      cumsum 
    runoff_data$period[runoff_data[[viscos_options()$name_COSmonth]] %in% out_of_period] <- 0
    # corrections for last year 
    max_year <- max(runoff_data[[viscos_options()$name_COSyear]])
    runoff_data %<>% dplyr::mutate(
      period = ifelse(
        (  (.[[viscos_options()$name_COSyear]] == max_year) &
           (.[[viscos_options()$name_COSmonth]] > end_month)  ),
                      0,
                      period
        )
      )
    return(runoff_data)
}
```

## `runoff_as_xts`
This function is a actually just a wrapper around the `xts::xts` function for 
the purposes of **visCOS**. `runoff_as_xts` depends on `magrittr`, `xts` and 
`zoo` (where the last one is acutally a dependency of `xts`). 
The function takes the runoff_data as input and makes sure that the data is a 
data.frame (check: `assert_dataframe` in the **defensive code** chapter), 
has no "junk" inside (`assert_chunk`, see also: `remove_chunk`) and 
has all the needed date/time columns (`assert_complete_date`, see also: 
`prepare_complete_date`). 

A notable quirk of the function is that the names of the header are put to lower
cases via the `tolower` funciton and possible leading zeros in the enumeration
of the basins are removed.

Currently it is not exported as users can use `xts` themselves perfectly well, 
and it is felt that the function does not provide enough added value for the 
user.

```r
# Convert runoff_data to xts-format
#
# Converts the runoff_data (class: data_frame) into an xts object
#
# @param runoff_data data_frame of the runoff_data (see: xxx)
# @return xts object of the runoff_data data.frame
# @export
runoff_as_xts <- function(runoff_data) {
  # pre
  require("zoo", quietly = TRUE, warn.conflicts = FALSE)
  require("xts", quietly = TRUE, warn.conflicts = FALSE)
  require("magrittr", quietly = TRUE)
  assert_dataframe(runoff_data)
  assert_chunk(runoff_data)
  assert_complete_date(runoff_data)
  # calculations:
  runoff_data %<>% remove_leading_zeros
  names(runoff_data) <- runoff_data %>% 
    remove_leading_zeros %>% 
    names %>%
    tolower
  name_posix <- viscos_options()$name_COSposix %>% tolower
  runoff_data_as_xts <- xts::xts(x = runoff_data,
                                 order.by = runoff_data[[name_posix]]) 
  # 
  return(runoff_data_as_xts)
}
```