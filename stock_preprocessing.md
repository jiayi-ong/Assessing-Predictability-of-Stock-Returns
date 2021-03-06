Code Appendix: Stock series pre-processing
================
Jia Yi Ong

``` r
library(dplyr)
library(lubridate)
library(stringr)
library(rlist)
library(moments)
```

## Load and Prepare raw data

Load data

``` r
SPTSE = read.csv("SPTSE.csv", stringsAsFactors = FALSE)
DAXI = read.csv("DAXI.csv", stringsAsFactors = FALSE)
SPX = read.csv("SPX.csv", stringsAsFactors = FALSE)
QE = read.csv("QE_TOTAL.csv", stringsAsFactors = FALSE)
SSE = read.csv("SSE_COMP.csv", stringsAsFactors = FALSE)
JSE = read.csv("JSE.csv", stringsAsFactors = FALSE)
PSI = read.csv("PSI.csv", stringsAsFactors = FALSE)
RTSI = read.csv("RTSI.csv", stringsAsFactors = FALSE)
MERVAL = read.csv("MERVAL.csv", stringsAsFactors = FALSE)
```

Format date and numbers. Remove NAs from missing values.

``` r
SPTSE = SPTSE %>% 
  select(Date, Close) %>%
  mutate(Date = dmy(Date)) %>%
  rename(Price = Close) %>%
  na.omit()

DAXI = DAXI %>% 
  select(Date, Close) %>%
  mutate(Date = dmy(Date),
         Close = as.numeric(Close)) %>%
  rename(Price = Close) %>%
  na.omit()

SPX = SPX %>% 
  select(Date, Price) %>%
  mutate(Date = dmy(Date)) %>%
  na.omit()

QE = QE %>% 
  select(Date, Price) %>%
  mutate(Date = dmy(Date))%>%
  na.omit()

SSE = SSE %>% 
  select(Date, Close) %>%
  mutate(Date = dmy(Date),
         Close = as.numeric(Close)) %>%
  rename(Price = Close) %>%
  na.omit()

JSE = JSE %>% 
  select(Date, Close) %>%
  mutate(Date = mdy(Date)) %>%
  rename(Price = Close) %>%
  na.omit()

PSI = PSI %>% 
  select(Date, Price) %>%
  mutate(Date = dmy(Date),
         Price = as.numeric(Price)) %>%
  na.omit()

RTSI = RTSI %>% 
  select(Date, Price) %>%
  mutate(Date = dmy(Date)) %>%
  na.omit()

MERVAL = MERVAL %>% 
  select(Date, Price) %>%
  mutate(Date = dmy(Date)) %>%
  na.omit()
```

## Define function to compute moments by month

Function to compute moments for each month in time interval

``` r
compute_moments = function(cleaned, first, last, sorted_tags){
  
  name = deparse(substitute(cleaned))
  
  processed = cleaned %>%
    
    # subset date interval
    filter(Date >= dmy(first) & Date <= dmy(last)) %>%
    
    # compute Year-Mon variable to group by
    mutate(mon_y = paste0("Y", year(Date), "M", month(Date))) %>%
    
    # for every Year-Mon group, summarize start and end of group
    # prices, compute difference, compute growth percentage
    group_by(mon_y) %>%
    summarize(mean = mean(Price),
              var = var(Price),
              skew = skewness(Price),
              kurt = kurtosis(Price)) %>%
    ungroup() %>%
    mutate(mon = as.numeric(substr(mon_y, start = 7, stop = 9))) %>%
    
    # sort by date tags
    arrange(factor(mon_y, levels = sorted_tags))
  
  return(processed)
}
```

## Define function to compute growth from start of month to end of every other month after

``` r
iter_growth = function(cleaned, first, last, sorted_tags,
                       start_YM, last_YM, n_ahead){
  
  name = deparse(substitute(cleaned))
  
  processed = cleaned %>%
    
    # subset date interval
    filter(Date >= dmy(first) & Date <= dmy(last)) %>%
    
    # compute Year-Mon variable to group by
    mutate(mon_y = paste0("Y", year(Date), "M", month(Date))) %>%
    
    # for every Year-Mon group, summarize start 
    # and end of group prices
    group_by(mon_y) %>%
    summarize(start = head(Price, 1),
              end = tail(Price, 1)) %>%
    
    # sort by date tags
    arrange(factor(mon_y, levels = sorted_tags))
  
  N = nrow(processed)
  start = which(processed$mon_y == start_YM)
  end = which(processed$mon_y == last_YM)
  table = c()

  for (i in start:end){
    growths = c()
    
    for (j in i:(i+n_ahead)){
      g = processed$end[j] - processed$start[i]
      growths = append(growths, g)
    }
    
    table = rbind(table, ifelse(growths < 0, 0, 1))
  }
  
  colnames(table) = c("current", paste0(1:n_ahead, "-ahead"))
  table = as.data.frame(table) %>%
    mutate(Index = name,
           mon_y = processed$mon_y[start:end],
           mon = as.numeric(substr(mon_y, start = 7, stop = 9)))
  
  return(table)
}
```

