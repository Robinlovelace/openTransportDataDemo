Open source software for reproducible transport data analysis: from
zones to route networks
================

<!-- README.md is generated from README.Rmd. Please edit that file -->
<!-- badges: start -->

[![.github/workflows/render-rmarkdown.yaml](https://github.com/Robinlovelace/odjitter/actions/workflows/render-rmarkdown.yaml/badge.svg)](https://github.com/Robinlovelace/odjitter/actions/workflows/render-rmarkdown.yaml)
<!-- badges: end -->

# Introduction

This repo was created to support at Open Data Manchester’s event on open
transport data, but it should be useful beyond that event, for anyone
wanting to get, analyse and model transport data with open source
software for transparent and evidence-based decision-making.

The amount of open data on transport systems can be overwhelming,
especially when much of it is hard to download, let alone visualise,
model and edit. In this talk, I will introduce tools that can help with
using open transport data to generate new evidence and analysis in
support of positive changes on vital travel networks in Chorlton and
beyond. I will show how to download and work with data on road networks,
road traffic casualties, and travel behaviour in R, a statistical
programming language with outstanding visualisation, geographic analysis
and statistical modelling capabilities. No need to ‘live code’ during
the session: all the scripts to reproduce the outputs from the
presentations will be provided in the open to support collaborative
transport planning research.

# Set-up

To reproduce the code in this repo you will need to have R installed
and, most likely, an IDE for R such as RStudio (recommended unless you
already have a favourite coding tool that has good support for R such as
VSCode).

If you’re new to R, it may be worth reading up on introductory material
such as the free and open source resource *Reproducible Road Safety with
R* (Lovelace 2020) tutorial. See [Section
1.5](https://itsleeds.github.io/rrsrr/introduction.html#installing-r-and-rstudio)
of that tutorial to install R/RStudio and [Section
3](https://itsleeds.github.io/rrsrr/rstudio.html) on getting started
with the powerful RStudio editor. A strength of R is the number of high
quality and open access
[tutorials](https://education.rstudio.com/learn/beginner/),
[books](https://education.rstudio.com/learn/beginner/) and videos to get
started.

If you have R installed, you should be able to run all the code in this
example and reproduce the results.

The first step is to install some packages, by entering the following
commands into the R console:

``` r
pkgs = c(
  "pct",
  "stats19",
  "osmextract",
  "tmap",
  "stplanr",
  "od",
  "dplyr"
)
```

You can install these packages as follows:

``` r
install.packages(pkgs)
```

You can load these packages one-by-one with `library(pct)`, or all at
once as follows:

``` r
lapply(pkgs, library, character.only = TRUE)[length(pkgs)]
#> Data provided under OGL v3.0. Cite the source and link to:
#> www.nationalarchives.gov.uk/doc/open-government-licence/version/3/
#> Data (c) OpenStreetMap contributors, ODbL 1.0. https://www.openstreetmap.org/copyright.
#> Check the package website, https://docs.ropensci.org/osmextract/, for more details.
#> 
#> Attaching package: 'od'
#> The following objects are masked from 'package:stplanr':
#> 
#>     od_id_character, od_id_max_min, od_id_order, od_id_szudzik,
#>     od_oneway, od_to_odmatrix, odmatrix_to_od
#> 
#> Attaching package: 'dplyr'
#> The following objects are masked from 'package:stats':
#> 
#>     filter, lag
#> The following objects are masked from 'package:base':
#> 
#>     intersect, setdiff, setequal, union
#> [[1]]
#>  [1] "dplyr"      "od"         "stplanr"    "tmap"       "osmextract"
#>  [6] "stats19"    "pct"        "sf"         "stats"      "graphics"  
#> [11] "grDevices"  "utils"      "datasets"   "methods"    "base"
```

One final line of code to set-up the environment is to switch `tmap`
into ‘view’ mode if you want to create interactive maps:

``` r
tmap_mode("view")
#> tmap mode set to interactive viewing
```

# Defining the study area

The first stage in many projects involving geographic data is defining
the study area. This is not always a straightforward or objective
process. In this case, the aim is to demonstrate how open data can be
downloaded and visualised with a focus on Chorlton and with a view to
getting the data into the transport simulation software A/B Street.

We will therefore select an area containing Chorlton and enough of the
surrounding area to enable modelling of trips to key destinations. As a
starting point, we will use a 2 km buffer around the straight line
between Chorlton and Manchester city centre to capture movement along
this transport corridor:

``` r
chorlton_point = tmaptools::geocode_OSM("chorlton, manchester")
manchester_point = tmaptools::geocode_OSM("manchester")
c_m_coordiantes = rbind(chorlton_point$coords, manchester_point$coords)
c_m_od = od::points_to_od(p = c_m_coordiantes, interzone_only = TRUE)
c_m_desire_line = od::odc_to_sf(c_m_od[-(1:2)])[1, ]
chorlton_buffer = stplanr::geo_buffer(c_m_desire_line, dist = 2000)
```

``` r
qtm(chorlton_buffer)
```

![](README_files/figure-gfm/unnamed-chunk-7-1.png)

``` r
sf::st_write(chorlton_buffer, "chorlton_buffer.geojson")
```

# Zone data from the PCT

TODO: Introduce what this section is doing? What’s the PCT?

``` r
head(pct::pct_regions$region_name)
#> [1] "london"                "greater-manchester"    "liverpool-city-region"
#> [4] "south-yorkshire"       "north-east"            "west-midlands"
# zones = pct::get_pct_zones("greater-manchester") # for smaller LSOA zones
zones = pct::get_pct_zones("greater-manchester", geography = "msoa")
names(zones)[1:20]
#>  [1] "geo_code"      "geo_name"      "lad11cd"       "lad_name"     
#>  [5] "all"           "bicycle"       "foot"          "car_driver"   
#>  [9] "car_passenger" "motorbike"     "train_tube"    "bus"          
#> [13] "taxi_other"    "govtarget_slc" "govtarget_sic" "govtarget_slw"
#> [17] "govtarget_siw" "govtarget_sld" "govtarget_sid" "govtarget_slp"
names_to_plot = c("bicycle", "foot", "car_driver", "bus")
plot(zones[names_to_plot] )
```

![](README_files/figure-gfm/unnamed-chunk-10-1.png)<!-- -->

To keep only zones whose centroids lie inside the study area we can use
the following spatial subsetting code:

``` r
zone_centroids = sf::st_centroid(zones)
#> Warning in st_centroid.sf(zones): st_centroid assumes attributes are constant
#> over geometries of x
#> Warning in st_centroid.sfc(st_geometry(x), of_largest_polygon =
#> of_largest_polygon): st_centroid does not give correct centroids for longitude/
#> latitude data
zone_centroids_chorlton = zone_centroids[chorlton_buffer, ]
#> although coordinates are longitude/latitude, st_intersects assumes that they are planar
#> although coordinates are longitude/latitude, st_intersects assumes that they are planar
zones = zones[zones$geo_code %in% zone_centroids_chorlton$geo_code, ]
```

Let’s plot the result, to get a handle on the level of walking and
cycling in the area (see interactive version of this map
[here](https://rpubs.com/RobinLovelace/772770), shown are LSOA results):

``` r
tm_shape(zones) +
  tm_fill(c("foot", "bicycle"), palette = "viridis") +
  tm_shape(chorlton_buffer) + tm_borders(lwd = 3)
```

![](https://i.imgur.com/oEuv1Zj.png)

# Desire line data from the pct package

The maps shown in the previous section establish that there is a decent
amount of cycling in the Chorlton area, at least according to the 2011
Census which is still a good proxy for travel patterns in 2021 due to
the inertia of travel behaviours to change (Goodman 2013).

You can get national OD (origin/destination, also called desire line)
data from the Census into R with the following command:

``` r
od_national = pct::get_od()
#> No region provided. Returning national OD data.
#> 
#> ── Column specification ────────────────────────────────────────────────────────
#> cols(
#>   `Area of residence` = col_character(),
#>   `Area of workplace` = col_character(),
#>   `All categories: Method of travel to work` = col_double(),
#>   `Work mainly at or from home` = col_double(),
#>   `Underground, metro, light rail, tram` = col_double(),
#>   Train = col_double(),
#>   `Bus, minibus or coach` = col_double(),
#>   Taxi = col_double(),
#>   `Motorcycle, scooter or moped` = col_double(),
#>   `Driving a car or van` = col_double(),
#>   `Passenger in a car or van` = col_double(),
#>   Bicycle = col_double(),
#>   `On foot` = col_double(),
#>   `Other method of travel to work` = col_double()
#> )
#> 
#> ── Column specification ────────────────────────────────────────────────────────
#> cols(
#>   MSOA11CD = col_character(),
#>   MSOA11NM = col_character(),
#>   BNGEAST = col_double(),
#>   BNGNORTH = col_double(),
#>   LONGITUDE = col_double(),
#>   LATITUDE = col_double()
#> )
od_national
#> # A tibble: 2,402,201 x 18
#>    geo_code1 geo_code2   all from_home light_rail train   bus  taxi motorbike
#>    <chr>     <chr>     <dbl>     <dbl>      <dbl> <dbl> <dbl> <dbl>     <dbl>
#>  1 E02000001 E02000001  1506         0         73    41    32     9         1
#>  2 E02000001 E02000014     2         0          2     0     0     0         0
#>  3 E02000001 E02000016     3         0          1     0     2     0         0
#>  4 E02000001 E02000025     1         0          0     1     0     0         0
#>  5 E02000001 E02000028     1         0          0     0     0     0         0
#>  6 E02000001 E02000051     1         0          1     0     0     0         0
#>  7 E02000001 E02000053     2         0          2     0     0     0         0
#>  8 E02000001 E02000057     1         0          1     0     0     0         0
#>  9 E02000001 E02000058     1         0          0     0     0     0         0
#> 10 E02000001 E02000059     1         0          0     0     0     1         0
#> # … with 2,402,191 more rows, and 9 more variables: car_driver <dbl>,
#> #   car_passenger <dbl>, bicycle <dbl>, foot <dbl>, other <dbl>,
#> #   geo_name1 <chr>, geo_name2 <chr>, la_1 <chr>, la_2 <chr>
```

Let’s keep only OD data that have a start and end point in the study
area (in a transport simulation, we may also want trips starting or
ending outside this area and passing through):

``` r
od = od_national %>% 
  filter(geo_code1 %in% zones$geo_code) %>% 
  filter(geo_code2 %in% zones$geo_code)
dim(od)
#> [1] 286  18
```

The result is nearly 300 rows of data representing movement between
origin and destination zone centroids. The data is non geographic,
however. To convert this non-geographic data into geographic desire
lines, you can use the `od_to_sf()` function in the `od` package as
follows:

``` r
desire_lines = od::od_to_sf(x = od, z = zones)
#> 0 origins with no match in zone ids
#> 0 destinations with no match in zone ids
#>  points not in od data removed.
```

We can plot the result as follows:

``` r
tmap_mode("plot")
#> tmap mode set to plotting
qtm(zones) +
  tm_shape(desire_lines) +
  tm_lines(c("foot", "bicycle"), palette = "Blues", style = "jenks", lwd = 3, alpha = 0.5)
```

![](README_files/figure-gfm/unnamed-chunk-16-1.png)<!-- -->

Note the OD data describes an aggregate pattern, between pairs of zones
– not between individual points-of-interest.

# Crash data from stats19

# Transport infrastructure data from osmextract

``` r
osm_data_full = osmextract::oe_get(zones)
osm_data_region = osm_data_full[chorlton_buffer, , op = sf::st_within]

q = "select * from multipolygons where building in ('house', 'residential', 'office', 'commercial', 'detached', 'yes')"
osm_data_polygons = osmextract::oe_get(zones, query = q)
osm_data_polygons_region = osm_data_polygons[chorlton_buffer, , op = sf::st_within]
qtm(zones) +
  qtm(osm_data_polygons_region)
```

# Scenarios of change

# Preparing data for A/B Street

``` r
remotes::install_github("a-b-street/abstr")

ablines = abstr::ab_scenario(
 houses = osm_data_polygons_region,
 buildings = osm_data_polygons_region,
 desire_lines = desire_lines,
 zones = zones,
 output_format = "sf"
)
```

# References

<div id="refs" class="references csl-bib-body hanging-indent">

<div id="ref-goodman_walking_2013" class="csl-entry">

Goodman, Anna. 2013. “Walking, Cycling and Driving to Work in the
English and Welsh 2011 Census: Trends, Socio-Economic Patterning and
Relevance to Travel Behaviour in General.” Edited by Harry Zhang. *PLoS
ONE* 8 (8): e71790. <https://doi.org/10.1371/journal.pone.0071790>.

</div>

<div id="ref-lovelace_reproducible_2020" class="csl-entry">

Lovelace, Robin. 2020. “Reproducible Road Safety Research with R.” Royal
Automotive Club Foundation.
<https://www.racfoundation.org/wp-content/uploads/Reproducible_road_safety_research_with_R_Lovelace_December_2020.pdf>.

</div>

</div>
