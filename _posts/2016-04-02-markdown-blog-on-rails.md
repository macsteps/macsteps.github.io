---
layout: post
title:  "A Markdown Blog in Rails"
date:   2016-04-02 6:30:21 -0700
categories: markdown redcarpet blog rails
---
# Creating a Markdown Blog on Rails

I enjoy writing in text editors more than word processors. Currently I'm using [Atom](Heroku Production Configurations) in Vim-mode. I'm not the Vim master, but I do prefer keeping my hands on the keyboard most of the time. It's just faster than moving my hands toward the trackpad (or mouse) constantly.

When I decided to start blogging about some of the work I'm doing, I learned about [Jekyll](https://github.com/jekyll/jekyll), which has [integration with Github](https://help.github.com/articles/using-jekyll-as-a-static-site-generator-with-github-pages/). I set it up quickly and it simply works. I can write simple documents in a text editor with some basic styling, preview it, and push it using Git. It's almost too easy.

I started working on a new eCommerce site, which could be used for my side business. I wanted to add a blog and would likely be the one writing the posts, at least initially. I wanted to:

+ add a blog
+ deepen my understanding of Rails
+ write the posts in [Markdown](https://en.wikipedia.org/wiki/Markdown)
+ use the same templates and styling as the reset of the site [^1]
+ make it fast (no reason to slow down blog posts with database calls) [^2]

[^1]: This would prove to be challenging with existing implementations.
[^2]: Still needed Rails and db interaction for the eCommerce side.

I found [Bloggy](https://github.com/zbruhnke/bloggy), which integrates Jekyll into Rails, but didn't work well with Ruby > 1.9 and hasn't been worked on since 2014. I also found [this](http://www.sitepoint.com/jekyll-rails/), which puts a Jekyll blog in a Rails app, but doesn't integrate the Rails styling, so it looks like another app.

There's probably just too little interest in this type of functionality, but I'm practicing Rails and real-world experience frequently has these kinds of strange requirements for various reasons (typically a requirement from the business side). In any case, I set out to meet these aforementioned requirements.

## Requirements

I'm using `Rails 4.2.6` and `bootstrap-sass`, so the only gem I needed was [Redcarpet](http://www.rubydoc.info/gems/redcarpet/3.3.4) to process the Markdown into HTML. Redcarpet is, in the creator/maintainers words:

> Redcarpet is a Ruby library for Markdown processing that smells like butterflies and popcorn.

Needed in Gemfile:

```
gem 'bootstrap-sass',           '~> 3.3', '>= 3.3.6'
gem 'redcarpet',                '~> 3.3', '>= 3.3.4'
```

Then run:

```
bundle install
```

That's it for setting up requirements. Let's code.

## Code

We'll need a controller with helper and views:

```
rails generate controller MarkdownBlog index show
```

I decided to call it markdown_blog, in case I wanted to add a traditional database-based blog later on. I use two actions: index and show. Deletes, edits will be done through Git.

Let's add a couple routes:

```
get 'blog/',              to: 'markdown_blog#index'
get 'blog/*path',         to: 'markdown_blog#show'
```

Note that `*path` will simple grab everything after `/blog` and put it in params[:path].

For the processing of Markdown to HTML, I used [this article from RichOnRails.com](https://richonrails.com/articles/rendering-markdown-with-redcarpet):

markdown_blogger_helper.rb:

```ruby
module MarkdownBlogHelper
  def md_to_html(text)
    options = {
      filter_html:     false,
      hard_wrap:       true,
      link_attributes: { rel: 'nofollow', target: "_blank" },
      space_after_headers: true,
      fenced_code_blocks: true,
    }

    extensions = {
      autolink:           true,
      superscript:        true,
      disable_indented_code_blocks: true,
      no_intra_emphasis:  true
    }

    renderer = Redcarpet::Render::HTML.new(options)
    markdown = Redcarpet::Markdown.new(renderer, extensions)

    markdown.render(text).html_safe
  end
end
```

The `filter_html` is important, because I want to be able to add html in markdown documents. You would set this to `true` if users supplied the markdown and then either disallow HTML or filter certain tags (e.g. `<script>`). `rel: 'nofollow'` is important, because I use links to external websites and I don't want [Google crawlers leaving my site](https://support.google.com/webmasters/answer/96569?hl=en).

Now we're set to add code to markdown_blog_controller.rb:

```ruby
class MarkdownBlogController < ApplicationController
  include MarkdownBlogHelper
  attr_reader :content

  def index
    posts = Dir.glob("#{Rails.root}/public/*/*/*.md")
    @content = Array.new
    posts.each do |post|
      discard, slug = post.split(/lnat_app\//)    # lnat_app is the name/directory of my app
      title, slug_final = get_slug_final(slug)
      pre_excerpt = File.foreach(post).first(1)
      # This next line does a couple things: 1. converts this array to a string
      # It's an array, because I was playing with multiple lines. In this case I could just use `read`
      # 2. change CSS class so index uses different CSS than show.
      pre_excerpt = pre_excerpt.join("").to_s.gsub(/col-md-offset-3 blog-main-img/, 'blog-excerpt-img')
      next if pre_excerpt.match(/^draft/)          # an easy way to ensure drafts aren't released early.
      excerpt = md_to_html(pre_excerpt)            # call to markdown_blog_helper.rb
      @content << [title, slug_final, excerpt]
    end
  end

  def show
    file_path = "#{Rails.root}/" + params[:path] + ".md"
    md_content = File.read(file_path)
    @content = md_to_html(md_content)
  end

  private

    def get_slug_final(slug)
      slug_fields = []
      slug_fields = slug.match(/(\w+?\/)(\w+?)\/(\d{4})\/(\S+?\.md)/)
      category = slug_fields[2]
      year = slug_fields[3]
      file_name = slug_fields[4]
      title = file_name.split(/-/).map(&:capitalize).join(' ').gsub(/\.md$/, '')
      slug_final = "public/#{category}/#{year}/#{file_name}"
      return title, slug_final
    end

end
```

There's likely some refactoring I could do here; however, this code works great. In `get_slug_final` I assemble the [slug](https://wincent.com/wiki/Rails_slugs) to ensure it's properly formed, although it's probably formed correctly to begin with.

I chose to add category, because it's an easy way for me to the the category and use it later for organization. I use the year, because I wouldn't want to see more than a couple hundred files in a single directory. For `pre_excerpt` I could have grabbed as many lines as I wanted, but just grab the first line, which I is where I put the image.

With that in place, let's look at the views. I use Bootstrap, because the [grid layout](http://getbootstrap.com/css/#grid-options) is easy to understand.

index.html.erb:

```
<h1>Blog</h1>

<% @content.each do |title, slug, excerpt| %>
  <div class="row blog-excerpt">
    <div class="col-md-2 col-md-offset-4">
      <h2 class="blog-excerpt-title"><%= link_to title, controller: "markdown_blog", action: "show", path: slug %></h2>
    </div>
    <div class="col-md-6">
      <%= excerpt %>
      <p><small><%= link_to "Read more >", controller: "markdown_blog", action: "show", path: slug %></small></p>
    </div>
  </div>
<% end %>
```

show.html.erb:

```
<div class="article">
  <%= @content %>
</div>
```

Some custom CSS, which was by far the most time-consuming aspect of this project. CSS looks straightforward, but [the devil is in the details](http://grammarist.com/words/devil-is-in-the-details-vs-god-is-in-the-detail/). Firebug and/or Chrome Developer Tools are very helpful to see what CSS is being used for each element of the page.

So I'll create a markdown file in `public\example-category\2016\markdown-file-name.md`. The first line will be an image:

```
<div class="row col-md-offset-3 blog-main-img"><img src="/assets/blog/2016/cool-image.jpg" /></div>
```

If you're testing this now, simply follow it with some markdown:

```
## markdown heading h2

some plan test with **some bold**

and a list:

1. caffeine
2. programming
3. nap
4. repeat
```

Start the rails server:

```
rails server
```

Check it out at `http://localhost:3000/blog`

That's it. Let me know if I left something out.

---
Footnotes:
