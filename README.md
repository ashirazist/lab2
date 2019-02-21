---
title: "STAT585 Lab#2"
author: "Team #10"
date: "February 20, 2019"
output:
  html_document:
    toc: true
    theme: united
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

### Prerequisite
```{r load package, message=F, results='hide'}
package_required <- c("ggplot2", "sf", "ggspatial","maps","maptools","rgeos","purrr","tidyverse")
sapply(package_required, require, character.only=TRUE)
```

# 1. Tolkien Middle Earth
```{r Tolkien Middle Earth}
p <- ggplot() +
  geom_sf(data = read_sf("data/ME-GIS/Coastline2.shp"), 
          colour="grey10", fill="grey90") +
  geom_sf(data = read_sf("data/ME-GIS/Rivers19.shp"), 
          colour="steelblue", size=0.3) +
  geom_sf(data = read_sf("data/ME-GIS/PrimaryRoads.shp"), 
          size = 0.7, colour="grey30") +
  geom_sf(data = read_sf("data/ME-GIS/Cities.shp")) +
  theme_bw()

p <- p +  geom_sf_text(aes(label = Name), data = read_sf("data/ME-GIS/Cities.shp")) +
  annotation_scale() +
  annotation_north_arrow()

print(p)
```

# 2. Australia map

```{r input shp data}
# sf read file
ozbig <- read_sf("data/gadm36_AUS_shp/gadm36_AUS_1.shp")

# thin the number of points used
oz_st <- maptools::thinnedSpatialPoly(
  as(ozbig, "Spatial"), tolerance = 0.1, 
  minarea = 0.001, topologyPreserve = TRUE)
oz <- st_as_sf(oz_st)
```

## 2.1 Take a glance

```{r glance}
head(oz)
oz$geometry[[1]]
is.list(oz$geometry[[1]])
oz$geometry[[1]][[1]]
class(oz$geometry[[1]][[1]][[1]])
```

* The result of `oz$geometry[[1]]` shows each of element in the `geometry` column is a `MULTIPOLYGON` object containning multiple lists. 
* The result of `is.list(oz$geometry[[1]])` shows the `MULTIPOLYGON` object is a `list` object
* The result of `oz$geometry[[1]][[1]]` shows each of this list has one sub-list containing a data with some structure.
* The result of `class(oz$geometry[[1]][[1]][[1]])` shows the data stored in the sub-lists is in `matrix` form.

## 2.2 Solution to extract the geometry infomation

* As identifying the `MULTIPOLYGON` objects are three-level `list` objects, we decide to reshape the objects with method of `purrr::flatten` to remove the top level of indexes.
* Then use `purrr::unnest` to remove the second level. 
* To remove the last level, we change the data structure from `list` to `tibble` (or `data.frame`) so that the use of `purrr:unnest` can extract the `x` and `y` in the `matrix` and paste them into two new columns, respectively.  

```{r reshape}
ozplus <- oz %>% select(NAME_1, geometry) %>% 
  group_by() %>% 
  mutate(coord = geometry %>% map(.f = function(m) flatten(.x=m)),
         region = row_number()) %>% 
  unnest

st_geometry(ozplus) <- NULL
ozplus <- ozplus %>% mutate(coord = coord %>% map(.f = function(m) as_tibble(m)),
                            group = row_number()) %>% 
  unnest
```

```{r plotS}
ozplus %>%  setNames(c("name", "region","group", "long", "lat")) %>% 
  ggplot(aes(x = long, y = lat, group = group)) + geom_polygon(color = "black")
```
