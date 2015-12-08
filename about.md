---
layout: page
title: About TADevelops
cover: /assets/images/covers/keyboard.jpg
---

TADevelops is a blog developed by the engineering team behind [TechnologyAdvice](http://www.technologyadvice.com). With an amazing group of developers and designers working with cutting edge technology the goal of TADevelops is to provide a platform for sharing with the community our experiences, knowledge and open source projects.

## The Team

<ul class="about-profiles">
{% for author in site.authors %}
  <li>
    <div>
    <span class="image" style="background-image: url('/assets/images/profiles/{{ author[1].pic }}');"></span>
    <strong>{{ author[1].name }}</strong>
    <p>{{ author[1].bio }}</p>
    <hr>
    <p>
    <strong>Posts By {{ author[1].name }}:</strong><br>
    <ul class="post-expander">
    {% assign counter = 0 %}
    {% for post in site.posts %}
      {% if post.author == author[0] %}
        {% assign counter=counter | plus:1 %}
        <li><a href="{{ post.url }}">{{ post.title }}</a> on {{ post.categories | array_to_sentence_string }}</li>
      {% endif %}
    {% endfor %}
    {% if counter == 0 %}None Yet! <a href="mailto:{{ author[1].email }}">Email {{ author[1].name }}</a> and request some content :){% endif %}
    </ul>
    </p>
    
    </div>
  </li>
{% endfor %}
</ul>

<script>
(function () {
  $('body').append('<style>.pe-hide { display: none; }');
  $('.post-expander').each(function () {
    var current = $(this);
    var maxItems = 3;
    var items = current.children('li');
    if (items.length > maxItems) {
      // Add expander
      $(this).append('<li class="pe-expander"><a href="#">Show More...</a></li>');
      // Hide items gt max
      items.each(function (i) {
        if (i >= maxItems) {
          $(this).addClass('pe-hide');
        }
      });
      // Expander click
      current.children('.pe-expander').on('click', function () {
        current.children('.pe-hide').removeClass('pe-hide');
        current.children('.pe-expander').remove();
      })
    }
  });
})();
</script>
