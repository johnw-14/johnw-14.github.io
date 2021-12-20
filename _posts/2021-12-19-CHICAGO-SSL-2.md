---
layout: post
title: "Mapping Chicago Predictive Policing Data by Census Tract"
always_allow_html: yes
output: 
  md_document: 
    variant: gfm
    preserve_yaml: true
---

------------------------------------------------------------------------

## Introduction

This post has been on the back burner for a few months while my company
goes through a new software implementation. I thought it best to focus
on the data work I get paid for. But I am finally back! (And then likely
gone again for a bit.)

This will be a bit of a mash-up of topics covered in two previous posts.
In [this post](https://johnw-14.github.io/2021/04/27/CHICAGO-SSL-1.html)
I looked at public data from a former predictive policing program in
Chicago. The program used an algorithm “to create a risk assessment
score known as the Strategic Subject List or ‘SSL.’ These scores reflect
an individual’s probability of being involved in a shooting incident
either as a victim or an offender.” In that post I did some basic
statistical tests to see how SSL scores compared between racial groups
of arrested individuals.

I now want to look at how the output SSL scores vary geographically in
Chicago. The data set provides the census tract of each arrest (in a
[separate
post](https://johnw-14.github.io/2021/05/26/CA-VACCINE-DIST.html) I
looked at census tracts and COVID-19 vaccine allocation in California).
For this post, I’m going to focus on mapping the SSL data only, but next
I want to examine how the geographic SSL score distribution compares to
other census data.

There are two articles I found particularly helpful in joining the SSL
data to public maps of census tracts. One is by Zev Ross
[here](http://zevross.com/blog/2015/10/14/manipulating-and-mapping-us-census-data-in-r-using-the-acs-tigris-and-leaflet-packages-3/),
and another from R-Journalism
[here](https://learn.r-journalism.com/en/mapping/ggplot_maps/mapping-census/).

Both of the tutorials start off with the “hard way” where you combine
spatial census tract data with your own table (SSL scores in this case).
You then use the resulting data frame to plot in ggplot. I will start
there for a basic visual of the geographic SSL distribution, and then
switch over to a more streamlined and visually pleasing option.

------------------------------------------------------------------------

## Load Packages in R

``` r
library(tidyverse)
library(rgdal)
library(rgeos)
library(maptools)
library(leaflet)
library(htmlwidgets)
library(here)
```

More information on the packages can be found in the [Zev Ross
post](http://zevross.com/blog/2015/10/14/manipulating-and-mapping-us-census-data-in-r-using-the-acs-tigris-and-leaflet-packages-3/)
mentioned earlier.

------------------------------------------------------------------------

## Setup

To map SSL scores by census tract, we’re naturally going to need a map
of census tracts in Chicago. I learned that census tract boundaries can
shift quite regularly, so I made sure to download a census tract
shapefile for Illinois in 2016 (which lines up with the SSL data we are
using). You can access the shapefiles
[here](https://www.census.gov/geographies/mapping-files/time-series/geo/carto-boundary-file.2016.html),
and adjust the year and map type as needed.

Now that we have our census tract shapefile, we can read it in from our
documents. The dsn and layer are just the folder and file names on my
computer, respectively.

``` r
tract <- readOGR(dsn="illinois_tracts_2016_17", layer = "cb_2016_17_tract_500k")
```

    ## OGR data source with driver: ESRI Shapefile 
    ## Source: "C:\Users\whele\Documents\johnw-14.github.io\docs\_posts\illinois_tracts_2016_17", layer: "cb_2016_17_tract_500k"
    ## with 3121 features
    ## It has 9 fields
    ## Integer64 fields read as strings:  ALAND AWATER

``` r
tract@data$TRACTCE <- as.character(tract@data$TRACTCE)
```

We also need to store the census tract ID (“TRACTCE” in the table) as a
character to join with the SSL data in a minute. To see if things worked
properly, we can quickly plot the tract data:

``` r
plot(tract)
```

![](/images/illinois-tracts-1.png)<!-- -->

Looks like Illinois to me! (After Google searching “illinois map”, of
course.) Chicago is the dense bit in the northeast corner, which we will
focus on with the SSL data.

Our tract data still needs to be converted from polygons to a data frame
(more technical background in the Zev Ross post). We do this by using
the fortify function in ggplot2. (Note: I initially got an error here,
“Error: isTRUE(gpclibPermitStatus()) is not TRUE”. I had to install
rgeos as well, and then reinstall rgdal from the source. More on that
issue
[here](https://stackoverflow.com/questions/30790036/error-istruegpclibpermitstatus-is-not-true)).

``` r
ggtract <- fortify(tract, region = "GEOID")
```

Now we have our data frame of the census tract boundaries. Next, we need
to structure the SSL data so they can be joined together. I decided to
start with simply the count of arrests and mean SSL score for
individuals arrested in each census tract (storing the census tract ID
as a character as well).

``` r
SSL_data_by_tract <- ChicagoPD_data %>%
  group_by(CENSUS.TRACT) %>%
  summarise(count = n(), meanSSL = mean(SSL.SCORE))
SSL_data_by_tract$CENSUS.TRACT <- as.character(SSL_data_by_tract$CENSUS.TRACT)
```

We are now ready to join the two together. Since the census tracts from
the shapefile are still all of Illinois, I filter out any census tracts
with no arrest data (and no SSL scores to look at).

``` r
SSL_tract_join <- left_join(ggtract, SSL_data_by_tract, by = c("id" = "CENSUS.TRACT"), copy = TRUE)
ggSSLtract <- SSL_tract_join %>%
  filter(!is.na(count))
```

------------------------------------------------------------------------

## Initial Heat Maps

With the combined data ready, we can use ggplot to get a quick heat map
of the mean SSL scores by census tract. I took my mapping cues from both
of the tutorials mentioned in the introduction.

``` r
ggplot() + geom_polygon(data = ggSSLtract, aes(x = long, y = lat, group = group, fill = meanSSL)) + scale_fill_gradient2(low = "deepskyblue2", mid = "white", high = "firebrick1", midpoint = 280)
```

![](/images/gg-SSL-map-1.png)<!-- -->

A quick Google search of “Chicago census tracts” will also confirm this
looks the way it should. It’s not the most visually pleasing heat map,
but it gives an initial idea of how the SSL scores look across the city.
Generally speaking, you see higher average scores towards the south of
the city. I could tinker with the parameters further to get some sharper
contrast and a tidier map, but rather than digging into that (this was
all the “hard way” remember), I’m going to jump over to a cleaner
option.

------------------------------------------------------------------------

## Improved Heat Maps

The steps from here closely follow the Zev Ross post. Rather than
converting the shapefile into a data frame for ggplot, we join the
summarized SSL data to the census tract shapefile directly. The added
“rec” column was a safety recommendation in the post.

``` r
df.polygon2 <- tract
df.polygon2@data$rec <- 1:nrow(df.polygon2@data)
tmp <- left_join(df.polygon2@data, SSL_data_by_tract, by = c("GEOID" = "CENSUS.TRACT"), copy = TRUE) %>% 
  arrange(rec)
df.polygon2@data <- tmp
```

Again, we want to filter the joined data to the Chicago area (i.e. only
census tracts with a non-zero count of arrests).

``` r
df.polygon2 <- df.polygon2[!is.na(df.polygon2$count),]
```

With that, we can now plot using leaflet. The popup information will
show the user the GEOID, count of arrests and mean SSL when they click
on a census tract in the map. The rest of the cosmetics follow Zev
Ross’s lead.

``` r
popup <- paste0("GEOID: ", df.polygon2$GEOID, "<br>", "Count of Arrests: ", df.polygon2$count, "<br>", "Mean SSL Score: ", round(df.polygon2$meanSSL))
pal <- colorNumeric(
  palette = "YlGnBu",
  domain = df.polygon2$meanSSL
)

map2<-leaflet() %>%
  addProviderTiles("CartoDB.Positron") %>%
  addPolygons(data = df.polygon2, 
              fillColor = ~pal(meanSSL), 
              color = "#b2aeae", # you need to use hex colors
              fillOpacity = 0.7, 
              weight = 1, 
              smoothFactor = 0.2,
              popup = popup) %>%
  addLegend(pal = pal, 
            values = df.polygon2$meanSSL, 
            position = "bottomright", 
            title = "Mean SSL Score") %>%
  saveWidget(here::here("html", "leaflet-SSL-map1.html"))
```
<iframe src="/html/leaflet-SSL-map1.html" height="600px" width="100%" style="border:none;"></iframe>

Much, much nicer than the ggplot version above. The proportions are
right, there’s a reference map underneath, popup information, and you
can zoom in and out. Looking at the distribution, there don’t appear to
be any distinct trends in mean SSL across the city (similar to before),
but we can see there are clearly individual tracts that jump out. As I
discussed in the previous COVID-19 vaccine allocation post, a lot of
those census tract discrepancies would be quickly lost if one were to
bin by postal code, community, or another larger area. I am definitely
interested to compare this to other census data by tract.

Now let’s look at a similar map focusing on count of arrests by census
tract.

``` r
pal2 <- colorNumeric(
  palette = "YlGnBu",
  domain = df.polygon2$count
)

map3 <- leaflet() %>%
  addProviderTiles("CartoDB.Positron") %>%
  addPolygons(data = df.polygon2, 
              fillColor = ~pal2(count), 
              color = "#b2aeae", # you need to use hex colors
              fillOpacity = 0.7, 
              weight = 1, 
              smoothFactor = 0.2,
              popup = popup) %>%
  addLegend(pal = pal2, 
            values = df.polygon2$count, 
            position = "bottomright", 
            title = "Count of Arrests") %>%
  saveWidget(here::here("html", "leaflet-SSL-map2.html"))
```
<iframe src="/html/leaflet-SSL-map2.html" height="600px" width="100%" style="border:none;"></iframe>

Here we see a few areas of the city immediately jump out. This obviously
leads into a much broader discussion about which areas of the city see
more arrests and why. That goes far outside the scope of one blog post,
but suffice to say the data will be heavily influenced by policing
practices and priorities. As I mentioned, overlaying this with other
census data would likely be enlightening as well. It’s also interesting
to note that the areas with the highest arrest counts don’t have an
immediate visual correlation with the mean SSL map above. Perhaps
something I will examine further in another post.

------------------------------------------------------------------------

## Conclusion

This post focused mostly on mapping data by census tract, rather than
digging into the SSL data further. A reminder that the [To Surveil and
Predict](https://citizenlab.ca/2020/09/to-surveil-and-predict-a-human-rights-analysis-of-algorithmic-policing-in-canada/)
report by Kate Robertson, Cynthia Khoo, and Yolanda Song is an excellent
resource for the issues that arise with algorithmic policing. And in
[this post](https://johnw-14.github.io/2021/04/27/CHICAGO-SSL-1.html) I
ran some basic statistical tests on the SSL data.

From a mapping perspective, we see how useful the leaflet package can be
for visualizing census tract data compared to forcing things together
for use in ggplot. There don’t appear to be high level trends in mean
SSL score by region in Chicago, but we do see certain census tracts
where the average stands out. When mapping by count of arrests, there is
a distinct set of census tracts with higher counts than the rest of the
city. As I alluded to earlier, it could be interesting to overlay either
of these measures with other census data, such as income level. Keeping
in mind that it wouldn’t exactly be ground-breaking to see arrests
and/or high SSL scores correlate with lower average income. There are
far larger criminal justice issues to discuss there, none of which can
be resolved with policing data alone.

Anyway, I hope you appreciated the improved maps as much as I did.
Thanks for reading. I will hopefully be back before long.
