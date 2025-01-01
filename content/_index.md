---
title: "GeoBrinkmann"
type: landing

date: "2025-01-01"
design:
  spacing: 3rem
sections:
- block: resume-biography-3
  content:
    button:
      text: Download CV
      url: uploads/resume.pdf
    text: ""
    username: admin
# - block: markdown
#   content:
#     subtitle: ""
#     text: "I am a Data Scientist at [GfK Geomarketing](https://www.gfk.com/home) where I develop new methodology for calculating international market data.<br>
#     Before, I was working as part of the [Digital Health Geography research group](https://www.geography.nat.fau.eu/research/cultural-geography/wg-digital-health/) as a Data Analyst with focus on machine learning, and the intersection of natural environment and human behavior by developing novel algorithms.<br><br>
#     On this website I would like to share some of my private projects where I solve geo-spatial problems with R.<br><br>"
#     title: "About me"
#   design:
#     columns: "1"
- block: collection
  content:
    count: 3
    filters:
      featured_only: false
      folders:
      - post
    title: "<br><br>\U0001F4DA Recent Posts"
    text: "[SEE ALL POSTS](post/)"
  design:
    spacing:
      padding:
      - 0
      - 0
      - 0
      - 0
    view: date-title-summary
  id: blogposts
- block: collection
  content:
    filters:
      featured_only: true
      folders:
      - publication
    title: "\U0001F4DA Featured Publications"
    text: "[SEE ALL PUBLICATIONS](publication/)"
  design:
    columns: 2
    view: article-grid
  id: papers
---
