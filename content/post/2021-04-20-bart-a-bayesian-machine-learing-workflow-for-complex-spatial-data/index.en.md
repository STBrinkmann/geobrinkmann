---
title: BART - A Bayesian machine learning workflow for complex spatial data
author: Sebastian Brinkmann
date: '2021-04-20'
slug: bart-a-bayesian-machine-learning-workflow-for-complex-spatial-data
categories: ["R"]
tags: ["R", "Bayesian", "Machine learning", "Health Geography", "COVID-19"]
subtitle: ''
summary: 'In our recent publication from 2020 we analyzed COVID-19 incidence rates using a multimethod approach. In this post I will present BART - a Bayesian machine learning algorithm - to model COVID-19 incidence rates.'
authors: []
lastmod: '2021-04-20T15:54:53+02:00'
featured: yes
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
output: html_document
bibliography: references.bib
---

<script src="{{< blogdown/postref >}}index.en_files/kePrint/kePrint.js"></script>
<link href="{{< blogdown/postref >}}index.en_files/lightable/lightable.css" rel="stylesheet" />
<script src="{{< blogdown/postref >}}index.en_files/kePrint/kePrint.js"></script>

<link href="{{< blogdown/postref >}}index.en_files/lightable/lightable.css" rel="stylesheet" />

<style>
body {
text-align: justify}
</style>

