{% for post in site.posts reversed %}

{% if post.series == include.series-name %}

<li> <a href="{{ post.url }}">{{ post.title }}</a> </li>
{% endif %}

{% endfor %}
