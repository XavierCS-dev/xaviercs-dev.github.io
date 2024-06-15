---
title: Welcome to XavierCS!
layout: post
author: Xavier
tags: XavierCS Xavier Jekyll Ruby HTML CSS Github-Pages SSG TeXt
---

Hi, and welcome to xaviercs.com. Today we will discuss how this website is put together,
what it will be used for, and potential future content.

## How this website is constructed

At the time of writing, the amount of CSS and HTML written by myself is very minimal, though this will definitely change as I update the site and add necessary functionality.
The heavy lifting has been done by a combination of the [Jekyll](https://jekyllrb.com/) Static Site Generator (SSG) and the beautiful [TeXt Theme](https://github.com/kitian616/jekyll-TeXt-theme).

Revision 1: Now using [Yat!](https://github.com/jeffreytse/jekyll-theme-yat)

Revision 2: Currently using the default [minima](https://github.com/jekyll/minima) theme, but I plan to add search functionality.

### Static Site Generators (SSGs)

Static Site Generators or SSGs generate full static websites with HTML and CSS using HTML/CSS templates, and content such as images or text. SSGs often form part of ["JAMstack"](https://www.cloudflare.com/en-gb/learning/performance/what-is-jamstack/).

Static websites are simple websites that
are mostly made up of HTML, CSS and Javascript, and always load the same way, as opposed to a dynamic website, which may change due to factors such as time or location.
This gives static sites an advantage in speed and predictability, for both the user and the host. The lack of a database also significantly increases the security of static sites.

In fact, static websites are so fast, many services such as Github via [GitHub Pages](https://pages.github.com/), offer free hosting for static sites. Coincidentally (or not) Github Pages runs on Jekyll, and hosts this website!

### Jekyll

Jekyll is a Static Site Generator written in Ruby with the goal of creating fast and beautiful blog-aware static websites without the need for databases.
Jekyll is Free and Open Source Software (FOSS) used by companies such as [Spotify, Mozilla, Twitch and Github](https://jekyllrb.com/showcase/) for creating functional static sites.

A huge advantage of Jekyll which made it my first choice for this website is the fact that it is blog-aware. This means that Jekyll makes it easy to utilise custom layouts for pages and articles
as well as take advantage of permalinks, tags and categories to improve the look of the website and boost SEO. You may have seen on the home page that all the posts are added and come with
a preview automagically! The way Jekyll does this is by utilising variables to copy and paste content from markdown files.

For this site, I am using the custom TeXt theme, which comes with its own layouts, categories and conventions, and have most of the CSS and HTML written for me.

The beauty of this setup is creating articles and site pages is incredibly easy. In order to do so, you must first create a markdown file with the date and name separated by dashes, either
in the main folder, or the "_posts" folder. For example:

```
2022-07-22-Welcome-to-XavierCyber.md
```

Once you have done that, you must then add variables such as the title, category and type to the top of the file, so Jekyll and your layout know how to display the page. In the case of this page,
the variables look something like this:

```
---
title: Welcome to XavierCS!
type: article
aside:
    toc: false
tags: XavierCS Xavier Jekyll Ruby HTML CSS Github-Pages SSG TeXt
article_header:
  type: cover
  image:
    src: assets/about-header.png
---
```

The rest is just [markdown formatting](https://www.markdownguide.org/basic-syntax/).


### Github Pages

Github Pages is a free static website hosting service that uses Jekyll, and is where this site is being hosted. Github pages uses Jekyll to host static sites, so you can drag 'n' drop your HTML and CSS files straight into a repository called:

```
[username].github.io
```

Even using your own domain is free, and of course, the integration with git, which is why Github Pages is so awesome! There are caveats though such as a size limit of 1GB
and a soft bandwidth limit of 100GB. This site is unlikely to exceed the bandwidth limit, but may exceed the size limit in the far future, which is when I will have to move the website
over to another host.


## The future of this site

Primarily, this site will serve as a place with articles about projects I have undertaken with tutorial-like reads wherever possible, so anyone can follow along or even attempt
the same projects for themselves. Mostly likely this site will contain more niche and specific information rather than broad tutorials or articles such as "the top 10 popular operating systems", but will still remain interesting nonetheless.

I will post articles and update pages whenever and wherever I can, though this won't be consistent. I can't be sure of the hosting situation in the future, but will try to have the best up-time possible.

Thank you for reading my welcome post!
