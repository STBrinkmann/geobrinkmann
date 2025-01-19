---
title: Nürnberg - 3. Greenness Accessibility Index (GACI)
author: ''
date: '2025-01-15'
slug: nuernberg-greenness-accessibility-index
categories: ["R"]
tags: ["Nürnberg", "GACI", "R", "Greenness Accessibility", "Greenspace", "CGEI"]
summary: "By integrating realistic walking routes and park size weighting, the Greenspace Accessibility Index (GACI) offers a more practical and community-focused measure of how easily residents can reach quality public parks."
output: blogdown::html_page
image:
  focal_point: ""
  placement: 2
  preview_only: true
bibliography: references.bib
---

## Introduction

The **accessibility of greenspace** is a critical factor for urban health and well-being, focusing on how easily people can physically or legally reach parks, community gardens, or forests. Scholars typically assess accessibility by measuring walking or driving distance to the nearest park - a process that can rely on either **Euclidean** or **road network** analyses. However, information on structural or legal access to these areas is not always readily available, particularly at larger geographic scales (Labib, Lindley, and Huck 2020). Despite these challenges, greenspace accessibility is highly relevant for both **instoration** (building capacities) and **restoration** (restoring capacities) (Markevych et al. 2017).

Research indicates that **access to green areas** can foster **social cohesion**, described as a sense of belonging and mutual respect among neighbors (Holtan, Dieterlen, and Sullivan 2014; Weinstein et al. 2015). Greater social cohesion has in turn been linked to **improved mental health outcomes** and enhanced well-being (Fone et al. 2014; Williams et al. 2020), potentially moderating the impacts of stressful life events (Kingsbury et al. 2019). Alongside these social benefits, accessible greenspaces also promote **physical activity** - an important tool for managing mental health conditions and boosting overall health (Lachowycz and Jones 2011; Duncan et al. 2014; McGrath, Hopkins, and Hinckson 2015). Studies suggest that “green exercise” yields higher psychological gains than similar activities in less vegetated environments (Pretty et al. 2005; Mitchell 2013).

Nonetheless, not all greenspaces are equally beneficial for social interaction or physical activity. A **large public park** with amenities to support community gatherings, for instance, may offer different health benefits compared to a **small patch of roadside greenery** (Markevych et al. 2017). Therefore, refined measures of greenspace accessibility - such as location-specific amenities, park size, and the presence of safe walkable routes - are needed to understand how and why some spaces provide stronger instoration and restoration benefits than others (Giles-Corti et al. 2005).

## Data

The relevant spatial datasets have been uploaded on GitHub: administrative boundaries (AOI), parks and water mask shapefiles.

``` r
library(sf)
library(dplyr)
library(terra)
library(CGEI)

data_repo <- "https://github.com/STBrinkmann/SpatData_Nbg/raw/refs/heads/master/01_analysis/0102_data/02_processed/"

# AOI
aoi <- read_sf(paste0(data_repo, "01_Nbg_Bezirke.gpkg")) %>%
  filter(code_bez != "97") %>% # This is not available in the GAVI
  group_by(code_bez, name_bez) %>%
  summarise() %>%
  ungroup()

# Water mask
water_mask <- read_sf(paste0(data_repo, "02_Nbg_Water.gpkg"))

# Parks (>1ha)
osm_parks <- read_sf(paste0(data_repo, "02_Nbg_Greenspace.gpkg")) %>%
  mutate(name = stringr::str_pad(1:n(), 3, pad = "0")) %>%
  mutate(size_ha = st_area(geom) %>%
           units::set_units(value = ha) %>%
           as.numeric()) %>%
  filter(size_ha > 1) %>%
  mutate(size_log = log(size_ha)) %>%
  relocate(geom, .after = last_col())
```

<img src="{{< blogdown/postref >}}index.en_files/figure-html/unnamed-chunk-2-1.png" width="960" />

## GACI

The **Greenspace Accessibility Index (GACI)** (Labib, Lindley, and Huck 2021) quantifies the ease with which the public can access **urban parks**. Rather than relying solely on Euclidean distance, GACI explicitly incorporates:

1.  **Park Access Points**  
    Identifiable entry sites - typically at the intersection of park boundaries with streets or footpaths - where users can legally and physically enter a park.

2.  **Network-Based Distance**  
    Walking distance is determined through road and footpath networks, capturing realistic travel times rather than simple straight-line distances.

