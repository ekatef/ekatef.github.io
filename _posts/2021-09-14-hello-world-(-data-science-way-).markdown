The climate data are a perfect data science playground with a nice side effect being a better feeling of the climate change. Actually, both effects I know from my own experience.

There are nowadays a lot of beautiful climate archives available. (Thanks to the open science and open data concepts!) However, I believe that it's of key importance to always keep in mind where are from the data you are using. And it's worth to start with the most basic datasets.

The basic part of the climate science are observations. The meteorological measurements are behind all the climate models. So, let's take the meteorological records for some location and check what they look like. We will use R language which has excellent data science functionality for research tasks.

### Data selection

My location choice is Angermuende town in Brandenburg area. I remember it as a small, beautiful and very quite place with a wonderful flair of its long and dramatic history. Highly recommended to visit on case you visit Berlin area and need a little break from the city buzz. 

![Angermuende](/assets/Angermuende.JPG){:class="img-responsive"}

What is going there with the climate? The most honest answer could give a meteorological record of the local observation station. That is exactly what does show the thermometer of a weather station at the given time moments without any computational artifacts. 

The observation records throughout Germany are kindly provided by the German Meteorological Office on the Open Data-Server via [https://opendata.dwd.de](https://opendata.dwd.de). We are interested in the weather observations archive and the hourly air temperature measurements in particular. That is we have to send a request via the following url

```R
data_dir_url <- "https://opendata.dwd.de/climate_environment/CDC/observations_germany/climate/hourly/air_temperature/historical/"
```

Angermuende meteorological station is referred in the National observation network under 00164; observations are available since 1956 with the files names on the server as follows 

```R
st_id <- "00164"
# the names are hardcoded according to the observations logic (sorry!)
tas_zip_on_server <- "stundenwerte_TU_00164_19560101_20201231_hist.zip"
tas_txt_on_server <- "produkt_tu_stunde_19560101_20201231_00164.txt"
tas_zip_url <- paste0(data_dir_url, tas_zip_on_server)
```

The zip and txt names of the data file were set manually according to the data structure on the server. We will need an archive name `tas_zip_url` to load the zipped data and a name of the compressed file `tas_txt_on_server` to read it. Of course, it is possible to generate both names automatically, but it hardly makes sense for a single request.

## DATA INPUT

First of all, we have to load a data file from the data server by the `tas_zip_url` address to a specified destination on the disc `tas_zip_file`. 

```R
tas_zip_file <- "tas_daily_st_00164.zip"
dnl_ok <- download.file(tas_zip_url, destfile = tas_zip_file)
```
The `dnl_ok` variable below is a download status only. A `download.file()` function is running [`wget`](https://www.gnu.org/software/wget/manual/wget.html) behind the scene and is used mainly for its side-effect.

Now we have to read the zip-file content into a data frame. This operation is luckily allowed by formatting of our data. Two things are important:
1. The name of a compressed file should be specified. Here is a point where we need our `tas_txt_on_server` variable.
2. We should remember to close a zip-file connection after data are read.

```R
zip_data <- read.table(
    unz(description = tas_zip_file, filename = tas_txt_on_server),
        sep = ";", header = TRUE, stringsAsFactors = FALSE)
close(file(tas_zip_file))
```

Note `stringsAsFactors = FALSE`. That is a kind of R magic to avoid a silent transformation of string variables to a factor type. A quick look on the results structure

```R
str(zip_data)

>'data.frame':  569799 obs. of  6 variables:
> $ STATIONS_ID: int  164 164 164 164 164 164 164 164 164 164 ...
> $ MESS_DATUM : int  1956010101 1956010102 1956010103 1956010104 1956010105 1956010106 1956010107 1956010108 1956010109 1956010110 ...
> $ QN_9       : int  5 5 5 5 5 5 5 5 5 5 ...
> $ TT_TU      : num  1.9 2 1 0.8 0.8 0.6 -0.1 0 0.3 0.8 ...
> $ RF_TU      : num  85 86 91 93 86 85 87 89 93 88 ...
> $ eor        : chr  "eor" "eor" "eor" "eor" ...
```

The `TT_TU` column contains hourly observations data on the air temperature we are interested while the `MESS_DATUM` column is a corresponded time stamp. A quick sanity check results in a conclusion that missed data were not recognized during data input

```R
any(is.na(zip_data))
>[1] FALSE
range(zip_data$TT_TU, na.rm = TRUE)
> [1] -999.0   37.3
```

Obviously, the missed temperature values are coded with the `-999` values. A `na.strings` is intended to fix this issue; unfortunately, the whitespaces are crucial to make `na.strings` work properly: `na.strings = "-999` does not work while  `na.strings = "  -999"` does the trick. That is not very convenient and the brute force approach is a better option:

```R
zip_data$TT_TU[zip_data$TT_TU < -100] <- NA
range(zip_data$TT_TU, na.rm = TRUE)
>[1] -28.1  37.3
```

So, now we have introduced the daily meteorological data into our programming workflow and are ready to process them. 

## Look on the data

We will choose here a quick-and-easy way and apply the `tidyverse` suite. I truly think that each R person should be able to do all basic operations by in-built language tools like NA replacement above, but `tidyverse` allows for cleaner calculations and is just perfect for our exploratory analysis.

Start with checking whether the required package is installed and fix it if it's not:

```R
# extract a package number from the whole list of installed ones
i_tidyverse <- grep(installed.packages()[, "Package"], pattern = "tidyverse")
# !NB this line downloads the package if it's not installed
if ( length(i_tidyverse) == 0 ) install.packages("tidyverse")

library(tidyverse)
library(lubridate)
```

The `lubridate` package makes life so very much easier when working with dates and times. This package is installed as a part of the `tidyverse` suite but requires for a separate loading with the `library()` function.

A quick look on the day-to-day air temperature dynamics can be done like that

```R
tas_day_plot <- zip_data %>%
    ggplot(aes(x = ymd_h(MESS_DATUM), y = TT_TU)) +
    geom_line() 
ggsave(paste0("tas_adv", st_id ,".jpeg"))
```

![as_ann_adv0016](/assets/tas_adv00164.jpeg)

A good moment to remember that the climate variability is a big thing and dynamic chaos is one of the climate system features… It seems there is a tendency towards the temperature increase but this trend is covered up with day-to-day climate variability and seasonality.  The simplest way to discover a robust climate signal is a simple aggregation:

```R
data_ann <- zip_data %>% 
    mutate(
        year = year(ymd_h(zip_data$MESS_DATUM))
    ) %>%
    group_by(year) %>%
    summarize(tas_ann = mean(TT_TU, na.rm = TRUE), n_ann = n())
```

The `na.rm = TRUE` means that missed values are simply ignored in the annual averages calculations. That is needed to use all the available data in the most complete way possible (otherwise we could loss the whole year due to one missed hour). However, an additional check should be done to ensure that the data gaps are not crucially big. We'll implement tests keeping in mind two possible flaws types:

1) not all dates may present in the `MESS_DATUM`;

2) some measurements may be not available for each year in the `TT_TU` column.