## Compute growths and export data (function 1)

Create vector of year-month labels to sort dataframe. Define first and
last of time interval

``` r
sorted_tags = paste0(paste0("Y", sort(rep(2014:2020, 12)), "M"), 1:12)
first = "01-Jan-14"
last = "31-Dec-20"
```

``` r
export = compute_moments(SPTSE, first, last, sorted_tags)
write.csv(export, "c1_ex1_can_sptse.csv", row.names = FALSE)

export = compute_moments(DAXI, first, last, sorted_tags)
write.csv(export, "c1_ex1_ger_daxi.csv", row.names = FALSE)

export = compute_moments(SPX, first, last, sorted_tags)
write.csv(export, "c1_ex1_us_spx.csv", row.names = FALSE)

export = compute_moments(QE, first, last, sorted_tags)
write.csv(export, "c4_ex1_qat_qe.csv", row.names = FALSE)

export = compute_moments(SSE, first, last, sorted_tags)
write.csv(export, "c3_ex1_chn_sse.csv", row.names = FALSE)

export = compute_moments(JSE, first, last, sorted_tags)
write.csv(export, "c3_ex1_safr_jse.csv", row.names = FALSE)

export = compute_moments(PSI, first, last, sorted_tags)
write.csv(export, "c2_ex1_por_psi.csv", row.names = FALSE)

export = compute_moments(RTSI, first, last, sorted_tags)
write.csv(export, "c2_ex1_rus_rtsi.csv", row.names = FALSE)

export = compute_moments(MERVAL, first, last, sorted_tags)
write.csv(export, "c2_ex1_arg_merval.csv", row.names = FALSE)
```

## Example of computed growths

``` r
export %>% head()
```

    ## # A tibble: 6 x 6
    ##   mon_y    mean     var     skew  kurt   mon
    ##   <chr>   <dbl>   <dbl>    <dbl> <dbl> <dbl>
    ## 1 Y2014M1 5631.  49201.  0.00670  1.77     1
    ## 2 Y2014M2 5893.  36203. -0.859    3.45     2
    ## 3 Y2014M3 5937.  44052.  0.557    2.10     3
    ## 4 Y2014M4 6543.  23063.  0.645    2.15     4
    ## 5 Y2014M5 7166. 125388.  0.484    1.71     5
    ## 6 Y2014M6 7839.  69003. -0.655    3.10     6

## Compute growths and export data (function 2)

Create vector of year-month labels to sort dataframe. Define first and
last of time interval

``` r
sorted_tags = paste0(paste0("Y", sort(rep(2014:2020, 12)), "M"), 1:12)
sorted_tags = c(sorted_tags, "Y2021M1", "Y2021M2")
first = "01-Jan-14"; last = "31-Mar-21"
s = "Y2014M2"; e = "Y2021M1"; n_ahead = 2
```

``` r
export = iter_growth(SPTSE, first, last, sorted_tags, s, e, n_ahead)
write.csv(export, "c1_ex2_can_sptse.csv", row.names = FALSE)

export = iter_growth(DAXI, first, last, sorted_tags, s, e, n_ahead)
write.csv(export, "c1_ex2_ger_daxi.csv", row.names = FALSE)

export = iter_growth(SPX, first, last, sorted_tags, s, e, n_ahead)
write.csv(export, "c1_ex2_us_spx.csv", row.names = FALSE)

export = iter_growth(QE, first, last, sorted_tags, s, e, n_ahead)
write.csv(export, "c4_ex2_qat_qe.csv", row.names = FALSE)

export = iter_growth(SSE, first, last, sorted_tags, s, e, n_ahead)
write.csv(export, "c3_ex2_chn_sse.csv", row.names = FALSE)

export = iter_growth(JSE, first, last, sorted_tags, s, e, n_ahead)
write.csv(export, "c3_ex2_safr_jse.csv", row.names = FALSE)

export = iter_growth(PSI, first, last, sorted_tags, s, e, n_ahead)
write.csv(export, "c2_ex2_por_psi.csv", row.names = FALSE)

export = iter_growth(RTSI, first, last, sorted_tags, s, e, n_ahead)
write.csv(export, "c2_ex2_rus_rtsi.csv", row.names = FALSE)

export = iter_growth(MERVAL, first, last, sorted_tags, s, e, n_ahead)
write.csv(export, "c2_ex2_arg_merval.csv", row.names = FALSE)
```

## Example of computed growths

``` r
export %>% head()
```

    ##   current 1-ahead 2-ahead  Index   mon_y mon
    ## 1       1       0       1 MERVAL Y2014M2   2
    ## 2       0       1       1 MERVAL Y2014M3   3
    ## 3       0       1       1 MERVAL Y2014M4   4
    ## 4       0       0       1 MERVAL Y2014M5   5
    ## 5       0       1       1 MERVAL Y2014M6   6
    ## 6       0       1       1 MERVAL Y2014M7   7
