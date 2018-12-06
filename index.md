## Hi there!
Welcome to my personal page.
This site currently serves as a collection of useful code snippets during my coding adventures (mostly F# and Python).

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>