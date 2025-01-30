---
layout: page
title: About
permalink: /about/
---

<div class="profile-container" markdown="1">


![Profile Picture](/assets/images/olymp_profile_scaled.jpg)

# Marko Ivanovic

### Senior Software Engineer

With over 10 years of experience in software development, learned to be pragmatic and have a product mindset. Passionate about AI.

Skilled in C++, C, Python, JavaScript.
<3 Linux.


#### Connect With Me

<div class="social-links">
{%- if site.github_username -%}
<a href="https://github.com/{{ site.github_username| cgi_escape | escape }}" class="social-link">
  <svg class="svg-icon"><use xlink:href="{{ '/assets/minima-social-icons.svg#github' | relative_url }}"></use></svg>
  GitHub
</a>
{%- endif -%}

{%- if site.linkedin_username -%}
<a href="https://www.linkedin.com/in/{{ site.linkedin_username| cgi_escape | escape }}" class="social-link">
  <svg class="svg-icon"><use xlink:href="{{ '/assets/minima-social-icons.svg#linkedin' | relative_url }}"></use></svg>
  LinkedIn
</a>
{%- endif -%}

{%- if site.stackoverflow_username -%}
<a href="https://stackoverflow.com/users/{{ site.stackoverflow_username| escape }}" class="social-link">
  <svg class="svg-icon"><use xlink:href="{{ '/assets/minima-social-icons.svg#stackoverflow' | relative_url }}"></use></svg>
  Stack Overflow
</a>
{%- endif -%}
</div>

</div>

<style>
.profile-intro {
  display: flex;
  gap: 30px;
  align-items: flex-start;
  margin-bottom: 40px;
}

.profile-image-container {
  flex-shrink: 0;
}

.profile-image {
  width: 150px;
  height: 150px;
  border-radius: 50%;
  object-fit: cover;
}

.profile-text {
  flex-grow: 1;
}

.profile-text h1 {
  margin-top: 0;
}

@media (max-width: 600px) {
  .profile-intro {
    flex-direction: column;
    align-items: center;
    text-align: center;
  }

  .profile-image {
    margin-bottom: 20px;
  }
}

.social-links {
  display: flex;
  gap: 20px;
  margin: 20px 0;
  flex-wrap: wrap;
}
</style>


## Key Projects and contributions

### Yandex browser (Chromium based)
- Worked a long side very talented individuals to make our users happy, and company to earn more money, kept Yandex.Browser in sync with Chromium base, and maintained
- [Contributed]( https://chromium-review.googlesource.com/q/status:merged+author:ivanovich%2540yandex-team.ru) to open source Chromium project
- Technologies used: C++ (some Java, Kotlin, JavaScript, and Python)

### NikaIPTV streaming cluster
- Side project. Transformed abandoned, almost unmaintainable code, into profitable SaaS business with less bugs and more features, using all skills at my disposal and working under pressure. Learned to **value users** and **business** side of projects.
- Technologies used: C, Linux, Node.js
- We had about 20-30 B2B partners with about 40k active end users streaming video each day, at its peak.

### Spectral Python
- One of the major [contributors](https://github.com/spectralpython/spectral/pulls?q=is%3Apr+author%3Akormang) to Spectral Python (machine learning and hyperspectral image analysis library).
- Invented and contributed fastest algorithm (at least publicly available) for computing upper convex hull, applied it to computing continuum of spectral signatures.


## Education

**Lobachevsky State University Nizhny Novgorod, Russian Federation**
  - MSc in Fundamental Informatics
  - _September 2018 - June 2020_
  - Thesis: Methods of processing and analysis of hyperspectral images
  - Received Red diploma (the equivalent of an Honours Degree in Russia)

**Faculty of Electrical Engineering, Banja Luka, Bosnia and Herzegovina**
  - BSc in Electrical Engineering
  - _September 2011 - September 2016_
  - Thesis:  Unsupervised Feature Learning for Aerial Image Classification
  - Grade: 9.17 / 10



<style>
.profile-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.profile-image {
  width: 200px;
  height: 200px;
  border-radius: 50%;
  margin: 0 auto 20px;
  display: block;
}

.social-link {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 20px;
  border-radius: 5px;
  background-color: #f5f5f5;
  text-decoration: none;
  color: inherit;
  transition: background-color 0.3s;
}

.social-link:hover {
  background-color: #e0e0e0;
}

.svg-icon {
  width: 20px;
  height: 20px;
}
</style>