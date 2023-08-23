Working on my new blog
=======================

I'm working on my new blog where I will write about my insane quest to take 12 certifications in 12 months, and about my other projects and things i find interiesting, annoying or both.

For fun I'm doing this with Jekyll and GitHub Pages, and I'm using the [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/) theme.

<h1>Latest Posts</h1>

<ul>
  {% for post in site.posts %}
    <li>
      <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
      {{ post.excerpt }}
    </li>
  {% endfor %}
</ul>