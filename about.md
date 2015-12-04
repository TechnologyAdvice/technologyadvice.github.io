---
layout: page
title: About TADevelops
cover: /assets/images/covers/keyboard.jpg
---

<style>
  ul.about-profiles {
    display: block;
    width: 100%;
    margin: 0;
    padding: 0;
  }
  
  ul.about-profiles li {
    display: block;
    margin: 25px 0;
    padding: 0;
  }
  
  ul.about-profiles li div {
    display: block;
    position: relative;
    margin: 0;
    padding: 15px;
    border: 1px solid #e8e8e8;
    border-radius: 3px;
    background-color: #fbfbfb;
    transition: .3s ease background-color, .3s ease box-shadow;
  }
  
  ul.about-profiles li div:hover {
    box-shadow: 0 0 15px 0 rgba(0,0,0,.1);
    background-color: #fff;
  }
  
  ul.about-profiles li div p {
    font-size: .75em;
  }
  
  ul.about-profiles li div span.image {
    display: block;
    float: left;
    border: 1px solid #e8e8e8;
    padding: 4px;
    margin: 0 15px 0 0;
    width: 50px;
    height: 50px;
    border-radius: 50%;
    background-size: cover;
    background-repeat: no-repeat;
    background-position: center center;
  }
</style>


TADevelops is a blog developed by the engineering team behind [TechnologyAdvice](http://www.technologyadvice.com). With an amazing group of developers and designers working with cutting edge technology the goal of TADevelops is to provide a platform for sharing with the community our experiences, knowledge and open source projects.

## The Team

<ul class="about-profiles">
{% for author in site.authors %}
  <li>
    <div>
    <span class="image" style="background-image: url('/assets/images/profiles/{{ author[1].pic }}');"></span>
    <strong>{{ author[1].name }}</strong>
    <p>{{ author[1].bio }}</p>
    </div>
  </li>
{% endfor %}
</ul>