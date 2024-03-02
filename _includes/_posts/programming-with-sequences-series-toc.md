{% for post in site.posts reversed | where: 'series', programming-with-sequences %}

{% if post.series == page.series %}

<li> <a href="{{ post.url }}">{{ post.title }}</a> </li>
{% endif %}
{% endfor %}

<!-- https://digitaldrummerj.me/blogging-on-github-part-13-creating-an-article-series/ -->