An additional `n_ann_nominal` column will contain a nominal hours number for each year. The `lubridate` functionality works amazing for such kind of tasks. The `time_length()` and `interval()` functions will do the job in our case. A total missed values number is calculated with a classic R boolean-to-integer coercion with `sum(is.na())`:

```R
data_ann_check <- zip_data %>% 
    mutate(
        year = year(ymd_h(zip_data$MESS_DATUM))
    ) %>%
    group_by(year) %>%
    summarize(nas_ann = sum(is.na(TT_TU)), n_ann = n()) %>%
    mutate(
        n_ann_nominal = time_length(
            interval(
                ymd(paste0(as.character(year), "0101")), 
                ymd(paste0(as.character(year), "1231")) + 1
            ),
            unit = "hour"
            ),
        missed_n = (n_ann_nominal - n_ann) + nas_ann,
        missed_share = missed_n/n_ann_nominal

    )
```

The testing results looks more than satisfactory

```R
range(data_ann_check$nas_ann)
> [1]  0 23
range(data_ann_check$missed_n)
> [1]  0 23
range(data_ann_check$missed_share
> [1] 0.000000000 0.002625571
```

There is no missed dates and a number of missed hourly measurements is no more than 23 hours per year (less than 0.3%). No further checking or filtering seems to be needed and we may rely on our data to make some conclusions about the climate in Angermuende. Let's look on the annual temperature dynamics:

```R
tas_plot <- data_ann %>%
    ggplot(mapping = aes(
        x = year, y = tas_ann)
    ) +
    geom_line(color = "gray") +
    geom_point(color = "forestgreen") 
ggsave(paste0("tas_ann_adv", st_id ,".jpeg"))
```

![tas_ann_adv0016](/assets/tas_ann_adv00164.jpeg){:class="img-responsive"}

The first thing we may see is that the climate warming is a reality. The multi-annual temperature increase during the covered time period is more than 2C target set by the Paris Agreement as the high warming limit… The warming dynamics is quite complicated and an increasing temperature trend is combined with significant year-to-year variations. For example, the coldest observed year was 1998 when a warming tendency has still established. Which does not prevents the temperature rise during the following years. The next local temperature minimum in 2010 is noticeable warmer as 1998 was. 

 The climate natural variability seems to be quasi-periodical. For example, especially hot years are observed in Argenmuende each 8-10 years. Such oscillations and theirs reasons directly relate to the big research questions of the modern climate science. Distinguish between the natural variability features and the climate change is currently one of the biggest challenges. However, quite basic data science tools allow to touch the spectral parameters of the ambient temperature getting a feeling of the natural climate variability. But that is a little different topic which I hope to show in one of the further posts :)

