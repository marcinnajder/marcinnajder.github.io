---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
---

<ul>
  {% for post in site.posts %}
    <li>
      <time datetime="{{ post.date | date: "%Y-%m-%d" }}">{{ post.date | date: "%Y-%m-%d" }}</time> <a href="{{ post.url }}">{{ post.title }}</a>
      
      <!-- {% for tag in post.tags %} <span>{{tag}}</span> {% endfor %} -->
    </li>

{% endfor %}

</ul>

<!-- {% for tag in site.tags %}

  <h3>{{ tag[0] }}</h3>
  <ul>
    {% for post in tag[1] %}
      <li><a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
{% endfor %} -->
