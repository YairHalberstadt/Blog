I am a software developer specialising in C#, among other technologies. I hope to use this blog to discuss some of my thoughts, ideas, problems, and solutions, on programming in general, and C# in particular.
 
  {% for post in site.posts %}
  <article>
    <h2>
      <a href="{{site.baseurl}}{{ post.url }}">
        {{ post.title }}
      </a>
    </h2>
    <time datetime="{{ post.date | date: "%Y-%m-%d" }}">{{ post.date | date_to_long_string }}</time>
    {{ post.content }}
  </article>
{% endfor %}