In our [recent publication from 2020](https://geobrinkmann.com/publication/scarpone2020/), we analyzed COVID-19 incidence and socioeconomic, infrastructural, and built environment characteristics using a multimethod approach. Germany has 401 counties, which vary greatly in size. Counties in Southern Germany are generally smaller and have higher population densities. The figure below shows the natural log-transformed, age-adjusted COVID-19 incidence rates as of April 1, 2020, highlighting spatial differences between the northeast and the south-southwest.
<img src="Incidence_rates_map.svg" style="width:60.0%" />

After conducting a thorough spatial exploratory analysis, we applied a *Bayesian Additive Regression Trees* (BART; (Chipman, George, and McCulloch 2010)) model to identify socioeconomic and built environment covariates associated with COVID-19 incidence. BART is an ensemble-of-trees method, similar to random forests (Breiman 2001) and stochastic gradient boosting (Friedman 2002). *Tree-based regression models* are flexible enough to capture interactions and non-linearities. Summation-of-trees models (such as BART or random forests) can capture even more complex relationships than single-tree models. However, BART differs from these common algorithms because it uses an underlying *Bayesian probability* model rather than a purely algorithmic approach (Kapelner and Bleich 2016). One key advantage of the Bayesian framework is that it computes posterior distributions to approximate the model parameters. These priors help prevent any single regression tree from dominating the ensemble, thus lowering the risk of overfitting (Kapelner and Bleich 2016; Scarpone et al. 2020).

For our analysis, we used the R package `bartMachine` (Kapelner and Bleich 2016). One caveat is that `bartMachine` does not support *tibble* objects as input features, but aside from that it is intuitive and simple to use.  
In this post, I will illustrate how we used the same data from our COVID-19 paper to analyze age-adjusted incidence rates in German counties using BART. Even though the BART model has strong predictive capabilities, we will *not* use it to forecast new COVID-19 cases. Instead, we will use it as an exploratory tool to understand which factors influence the spread of COVID-19 and how they interact with incidence rates.

I want to emphasize that we are only examining data from a single date, April 1, 2020, covering the first wave of the pandemic in Germany. A friend of mine is currently investigating which factors contributed to the second and third waves. Although some variables remain important across different time points, others gain or lose relevance, and in some cases the direction of their effects appears reversed.

In sum, BART is used here as an exploratory tool, and we will generate *Partial Dependence Plots (PDPs)* to visualize and interpret the marginal effects of key predictors.

If you are interested in more detail about pre-modeling exploratory data analysis or data engineering, please see the [paper](https://geobrinkmann.com/publication/scarpone2020/) or [contact me](https://geobrinkmann.com/#contact).

## Data download and pre-processing

First, we need to load the packages and set the memory size and number of cores for `bartMachine`.

``` r
# Set to use 45 GB memory - adjust this to your resources
options(java.parameters = "-Xmx45g")

# Load packages
library(bartMachine)
library(dplyr)

# Set to run on 20 threads - adjust this to your resources
set_bart_machine_num_cores(20)
```

    ## bartMachine now using 20 cores.

*Tip:* Double-check that the correct amount of RAM is being allocated after loading `bartMachine.` If the message shows a different number, try manually typing out the `java.parameters` string instead of copy-pasting.

Next, we download the data and normalize the church density variable (`Rch_den`).e.

``` r
# Function for linear stretching. New range: 0-1
range0_1 <- function(x){(x-min(x))/(max(x)-min(x))}

# Download data
data <- read.csv("https://github.com/CHEST-Lab/BART_Covid-19/raw/master/Data/GermanyTraining.csv",
                 stringsAsFactors = F) %>%
  mutate(Rch_den = range0_1(Rch_den),
         NUTS2_Fact = as.factor(NUTS2_Fact), 
         BL_ID = as.factor(BL),
         S_109 = as.factor(S_109))

# Select variables: Lat/ Long, BL, NUTS2, socioeconomic, build environment and age adjusted case rate
data <- data[c(374, 3, 4, 5, 38, 28, 65:372)]
```

<table style="width:90%; font-family: &quot;Arial Narrow&quot;, &quot;Source Sans Pro&quot;, sans-serif; width: auto !important; margin-left: auto; margin-right: auto;" class=" lightable-classic lightable-striped">
<thead>
<tr>
<th style="text-align:right;font-weight: bold;">
AdjRate
</th>
<th style="text-align:right;font-weight: bold;">
X
</th>
<th style="text-align:right;font-weight: bold;">
Y
</th>
<th style="text-align:left;font-weight: bold;">
NUTS2_Fact
</th>
<th style="text-align:left;font-weight: bold;">
BL_ID
</th>
<th style="text-align:right;font-weight: bold;">
EWZ
</th>
<th style="text-align:right;font-weight: bold;">
S_001
</th>
<th style="text-align:right;font-weight: bold;">
S_002
</th>
<th style="text-align:right;font-weight: bold;">
S_003
</th>
<th style="text-align:right;font-weight: bold;">
S_004
</th>
<th style="text-align:right;font-weight: bold;">
S_005
</th>
<th style="text-align:right;font-weight: bold;">
S_006
</th>
<th style="text-align:right;font-weight: bold;">
S_007
</th>
<th style="text-align:right;font-weight: bold;">
S_008
</th>
<th style="text-align:right;font-weight: bold;">
S_009
</th>
<th style="text-align:right;font-weight: bold;">
S_010
</th>
<th style="text-align:right;font-weight: bold;">
S_011
</th>
<th style="text-align:right;font-weight: bold;">
S_012
</th>
<th style="text-align:right;font-weight: bold;">
S_013
</th>
<th style="text-align:right;font-weight: bold;">
S_014
</th>
<th style="text-align:right;font-weight: bold;">
S_015
</th>
<th style="text-align:right;font-weight: bold;">
S_016
</th>
<th style="text-align:right;font-weight: bold;">
S_017
</th>
<th style="text-align:right;font-weight: bold;">
S_018
</th>
<th style="text-align:right;font-weight: bold;">
S_019
</th>
<th style="text-align:right;font-weight: bold;">
S_020
</th>
<th style="text-align:right;font-weight: bold;">
S_021
</th>
<th style="text-align:right;font-weight: bold;">
S_022
</th>
<th style="text-align:right;font-weight: bold;">
S_023
</th>
<th style="text-align:right;font-weight: bold;">
S_024
</th>
<th style="text-align:right;font-weight: bold;">
S_025
</th>
<th style="text-align:right;font-weight: bold;">
S_026
</th>
<th style="text-align:right;font-weight: bold;">
S_027
</th>
<th style="text-align:right;font-weight: bold;">
S_028
</th>
<th style="text-align:right;font-weight: bold;">
S_029
</th>
<th style="text-align:right;font-weight: bold;">
S_030
</th>
<th style="text-align:right;font-weight: bold;">
S_031
</th>
<th style="text-align:right;font-weight: bold;">
S_032
</th>
<th style="text-align:right;font-weight: bold;">
S_033
</th>
<th style="text-align:right;font-weight: bold;">
S_034
</th>
<th style="text-align:right;font-weight: bold;">
S_035
</th>
<th style="text-align:right;font-weight: bold;">
S_036
</th>
<th style="text-align:right;font-weight: bold;">
S_037
</th>
<th style="text-align:right;font-weight: bold;">
S_038
</th>
<th style="text-align:right;font-weight: bold;">
S_039
</th>
<th style="text-align:right;font-weight: bold;">
S_040
</th>
<th style="text-align:right;font-weight: bold;">
S_041
</th>
<th style="text-align:right;font-weight: bold;">
S_042
</th>
<th style="text-align:right;font-weight: bold;">
S_043
</th>
<th style="text-align:right;font-weight: bold;">
S_044
</th>
<th style="text-align:right;font-weight: bold;">
S_045
</th>
<th style="text-align:right;font-weight: bold;">
S_046
</th>
<th style="text-align:right;font-weight: bold;">
S_047
</th>
<th style="text-align:right;font-weight: bold;">
S_048
</th>
<th style="text-align:right;font-weight: bold;">
S_049
</th>
<th style="text-align:right;font-weight: bold;">
S_050
</th>
<th style="text-align:right;font-weight: bold;">
S_051
</th>
<th style="text-align:right;font-weight: bold;">
S_052
</th>
<th style="text-align:right;font-weight: bold;">
S_053
</th>
<th style="text-align:right;font-weight: bold;">
S_054
</th>
<th style="text-align:right;font-weight: bold;">
S_055
</th>
<th style="text-align:right;font-weight: bold;">
S_056
</th>
<th style="text-align:right;font-weight: bold;">
S_057
</th>
<th style="text-align:right;font-weight: bold;">
S_058
</th>
<th style="text-align:right;font-weight: bold;">
S_059
</th>
<th style="text-align:right;font-weight: bold;">
S_060
</th>
<th style="text-align:right;font-weight: bold;">
S_061
</th>
<th style="text-align:right;font-weight: bold;">
S_062
</th>
<th style="text-align:right;font-weight: bold;">
S_063
</th>
<th style="text-align:right;font-weight: bold;">
S_064
</th>
<th style="text-align:right;font-weight: bold;">
S_065
</th>
<th style="text-align:right;font-weight: bold;">
S_066
</th>
<th style="text-align:right;font-weight: bold;">
S_067
</th>
<th style="text-align:right;font-weight: bold;">
S_068
</th>
<th style="text-align:right;font-weight: bold;">
S_069
</th>
<th style="text-align:right;font-weight: bold;">
S_070
</th>
<th style="text-align:right;font-weight: bold;">
S_071
</th>
<th style="text-align:right;font-weight: bold;">
S_072
</th>
<th style="text-align:right;font-weight: bold;">
S_073
</th>
<th style="text-align:right;font-weight: bold;">
S_074
</th>
<th style="text-align:right;font-weight: bold;">
S_075
</th>
<th style="text-align:right;font-weight: bold;">
S_076
</th>
<th style="text-align:right;font-weight: bold;">
S_077
</th>
<th style="text-align:right;font-weight: bold;">
S_078
</th>
<th style="text-align:right;font-weight: bold;">
S_079
</th>
<th style="text-align:right;font-weight: bold;">
S_080
</th>
<th style="text-align:right;font-weight: bold;">
S_081
</th>
<th style="text-align:right;font-weight: bold;">
S_082
</th>
<th style="text-align:right;font-weight: bold;">
S_083
</th>
<th style="text-align:right;font-weight: bold;">
S_084
</th>
<th style="text-align:right;font-weight: bold;">
S_085
</th>
<th style="text-align:right;font-weight: bold;">
S_086
</th>
<th style="text-align:right;font-weight: bold;">
S_087
</th>
<th style="text-align:right;font-weight: bold;">
S_088
</th>
<th style="text-align:right;font-weight: bold;">
S_089
</th>
<th style="text-align:right;font-weight: bold;">
S_090
</th>
<th style="text-align:right;font-weight: bold;">
S_091
</th>
<th style="text-align:right;font-weight: bold;">
S_092
</th>
<th style="text-align:right;font-weight: bold;">
S_093
</th>
<th style="text-align:right;font-weight: bold;">
S_094
</th>
<th style="text-align:right;font-weight: bold;">
S_095
</th>
<th style="text-align:right;font-weight: bold;">
S_096
</th>
<th style="text-align:right;font-weight: bold;">
S_097
</th>
<th style="text-align:right;font-weight: bold;">
S_098
</th>
<th style="text-align:right;font-weight: bold;">
S_099
</th>
<th style="text-align:right;font-weight: bold;">
S_100
</th>
<th style="text-align:right;font-weight: bold;">
S_101
</th>
<th style="text-align:right;font-weight: bold;">
S_102
</th>
<th style="text-align:right;font-weight: bold;">
S_103
</th>
<th style="text-align:right;font-weight: bold;">
S_104
</th>
<th style="text-align:right;font-weight: bold;">
S_105
</th>
<th style="text-align:right;font-weight: bold;">
S_106
</th>
<th style="text-align:right;font-weight: bold;">
S_107
</th>
<th style="text-align:right;font-weight: bold;">
S_108
</th>
<th style="text-align:left;font-weight: bold;">
S_109
</th>
<th style="text-align:right;font-weight: bold;">
S_110
</th>
<th style="text-align:right;font-weight: bold;">
S_111
</th>
<th style="text-align:right;font-weight: bold;">
S_112
</th>
<th style="text-align:right;font-weight: bold;">
S_113
</th>
<th style="text-align:right;font-weight: bold;">
S_114
</th>
<th style="text-align:right;font-weight: bold;">
S_115
</th>
<th style="text-align:right;font-weight: bold;">
S_116
</th>
<th style="text-align:right;font-weight: bold;">
S_117
</th>
<th style="text-align:right;font-weight: bold;">
S_118
</th>
<th style="text-align:right;font-weight: bold;">
S_119
</th>
<th style="text-align:right;font-weight: bold;">
S_120
</th>
<th style="text-align:right;font-weight: bold;">
S_121
</th>
<th style="text-align:right;font-weight: bold;">
S_122
</th>
<th style="text-align:right;font-weight: bold;">
S_123
</th>
<th style="text-align:right;font-weight: bold;">
S_124
</th>
<th style="text-align:right;font-weight: bold;">
S_125
</th>
<th style="text-align:right;font-weight: bold;">
S_126
</th>
<th style="text-align:right;font-weight: bold;">
S_127
</th>
<th style="text-align:right;font-weight: bold;">
S_128
</th>
<th style="text-align:right;font-weight: bold;">
S_129
</th>
<th style="text-align:right;font-weight: bold;">
S_130
</th>
<th style="text-align:right;font-weight: bold;">
S_131
</th>
<th style="text-align:right;font-weight: bold;">
S_132
</th>
<th style="text-align:right;font-weight: bold;">
S_133
</th>
<th style="text-align:right;font-weight: bold;">
S_134
</th>
<th style="text-align:right;font-weight: bold;">
S_135
</th>
<th style="text-align:right;font-weight: bold;">
S_136
</th>
<th style="text-align:right;font-weight: bold;">
S_137
</th>
<th style="text-align:right;font-weight: bold;">
S_138
</th>
<th style="text-align:right;font-weight: bold;">
S_139
</th>
<th style="text-align:right;font-weight: bold;">
S_140
</th>
<th style="text-align:right;font-weight: bold;">
S_141
</th>
<th style="text-align:right;font-weight: bold;">
S_142
</th>
<th style="text-align:right;font-weight: bold;">
S_143
</th>
<th style="text-align:right;font-weight: bold;">
S_144
</th>
<th style="text-align:right;font-weight: bold;">
S_145
</th>
<th style="text-align:right;font-weight: bold;">
S_146
</th>
<th style="text-align:right;font-weight: bold;">
S_147
</th>
<th style="text-align:right;font-weight: bold;">
S_148
</th>
<th style="text-align:right;font-weight: bold;">
S_149
</th>
<th style="text-align:right;font-weight: bold;">
S_150
</th>
<th style="text-align:right;font-weight: bold;">
S_151
</th>
<th style="text-align:right;font-weight: bold;">
S_152
</th>
<th style="text-align:right;font-weight: bold;">
S_153
</th>
<th style="text-align:right;font-weight: bold;">
S_154
</th>
<th style="text-align:right;font-weight: bold;">
S_155
</th>
<th style="text-align:right;font-weight: bold;">
S_156
</th>
<th style="text-align:right;font-weight: bold;">
S_157
</th>
<th style="text-align:right;font-weight: bold;">
S_158
</th>
<th style="text-align:right;font-weight: bold;">
S_159
</th>
<th style="text-align:right;font-weight: bold;">
S_160
</th>
<th style="text-align:right;font-weight: bold;">
S_161
</th>
<th style="text-align:right;font-weight: bold;">
S_162
</th>
<th style="text-align:right;font-weight: bold;">
S_163
</th>
<th style="text-align:right;font-weight: bold;">
S_164
</th>
<th style="text-align:right;font-weight: bold;">
S_165
</th>
<th style="text-align:right;font-weight: bold;">
S_166
</th>
<th style="text-align:right;font-weight: bold;">
S_167
</th>
<th style="text-align:right;font-weight: bold;">
S_168
</th>
<th style="text-align:right;font-weight: bold;">
S_169
</th>
<th style="text-align:right;font-weight: bold;">
S_170
</th>
<th style="text-align:right;font-weight: bold;">
S_171
</th>
<th style="text-align:right;font-weight: bold;">
S_172
</th>
<th style="text-align:right;font-weight: bold;">
S_173
</th>
<th style="text-align:right;font-weight: bold;">
S_174
</th>
<th style="text-align:right;font-weight: bold;">
S_175
</th>
<th style="text-align:right;font-weight: bold;">
S_176
</th>
<th style="text-align:right;font-weight: bold;">
M_km2
</th>
<th style="text-align:right;font-weight: bold;">
T_km2
</th>
<th style="text-align:right;font-weight: bold;">
P_km2
</th>
<th style="text-align:right;font-weight: bold;">
S_km2
</th>
<th style="text-align:right;font-weight: bold;">
T_km2_1
</th>
<th style="text-align:right;font-weight: bold;">
All_R_2
</th>
<th style="text-align:right;font-weight: bold;">
M_pop
</th>
<th style="text-align:right;font-weight: bold;">
T_pop
</th>
<th style="text-align:right;font-weight: bold;">
P_pop
</th>
<th style="text-align:right;font-weight: bold;">
S_pop
</th>
<th style="text-align:right;font-weight: bold;">
T_pop_1
</th>
<th style="text-align:right;font-weight: bold;">
All_R_p
</th>
<th style="text-align:right;font-weight: bold;">
tot_R
</th>
<th style="text-align:right;font-weight: bold;">
lib_km2
</th>
<th style="text-align:right;font-weight: bold;">
uni_km2
</th>
<th style="text-align:right;font-weight: bold;">
sch_km2
</th>
<th style="text-align:right;font-weight: bold;">
cll_km2
</th>
<th style="text-align:right;font-weight: bold;">
kid_km2
</th>
<th style="text-align:right;font-weight: bold;">
ply_km2
</th>
<th style="text-align:right;font-weight: bold;">
the_km2
</th>
<th style="text-align:right;font-weight: bold;">
ngh_km2
</th>
<th style="text-align:right;font-weight: bold;">
cin_km2
</th>
<th style="text-align:right;font-weight: bold;">
pub_km2
</th>
<th style="text-align:right;font-weight: bold;">
bar_km2
</th>
<th style="text-align:right;font-weight: bold;">
std_km2
</th>
<th style="text-align:right;font-weight: bold;">
sc_km2
</th>
<th style="text-align:right;font-weight: bold;">
hsp_km2
</th>
<th style="text-align:right;font-weight: bold;">
doc_km2
</th>
<th style="text-align:right;font-weight: bold;">
mll_km2
</th>
<th style="text-align:right;font-weight: bold;">
dc_km2
</th>
<th style="text-align:right;font-weight: bold;">
con_km2
</th>
<th style="text-align:right;font-weight: bold;">
sup_km2
</th>
<th style="text-align:right;font-weight: bold;">
pst_km2
</th>
<th style="text-align:right;font-weight: bold;">
th_km2
</th>
<th style="text-align:right;font-weight: bold;">
cc_km2
</th>
<th style="text-align:right;font-weight: bold;">
nh_km2
</th>
<th style="text-align:right;font-weight: bold;">
ac_km2
</th>
<th style="text-align:right;font-weight: bold;">
phr_km2
</th>
<th style="text-align:right;font-weight: bold;">
rst_km2
</th>
<th style="text-align:right;font-weight: bold;">
ff_km2
</th>
<th style="text-align:right;font-weight: bold;">
caf_km2
</th>
<th style="text-align:right;font-weight: bold;">
fc_km2
</th>
<th style="text-align:right;font-weight: bold;">
bir_km2
</th>
<th style="text-align:right;font-weight: bold;">
bak_km2
</th>
<th style="text-align:right;font-weight: bold;">
chm_km2
</th>
<th style="text-align:right;font-weight: bold;">
dot_km2
</th>
<th style="text-align:right;font-weight: bold;">
har_km2
</th>
<th style="text-align:right;font-weight: bold;">
prk_km2
</th>
<th style="text-align:right;font-weight: bold;">
Rch_km2
</th>
<th style="text-align:right;font-weight: bold;">
Rmu_km2
</th>
<th style="text-align:right;font-weight: bold;">
Rth_km2
</th>
<th style="text-align:right;font-weight: bold;">
All_P_2
</th>
<th style="text-align:right;font-weight: bold;">
lib_pop
</th>
<th style="text-align:right;font-weight: bold;">
uni_pop
</th>
<th style="text-align:right;font-weight: bold;">
sch_pop
</th>
<th style="text-align:right;font-weight: bold;">
coll_pp
</th>
<th style="text-align:right;font-weight: bold;">
kid_pop
</th>
<th style="text-align:right;font-weight: bold;">
play_pp
</th>
<th style="text-align:right;font-weight: bold;">
thea_pp
</th>
<th style="text-align:right;font-weight: bold;">
nigh_pp
</th>
<th style="text-align:right;font-weight: bold;">
cin_pop
</th>
<th style="text-align:right;font-weight: bold;">
pub_pop
</th>
<th style="text-align:right;font-weight: bold;">
bar_pop
</th>
<th style="text-align:right;font-weight: bold;">
stad_pp
</th>
<th style="text-align:right;font-weight: bold;">
sc_pop
</th>
<th style="text-align:right;font-weight: bold;">
hosp_pp
</th>
<th style="text-align:right;font-weight: bold;">
doc_pop
</th>
<th style="text-align:right;font-weight: bold;">
mall_pp
</th>
<th style="text-align:right;font-weight: bold;">
dc_pop
</th>
<th style="text-align:right;font-weight: bold;">
con_pop
</th>
<th style="text-align:right;font-weight: bold;">
sup_pop
</th>
<th style="text-align:right;font-weight: bold;">
post_pp
</th>
<th style="text-align:right;font-weight: bold;">
th_pop
</th>
<th style="text-align:right;font-weight: bold;">
cc_pop
</th>
<th style="text-align:right;font-weight: bold;">
nh_pop
</th>
<th style="text-align:right;font-weight: bold;">
ac_pop
</th>
<th style="text-align:right;font-weight: bold;">
phar_pp
</th>
<th style="text-align:right;font-weight: bold;">
rest_pp
</th>
<th style="text-align:right;font-weight: bold;">
ff_pop
</th>
<th style="text-align:right;font-weight: bold;">
cafe_pp
</th>
<th style="text-align:right;font-weight: bold;">
fc_pop
</th>
<th style="text-align:right;font-weight: bold;">
bier_pp
</th>
<th style="text-align:right;font-weight: bold;">
bake_pp
</th>
<th style="text-align:right;font-weight: bold;">
chem_pp
</th>
<th style="text-align:right;font-weight: bold;">
doit_pp
</th>
<th style="text-align:right;font-weight: bold;">
hair_pp
</th>
<th style="text-align:right;font-weight: bold;">
park_pp
</th>
<th style="text-align:right;font-weight: bold;">
Rch_pop
</th>
<th style="text-align:right;font-weight: bold;">
Rmu_pop
</th>
<th style="text-align:right;font-weight: bold;">
Roth_pp
</th>
<th style="text-align:right;font-weight: bold;">
All_P_p
</th>
<th style="text-align:right;font-weight: bold;">
tot_P
</th>
<th style="text-align:right;font-weight: bold;">
lib_den
</th>
<th style="text-align:right;font-weight: bold;">
uni_den
</th>
<th style="text-align:right;font-weight: bold;">
sch_den
</th>
<th style="text-align:right;font-weight: bold;">
coll_dn
</th>
<th style="text-align:right;font-weight: bold;">
kid_den
</th>
<th style="text-align:right;font-weight: bold;">
play_dn
</th>
<th style="text-align:right;font-weight: bold;">
thea_dn
</th>
<th style="text-align:right;font-weight: bold;">
nigh_dn
</th>
<th style="text-align:right;font-weight: bold;">
cin_den
</th>
<th style="text-align:right;font-weight: bold;">
pub_den
</th>
<th style="text-align:right;font-weight: bold;">
bar_den
</th>
<th style="text-align:right;font-weight: bold;">
stad_dn
</th>
<th style="text-align:right;font-weight: bold;">
sc_den
</th>
<th style="text-align:right;font-weight: bold;">
hosp_dn
</th>
<th style="text-align:right;font-weight: bold;">
doc_den
</th>
<th style="text-align:right;font-weight: bold;">
mall_dn
</th>
<th style="text-align:right;font-weight: bold;">
dc_den
</th>
<th style="text-align:right;font-weight: bold;">
con_den
</th>
<th style="text-align:right;font-weight: bold;">
sup_den
</th>
<th style="text-align:right;font-weight: bold;">
post_dn
</th>
<th style="text-align:right;font-weight: bold;">
th_den
</th>
<th style="text-align:right;font-weight: bold;">
cc_den
</th>
<th style="text-align:right;font-weight: bold;">
nh_den
</th>
<th style="text-align:right;font-weight: bold;">
ac_den
</th>
<th style="text-align:right;font-weight: bold;">
phar_dn
</th>
<th style="text-align:right;font-weight: bold;">
rest_dn
</th>
<th style="text-align:right;font-weight: bold;">
ff_den
</th>
<th style="text-align:right;font-weight: bold;">
cafe_dn
</th>
<th style="text-align:right;font-weight: bold;">
fc_den
</th>
<th style="text-align:right;font-weight: bold;">
bier_dn
</th>
<th style="text-align:right;font-weight: bold;">
bake_dn
</th>
<th style="text-align:right;font-weight: bold;">
chem_dn
</th>
<th style="text-align:right;font-weight: bold;">
doit_dn
</th>
<th style="text-align:right;font-weight: bold;">
hair_dn
</th>
<th style="text-align:right;font-weight: bold;">
park_dn
</th>
<th style="text-align:right;font-weight: bold;">
Rch_den
</th>
<th style="text-align:right;font-weight: bold;">
Rmu_den
</th>
<th style="text-align:right;font-weight: bold;">
Roth_dn
</th>
<th style="text-align:right;font-weight: bold;">
All_P_d
</th>
<th style="text-align:right;font-weight: bold;">
Pop_Den
</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:right;">
27.37
</td>
<td style="text-align:right;">
9.44
</td>
<td style="text-align:right;">
54.78
</td>
<td style="text-align:left;">
108
</td>
<td style="text-align:left;">
Schleswig-Holstein
</td>
<td style="text-align:right;">
89504
</td>
<td style="text-align:right;">
8.9
</td>
<td style="text-align:right;">
76.0
</td>
<td style="text-align:right;">
19.3
</td>
<td style="text-align:right;">
7.9
</td>
<td style="text-align:right;">
12.5
</td>
<td style="text-align:right;">
43.8
</td>
<td style="text-align:right;">
14.5
</td>
<td style="text-align:right;">
60.3
</td>
<td style="text-align:right;">
8.1
</td>
<td style="text-align:right;">
31.8
</td>
<td style="text-align:right;">
94.8
</td>
<td style="text-align:right;">
24.1
</td>
<td style="text-align:right;">
-10.9
</td>
<td style="text-align:right;">
72.1
</td>
<td style="text-align:right;">
27.7
</td>
<td style="text-align:right;">
44.1
</td>
<td style="text-align:right;">
51.8
</td>
<td style="text-align:right;">
18.8
</td>
<td style="text-align:right;">
48.6
</td>
<td style="text-align:right;">
22.8
</td>
<td style="text-align:right;">
39.9
</td>
<td style="text-align:right;">
17.2
</td>
<td style="text-align:right;">
32.9
</td>
<td style="text-align:right;">
85.0
</td>
<td style="text-align:right;">
74.5
</td>
<td style="text-align:right;">
2.9
</td>
<td style="text-align:right;">
2.6
</td>
<td style="text-align:right;">
5.5
</td>
<td style="text-align:right;">
9.8
</td>
<td style="text-align:right;">
11.9
</td>
<td style="text-align:right;">
9.1
</td>
<td style="text-align:right;">
23.8
</td>
<td style="text-align:right;">
19.6
</td>
<td style="text-align:right;">
9.6
</td>
<td style="text-align:right;">
20.2
</td>
<td style="text-align:right;">
10.6
</td>
<td style="text-align:right;">
8.2
</td>
<td style="text-align:right;">
2.4
</td>
<td style="text-align:right;">
42.2
</td>
<td style="text-align:right;">
6.1
</td>
<td style="text-align:right;">
6.6
</td>
<td style="text-align:right;">
3.0
</td>
<td style="text-align:right;">
19.0
</td>
<td style="text-align:right;">
30.2
</td>
<td style="text-align:right;">
14.3
</td>
<td style="text-align:right;">
10.0
</td>
<td style="text-align:right;">
11.7
</td>
<td style="text-align:right;">
0.75
</td>
<td style="text-align:right;">
78.80
</td>
<td style="text-align:right;">
2.3
</td>
<td style="text-align:right;">
70.3
</td>
<td style="text-align:right;">
48.2
</td>
<td style="text-align:right;">
34.0
</td>
<td style="text-align:right;">
92.1
</td>
<td style="text-align:right;">
109.0
</td>
<td style="text-align:right;">
44.7
</td>
<td style="text-align:right;">
13.2
</td>
<td style="text-align:right;">
91.2
</td>
<td style="text-align:right;">
9.8
</td>
<td style="text-align:right;">
29.5
</td>
<td style="text-align:right;">
6.1
</td>
<td style="text-align:right;">
8.5
</td>
<td style="text-align:right;">
15.7
</td>
<td style="text-align:right;">
14.0
</td>
<td style="text-align:right;">
2.91
</td>
<td style="text-align:right;">
1527
</td>
<td style="text-align:right;">
2333
</td>
<td style="text-align:right;">
1278.6
</td>
<td style="text-align:right;">
2986
</td>
<td style="text-align:right;">
16.0
</td>
<td style="text-align:right;">
17.6
</td>
<td style="text-align:right;">
24258
</td>
<td style="text-align:right;">
11.3
</td>
<td style="text-align:right;">
3375
</td>
<td style="text-align:right;">
2992
</td>
<td style="text-align:right;">
3373
</td>
<td style="text-align:right;">
2965.5
</td>
<td style="text-align:right;">
3.8
</td>
<td style="text-align:right;">
52.6
</td>
<td style="text-align:right;">
24.4
</td>
<td style="text-align:right;">
6.4
</td>
<td style="text-align:right;">
40.9
</td>
<td style="text-align:right;">
1.0
</td>
<td style="text-align:right;">
14.9
</td>
<td style="text-align:right;">
4568.9
</td>
<td style="text-align:right;">
444.4
</td>
<td style="text-align:right;">
50.4
</td>
<td style="text-align:right;">
15.6
</td>
<td style="text-align:right;">
27.1
</td>
<td style="text-align:right;">
-2.4
</td>
<td style="text-align:right;">
9.47
</td>
<td style="text-align:right;">
369.1
</td>
<td style="text-align:right;">
29.6
</td>
<td style="text-align:right;">
30.1
</td>
<td style="text-align:right;">
100.3
</td>
<td style="text-align:right;">
55.7
</td>
<td style="text-align:right;">
123.9
</td>
<td style="text-align:right;">
35.2
</td>
<td style="text-align:right;">
15.8
</td>
<td style="text-align:right;">
92.5
</td>
<td style="text-align:right;">
45.2
</td>
<td style="text-align:right;">
69.1
</td>
<td style="text-align:right;">
7.0
</td>
<td style="text-align:right;">
371.4
</td>
<td style="text-align:right;">
1282.4
</td>
<td style="text-align:right;">
1.3
</td>
<td style="text-align:right;">
767.3
</td>
<td style="text-align:right;">
0.0
</td>
<td style="text-align:left;">
2
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
156
</td>
<td style="text-align:right;">
2309
</td>
<td style="text-align:right;">
13725
</td>
<td style="text-align:right;">
16.2
</td>
<td style="text-align:right;">
23.5
</td>
<td style="text-align:right;">
19.6
</td>
<td style="text-align:right;">
26.5
</td>
<td style="text-align:right;">
7.2
</td>
<td style="text-align:right;">
19.1
</td>
<td style="text-align:right;">
7.41
</td>
<td style="text-align:right;">
25.6
</td>
<td style="text-align:right;">
134.5
</td>
<td style="text-align:right;">
6.24
</td>
<td style="text-align:right;">
1.80
</td>
<td style="text-align:right;">
5.1
</td>
<td style="text-align:right;">
5
</td>
<td style="text-align:right;">
98
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
500
</td>
<td style="text-align:right;">
92
</td>
<td style="text-align:right;">
543
</td>
<td style="text-align:right;">
90
</td>
<td style="text-align:right;">
602
</td>
<td style="text-align:right;">
89
</td>
<td style="text-align:right;">
180
</td>
<td style="text-align:right;">
100
</td>
<td style="text-align:right;">
478
</td>
<td style="text-align:right;">
52.4
</td>
<td style="text-align:right;">
34.2
</td>
<td style="text-align:right;">
27.7
</td>
<td style="text-align:right;">
14.0
</td>
<td style="text-align:right;">
7.7
</td>
<td style="text-align:right;">
3.6
</td>
<td style="text-align:right;">
3.2
</td>
<td style="text-align:right;">
2.3
</td>
<td style="text-align:right;">
1.5
</td>
<td style="text-align:right;">
19.8
</td>
<td style="text-align:right;">
3.5
</td>
<td style="text-align:right;">
40.1
</td>
<td style="text-align:right;">
1.6
</td>
<td style="text-align:right;">
131.7
</td>
<td style="text-align:right;">
30.9
</td>
<td style="text-align:right;">
6.23
</td>
<td style="text-align:right;">
42.5
</td>
<td style="text-align:right;">
62.0
</td>
<td style="text-align:right;">
18.1
</td>
<td style="text-align:right;">
25.3
</td>
<td style="text-align:right;">
5.8
</td>
<td style="text-align:right;">
67.8
</td>
<td style="text-align:right;">
8.5
</td>
<td style="text-align:right;">
1.9
</td>
<td style="text-align:right;">
35.2
</td>
<td style="text-align:right;">
92.5
</td>
<td style="text-align:right;">
100.0
</td>
<td style="text-align:right;">
42826.6
</td>
<td style="text-align:right;">
4.1
</td>
<td style="text-align:right;">
51.8
</td>
<td style="text-align:right;">
24.8
</td>
<td style="text-align:right;">
97.0
</td>
<td style="text-align:right;">
34.8
</td>
<td style="text-align:right;">
48.8
</td>
<td style="text-align:right;">
16.4
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.62
</td>
<td style="text-align:right;">
0.44
</td>
<td style="text-align:right;">
0.44
</td>
<td style="text-align:right;">
0.95
</td>
<td style="text-align:right;">
2.45
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
33.81
</td>
<td style="text-align:right;">
24.40
</td>
<td style="text-align:right;">
24.01
</td>
<td style="text-align:right;">
52.45
</td>
<td style="text-align:right;">
134.66
</td>
<td style="text-align:right;">
120.52
</td>
<td style="text-align:right;">
0.08
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.08
</td>
<td style="text-align:right;">
0.06
</td>
<td style="text-align:right;">
0.30
</td>
<td style="text-align:right;">
1.67
</td>
<td style="text-align:right;">
0.10
</td>
<td style="text-align:right;">
0.12
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0.91
</td>
<td style="text-align:right;">
0.08
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.08
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.91
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0.30
</td>
<td style="text-align:right;">
0.41
</td>
<td style="text-align:right;">
0.20
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.12
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.06
</td>
<td style="text-align:right;">
0.37
</td>
<td style="text-align:right;">
1.71
</td>
<td style="text-align:right;">
1.24
</td>
<td style="text-align:right;">
0.85
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
1.02
</td>
<td style="text-align:right;">
0.22
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
1.26
</td>
<td style="text-align:right;">
0.02
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0.06
</td>
<td style="text-align:right;">
12.46
</td>
<td style="text-align:right;">
4.47
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
4.47
</td>
<td style="text-align:right;">
3.35
</td>
<td style="text-align:right;">
16.76
</td>
<td style="text-align:right;">
91.62
</td>
<td style="text-align:right;">
5.59
</td>
<td style="text-align:right;">
6.70
</td>
<td style="text-align:right;">
2.23
</td>
<td style="text-align:right;">
50.28
</td>
<td style="text-align:right;">
4.47
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
4.47
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
50.28
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
2.23
</td>
<td style="text-align:right;">
16.76
</td>
<td style="text-align:right;">
22.35
</td>
<td style="text-align:right;">
11.17
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
6.70
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
3.35
</td>
<td style="text-align:right;">
20.11
</td>
<td style="text-align:right;">
93.85
</td>
<td style="text-align:right;">
68.15
</td>
<td style="text-align:right;">
46.93
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
55.86
</td>
<td style="text-align:right;">
12.29
</td>
<td style="text-align:right;">
2.23
</td>
<td style="text-align:right;">
69.27
</td>
<td style="text-align:right;">
1.12
</td>
<td style="text-align:right;">
2.23
</td>
<td style="text-align:right;">
2.23
</td>
<td style="text-align:right;">
3.35
</td>
<td style="text-align:right;">
684.89
</td>
<td style="text-align:right;">
613
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.16
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.27
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.26
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.03
</td>
<td style="text-align:right;">
0.03
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.02
</td>
<td style="text-align:right;">
0.40
</td>
<td style="text-align:right;">
0.29
</td>
<td style="text-align:right;">
0.18
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.23
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.32
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.03
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.33
</td>
<td style="text-align:right;">
4566
</td>
</tr>
<tr>
<td style="text-align:right;">
49.54
</td>
<td style="text-align:right;">
10.13
</td>
<td style="text-align:right;">
54.32
</td>
<td style="text-align:left;">
108
</td>
<td style="text-align:left;">
Schleswig-Holstein
</td>
<td style="text-align:right;">
247548
</td>
<td style="text-align:right;">
9.1
</td>
<td style="text-align:right;">
72.0
</td>
<td style="text-align:right;">
22.6
</td>
<td style="text-align:right;">
6.6
</td>
<td style="text-align:right;">
8.9
</td>
<td style="text-align:right;">
33.1
</td>
<td style="text-align:right;">
14.9
</td>
<td style="text-align:right;">
63.2
</td>
<td style="text-align:right;">
8.8
</td>
<td style="text-align:right;">
37.8
</td>
<td style="text-align:right;">
310.5
</td>
<td style="text-align:right;">
3.2
</td>
<td style="text-align:right;">
136.2
</td>
<td style="text-align:right;">
67.3
</td>
<td style="text-align:right;">
32.4
</td>
<td style="text-align:right;">
38.9
</td>
<td style="text-align:right;">
51.5
</td>
<td style="text-align:right;">
19.2
</td>
<td style="text-align:right;">
48.9
</td>
<td style="text-align:right;">
21.3
</td>
<td style="text-align:right;">
39.3
</td>
<td style="text-align:right;">
17.5
</td>
<td style="text-align:right;">
32.6
</td>
<td style="text-align:right;">
79.2
</td>
<td style="text-align:right;">
67.7
</td>
<td style="text-align:right;">
2.8
</td>
<td style="text-align:right;">
2.5
</td>
<td style="text-align:right;">
5.3
</td>
<td style="text-align:right;">
9.4
</td>
<td style="text-align:right;">
11.0
</td>
<td style="text-align:right;">
10.2
</td>
<td style="text-align:right;">
26.5
</td>
<td style="text-align:right;">
19.1
</td>
<td style="text-align:right;">
8.8
</td>
<td style="text-align:right;">
18.5
</td>
<td style="text-align:right;">
9.7
</td>
<td style="text-align:right;">
7.5
</td>
<td style="text-align:right;">
2.3
</td>
<td style="text-align:right;">
41.5
</td>
<td style="text-align:right;">
3.4
</td>
<td style="text-align:right;">
4.9
</td>
<td style="text-align:right;">
2.3
</td>
<td style="text-align:right;">
17.8
</td>
<td style="text-align:right;">
26.7
</td>
<td style="text-align:right;">
1.6
</td>
<td style="text-align:right;">
10.1
</td>
<td style="text-align:right;">
10.0
</td>
<td style="text-align:right;">
0.64
</td>
<td style="text-align:right;">
80.05
</td>
<td style="text-align:right;">
5.0
</td>
<td style="text-align:right;">
73.5
</td>
<td style="text-align:right;">
31.1
</td>
<td style="text-align:right;">
41.7
</td>
<td style="text-align:right;">
90.4
</td>
<td style="text-align:right;">
140.6
</td>
<td style="text-align:right;">
31.4
</td>
<td style="text-align:right;">
13.4
</td>
<td style="text-align:right;">
127.9
</td>
<td style="text-align:right;">
9.4
</td>
<td style="text-align:right;">
30.3
</td>
<td style="text-align:right;">
5.1
</td>
<td style="text-align:right;">
7.3
</td>
<td style="text-align:right;">
13.9
</td>
<td style="text-align:right;">
20.5
</td>
<td style="text-align:right;">
4.56
</td>
<td style="text-align:right;">
1564
</td>
<td style="text-align:right;">
27
</td>
<td style="text-align:right;">
1293.2
</td>
<td style="text-align:right;">
3304
</td>
<td style="text-align:right;">
12.1
</td>
<td style="text-align:right;">
14.5
</td>
<td style="text-align:right;">
35
</td>
<td style="text-align:right;">
18.2
</td>
<td style="text-align:right;">
4015
</td>
<td style="text-align:right;">
3296
</td>
<td style="text-align:right;">
3693
</td>
<td style="text-align:right;">
3844.7
</td>
<td style="text-align:right;">
6.0
</td>
<td style="text-align:right;">
54.4
</td>
<td style="text-align:right;">
28.7
</td>
<td style="text-align:right;">
5.8
</td>
<td style="text-align:right;">
27.7
</td>
<td style="text-align:right;">
0.2
</td>
<td style="text-align:right;">
5.0
</td>
<td style="text-align:right;">
886.5
</td>
<td style="text-align:right;">
383.0
</td>
<td style="text-align:right;">
51.7
</td>
<td style="text-align:right;">
17.1
</td>
<td style="text-align:right;">
31.5
</td>
<td style="text-align:right;">
7.3
</td>
<td style="text-align:right;">
9.84
</td>
<td style="text-align:right;">
295.2
</td>
<td style="text-align:right;">
30.1
</td>
<td style="text-align:right;">
26.0
</td>
<td style="text-align:right;">
74.6
</td>
<td style="text-align:right;">
67.8
</td>
<td style="text-align:right;">
88.6
</td>
<td style="text-align:right;">
34.9
</td>
<td style="text-align:right;">
30.2
</td>
<td style="text-align:right;">
91.0
</td>
<td style="text-align:right;">
67.0
</td>
<td style="text-align:right;">
39.6
</td>
<td style="text-align:right;">
7.3
</td>
<td style="text-align:right;">
399.2
</td>
<td style="text-align:right;">
2269.4
</td>
<td style="text-align:right;">
74.5
</td>
<td style="text-align:right;">
755.3
</td>
<td style="text-align:right;">
53.4
</td>
<td style="text-align:left;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
209
</td>
<td style="text-align:right;">
3112
</td>
<td style="text-align:right;">
22801
</td>
<td style="text-align:right;">
16.9
</td>
<td style="text-align:right;">
17.7
</td>
<td style="text-align:right;">
17.4
</td>
<td style="text-align:right;">
26.2
</td>
<td style="text-align:right;">
9.8
</td>
<td style="text-align:right;">
12.9
</td>
<td style="text-align:right;">
5.25
</td>
<td style="text-align:right;">
30.2
</td>
<td style="text-align:right;">
146.1
</td>
<td style="text-align:right;">
7.31
</td>
<td style="text-align:right;">
1.91
</td>
<td style="text-align:right;">
6.6
</td>
<td style="text-align:right;">
2
</td>
<td style="text-align:right;">
64
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
460
</td>
<td style="text-align:right;">
92
</td>
<td style="text-align:right;">
415
</td>
<td style="text-align:right;">
94
</td>
<td style="text-align:right;">
590
</td>
<td style="text-align:right;">
89
</td>
<td style="text-align:right;">
193
</td>
<td style="text-align:right;">
100
</td>
<td style="text-align:right;">
436
</td>
<td style="text-align:right;">
49.1
</td>
<td style="text-align:right;">
30.3
</td>
<td style="text-align:right;">
27.1
</td>
<td style="text-align:right;">
13.1
</td>
<td style="text-align:right;">
4.6
</td>
<td style="text-align:right;">
3.4
</td>
<td style="text-align:right;">
2.0
</td>
<td style="text-align:right;">
1.3
</td>
<td style="text-align:right;">
0.9
</td>
<td style="text-align:right;">
17.9
</td>
<td style="text-align:right;">
2.9
</td>
<td style="text-align:right;">
21.0
</td>
<td style="text-align:right;">
2.0
</td>
<td style="text-align:right;">
120.2
</td>
<td style="text-align:right;">
30.6
</td>
<td style="text-align:right;">
6.96
</td>
<td style="text-align:right;">
45.7
</td>
<td style="text-align:right;">
66.4
</td>
<td style="text-align:right;">
18.3
</td>
<td style="text-align:right;">
30.0
</td>
<td style="text-align:right;">
5.9
</td>
<td style="text-align:right;">
108.1
</td>
<td style="text-align:right;">
7.3
</td>
<td style="text-align:right;">
1.7
</td>
<td style="text-align:right;">
34.9
</td>
<td style="text-align:right;">
91.0
</td>
<td style="text-align:right;">
0.0
</td>
<td style="text-align:right;">
45821.3
</td>
<td style="text-align:right;">
4.6
</td>
<td style="text-align:right;">
51.5
</td>
<td style="text-align:right;">
26.0
</td>
<td style="text-align:right;">
97.5
</td>
<td style="text-align:right;">
38.3
</td>
<td style="text-align:right;">
48.1
</td>
<td style="text-align:right;">
13.6
</td>
<td style="text-align:right;">
0.20
</td>
<td style="text-align:right;">
0.69
</td>
<td style="text-align:right;">
0.14
</td>
<td style="text-align:right;">
0.68
</td>
<td style="text-align:right;">
0.66
</td>
<td style="text-align:right;">
2.37
</td>
<td style="text-align:right;">
9.06
</td>
<td style="text-align:right;">
31.13
</td>
<td style="text-align:right;">
6.29
</td>
<td style="text-align:right;">
30.83
</td>
<td style="text-align:right;">
30.10
</td>
<td style="text-align:right;">
107.42
</td>
<td style="text-align:right;">
265.92
</td>
<td style="text-align:right;">
0.11
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.06
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0.33
</td>
<td style="text-align:right;">
0.53
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0.11
</td>
<td style="text-align:right;">
0.06
</td>
<td style="text-align:right;">
0.51
</td>
<td style="text-align:right;">
0.33
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.21
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.67
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.05
</td>
<td style="text-align:right;">
0.17
</td>
<td style="text-align:right;">
0.56
</td>
<td style="text-align:right;">
0.25
</td>
<td style="text-align:right;">
0.02
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.02
</td>
<td style="text-align:right;">
0.61
</td>
<td style="text-align:right;">
1.34
</td>
<td style="text-align:right;">
1.35
</td>
<td style="text-align:right;">
0.81
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.93
</td>
<td style="text-align:right;">
0.17
</td>
<td style="text-align:right;">
0.08
</td>
<td style="text-align:right;">
1.10
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.16
</td>
<td style="text-align:right;">
0.06
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
10.75
</td>
<td style="text-align:right;">
4.85
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
2.83
</td>
<td style="text-align:right;">
1.62
</td>
<td style="text-align:right;">
14.95
</td>
<td style="text-align:right;">
23.83
</td>
<td style="text-align:right;">
2.02
</td>
<td style="text-align:right;">
4.85
</td>
<td style="text-align:right;">
2.83
</td>
<td style="text-align:right;">
23.03
</td>
<td style="text-align:right;">
14.95
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
9.70
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
30.30
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
2.42
</td>
<td style="text-align:right;">
7.68
</td>
<td style="text-align:right;">
25.45
</td>
<td style="text-align:right;">
11.31
</td>
<td style="text-align:right;">
0.81
</td>
<td style="text-align:right;">
1.62
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.81
</td>
<td style="text-align:right;">
27.47
</td>
<td style="text-align:right;">
60.59
</td>
<td style="text-align:right;">
61.40
</td>
<td style="text-align:right;">
36.76
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
42.01
</td>
<td style="text-align:right;">
7.68
</td>
<td style="text-align:right;">
3.64
</td>
<td style="text-align:right;">
49.69
</td>
<td style="text-align:right;">
0.40
</td>
<td style="text-align:right;">
7.27
</td>
<td style="text-align:right;">
2.83
</td>
<td style="text-align:right;">
1.62
</td>
<td style="text-align:right;">
487.18
</td>
<td style="text-align:right;">
1206
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.02
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.05
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.09
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.05
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.06
</td>
<td style="text-align:right;">
0.21
</td>
<td style="text-align:right;">
0.29
</td>
<td style="text-align:right;">
0.13
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.10
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.20
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.10
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.30
</td>
<td style="text-align:right;">
6862
</td>
</tr>
<tr>
<td style="text-align:right;">
43.20
</td>
<td style="text-align:right;">
10.73
</td>
<td style="text-align:right;">
53.87
</td>
<td style="text-align:left;">
108
</td>
<td style="text-align:left;">
Schleswig-Holstein
</td>
<td style="text-align:right;">
217198
</td>
<td style="text-align:right;">
8.6
</td>
<td style="text-align:right;">
69.7
</td>
<td style="text-align:right;">
26.5
</td>
<td style="text-align:right;">
7.0
</td>
<td style="text-align:right;">
8.6
</td>
<td style="text-align:right;">
35.7
</td>
<td style="text-align:right;">
17.2
</td>
<td style="text-align:right;">
58.0
</td>
<td style="text-align:right;">
8.5
</td>
<td style="text-align:right;">
36.3
</td>
<td style="text-align:right;">
176.7
</td>
<td style="text-align:right;">
18.6
</td>
<td style="text-align:right;">
52.5
</td>
<td style="text-align:right;">
75.6
</td>
<td style="text-align:right;">
24.3
</td>
<td style="text-align:right;">
40.5
</td>
<td style="text-align:right;">
56.1
</td>
<td style="text-align:right;">
18.5
</td>
<td style="text-align:right;">
48.4
</td>
<td style="text-align:right;">
21.0
</td>
<td style="text-align:right;">
46.9
</td>
<td style="text-align:right;">
17.6
</td>
<td style="text-align:right;">
31.9
</td>
<td style="text-align:right;">
81.1
</td>
<td style="text-align:right;">
85.6
</td>
<td style="text-align:right;">
2.7
</td>
<td style="text-align:right;">
2.5
</td>
<td style="text-align:right;">
5.2
</td>
<td style="text-align:right;">
10.0
</td>
<td style="text-align:right;">
8.2
</td>
<td style="text-align:right;">
7.0
</td>
<td style="text-align:right;">
25.0
</td>
<td style="text-align:right;">
21.5
</td>
<td style="text-align:right;">
10.5
</td>
<td style="text-align:right;">
23.2
</td>
<td style="text-align:right;">
12.7
</td>
<td style="text-align:right;">
9.6
</td>
<td style="text-align:right;">
3.1
</td>
<td style="text-align:right;">
44.7
</td>
<td style="text-align:right;">
2.2
</td>
<td style="text-align:right;">
6.0
</td>
<td style="text-align:right;">
3.1
</td>
<td style="text-align:right;">
19.5
</td>
<td style="text-align:right;">
36.1
</td>
<td style="text-align:right;">
1.7
</td>
<td style="text-align:right;">
9.1
</td>
<td style="text-align:right;">
12.7
</td>
<td style="text-align:right;">
0.72
</td>
<td style="text-align:right;">
79.62
</td>
<td style="text-align:right;">
1.0
</td>
<td style="text-align:right;">
70.7
</td>
<td style="text-align:right;">
54.5
</td>
<td style="text-align:right;">
43.0
</td>
<td style="text-align:right;">
104.4
</td>
<td style="text-align:right;">
49.2
</td>
<td style="text-align:right;">
22.7
</td>
<td style="text-align:right;">
16.3
</td>
<td style="text-align:right;">
60.1
</td>
<td style="text-align:right;">
10.0
</td>
<td style="text-align:right;">
25.6
</td>
<td style="text-align:right;">
4.6
</td>
<td style="text-align:right;">
11.1
</td>
<td style="text-align:right;">
12.8
</td>
<td style="text-align:right;">
14.2
</td>
<td style="text-align:right;">
2.81
</td>
<td style="text-align:right;">
163
</td>
<td style="text-align:right;">
2593
</td>
<td style="text-align:right;">
1298.6
</td>
<td style="text-align:right;">
3036
</td>
<td style="text-align:right;">
15.1
</td>
<td style="text-align:right;">
17.0
</td>
<td style="text-align:right;">
26859
</td>
<td style="text-align:right;">
23.9
</td>
<td style="text-align:right;">
3784
</td>
<td style="text-align:right;">
3055
</td>
<td style="text-align:right;">
3324
</td>
<td style="text-align:right;">
2718.6
</td>
<td style="text-align:right;">
5.7
</td>
<td style="text-align:right;">
37.1
</td>
<td style="text-align:right;">
56.8
</td>
<td style="text-align:right;">
15.4
</td>
<td style="text-align:right;">
152.6
</td>
<td style="text-align:right;">
1.7
</td>
<td style="text-align:right;">
16.8
</td>
<td style="text-align:right;">
2028.8
</td>
<td style="text-align:right;">
442.1
</td>
<td style="text-align:right;">
52.6
</td>
<td style="text-align:right;">
19.6
</td>
<td style="text-align:right;">
27.3
</td>
<td style="text-align:right;">
16.0
</td>
<td style="text-align:right;">
9.15
</td>
<td style="text-align:right;">
393.9
</td>
<td style="text-align:right;">
20.5
</td>
<td style="text-align:right;">
36.0
</td>
<td style="text-align:right;">
120.0
</td>
<td style="text-align:right;">
44.7
</td>
<td style="text-align:right;">
156.1
</td>
<td style="text-align:right;">
34.3
</td>
<td style="text-align:right;">
22.3
</td>
<td style="text-align:right;">
92.3
</td>
<td style="text-align:right;">
61.7
</td>
<td style="text-align:right;">
35.4
</td>
<td style="text-align:right;">
6.8
</td>
<td style="text-align:right;">
375.7
</td>
<td style="text-align:right;">
3039.8
</td>
<td style="text-align:right;">
106.4
</td>
<td style="text-align:right;">
724.8
</td>
<td style="text-align:right;">
36.9
</td>
<td style="text-align:left;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
101
</td>
<td style="text-align:right;">
1454
</td>
<td style="text-align:right;">
22491
</td>
<td style="text-align:right;">
16.1
</td>
<td style="text-align:right;">
25.6
</td>
<td style="text-align:right;">
16.5
</td>
<td style="text-align:right;">
31.0
</td>
<td style="text-align:right;">
9.3
</td>
<td style="text-align:right;">
20.1
</td>
<td style="text-align:right;">
6.82
</td>
<td style="text-align:right;">
26.6
</td>
<td style="text-align:right;">
132.3
</td>
<td style="text-align:right;">
6.25
</td>
<td style="text-align:right;">
1.89
</td>
<td style="text-align:right;">
5.7
</td>
<td style="text-align:right;">
3
</td>
<td style="text-align:right;">
9
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
532
</td>
<td style="text-align:right;">
90
</td>
<td style="text-align:right;">
707
</td>
<td style="text-align:right;">
82
</td>
<td style="text-align:right;">
705
</td>
<td style="text-align:right;">
88
</td>
<td style="text-align:right;">
276
</td>
<td style="text-align:right;">
99
</td>
<td style="text-align:right;">
444
</td>
<td style="text-align:right;">
43.6
</td>
<td style="text-align:right;">
31.3
</td>
<td style="text-align:right;">
17.9
</td>
<td style="text-align:right;">
15.0
</td>
<td style="text-align:right;">
3.9
</td>
<td style="text-align:right;">
3.0
</td>
<td style="text-align:right;">
1.6
</td>
<td style="text-align:right;">
0.7
</td>
<td style="text-align:right;">
0.6
</td>
<td style="text-align:right;">
43.2
</td>
<td style="text-align:right;">
7.8
</td>
<td style="text-align:right;">
19.7
</td>
<td style="text-align:right;">
2.4
</td>
<td style="text-align:right;">
109.0
</td>
<td style="text-align:right;">
27.1
</td>
<td style="text-align:right;">
5.13
</td>
<td style="text-align:right;">
38.0
</td>
<td style="text-align:right;">
65.3
</td>
<td style="text-align:right;">
17.8
</td>
<td style="text-align:right;">
26.6
</td>
<td style="text-align:right;">
5.8
</td>
<td style="text-align:right;">
87.4
</td>
<td style="text-align:right;">
11.1
</td>
<td style="text-align:right;">
2.0
</td>
<td style="text-align:right;">
34.3
</td>
<td style="text-align:right;">
92.3
</td>
<td style="text-align:right;">
100.0
</td>
<td style="text-align:right;">
38078.1
</td>
<td style="text-align:right;">
4.0
</td>
<td style="text-align:right;">
56.1
</td>
<td style="text-align:right;">
29.2
</td>
<td style="text-align:right;">
95.9
</td>
<td style="text-align:right;">
37.0
</td>
<td style="text-align:right;">
48.1
</td>
<td style="text-align:right;">
14.8
</td>
<td style="text-align:right;">
0.25
</td>
<td style="text-align:right;">
0.18
</td>
<td style="text-align:right;">
0.23
</td>
<td style="text-align:right;">
0.44
</td>
<td style="text-align:right;">
0.45
</td>
<td style="text-align:right;">
1.54
</td>
<td style="text-align:right;">
24.51
</td>
<td style="text-align:right;">
17.43
</td>
<td style="text-align:right;">
22.51
</td>
<td style="text-align:right;">
42.43
</td>
<td style="text-align:right;">
43.38
</td>
<td style="text-align:right;">
150.27
</td>
<td style="text-align:right;">
326.38
</td>
<td style="text-align:right;">
0.02
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.31
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.75
</td>
<td style="text-align:right;">
0.62
</td>
<td style="text-align:right;">
0.07
</td>
<td style="text-align:right;">
0.03
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.44
</td>
<td style="text-align:right;">
0.09
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.16
</td>
<td style="text-align:right;">
0.02
</td>
<td style="text-align:right;">
0.35
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.02
</td>
<td style="text-align:right;">
0.17
</td>
<td style="text-align:right;">
0.46
</td>
<td style="text-align:right;">
0.51
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.05
</td>
<td style="text-align:right;">
0.07
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0.26
</td>
<td style="text-align:right;">
1.23
</td>
<td style="text-align:right;">
0.88
</td>
<td style="text-align:right;">
0.48
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.05
</td>
<td style="text-align:right;">
0.50
</td>
<td style="text-align:right;">
0.11
</td>
<td style="text-align:right;">
0.06
</td>
<td style="text-align:right;">
0.79
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.02
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
8.62
</td>
<td style="text-align:right;">
2.30
</td>
<td style="text-align:right;">
0.92
</td>
<td style="text-align:right;">
30.39
</td>
<td style="text-align:right;">
0.46
</td>
<td style="text-align:right;">
73.21
</td>
<td style="text-align:right;">
60.77
</td>
<td style="text-align:right;">
6.91
</td>
<td style="text-align:right;">
3.22
</td>
<td style="text-align:right;">
1.38
</td>
<td style="text-align:right;">
42.82
</td>
<td style="text-align:right;">
9.21
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
15.65
</td>
<td style="text-align:right;">
1.84
</td>
<td style="text-align:right;">
34.53
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
2.30
</td>
<td style="text-align:right;">
16.11
</td>
<td style="text-align:right;">
45.12
</td>
<td style="text-align:right;">
49.26
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
4.60
</td>
<td style="text-align:right;">
6.45
</td>
<td style="text-align:right;">
3.68
</td>
<td style="text-align:right;">
25.78
</td>
<td style="text-align:right;">
120.17
</td>
<td style="text-align:right;">
86.10
</td>
<td style="text-align:right;">
46.96
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
4.60
</td>
<td style="text-align:right;">
48.80
</td>
<td style="text-align:right;">
10.59
</td>
<td style="text-align:right;">
5.52
</td>
<td style="text-align:right;">
77.35
</td>
<td style="text-align:right;">
0.46
</td>
<td style="text-align:right;">
0.92
</td>
<td style="text-align:right;">
1.84
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
840.25
</td>
<td style="text-align:right;">
1825
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.02
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.06
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.03
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.03
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.23
</td>
<td style="text-align:right;">
0.10
</td>
<td style="text-align:right;">
0.08
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.14
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.13
</td>
<td style="text-align:right;">
6487
</td>
</tr>
<tr>
<td style="text-align:right;">
31.48
</td>
<td style="text-align:right;">
9.98
</td>
<td style="text-align:right;">
54.08
</td>
<td style="text-align:left;">
108
</td>
<td style="text-align:left;">
Schleswig-Holstein
</td>
<td style="text-align:right;">
79487
</td>
<td style="text-align:right;">
9.2
</td>
<td style="text-align:right;">
75.6
</td>
<td style="text-align:right;">
25.8
</td>
<td style="text-align:right;">
9.5
</td>
<td style="text-align:right;">
11.5
</td>
<td style="text-align:right;">
48.9
</td>
<td style="text-align:right;">
17.9
</td>
<td style="text-align:right;">
63.7
</td>
<td style="text-align:right;">
8.9
</td>
<td style="text-align:right;">
38.3
</td>
<td style="text-align:right;">
100.6
</td>
<td style="text-align:right;">
12.5
</td>
<td style="text-align:right;">
9.7
</td>
<td style="text-align:right;">
81.8
</td>
<td style="text-align:right;">
18.1
</td>
<td style="text-align:right;">
43.1
</td>
<td style="text-align:right;">
56.2
</td>
<td style="text-align:right;">
19.5
</td>
<td style="text-align:right;">
50.5
</td>
<td style="text-align:right;">
21.2
</td>
<td style="text-align:right;">
46.2
</td>
<td style="text-align:right;">
16.9
</td>
<td style="text-align:right;">
27.2
</td>
<td style="text-align:right;">
82.1
</td>
<td style="text-align:right;">
78.1
</td>
<td style="text-align:right;">
2.7
</td>
<td style="text-align:right;">
2.6
</td>
<td style="text-align:right;">
5.3
</td>
<td style="text-align:right;">
11.3
</td>
<td style="text-align:right;">
8.2
</td>
<td style="text-align:right;">
6.5
</td>
<td style="text-align:right;">
24.3
</td>
<td style="text-align:right;">
21.8
</td>
<td style="text-align:right;">
10.4
</td>
<td style="text-align:right;">
22.6
</td>
<td style="text-align:right;">
12.2
</td>
<td style="text-align:right;">
9.4
</td>
<td style="text-align:right;">
2.8
</td>
<td style="text-align:right;">
44.2
</td>
<td style="text-align:right;">
3.1
</td>
<td style="text-align:right;">
5.2
</td>
<td style="text-align:right;">
4.5
</td>
<td style="text-align:right;">
21.0
</td>
<td style="text-align:right;">
35.3
</td>
<td style="text-align:right;">
1.0
</td>
<td style="text-align:right;">
8.9
</td>
<td style="text-align:right;">
14.4
</td>
<td style="text-align:right;">
0.71
</td>
<td style="text-align:right;">
78.65
</td>
<td style="text-align:right;">
4.3
</td>
<td style="text-align:right;">
66.0
</td>
<td style="text-align:right;">
62.3
</td>
<td style="text-align:right;">
39.6
</td>
<td style="text-align:right;">
99.0
</td>
<td style="text-align:right;">
0.0
</td>
<td style="text-align:right;">
0.0
</td>
<td style="text-align:right;">
16.1
</td>
<td style="text-align:right;">
0.0
</td>
<td style="text-align:right;">
11.3
</td>
<td style="text-align:right;">
29.8
</td>
<td style="text-align:right;">
6.0
</td>
<td style="text-align:right;">
7.2
</td>
<td style="text-align:right;">
15.0
</td>
<td style="text-align:right;">
6.9
</td>
<td style="text-align:right;">
0.92
</td>
<td style="text-align:right;">
1572
</td>
<td style="text-align:right;">
2464
</td>
<td style="text-align:right;">
1316.4
</td>
<td style="text-align:right;">
2842
</td>
<td style="text-align:right;">
17.9
</td>
<td style="text-align:right;">
21.6
</td>
<td style="text-align:right;">
42238
</td>
<td style="text-align:right;">
14.9
</td>
<td style="text-align:right;">
3433
</td>
<td style="text-align:right;">
2872
</td>
<td style="text-align:right;">
3012
</td>
<td style="text-align:right;">
2184.3
</td>
<td style="text-align:right;">
5.3
</td>
<td style="text-align:right;">
50.7
</td>
<td style="text-align:right;">
48.2
</td>
<td style="text-align:right;">
4.5
</td>
<td style="text-align:right;">
40.5
</td>
<td style="text-align:right;">
0.5
</td>
<td style="text-align:right;">
6.0
</td>
<td style="text-align:right;">
7671.7
</td>
<td style="text-align:right;">
563.9
</td>
<td style="text-align:right;">
55.4
</td>
<td style="text-align:right;">
23.8
</td>
<td style="text-align:right;">
32.8
</td>
<td style="text-align:right;">
7.5
</td>
<td style="text-align:right;">
8.62
</td>
<td style="text-align:right;">
448.4
</td>
<td style="text-align:right;">
23.8
</td>
<td style="text-align:right;">
30.4
</td>
<td style="text-align:right;">
126.6
</td>
<td style="text-align:right;">
51.9
</td>
<td style="text-align:right;">
158.6
</td>
<td style="text-align:right;">
29.3
</td>
<td style="text-align:right;">
13.0
</td>
<td style="text-align:right;">
85.5
</td>
<td style="text-align:right;">
40.1
</td>
<td style="text-align:right;">
62.2
</td>
<td style="text-align:right;">
7.6
</td>
<td style="text-align:right;">
337.3
</td>
<td style="text-align:right;">
1650.6
</td>
<td style="text-align:right;">
35.2
</td>
<td style="text-align:right;">
775.4
</td>
<td style="text-align:right;">
3.5
</td>
<td style="text-align:left;">
2
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
1108
</td>
<td style="text-align:right;">
1645
</td>
<td style="text-align:right;">
26087
</td>
<td style="text-align:right;">
15.6
</td>
<td style="text-align:right;">
21.3
</td>
<td style="text-align:right;">
17.5
</td>
<td style="text-align:right;">
22.5
</td>
<td style="text-align:right;">
8.5
</td>
<td style="text-align:right;">
16.1
</td>
<td style="text-align:right;">
8.62
</td>
<td style="text-align:right;">
25.2
</td>
<td style="text-align:right;">
133.9
</td>
<td style="text-align:right;">
7.76
</td>
<td style="text-align:right;">
1.99
</td>
<td style="text-align:right;">
6.4
</td>
<td style="text-align:right;">
7
</td>
<td style="text-align:right;">
47
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
588
</td>
<td style="text-align:right;">
85
</td>
<td style="text-align:right;">
645
</td>
<td style="text-align:right;">
82
</td>
<td style="text-align:right;">
682
</td>
<td style="text-align:right;">
81
</td>
<td style="text-align:right;">
187
</td>
<td style="text-align:right;">
100
</td>
<td style="text-align:right;">
514
</td>
<td style="text-align:right;">
54.5
</td>
<td style="text-align:right;">
38.6
</td>
<td style="text-align:right;">
25.9
</td>
<td style="text-align:right;">
14.7
</td>
<td style="text-align:right;">
4.4
</td>
<td style="text-align:right;">
3.6
</td>
<td style="text-align:right;">
1.6
</td>
<td style="text-align:right;">
1.6
</td>
<td style="text-align:right;">
1.4
</td>
<td style="text-align:right;">
13.9
</td>
<td style="text-align:right;">
2.4
</td>
<td style="text-align:right;">
26.6
</td>
<td style="text-align:right;">
2.1
</td>
<td style="text-align:right;">
129.8
</td>
<td style="text-align:right;">
31.3
</td>
<td style="text-align:right;">
5.06
</td>
<td style="text-align:right;">
39.5
</td>
<td style="text-align:right;">
61.9
</td>
<td style="text-align:right;">
17.5
</td>
<td style="text-align:right;">
25.0
</td>
<td style="text-align:right;">
4.8
</td>
<td style="text-align:right;">
70.6
</td>
<td style="text-align:right;">
7.2
</td>
<td style="text-align:right;">
2.4
</td>
<td style="text-align:right;">
29.3
</td>
<td style="text-align:right;">
85.5
</td>
<td style="text-align:right;">
100.0
</td>
<td style="text-align:right;">
39620.9
</td>
<td style="text-align:right;">
4.5
</td>
<td style="text-align:right;">
56.2
</td>
<td style="text-align:right;">
27.7
</td>
<td style="text-align:right;">
97.9
</td>
<td style="text-align:right;">
34.7
</td>
<td style="text-align:right;">
48.2
</td>
<td style="text-align:right;">
17.1
</td>
<td style="text-align:right;">
0.13
</td>
<td style="text-align:right;">
0.11
</td>
<td style="text-align:right;">
0.30
</td>
<td style="text-align:right;">
0.43
</td>
<td style="text-align:right;">
0.56
</td>
<td style="text-align:right;">
1.52
</td>
<td style="text-align:right;">
11.25
</td>
<td style="text-align:right;">
9.52
</td>
<td style="text-align:right;">
26.65
</td>
<td style="text-align:right;">
38.50
</td>
<td style="text-align:right;">
50.19
</td>
<td style="text-align:right;">
136.11
</td>
<td style="text-align:right;">
108.19
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.03
</td>
<td style="text-align:right;">
0.27
</td>
<td style="text-align:right;">
0.03
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.13
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.06
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.10
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.03
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.25
</td>
<td style="text-align:right;">
0.06
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.32
</td>
<td style="text-align:right;">
0.56
</td>
<td style="text-align:right;">
0.24
</td>
<td style="text-align:right;">
0.21
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.32
</td>
<td style="text-align:right;">
0.03
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.13
</td>
<td style="text-align:right;">
0.03
</td>
<td style="text-align:right;">
0.03
</td>
<td style="text-align:right;">
0.03
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
2.96
</td>
<td style="text-align:right;">
1.26
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
2.52
</td>
<td style="text-align:right;">
23.90
</td>
<td style="text-align:right;">
2.52
</td>
<td style="text-align:right;">
1.26
</td>
<td style="text-align:right;">
1.26
</td>
<td style="text-align:right;">
11.32
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
5.03
</td>
<td style="text-align:right;">
1.26
</td>
<td style="text-align:right;">
8.81
</td>
<td style="text-align:right;">
1.26
</td>
<td style="text-align:right;">
2.52
</td>
<td style="text-align:right;">
1.26
</td>
<td style="text-align:right;">
22.65
</td>
<td style="text-align:right;">
5.03
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
1.26
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
1.26
</td>
<td style="text-align:right;">
28.94
</td>
<td style="text-align:right;">
50.32
</td>
<td style="text-align:right;">
21.39
</td>
<td style="text-align:right;">
18.87
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
28.94
</td>
<td style="text-align:right;">
2.52
</td>
<td style="text-align:right;">
1.26
</td>
<td style="text-align:right;">
11.32
</td>
<td style="text-align:right;">
2.52
</td>
<td style="text-align:right;">
2.52
</td>
<td style="text-align:right;">
2.52
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
265.45
</td>
<td style="text-align:right;">
211
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.05
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.05
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.02
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.05
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.13
</td>
<td style="text-align:right;">
0.16
</td>
<td style="text-align:right;">
0.08
</td>
<td style="text-align:right;">
0.09
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.17
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.10
</td>
<td style="text-align:right;">
3521
</td>
</tr>
<tr>
<td style="text-align:right;">
19.75
</td>
<td style="text-align:right;">
9.11
</td>
<td style="text-align:right;">
54.13
</td>
<td style="text-align:left;">
108
</td>
<td style="text-align:left;">
Schleswig-Holstein
</td>
<td style="text-align:right;">
133210
</td>
<td style="text-align:right;">
6.8
</td>
<td style="text-align:right;">
55.4
</td>
<td style="text-align:right;">
19.3
</td>
<td style="text-align:right;">
7.6
</td>
<td style="text-align:right;">
13.7
</td>
<td style="text-align:right;">
43.0
</td>
<td style="text-align:right;">
17.8
</td>
<td style="text-align:right;">
41.3
</td>
<td style="text-align:right;">
6.3
</td>
<td style="text-align:right;">
30.5
</td>
<td style="text-align:right;">
52.5
</td>
<td style="text-align:right;">
9.3
</td>
<td style="text-align:right;">
16.6
</td>
<td style="text-align:right;">
94.1
</td>
<td style="text-align:right;">
5.8
</td>
<td style="text-align:right;">
54.3
</td>
<td style="text-align:right;">
55.8
</td>
<td style="text-align:right;">
19.7
</td>
<td style="text-align:right;">
45.2
</td>
<td style="text-align:right;">
23.3
</td>
<td style="text-align:right;">
50.1
</td>
<td style="text-align:right;">
16.1
</td>
<td style="text-align:right;">
30.0
</td>
<td style="text-align:right;">
81.1
</td>
<td style="text-align:right;">
130.9
</td>
<td style="text-align:right;">
2.4
</td>
<td style="text-align:right;">
2.4
</td>
<td style="text-align:right;">
4.8
</td>
<td style="text-align:right;">
11.2
</td>
<td style="text-align:right;">
7.8
</td>
<td style="text-align:right;">
5.2
</td>
<td style="text-align:right;">
22.6
</td>
<td style="text-align:right;">
23.8
</td>
<td style="text-align:right;">
11.9
</td>
<td style="text-align:right;">
24.6
</td>
<td style="text-align:right;">
12.7
</td>
<td style="text-align:right;">
9.9
</td>
<td style="text-align:right;">
2.8
</td>
<td style="text-align:right;">
45.8
</td>
<td style="text-align:right;">
0.4
</td>
<td style="text-align:right;">
6.1
</td>
<td style="text-align:right;">
2.4
</td>
<td style="text-align:right;">
20.5
</td>
<td style="text-align:right;">
39.2
</td>
<td style="text-align:right;">
4.6
</td>
<td style="text-align:right;">
8.0
</td>
<td style="text-align:right;">
13.3
</td>
<td style="text-align:right;">
0.77
</td>
<td style="text-align:right;">
79.28
</td>
<td style="text-align:right;">
3.7
</td>
<td style="text-align:right;">
72.2
</td>
<td style="text-align:right;">
23.1
</td>
<td style="text-align:right;">
38.6
</td>
<td style="text-align:right;">
99.3
</td>
<td style="text-align:right;">
13.7
</td>
<td style="text-align:right;">
13.7
</td>
<td style="text-align:right;">
19.7
</td>
<td style="text-align:right;">
17.5
</td>
<td style="text-align:right;">
11.2
</td>
<td style="text-align:right;">
27.8
</td>
<td style="text-align:right;">
3.6
</td>
<td style="text-align:right;">
8.6
</td>
<td style="text-align:right;">
12.7
</td>
<td style="text-align:right;">
6.8
</td>
<td style="text-align:right;">
0.96
</td>
<td style="text-align:right;">
1793
</td>
<td style="text-align:right;">
2317
</td>
<td style="text-align:right;">
1260.8
</td>
<td style="text-align:right;">
2914
</td>
<td style="text-align:right;">
12.8
</td>
<td style="text-align:right;">
17.3
</td>
<td style="text-align:right;">
36985
</td>
<td style="text-align:right;">
13.7
</td>
<td style="text-align:right;">
3343
</td>
<td style="text-align:right;">
2924
</td>
<td style="text-align:right;">
3248
</td>
<td style="text-align:right;">
844.5
</td>
<td style="text-align:right;">
0.5
</td>
<td style="text-align:right;">
11.1
</td>
<td style="text-align:right;">
58.6
</td>
<td style="text-align:right;">
3.8
</td>
<td style="text-align:right;">
402.7
</td>
<td style="text-align:right;">
10.2
</td>
<td style="text-align:right;">
9.5
</td>
<td style="text-align:right;">
1607.3
</td>
<td style="text-align:right;">
518.4
</td>
<td style="text-align:right;">
46.1
</td>
<td style="text-align:right;">
25.3
</td>
<td style="text-align:right;">
21.7
</td>
<td style="text-align:right;">
0.0
</td>
<td style="text-align:right;">
5.26
</td>
<td style="text-align:right;">
411.9
</td>
<td style="text-align:right;">
23.6
</td>
<td style="text-align:right;">
28.8
</td>
<td style="text-align:right;">
125.9
</td>
<td style="text-align:right;">
43.7
</td>
<td style="text-align:right;">
130.8
</td>
<td style="text-align:right;">
20.6
</td>
<td style="text-align:right;">
2.9
</td>
<td style="text-align:right;">
85.1
</td>
<td style="text-align:right;">
7.9
</td>
<td style="text-align:right;">
54.4
</td>
<td style="text-align:right;">
8.3
</td>
<td style="text-align:right;">
366.3
</td>
<td style="text-align:right;">
1831.6
</td>
<td style="text-align:right;">
53.4
</td>
<td style="text-align:right;">
824.7
</td>
<td style="text-align:right;">
7.9
</td>
<td style="text-align:left;">
2
</td>
<td style="text-align:right;">
74
</td>
<td style="text-align:right;">
2
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
93
</td>
<td style="text-align:right;">
122
</td>
<td style="text-align:right;">
11103
</td>
<td style="text-align:right;">
11.8
</td>
<td style="text-align:right;">
13.2
</td>
<td style="text-align:right;">
19.9
</td>
<td style="text-align:right;">
20.2
</td>
<td style="text-align:right;">
6.9
</td>
<td style="text-align:right;">
24.2
</td>
<td style="text-align:right;">
5.35
</td>
<td style="text-align:right;">
19.1
</td>
<td style="text-align:right;">
94.6
</td>
<td style="text-align:right;">
7.64
</td>
<td style="text-align:right;">
1.91
</td>
<td style="text-align:right;">
6.1
</td>
<td style="text-align:right;">
14
</td>
<td style="text-align:right;">
71
</td>
<td style="text-align:right;">
19
</td>
<td style="text-align:right;">
65
</td>
<td style="text-align:right;">
14
</td>
<td style="text-align:right;">
1864
</td>
<td style="text-align:right;">
51
</td>
<td style="text-align:right;">
248
</td>
<td style="text-align:right;">
41
</td>
<td style="text-align:right;">
1823
</td>
<td style="text-align:right;">
50
</td>
<td style="text-align:right;">
576
</td>
<td style="text-align:right;">
87
</td>
<td style="text-align:right;">
583
</td>
<td style="text-align:right;">
70.8
</td>
<td style="text-align:right;">
73.5
</td>
<td style="text-align:right;">
-15.8
</td>
<td style="text-align:right;">
16.7
</td>
<td style="text-align:right;">
4.0
</td>
<td style="text-align:right;">
3.1
</td>
<td style="text-align:right;">
-0.1
</td>
<td style="text-align:right;">
0.5
</td>
<td style="text-align:right;">
0.5
</td>
<td style="text-align:right;">
84.3
</td>
<td style="text-align:right;">
11.7
</td>
<td style="text-align:right;">
2.6
</td>
<td style="text-align:right;">
4.9
</td>
<td style="text-align:right;">
91.7
</td>
<td style="text-align:right;">
16.0
</td>
<td style="text-align:right;">
1.77
</td>
<td style="text-align:right;">
31.4
</td>
<td style="text-align:right;">
70.1
</td>
<td style="text-align:right;">
11.8
</td>
<td style="text-align:right;">
18.9
</td>
<td style="text-align:right;">
2.5
</td>
<td style="text-align:right;">
47.2
</td>
<td style="text-align:right;">
8.6
</td>
<td style="text-align:right;">
1.9
</td>
<td style="text-align:right;">
20.6
</td>
<td style="text-align:right;">
85.1
</td>
<td style="text-align:right;">
63.0
</td>
<td style="text-align:right;">
31456.6
</td>
<td style="text-align:right;">
2.8
</td>
<td style="text-align:right;">
55.8
</td>
<td style="text-align:right;">
26.3
</td>
<td style="text-align:right;">
71.8
</td>
<td style="text-align:right;">
32.9
</td>
<td style="text-align:right;">
47.4
</td>
<td style="text-align:right;">
19.7
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.08
</td>
<td style="text-align:right;">
0.28
</td>
<td style="text-align:right;">
0.25
</td>
<td style="text-align:right;">
0.65
</td>
<td style="text-align:right;">
41.70
</td>
<td style="text-align:right;">
6.89
</td>
<td style="text-align:right;">
90.28
</td>
<td style="text-align:right;">
297.00
</td>
<td style="text-align:right;">
262.48
</td>
<td style="text-align:right;">
698.35
</td>
<td style="text-align:right;">
930.27
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.02
</td>
<td style="text-align:right;">
0.03
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.02
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.06
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.02
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.02
</td>
<td style="text-align:right;">
0.07
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0.03
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.05
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.45
</td>
<td style="text-align:right;">
4.50
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
6.01
</td>
<td style="text-align:right;">
0.75
</td>
<td style="text-align:right;">
18.77
</td>
<td style="text-align:right;">
28.53
</td>
<td style="text-align:right;">
1.50
</td>
<td style="text-align:right;">
0.75
</td>
<td style="text-align:right;">
1.50
</td>
<td style="text-align:right;">
25.52
</td>
<td style="text-align:right;">
4.50
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
6.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
60.06
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
12.01
</td>
<td style="text-align:right;">
9.01
</td>
<td style="text-align:right;">
19.52
</td>
<td style="text-align:right;">
0.75
</td>
<td style="text-align:right;">
7.51
</td>
<td style="text-align:right;">
0.75
</td>
<td style="text-align:right;">
1.50
</td>
<td style="text-align:right;">
20.27
</td>
<td style="text-align:right;">
78.82
</td>
<td style="text-align:right;">
37.53
</td>
<td style="text-align:right;">
27.78
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
52.55
</td>
<td style="text-align:right;">
6.01
</td>
<td style="text-align:right;">
4.50
</td>
<td style="text-align:right;">
38.29
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
6.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
481.20
</td>
<td style="text-align:right;">
641
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.02
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.03
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
4041
</td>
</tr>
<tr>
<td style="text-align:right;">
54.83
</td>
<td style="text-align:right;">
10.60
</td>
<td style="text-align:right;">
53.59
</td>
<td style="text-align:left;">
108
</td>
<td style="text-align:left;">
Schleswig-Holstein
</td>
<td style="text-align:right;">
197264
</td>
<td style="text-align:right;">
5.5
</td>
<td style="text-align:right;">
44.9
</td>
<td style="text-align:right;">
22.6
</td>
<td style="text-align:right;">
6.4
</td>
<td style="text-align:right;">
11.4
</td>
<td style="text-align:right;">
32.3
</td>
<td style="text-align:right;">
22.3
</td>
<td style="text-align:right;">
44.5
</td>
<td style="text-align:right;">
6.3
</td>
<td style="text-align:right;">
35.9
</td>
<td style="text-align:right;">
114.1
</td>
<td style="text-align:right;">
22.7
</td>
<td style="text-align:right;">
19.5
</td>
<td style="text-align:right;">
90.7
</td>
<td style="text-align:right;">
9.2
</td>
<td style="text-align:right;">
47.5
</td>
<td style="text-align:right;">
58.2
</td>
<td style="text-align:right;">
20.0
</td>
<td style="text-align:right;">
51.2
</td>
<td style="text-align:right;">
19.8
</td>
<td style="text-align:right;">
45.4
</td>
<td style="text-align:right;">
17.1
</td>
<td style="text-align:right;">
32.5
</td>
<td style="text-align:right;">
82.1
</td>
<td style="text-align:right;">
136.0
</td>
<td style="text-align:right;">
2.8
</td>
<td style="text-align:right;">
2.8
</td>
<td style="text-align:right;">
5.6
</td>
<td style="text-align:right;">
11.8
</td>
<td style="text-align:right;">
6.8
</td>
<td style="text-align:right;">
4.9
</td>
<td style="text-align:right;">
24.9
</td>
<td style="text-align:right;">
23.8
</td>
<td style="text-align:right;">
10.2
</td>
<td style="text-align:right;">
22.3
</td>
<td style="text-align:right;">
12.0
</td>
<td style="text-align:right;">
9.3
</td>
<td style="text-align:right;">
2.8
</td>
<td style="text-align:right;">
44.6
</td>
<td style="text-align:right;">
4.3
</td>
<td style="text-align:right;">
6.1
</td>
<td style="text-align:right;">
2.5
</td>
<td style="text-align:right;">
22.2
</td>
<td style="text-align:right;">
35.0
</td>
<td style="text-align:right;">
8.3
</td>
<td style="text-align:right;">
9.1
</td>
<td style="text-align:right;">
12.1
</td>
<td style="text-align:right;">
0.87
</td>
<td style="text-align:right;">
80.58
</td>
<td style="text-align:right;">
3.9
</td>
<td style="text-align:right;">
76.8
</td>
<td style="text-align:right;">
10.3
</td>
<td style="text-align:right;">
36.3
</td>
<td style="text-align:right;">
100.1
</td>
<td style="text-align:right;">
0.0
</td>
<td style="text-align:right;">
0.0
</td>
<td style="text-align:right;">
16.7
</td>
<td style="text-align:right;">
0.0
</td>
<td style="text-align:right;">
11.8
</td>
<td style="text-align:right;">
22.5
</td>
<td style="text-align:right;">
4.0
</td>
<td style="text-align:right;">
9.6
</td>
<td style="text-align:right;">
10.8
</td>
<td style="text-align:right;">
10.6
</td>
<td style="text-align:right;">
1.11
</td>
<td style="text-align:right;">
1903
</td>
<td style="text-align:right;">
2404
</td>
<td style="text-align:right;">
1257.6
</td>
<td style="text-align:right;">
2832
</td>
<td style="text-align:right;">
10.1
</td>
<td style="text-align:right;">
11.7
</td>
<td style="text-align:right;">
54979
</td>
<td style="text-align:right;">
20.9
</td>
<td style="text-align:right;">
3404
</td>
<td style="text-align:right;">
287
</td>
<td style="text-align:right;">
3031
</td>
<td style="text-align:right;">
1344.4
</td>
<td style="text-align:right;">
0.9
</td>
<td style="text-align:right;">
11.5
</td>
<td style="text-align:right;">
59.8
</td>
<td style="text-align:right;">
26.0
</td>
<td style="text-align:right;">
1673.3
</td>
<td style="text-align:right;">
3.3
</td>
<td style="text-align:right;">
5.0
</td>
<td style="text-align:right;">
2476.9
</td>
<td style="text-align:right;">
490.2
</td>
<td style="text-align:right;">
47.9
</td>
<td style="text-align:right;">
23.3
</td>
<td style="text-align:right;">
19.9
</td>
<td style="text-align:right;">
12.7
</td>
<td style="text-align:right;">
3.14
</td>
<td style="text-align:right;">
406.7
</td>
<td style="text-align:right;">
25.6
</td>
<td style="text-align:right;">
28.4
</td>
<td style="text-align:right;">
111.1
</td>
<td style="text-align:right;">
56.2
</td>
<td style="text-align:right;">
126.2
</td>
<td style="text-align:right;">
31.7
</td>
<td style="text-align:right;">
19.2
</td>
<td style="text-align:right;">
83.1
</td>
<td style="text-align:right;">
38.5
</td>
<td style="text-align:right;">
40.9
</td>
<td style="text-align:right;">
7.5
</td>
<td style="text-align:right;">
474.9
</td>
<td style="text-align:right;">
1251.8
</td>
<td style="text-align:right;">
52.9
</td>
<td style="text-align:right;">
791.1
</td>
<td style="text-align:right;">
1.6
</td>
<td style="text-align:left;">
2
</td>
<td style="text-align:right;">
36
</td>
<td style="text-align:right;">
3
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
155
</td>
<td style="text-align:right;">
193
</td>
<td style="text-align:right;">
35799
</td>
<td style="text-align:right;">
8.6
</td>
<td style="text-align:right;">
10.3
</td>
<td style="text-align:right;">
27.4
</td>
<td style="text-align:right;">
13.0
</td>
<td style="text-align:right;">
7.8
</td>
<td style="text-align:right;">
14.9
</td>
<td style="text-align:right;">
4.62
</td>
<td style="text-align:right;">
13.5
</td>
<td style="text-align:right;">
71.8
</td>
<td style="text-align:right;">
7.77
</td>
<td style="text-align:right;">
2.02
</td>
<td style="text-align:right;">
6.9
</td>
<td style="text-align:right;">
10
</td>
<td style="text-align:right;">
30
</td>
<td style="text-align:right;">
20
</td>
<td style="text-align:right;">
31
</td>
<td style="text-align:right;">
13
</td>
<td style="text-align:right;">
1545
</td>
<td style="text-align:right;">
63
</td>
<td style="text-align:right;">
2042
</td>
<td style="text-align:right;">
49
</td>
<td style="text-align:right;">
1804
</td>
<td style="text-align:right;">
48
</td>
<td style="text-align:right;">
343
</td>
<td style="text-align:right;">
94
</td>
<td style="text-align:right;">
572
</td>
<td style="text-align:right;">
71.7
</td>
<td style="text-align:right;">
79.7
</td>
<td style="text-align:right;">
-52.4
</td>
<td style="text-align:right;">
15.6
</td>
<td style="text-align:right;">
4.1
</td>
<td style="text-align:right;">
2.9
</td>
<td style="text-align:right;">
-0.5
</td>
<td style="text-align:right;">
0.2
</td>
<td style="text-align:right;">
0.3
</td>
<td style="text-align:right;">
23.6
</td>
<td style="text-align:right;">
3.4
</td>
<td style="text-align:right;">
4.9
</td>
<td style="text-align:right;">
3.2
</td>
<td style="text-align:right;">
94.4
</td>
<td style="text-align:right;">
15.0
</td>
<td style="text-align:right;">
1.85
</td>
<td style="text-align:right;">
21.7
</td>
<td style="text-align:right;">
61.0
</td>
<td style="text-align:right;">
9.1
</td>
<td style="text-align:right;">
13.4
</td>
<td style="text-align:right;">
2.6
</td>
<td style="text-align:right;">
41.8
</td>
<td style="text-align:right;">
9.6
</td>
<td style="text-align:right;">
1.7
</td>
<td style="text-align:right;">
31.7
</td>
<td style="text-align:right;">
83.1
</td>
<td style="text-align:right;">
71.7
</td>
<td style="text-align:right;">
21795.1
</td>
<td style="text-align:right;">
2.6
</td>
<td style="text-align:right;">
58.2
</td>
<td style="text-align:right;">
27.4
</td>
<td style="text-align:right;">
92.6
</td>
<td style="text-align:right;">
27.3
</td>
<td style="text-align:right;">
47.7
</td>
<td style="text-align:right;">
25.1
</td>
<td style="text-align:right;">
0.09
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.14
</td>
<td style="text-align:right;">
0.24
</td>
<td style="text-align:right;">
0.30
</td>
<td style="text-align:right;">
0.78
</td>
<td style="text-align:right;">
58.77
</td>
<td style="text-align:right;">
5.10
</td>
<td style="text-align:right;">
90.62
</td>
<td style="text-align:right;">
153.91
</td>
<td style="text-align:right;">
193.65
</td>
<td style="text-align:right;">
502.04
</td>
<td style="text-align:right;">
990.34
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0.10
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.03
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.03
</td>
<td style="text-align:right;">
0.02
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.03
</td>
<td style="text-align:right;">
0.11
</td>
<td style="text-align:right;">
0.05
</td>
<td style="text-align:right;">
0.05
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.06
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.04
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.69
</td>
<td style="text-align:right;">
3.55
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
9.12
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
24.33
</td>
<td style="text-align:right;">
64.38
</td>
<td style="text-align:right;">
1.52
</td>
<td style="text-align:right;">
0.51
</td>
<td style="text-align:right;">
1.52
</td>
<td style="text-align:right;">
8.11
</td>
<td style="text-align:right;">
2.53
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
4.06
</td>
<td style="text-align:right;">
0.51
</td>
<td style="text-align:right;">
18.76
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
1.01
</td>
<td style="text-align:right;">
8.62
</td>
<td style="text-align:right;">
19.77
</td>
<td style="text-align:right;">
14.70
</td>
<td style="text-align:right;">
6.08
</td>
<td style="text-align:right;">
7.60
</td>
<td style="text-align:right;">
0.51
</td>
<td style="text-align:right;">
1.01
</td>
<td style="text-align:right;">
20.78
</td>
<td style="text-align:right;">
70.97
</td>
<td style="text-align:right;">
32.95
</td>
<td style="text-align:right;">
31.94
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
2.03
</td>
<td style="text-align:right;">
39.03
</td>
<td style="text-align:right;">
8.11
</td>
<td style="text-align:right;">
3.04
</td>
<td style="text-align:right;">
27.37
</td>
<td style="text-align:right;">
0.51
</td>
<td style="text-align:right;">
3.55
</td>
<td style="text-align:right;">
1.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
439.51
</td>
<td style="text-align:right;">
867
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.02
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0.02
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.00
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.01
</td>
<td style="text-align:right;">
4296
</td>
</tr>
</tbody>
</table>

The first column (`AdjRate`) contains age-adjusted incidence rates. The `X` and `Y` columns represent longitude and latitude. `NUTS2_Fact` represents government regions (Regierungsbezirke), `BL_ID` is the federal state (Bundesland), and `EWZ` is the population. The remaining columns cover socioeconomic, infrastructural, and built environment features. We will filter out only the most relevant features later.

BART requires a *data.frame* of predictors and a separate vector for the response variable:

``` r
psych::skew(data$AdjRate)
```

    ## [1] 3.64179

``` r
# Response variable
y <- log(data$AdjRate)

# Data.frame of predictors
data_bm <- select(data, -c(AdjRate))
```

Because `AdjRate` is highly skewed (skewness = 3.64), we apply a log transformation. Though BART is non-parametric and can handle skewed data, transforming the response variable aids in visually interpreting results.

Below is a summary of the log-transformed `AdjRate`:

<table class=" lightable-classic lightable-striped" style="font-family: &quot;Arial Narrow&quot;, &quot;Source Sans Pro&quot;, sans-serif; width: auto !important; margin-left: auto; margin-right: auto;">
<thead>
<tr>
<th style="text-align:right;font-weight: bold;">
Mean
</th>
<th style="text-align:right;font-weight: bold;">
SD
</th>
<th style="text-align:right;font-weight: bold;">
Min
</th>
<th style="text-align:right;font-weight: bold;">
Max
</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:right;">
4.13
</td>
<td style="text-align:right;">
0.7
</td>
<td style="text-align:right;">
1.75
</td>
<td style="text-align:right;">
6.51
</td>
</tr>
</tbody>
</table>

## First BART model

We can use default hyperparameter values for BART, but `bartMachine` offers a convenient tuning function called `bartMachineCV` to find optimal values. Because that process can be time-consuming, I’ve omitted it here. Below, I simply build a BART model using those *pre-determined* optimal hyperparameters.

``` r
bm_All <- bartMachine(X = data_bm, y = y, 
                      k=2, nu=3, q=0.9, num_trees=100, 
                      num_iterations_after_burn_in=2000, 
                      num_burn_in = 300, 
                      seed = 1234, verbose = FALSE)
```

    ## Warning in build_bart_machine(X = X, y = y, Xy = Xy, num_trees = num_trees, : Setting the seed when using parallelization does not result in deterministic output.
    ## If you need deterministic output, you must run "set_bart_machine_num_cores(1)" and then build the BART model with the set seed.

``` r
summary(bm_All)
```

    ## bartMachine v1.3.4.1 for regression
    ## 
    ## training data size: n = 401 and p = 366 
    ## built in 6.1 secs on 20 cores, 100 trees, 300 burn-in and 2000 post. samples
    ## 
    ## sigsq est for y beforehand: 0.024 
    ## avg sigsq estimate after burn-in: 0.09589 
    ## 
    ## in-sample statistics:
    ##  L1 = 60.66 
    ##  L2 = 15.61 
    ##  rmse = 0.2 
    ##  Pseudo-Rsq = 0.9193
    ## p-val for shapiro-wilk test of normality of residuals: 0.00015 
    ## p-val for zero-mean noise: 0.97071

The Pseudo-R² of 0.92 and RMSE of 0.2 look promising. Let’s check the error assumptions next:

``` r
check_bart_error_assumptions(bm_All)
```

![](errorAssumption.png)

Both diagnostic plots look reasonable. In the second one (Assessment of Heteroskedasticity), there is a mild pattern, but not a stark one. Overall, the model seems to fit well, though it might struggle slightly with extreme values.

``` r
plot_y_vs_yhat(bm_All, credible_intervals = TRUE)
```

![](yVSyhat.png)

The “Fitted vs. Actual Values” plot also demonstrates good model performance. Again, we see that the model tends to *over-predict* extremely low values and *under-predict* extremely high values. We could also map the residuals to check for spatial clustering, but let’s first reduce the model’s complexity by removing non-important predictors.

## Variable Selection

Although BART yields *excellent* R² values, part of its power lies in its ability to handle high-dimensional data. In large datasets, often only a fraction of the predictors significantly influence the response variable. *Occam’s razor* suggests favoring simpler models over overly complex ones.

The function `var_selection_by_permute` (introduced by (Bleich et al. 2014)) helps reduce model complexity. Feel free to skip the following code and just trust my results—this step can be time-consuming:

``` r
# Leave the num_trees_for_permute small, to force variables to compete for entry into the model!
var_sel <- bartMachine::var_selection_by_permute_cv(bm_All, num_trees_for_permute = 20)

# Look at the most important variables
var_sel$important_vars_cv
```

    ##  [1] "BL_ID_Baden-Württemberg" "BL_ID_Bayern"           
    ##  [3] "BL_ID_Hessen"            "ff_pop"                 
    ##  [5] "hair_pp"                 "thea_pp"                
    ##  [7] "NUTS2_Fact_11"           "NUTS2_Fact_27"          
    ##  [9] "NUTS2_Fact_40"           "NUTS2_Fact_71"          
    ## [11] "S_170"                   "play_dn"                
    ## [13] "bir_km2"                 "cc_pop"                 
    ## [15] "sch_den"                 "kid_den"                
    ## [17] "Rch_den"                 "S_109_2"                
    ## [19] "EWZ"                     "Pop_Den"                
    ## [21] "S_004"                   "S_006"                  
    ## [23] "S_020"                   "S_051"                  
    ## [25] "S_054"                   "S_066"                  
    ## [27] "S_070"                   "S_080"                  
    ## [29] "S_104"                   "S_107"                  
    ## [31] "S_115"                   "S_123"                  
    ## [33] "S_130"                   "S_146"                  
    ## [35] "S_153"                   "X"                      
    ## [37] "Y"

Below is the subset of important variables (grouped thematically). I also removed *NUTS2_Fact* and *BL_ID*:

``` r
data_subset <- data_bm %>%
  select(c(
    # Geographical Units
    X, #Longitude
    Y, #Latitude
    
    # Political units
    S_109, #Rural/Urban
    
    # Socioeconomic
    EWZ, #Population
    Pop_Den, #Population density
    S_004, #Unemployment rate under 25
    S_006, #Household income per capita 
    S_020, #Employment rate 15-<30
    S_051, #Voter participation
    S_054, #Apprenticeship positions
    S_066, #Household income
    S_070, #Deptors rate
    S_080, #Recreationl Space
    S_104, #Income tax
    S_107, #Steuerkraft
    S_115, #Regional population potential
    S_123, #Child poverty
    S_130, #IC train station access
    S_146, #Commuters >150km
    S_153, #Foreign guests in tourist establishments
    S_170, #Longterm unemployment rate
    
    # Built environment
    Rch_den, #Church density
    play_dn, #Playground density
    bir_km2, #Biergarten per km²
    ff_pop, #Fast food places per capita
    hair_pp, #Hairdresser per capita
    thea_pp, #Theatre per capita
    cc_pop, #Community centre density
    sch_den, #School density
    kid_den #Kindergarten density
  ))
```

## Second BART model

With this subset of key predictors, I now build a new BART model (again using *pre-determined* optimal hyperparameters):

``` r
bm_final <- bartMachine(X = data_subset, y = y, 
                        k=3, nu=3, q=0.99, num_trees=225,
                        seed = 1234, verbose = FALSE)
```

    ## Warning in build_bart_machine(X = X, y = y, Xy = Xy, num_trees = num_trees, : Setting the seed when using parallelization does not result in deterministic output.
    ## If you need deterministic output, you must run "set_bart_machine_num_cores(1)" and then build the BART model with the set seed.

``` r
summary(bm_final)
```

    ## bartMachine v1.3.4.1 for regression
    ## 
    ## training data size: n = 401 and p = 31 
    ## built in 5.9 secs on 20 cores, 225 trees, 250 burn-in and 1000 post. samples
    ## 
    ## sigsq est for y beforehand: 0.185 
    ## avg sigsq estimate after burn-in: 0.126 
    ## 
    ## in-sample statistics:
    ##  L1 = 89.17 
    ##  L2 = 33.3 
    ##  rmse = 0.29 
    ##  Pseudo-Rsq = 0.8279
    ## p-val for shapiro-wilk test of normality of residuals: 0.00128 
    ## p-val for zero-mean noise: 0.99946

Compared to the first model, the Pseudo-R² decreases from 0.92 to 0.83, equating to a 10% reduction in explained variance. The RMSE increased from 0.2 to 0.29, indicating that the final model predicts the age-adjusted incidence rate of COVID-19 with an accuracy of roughly +/− 1.3 cases per 100,000. Let’s again check the diagnostic plots:

``` r
check_bart_error_assumptions(bm_final)
```

![](errorAssumption2.png) The new model’s Q-Q and residuals plots look fine, but as expected, it performs slightly worse on extreme values.

``` r
plot_y_vs_yhat(bm_final, credible_intervals = TRUE)
```

![](yVSyhat2.png)

The “Fitted vs. Actual Values” plot shows *okay* performance, though more points lie outside the confidence intervals. Next, let’s map the residuals to check for any spatial clustering.

## Spatial autocorrelation

We can quickly visualize residuals by linking them to a shapefile of German counties (*NUTS3*). Any obvious clustering of positive or negative residuals would indicate spatial autocorrelation.

``` r
library(sf)
library(RColorBrewer)
library(tmap)

# Download shapefile
shp <- read_sf("https://github.com/STBrinkmann/data/raw/main/RKI_sf.gpkg")

# Sort shapefile, that it has the same order as the data_subset
shp <- shp[order(match(shp$EWZ, data_subset$EWZ)),]

# Join residuals to shapefile, then map residuals
shp$resid <- bm_final$residuals
tm_shape(shp) + 
  tm_polygons(col="resid", 
              title="BART Machine Residuals\n(log incidence rate)", 
              breaks=seq(-1.75, 1.75, 0.5), midpoint=NA, palette="RdBu") + 
  tm_layout(frame = FALSE,
            inner.margins=c(0.02, 0.02, 0.02, 0.20),
            legend.position = c(0.7, 0.22),
            legend.frame = TRUE,
            legend.outside = FALSE, 
            bg.color = "white")
```

<img src="{{< blogdown/postref >}}index.en_files/figure-html/unnamed-chunk-15-1.png" width="672" />

No clear geographical pattern emerges, suggesting no strong spatial clusters. For a formal test, we can compute *Moran’s I*. I will not explain Moran’s I in this post, but I would highly recommend these posts: [Intro to GIS and Spatial Analysis](https://mgimond.github.io/Spatial/spatial-autocorrelation-in-r.html#app8_3) and [Spatial autocorrelation analysis in R](https://rpubs.com/quarcs-lab/spatial-autocorrelation).

``` r
library(spdep)
# Define neighboring polygons
nb <- poly2nb(shp, queen=TRUE)

# Assign weights to each neighboring polygon
lw <- nb2listw(nb, style="B", zero.policy=TRUE)

# Compute Moran's I statistic using a Monte-Carlo simulation 
MC <- moran.mc(shp$resid, lw, nsim=99)
MC
```

    ## 
    ## 	Monte-Carlo simulation of Moran I
    ## 
    ## data:  shp$resid 
    ## weights: lw  
    ## number of simulations + 1: 100 
    ## 
    ## statistic = 0.059006, observed rank = 100, p-value = 0.01
    ## alternative hypothesis: greater

Moran’s I ranges from -1 to 1, where values closer to ±1 indicate strong positive/negative spatial autocorrelation, and 0 implies randomness. With a Moran’s I of 0.06 (p=0.01), we conclude there is no significant spatial autocorrelation in the residuals.

## Partial Dependence Plots

So far, we’ve built and refined a BART model to predict age-adjusted incidence rates of COVID-19 in German counties. However, our main goal is to explore the data rather than simply predict. BART’s high R² suggests it captures these non-linear relationships, and Partial Dependence Plots (PDPs) let us visualize and interpret them (Friedman 2002; Scarpone et al. 2017, 2020).

Although the [pdp](https://cran.r-project.org/web/packages/pdp/index.html) is great for computing PDPs, it does not directly support `bartMachine`. Fortunately, `bartMachine` provides its own PDP function, `pd_plot`. Below is a demonstration using `S_115` (Regional Population Potential):

``` r
# Set parameters to plot PDP top and histogram bottom
par(mfrow = c(2,1))

pd_plot(bm_final, "S_115", levs = c(0.0, seq(0, 1, 0.1), 1))
```

    ## ...........

``` r
hist(bm_final$X$S_115, 20, 
     main = "Histogram of the Regional Population Potential",
     xlab = "Regional Population Potential")
```

<img src="{{< blogdown/postref >}}index.en_files/figure-html/unnamed-chunk-17-1.png" width="768" />

*Regional population potential* measures the likelihood of direct human interactions in a region. This PDP suggests minimal marginal change in incidence at the lower end of the distribution, implying less chance of viral contagion in areas with fewer human interactions. The curve rises notably between the 20th and 80th percentiles (14,016 to 47,067), indicating a strong non-linear effect on incidence rates.

To log-transform the x-axis for clarity, you would need a custom PDP function, since `pd_plot` does not support that directly. I wrote a variant that returns ggplot2 objects [GitHub](https://github.com/CHEST-Lab/BART_Covid-19/blob/master/pdPlotGG.R)), which lets you customize plots extensively.

Below is a *multi-panel* PDP from our publication, showing the top 10 variables. You can see more details and interpretation of these variables [here](https://ij-healthgeographics.biomedcentral.com/articles/10.1186/s12942-020-00225-1#Sec18).

[![Partial Dependence Plots (PDP) of the 10 most prevalent variables in the final Bayesian Additive Regression Tree (BART) model. Histograms are shown for the entire country (green), for only the low rates region (LRR, teal), and for only the high rates region (HRR, purple). The PDPs indicate marginal changes in the predicted (log-transformed, age-adjusted) incidence rate per 100,000 residents (upper y-axis) for different values of each independent variable (x-axis)](WallOfShame.svg)](https://ij-healthgeographics.biomedcentral.com/articles/10.1186/s12942-020-00225-1/figures/6)

## Conclusion

In this post, we developed and refined a BART-based workflow for modeling age-adjusted COVID-19 incidence rates across German counties. We started by applying BART to a high-dimensional dataset of socioeconomic, infrastructural, and built environment predictors, then used a permutation-based variable selection technique to isolate the most important features. Although BART can handle dozens or even hundreds of predictors, our results showed that only a smaller subset of variables drove most of the explanatory power.

By visualizing these relationships with Partial Dependence Plots, we uncovered non-linear effects and interactions that more conventional methods might have missed. These plots also help “open the black box,” translating an otherwise opaque machine learning model into actionable insights. Although BART performed well overall, we observed that it tended to under-predict very high values and over-predict very low ones, highlighting the challenges of modeling extreme outcomes.

In future posts, I will focus on **out-of-sample validation** to confirm the model’s robustness and detect any lingering overfitting issues. I also plan to dive deeper into how the variable selection procedure can be tuned for different research contexts. Ultimately, BART’s Bayesian foundation—combined with flexible tree-ensemble strategies—offers a compelling toolkit for complex, spatially heterogeneous data like COVID-19 incidence.

## References

<div id="refs" class="references csl-bib-body hanging-indent" entry-spacing="0">

<div id="ref-bleich2014" class="csl-entry">

Bleich, Justin, Adam Kapelner, Edward I. George, and Shane T. Jensen. 2014. “Variable Selection for BART: An Application to Gene Regulation.” *The Annals of Applied Statistics* 8 (3). <https://doi.org/10.1214/14-aoas755>.

</div>

<div id="ref-breiman2001" class="csl-entry">

Breiman, Leo. 2001. “Statistical Modeling: The Two Cultures (with Comments and a Rejoinder by the Author).” *Statistical Science* 16 (3). <https://doi.org/10.1214/ss/1009213726>.

</div>

<div id="ref-Chipman2010" class="csl-entry">

Chipman, Hugh A., Edward I. George, and Robert E. McCulloch. 2010. “<span class="nocase">BART: Bayesian additive regressiontrees</span>.” *The Annals of Applied Statistics* 4 (1): 266–98. <https://doi.org/10.1214/09-AOAS285>.

</div>

<div id="ref-friedman2002" class="csl-entry">

Friedman, Jerome H. 2002. “Stochastic Gradient Boosting.” *Computational Statistics & Data Analysis* 38 (4): 367–78. <https://doi.org/10.1016/s0167-9473(01)00065-2>.

</div>

<div id="ref-kapelner2016" class="csl-entry">

Kapelner, Adam, and Justin Bleich. 2016. “bartMachine: Machine Learning with Bayesian Additive Regression Trees.” *Journal of Statistical Software* 70 (4). <https://doi.org/10.18637/jss.v070.i04>.

</div>

<div id="ref-scarpone2020" class="csl-entry">

Scarpone, Christopher, Sebastian T. Brinkmann, Tim Große, Daniel Sonnenwald, Martin Fuchs, and Blake Byron Walker. 2020. “A Multimethod Approach for County-Scale Geospatial Analysis of Emerging Infectious Diseases: A Cross-Sectional Case Study of COVID-19 Incidence in Germany.” *International Journal of Health Geographics* 19 (1). <https://doi.org/10.1186/s12942-020-00225-1>.

</div>

<div id="ref-scarpone2017" class="csl-entry">

Scarpone, Christopher, Margaret G. Schmidt, Chuck E. Bulmer, and Anders Knudby. 2017. “Semi-Automated Classification of Exposed Bedrock Cover in British Columbia’s Southern Mountains Using a Random Forest Approach.” *Geomorphology* 285 (May): 214–24. <https://doi.org/10.1016/j.geomorph.2017.02.013>.

</div>

</div>