3.  **Park Size Weighting**  
    Larger parks potentially attract users from a greater distance. Consequently, GACI considers both distance and the (log-transformed) size of the park.

4.  **Normalization and Classification**  
    Extreme values (e.g., very large distances) are trimmed to mitigate outliers, then final scores are **reclassified** - often via **Jenks natural breaks** - to yield an intuitive map of greenspace accessibility.

Together, these factors yield a spatial indicator that aligns more closely with **real-world access** than conventional 2D buffer analyses.

### 1. Identifying Park Access Points

Public parks are generally entered where roads or paths meet park boundaries. We **intersect** an OpenStreetMap (OSM)-based park polygon layer with an OSM road network. Where no intersection is found, park **centroids** serve as fallback access points. This step ensures each park has at least one feasible entry location.

``` r
# --- 1. Generate Road Network & Park Access Points ---

# A. Create buffered AOI boundary to mitigate edge effects
aoi_bbox <- aoi %>%
  st_buffer(5000) %>%
  st_transform(4326) %>%
  st_bbox()

# B. Download and filter OSM road network for walkable paths
osm_roads_raw <- opq(bbox = aoi_bbox) %>%
  add_osm_feature(key = "highway") %>%
  osmdata_sf(quiet = FALSE) %>%
  osm_poly2line()

osm_roads <- osm_roads_raw$osm_lines %>%
  select(highway) %>%
  filter(!highway %in% c("motorway", "motorway_link", "trunk", "trunk_link", "raceway")) %>%
  st_transform(st_crs(aoi)) %>%
  st_intersection(st_buffer(aoi, 5000))

# C. Segment roads for higher intersection accuracy
cores <- 12
if (Sys.info()[["sysname"]] == "Windows") {
  cl <- parallel::makeCluster(cores)
  osm_roads <- suppressWarnings(split(osm_roads, seq(from = 1, to = nrow(osm_roads), by = 200)))
  osm_roads <- parallel::parLapply(cl, osm_roads, function(x){
    nngeo::st_segments(sf::st_cast(x, "LINESTRING"), progress = FALSE)
  })
  parallel::stopCluster(cl)
} else {
  osm_roads <- suppressWarnings(split(osm_roads, seq(from = 1, to = nrow(osm_roads), by = 200))) %>%
    parallel::mclapply(function(x){
      nngeo::st_segments(sf::st_cast(x, "LINESTRING"), progress = FALSE)
    },
    mc.cores = cores, mc.preschedule = TRUE)
}

osm_roads <- st_as_sf(dplyr::as_tibble(data.table::rbindlist(osm_roads)))
osm_roads <- st_set_geometry(osm_roads, "geom")

st_geometry(osm_roads) <- st_geometry(osm_roads) %>%
  lapply(function(x) round(x, 0)) %>%
  st_sfc(crs = st_crs(osm_roads))

# D. Derive park centroids and road intersections
osm_park_centroids <- osm_parks %>%
  st_centroid() %>%
  select(name, size_ha, size_log)

# Helper function
st_multipoint_to_point <- function(x) {
  x.points <- x[which(sf::st_geometry_type(x) == "POINT"), ]
  x.multipoints <- x[which(sf::st_geometry_type(x) == "MULTIPOINT"), ]

  for (i in 1:nrow(x.multipoints)) {
    x.points <- x.points %>%
      dplyr::add_row(sf::st_cast(x.multipoints[i, ], "POINT"))
  }

  return(x.points)
}

osm_park_road_interesect <- osm_parks %>%
  st_boundary() %>%
  st_intersection(osm_roads) %>%
  select(name, size_ha, size_log) %>%
  st_multipoint_to_point()

# E. Combine both into final access point layer
park_accesspoints <- rbind(osm_park_centroids, osm_park_road_interesect)
```

### 2. Preparing Observer Locations

To represent potential origins of park visitors, we use a **10 m resolution grid** covering the entire area of interest. Points located **within parks** inherently have optimal accessibility, thus are excluded from further computation. Similarly, we remove points in water bodies.

``` r
# --- 2. Prepare Observer Locations ---

# A. Create a regular grid (10m resolution)
observer <- st_make_grid(aoi, cellsize = 50, what = "centers") %>%
  st_as_sf() %>%
  st_set_geometry("geom")
observer <- observer[aoi,]
observer <- observer %>%
  mutate(id = 1:n()) %>%
  relocate(id, .after = last_col())

# B. Remove points inside parks
obs_in_parks <- observer[osm_parks, ]
observer <- observer[-obs_in_parks$id, ]

# C. Remove points in water bodies
observer <- observer[-(observer[water_mask, ]$id), ]
```

