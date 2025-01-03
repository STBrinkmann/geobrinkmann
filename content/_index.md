---
title: "GeoBrinkmann"
type: landing

date: "2025-01-01"
design:
  spacing: 3rem
sections:
- block: resume-biography-3
  content:
    # button:
    #   text: Download CV
    #   url: uploads/resume.pdf
    text: ""
    username: admin
- block: collection
  content:
    count: 3
    filters:
      featured_only: false
      folders:
      - post
    title: "<br><br>\U0001F4DA Recent Posts"
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
- block: markdown
  content:
    title: Contact
    text: |
      <form name="contact" netlify style="max-width: 600px; margin: 0 auto; padding: 20px; background-color: #f9f9f9; border: 1px solid #ddd; border-radius: 8px;">
        <p style="margin-bottom: 1em;">
          <label style="display: block; font-weight: bold; margin-bottom: 0.5em;">Name:</label>
          <input type="text" name="name" style="width: 100%; padding: 10px; border: 1px solid #ccc; border-radius: 4px;" />
        </p>
        <p style="margin-bottom: 1em;">
          <label style="display: block; font-weight: bold; margin-bottom: 0.5em;">Email:</label>
          <input type="email" name="email" style="width: 100%; padding: 10px; border: 1px solid #ccc; border-radius: 4px;" />
        </p>
        <p style="margin-bottom: 1em;">
          <label style="display: block; font-weight: bold; margin-bottom: 0.5em;">Message:</label>
          <textarea name="message" style="width: 100%; padding: 10px; border: 1px solid #ccc; border-radius: 4px; height: 150px;"></textarea>
        </p>
        <p style="text-align: center;">
          <button type="submit" style="background-color: #007BFF; color: white; padding: 10px 20px; border: none; border-radius: 4px; cursor: pointer;">Send</button>
        </p>
      </form>
---
