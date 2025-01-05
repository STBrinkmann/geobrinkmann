---
authors:
- admin
date: "2020-05-07T00:00:00Z"
draft: false
featured: false
image:
  focal_point: ""
  placement: 2
  preview_only: true
lastmod: "2020-05-07T00:00:00Z"
projects: []
subtitle: "Text Mining and Topic Modelling using a Structural Topic Model"
summary: "In 2001 a project has been created to transform the former steel production site in Belval, Luxembourg into the Cité des Sciences. In 2015 the new campus has been opened. In this analysis I'm using a Structural Topic Model (STM) to explore how this topic is being represented in the news."
tags: ["Text Mining", "R", "STM"]
title: Text Mining - Belval Campus
editor_options: 
  markdown: 
    wrap: 72
---

In 2001, an extensive project was initiated to transform the former
steel production site in Belval, Luxembourg, into the [Cité des
Sciences](https://wwwde.uni.lu/fhse/belval_campus). This transformation
sought to revitalize a region historically characterized by heavy
industry and to integrate research, education, and cultural activities
within a single, modern complex. By
[2015](https://lequotidien.lu/luxembourg/le-campus-de-belval-en-un-clin-doeil/),
the new campus opened, signifying a notable milestone in Luxembourg's
shift from a traditionally industrial economy to a knowledge-based one.
![Campus
Belval](https://www.uni.lu/wp-content/uploads/sites/9/2023/07/Esch-Belval_areal-view2_2014.jpg)

The architectural design of the campus preserves elements of the steel
industry---such as the iconic blast furnaces---to visually connect
Belval's industrial past with its current role as a center of higher
education and research. In this post, I apply text mining techniques,
including a **Structural Topic Model (STM)**, to analyze how the Belval
Campus has been represented in news coverage over several years. This
analysis reveals shifting themes of historical significance,
transformation, and broader societal implications.

### Data Collection

The data for this analysis consists of 58 news articles obtained from
[Google
News](https://www.google.com/search?q=belval+campus+esch-sur-alzette&client=firefox-b-d&sxsrf=ALeKk0080OxF6oOpC3lb6hNxafFccNgYjA:1590592264605&source=lnms&tbm=nws&sa=X&ved=2ahUKEwi57Kf3qdTpAhU7ThUIHSw_CG0Q_AUoAXoECCwQAw&biw=1920&bih=966).
Since the articles were published in multiple languages, they were
translated into English using
[DeepL](https://www.deepl.com/en/translator) to establish a uniform
corpus for analysis. The text files were then standardized as follows:

```         
Title: Title_Name
DATE: dd.mm.yyyy
.
. Text
.
Title: Title_Name
DATE: dd.mm.yyyy
.
. Text
.
```

This standardized format ensures that all articles could be processed
systematically.

### Wordcloud Analysis

The purpose of creating a wordcloud was to visualize the frequency of
terms in the corpus, providing an initial overview of recurring
concepts. Words displayed with larger fonts appear more frequently.

![](https://raw.githubusercontent.com/STBrinkmann/Text_Mining_Belval/master/Plots/wordcloud.svg)

The figure above presents a wordcloud generated from the combined news
corpus. Notably, terms such as **"university"**, **"student"**,
**"building",** and **"campus"** appear in larger fonts, indicating
their prominence and reflecting the central role of academia in the
Belval transformation. Concurrently, words such as **"furnace",**
**"steel",** and **"industrial"** acknowledge the site's heritage as a
former steel production hub, highlighting the juxtaposition of
historical infrastructure with contemporary educational facilities.
Additional references to **"restaurant",** **"project",** and **"2022"**
suggest a broader scope of local development efforts, including dining
amenities and preparations for major cultural events. Overall, the
wordcloud underscores the importance of the university environment while
also emphasizing the ongoing transition from an industrial past to a
diverse, forward-looking district.

### Term Frequency-Inverse Document Frequency (TF-IDF)

To identify terms that were particularly distinctive in each year, a
two-step approach was adopted. First, the **number of articles published
between 2015 and 2020** was examined (see the first figure below),
revealing variations in media attention over time. Notably, 2019 stands
out with the highest number of articles, suggesting heightened public
interest in Belval's development during that year.

![](https://raw.githubusercontent.com/STBrinkmann/Text_Mining_Belval/master/Plots/Articles_Count.svg)

Second, a **TF-IDF analysis** was conducted to determine which words
were most characteristic of each year (see the figure below). **TF-IDF**
stands for *term frequency--inverse document frequency* and is a widely
used text-mining method that measures how important a word is within a
specific subset of documents compared to its usage in the overall
corpus. Words that appear frequently in a small number of documents but
rarely across all documents receive a higher score, highlighting their
uniqueness.

![](https://raw.githubusercontent.com/STBrinkmann/Text_Mining_Belval/master/Plots/tf_idf.svg)

-   **2015**: Terms such as *"listener", "canteen",* and *"restopolis"*
    suggest early-stage coverage focusing on student facilities and
    everyday campus life, highlighting Belval's emerging university
    environment.

-   **2016**: Words like *"monaco", "burger", "fashion",* and
    *"shopping"* hint at expanding commercial and leisure offerings,
    reflecting growing public interest in new retail spaces.

-   **2017**: The prominence of *"evening", "glow", "performance",
    "furnace",* and *"blast"* indicates the significance of nighttime
    cultural events and the steel industry's legacy as a focal point for
    tourism and artistic performances.

-   **2018**: References to *"tram", "lab", "challenges",* and
    *"tudiants"* (students) point to discussions surrounding
    transportation infrastructure, academic growth, and the obstacles
    faced during Belval's continued expansion.

-   **2019**: Terms such as *"beach", "interview", "shooting",
    "soldier",* and *"residences"* illustrate a broader range of social
    and cultural themes, reflecting a surge in reporting as Belval
    prepared for major events such as the 2022 European Capital of
    Culture.

-   **2020**: Words including *"committee", "discus", "financing",
    "planning", and "parliamentary"* suggest a shift toward policy
    considerations, funding mechanisms, and formal decision-making
    processes governing Belval's future development.

Collectively, these TF-IDF outcomes demonstrate how media attention
moved from campus-focused reporting to a more diverse discussion of
Belval's evolving social, cultural, and infrastructural landscape. This
shift illustrates the campus's growing impact on the region's identity,
transitioning from an industrial brownfield to a multifaceted center of
education, innovation, and cultural engagement.

### Structural Topic Model (STM)

In order to identify coherent themes within the corpus and track their
evolution over time, a **Structural Topic Model (STM)** was applied.
Unlike simpler topic modeling methods (e.g., Latent Dirichlet
Allocation), the STM can incorporate metadata---in this case,
publication year---into the modeling process. This approach not only
reveals the latent topics but also how each topic's prevalence shifts
from one year to the next.

![](https://raw.githubusercontent.com/STBrinkmann/Text_Mining_Belval/master/Plots/stm.svg)

The figure above presents the six extracted topics along with their most
salient terms. Each topic was manually labeled based on these terms and
the broader context of the articles:

-   **Campus Belval**: Emphasizes the university setting, featuring
    words such as "university", "student", and "campus".

-   **Events**: Centers on cultural or entertainment happenings, with
    notable keywords like "beach", "concert", and "party".

-   **Gastronomie**: Reflects discussions of food and dining options,
    including "restaurant", "canteen", and "restopolis".

-   **Kultur/Zukunft, Esch 2022**: Highlights cultural projects and
    future-oriented planning, especially related to Esch-sur-Alzette's
    role as European Capital of Culture 2022 (terms such as "culture",
    "capital", "planning").

-   **Stadtentwicklung**: Focuses on urban development, covering
    infrastructure (e.g., "tram", "city", "park"), transportation, and
    city planning.

-   **Strukturwandel**: Captures narratives around
    industrial-to-knowledge transformation, featuring references to the
    "furnace", "blast", and "industrial" past alongside "university" and
    "city".

The second figure illustrates how these topics are distributed across
the years:

![](https://raw.githubusercontent.com/STBrinkmann/Text_Mining_Belval/master/Plots/timeseries.svg)

Several observations emerge:

1.  **Early Years (2015--2016):**  
    Articles often concentrated on **Campus Belval** and
    **Strukturwandel**, reflecting public interest in the new
    university's establishment and the ongoing shift away from the steel
    industry.

2.  **Increasing Variety (2017--2018):**  
    **Stadtentwicklung** gains more attention, indicating growing
    discussions about infrastructure projects such as new tram lines and
    urban amenities. **Gastronomie** also appears more frequently,
    suggesting that an expanding dining scene and student life began to
    shape public discourse.

3.  **Peak in 2019:**  
    A clear spike in **Events** and **Kultur/Zukunft, Esch 2022**
    coincides with preparations for the European Capital of Culture,
    driving media coverage that centered on cultural programs,
    festivals, and large-scale community initiatives.

4.  **Recent Trends (2020):**  
    Focus on **Campus Belval** remains, but **Stadtentwicklung** topics
    persist, reflecting sustained interest in policies and planning
    efforts that continue to shape Belval's evolution as both a research
    hub and a culturally vibrant district.

Overall, the STM reveals that while Belval's transformation was
originally framed as an industrial redevelopment story, subsequent
coverage expanded to highlight student life, cultural events,
gastronomic offerings, and broader urban planning. This transition
underscores Belval's emergence as a multi-faceted site where historical
legacy, academic innovation, and cultural aspirations converge.

### Conclusion

This text mining analysis demonstrates how news coverage of the Belval
Campus has evolved over time, transitioning from early discussions
centered on the feasibility of repurposing a steel production site to
more diverse topics reflecting cultural initiatives, gastronomic
developments, and the broader evolution of the campus as a hub for
research and innovation. Although structural transformation and
industrial heritage were initially focal points, subsequent coverage
highlights the growth of student life, the emergence of cultural events,
and ongoing urban planning efforts. As such, Belval has come to embody a
multifaceted model of regional renewal in Luxembourg - one that bridges
historical identity with contemporary academic, cultural, and
infrastructural advancement.

For readers interested in replicating or extending this research, the
complete code is available on
[GitHub](https://github.com/STBrinkmann/Text_Mining_Belval). The
repository provides the necessary scripts and instructions for data
collection, preprocessing, and analysis, offering a foundation for
further investigations into text mining and topic modeling.