### 3. Calculating Network-Based Distance Using OSRM

We use the **Open Source Routing Machine (OSRM)** to derive walking distances via a **road network**. OSRM offers **fast and efficient** routing by pre-processing large OSM data sets and applying advanced routing algorithms (e.g., Multi-Level Dijkstra).

Below is an example workflow for **downloading** the relevant OSM `.pbf` (protocol buffer) file (here, the Mittelfranken region of Germany) and **running OSRM** in a **Docker** container. Ensure Docker is installed beforehand ([docs.docker.com/engine/install/ubuntu](https://docs.docker.com/engine/install/ubuntu)).

``` r
# Download OSM data for your region of interest
if(!file.exists("01_analysis/0101_data/osrm/mittelfranken-latest.osm.pbf")){
  dir.create("01_analysis/0101_data/osrm/", showWarnings = FALSE)
  default_timeout <- getOption("timeout")
  options(timeout = max(600, default_timeout))
  download.file(
    url = "https://download.geofabrik.de/europe/germany/bayern/mittelfranken-latest.osm.pbf",
    destfile = "01_analysis/0101_data/osrm/mittelfranken-latest.osm.pbf",
    mode="wb"
  )
  options(timeout = default_timeout)
}

# OSRM Docker commands:
# 1) Navigate to the appropriate folder:
#    cd 01_analysis/0101_data/osrm/
# 2) Extract, partition, and customize:
#    sudo docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-extract -p /opt/foot.lua /data/mittelfranken-latest.osm.pbf
#    sudo docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-partition /data/mittelfranken-latest.osrm
#    sudo docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-customize /data/mittelfranken-latest.osrm
# 3) Run the OSRM server:
#    sudo docker run -t -i -p 5000:5000 -v "${PWD}:/data" osrm/osrm-backend osrm-routed --max-table-size=50000 --algorithm mld /data/mittelfranken-latest.osrm
```

After setting up OSRM, we retrieve walking distances for each **observer location** to the **k nearest** park access points, and compute the walking distance or time. These distance values are subsequently **weighted** by park size, recognizing that larger parks may attract users from farther afield (Giles-Corti et al. 2005).

``` r
# --- 3. Network Analysis with OSRM ---

# The OSRM server is run locally via Docker (setup described earlier).

observer_park_duration <- function(j){
  # Identify k-nearest access points
  knn_access <- nngeo::st_nn(
    observer[j,],
    park_accesspoints,
    k = 2 * round(sqrt(nrow(park_accesspoints))),
    progress = FALSE
  ) %>% unlist()

  # Compute travel times using OSRM
  duration_table <- osrmTable(
    src = observer[j,],
    dst = park_accesspoints[knn_access,],
    osrm.server = "http://0.0.0.0:5000/",
    osrm.profile = "foot"
  )

  # Select best match (close & large)
  out <- tibble(
    id = observer[j,]$id,
    dist = as.numeric(duration_table$durations),
    size_ha = park_accesspoints[knn_access,]$size_ha,
    size_log = park_accesspoints[knn_access,]$size_log,
    name = park_accesspoints[knn_access,]$name
  ) %>%
    group_by(name) %>%
    filter(dist == min(dist)) %>%
    distinct() %>%
    ungroup() %>%
    # Normalize distance & size values
    mutate(dist_01 = dist / mean(dist) * 100,
           size_01 = size_log / mean(size_log) * 100) %>%
    mutate(dist_01 = 1 - dist_01 / max(dist_01),
           size_01 = size_01 / max(size_01)) %>%
    # Weight by distance vs. park size
    mutate(w = (1.91 / 0.85 * dist_01 + size_01) / 2) %>%
    filter(w == max(w)) %>%
    select(id, name, dist, size_ha, w)

  return(out)
}

# Split observer set into manageable chunks
obs_seq <- split(1:nrow(observer), ceiling(seq_along(1:nrow(observer))/250))

# Initialize an output CSV
tibble(
  id = as.numeric(),
  name = as.character(),
  dist = as.numeric(),
  size_ha = as.numeric(),
  w = as.numeric()
) %>%
  readr::write_csv("01_analysis/0101_data/osrm/access_table.csv", col_names = TRUE)

# Parallel loop over observer subsets
pb = pbmcapply::progressBar(min = 0, max = length(obs_seq))
stepi = 1
for(this_seq in obs_seq){
  dist_table <- parallel::mclapply(this_seq, observer_park_duration, mc.cores = 18) %>%
    do.call(rbind, .)

  readr::write_csv(dist_table, "01_analysis/0101_data/osrm/access_table.csv", append = TRUE)
  setTxtProgressBar(pb, stepi)
  stepi = stepi + 1
}
close(pb)

# Merge distance results with the observer SF
dist_table <- readr::read_csv("01_analysis/0101_data/osrm/access_table.csv")
dist_sf <- inner_join(observer, dist_table)

# Terminate OSRM server and cleanup if desired
unlink("01_analysis/0101_data/osrm/", recursive = TRUE)
```

### 4. Generating the GACI Raster

Finally, the weighted accessibility values (`w`) are **interpolated** into a continuous **10 m raster** using an **Inverse Distance Weighting (IDW)** approach, capping extreme values and **reclassifying** with **Jenks** to facilitate map visualization. Areas with lowest travel times to largest parks receive the most favorable accessibility scores.

``` r
# --- 4. Convert Access Values to GACI Raster ---

# A. Interpolate to 10m grid using IDW
gaci_raw <- CGEI::sf_interpolat_IDW(
  observer = dist_sf,
  v = "w",
  aoi = aoi,
  max_distance = Inf,
  n = 20,
  raster_res = 10,
  na_only = TRUE,
  cores = 22,
  progress = TRUE
)

# B. Park cells: assign ideal accessibility
park_cells <- terra::extract(gaci_raw, osm_parks, cells = TRUE, ID = FALSE)$cell
gaci_raw[park_cells] <- NA

# C. Reclassify via Jenks for readability
gaci <- CGEI:::reclassify_jenks(gaci_raw, n_classes = 9)
gaci[park_cells] <- 9

gaci <- gaci %>%
  crop(aoi) %>%
  mask(aoi)
names(gaci) <- "GACI"
```

Below is the **final GACI map** for Nürnberg, indicating how readily residents can access public parks at each location. Areas in **red** reflect relatively low accessibility (longer walking distances to suitably large parks), whereas **green** signifies more favorable access. Unsurprisingly, the **city center** exhibits poorer scores, likely due to limited large parks and a denser built environment. By contrast, districts near the **eastern greenbelt** and forested zones (e.g., Reichswald) show stronger accessibility, thanks to extensive parkland and well-connected footpaths. Furthermore, the **Wöhrder Wiese** and **Wöhrder See,** as well as the **Pegnitz** in the west (close to the *Westfriedhof*) form a crucial green belt, ensuring many neighborhoods in densely populated areas benefit from relatively high accessibility.

Overall, the GACI reveals not just *where* parks are lacking, but also *how easily* citizens can reach them - an essential insight for guiding equitable urban planning and public health initiatives.

<img src="{{< blogdown/postref >}}index.en_files/figure-html/unnamed-chunk-9-1.png" width="768" />

## References

<div id="refs" class="references csl-bib-body hanging-indent" entry-spacing="0">

<div id="ref-duncan2014" class="csl-entry">

Duncan, Michael, Neil Clarke, Samantha Birch, Jason Tallis, Joanne Hankey, Elizabeth Bryant, and Emma Eyre. 2014. “The Effect of Green Exercise on Blood Pressure, Heart Rate and Mood State in Primary School Children.” *International Journal of Environmental Research and Public Health* 11 (4): 3678–88. <https://doi.org/10.3390/ijerph110403678>.

</div>

<div id="ref-fone2014" class="csl-entry">

Fone, D., J. White, D. Farewell, M. Kelly, G. John, K. Lloyd, G. Williams, and F. Dunstan. 2014. “Effect of Neighbourhood Deprivation and Social Cohesion on Mental Health Inequality: A Multilevel Population-Based Longitudinal Study.” *Psychological Medicine* 44 (11): 2449–60. <https://doi.org/10.1017/s0033291713003255>.

</div>

<div id="ref-giles-corti2005" class="csl-entry">

Giles-Corti, Billie, Melissa H. Broomhall, Matthew Knuiman, Catherine Collins, Kate Douglas, Kevin Ng, Andrea Lange, and Robert J. Donovan. 2005. “Increasing Walking.” *American Journal of Preventive Medicine* 28 (2): 169–76. <https://doi.org/10.1016/j.amepre.2004.10.018>.

</div>

<div id="ref-holtan2014" class="csl-entry">

Holtan, Meghan T., Susan L. Dieterlen, and William C. Sullivan. 2014. “Social Life Under Cover.” *Environment and Behavior* 47 (5): 502–25. <https://doi.org/10.1177/0013916513518064>.

</div>

<div id="ref-kingsbury2019" class="csl-entry">

Kingsbury, Mila, Zahra Clayborne, Ian Colman, and James B. Kirkbride. 2019. “The Protective Effect of Neighbourhood Social Cohesion on Adolescent Mental Health Following Stressful Life Events.” *Psychological Medicine* 50 (8): 1292–99. <https://doi.org/10.1017/s0033291719001235>.

</div>

<div id="ref-labib2020_review" class="csl-entry">

Labib, S. M., Sarah Lindley, and Jonny J. Huck. 2020. “Spatial Dimensions of the Influence of Urban Green-Blue Spaces on Human Health: A Systematic Review.” *Environmental Research* 180 (January): 108869. <https://doi.org/10.1016/j.envres.2019.108869>.

</div>

<div id="ref-labib2021_greenspaceExposure" class="csl-entry">

———. 2021. “Estimating Multiple Greenspace Exposure Types and Their Associations with Neighbourhood Premature Mortality: A Socioecological Study.” *Science of The Total Environment* 789 (October): 147919. <https://doi.org/10.1016/j.scitotenv.2021.147919>.

</div>

<div id="ref-lachowycz2011" class="csl-entry">

Lachowycz, K., and A. P. Jones. 2011. “Greenspace and Obesity: A Systematic Review of the Evidence.” *Obesity Reviews* 12 (5): e183–89. <https://doi.org/10.1111/j.1467-789x.2010.00827.x>.

</div>

<div id="ref-markevych2017" class="csl-entry">

Markevych, Iana, Julia Schoierer, Terry Hartig, Alexandra Chudnovsky, Perry Hystad, Angel M. Dzhambov, Sjerp de Vries, et al. 2017. “Exploring Pathways Linking Greenspace to Health: Theoretical and Methodological Guidance.” *Environmental Research* 158 (October): 301–17. <https://doi.org/10.1016/j.envres.2017.06.028>.

</div>

<div id="ref-mcgrath2015" class="csl-entry">

McGrath, Leslie J., Will G. Hopkins, and Erica A. Hinckson. 2015. “Associations of Objectively Measured Built-Environment Attributes with Youth ModerateVigorous Physical Activity: A Systematic Review and Meta-Analysis.” *Sports Medicine* 45 (6): 841–65. <https://doi.org/10.1007/s40279-015-0301-3>.

</div>

<div id="ref-mitchell2013" class="csl-entry">

Mitchell, Richard. 2013. “Is Physical Activity in Natural Environments Better for Mental Health Than Physical Activity in Other Environments?” *Social Science & Medicine* 91 (August): 130–34. <https://doi.org/10.1016/j.socscimed.2012.04.012>.

</div>

<div id="ref-pretty2005" class="csl-entry">

Pretty, Jules, Jo Peacock, Martin Sellens, and Murray Griffin. 2005. “The Mental and Physical Health Outcomes of Green Exercise.” *International Journal of Environmental Health Research* 15 (5): 319–37. <https://doi.org/10.1080/09603120500155963>.

</div>

<div id="ref-weinstein2015" class="csl-entry">

Weinstein, Netta, Andrew Balmford, Cody R. DeHaan, Valerie Gladwell, Richard B. Bradbury, and Tatsuya Amano. 2015. “Seeing Community for the Trees: The Links Among Contact with Natural Environments, Community Cohesion, and Crime.” *BioScience* 65 (12): 1141–53. <https://doi.org/10.1093/biosci/biv151>.

</div>

<div id="ref-williams2020" class="csl-entry">

Williams, Andrew James, Kath Maguire, Karyn Morrissey, Tim Taylor, and Katrina Wyatt. 2020. “Social Cohesion, Mental Wellbeing and Health-Related Quality of Life Among a Cohort of Social Housing Residents in Cornwall: A Cross Sectional Study.” *BMC Public Health* 20 (1). <https://doi.org/10.1186/s12889-020-09078-6>.

</div>

</div>
