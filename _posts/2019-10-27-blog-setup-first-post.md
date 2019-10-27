---
title: "Blog Setup and First Post"
date: 2019-10-27
categories:
  - blog
tags:
  - Jekyll
  - update
---
I've had a lot of "I gotta blog this" moments recently and never got to satisfy
those urges because I had no good place to put anything.
The twitter format is just too small for many of these thoughts.

### Selecting the Platform

I have a dormant Wordpress site which may have been a good idea to resurrect.
But the database-driven sites don't really interest me and a lot of my life
revolves around GitHub these days, so it sort of makes sense to think of
GitHub as a good place to do this.
And having the content in GitHub seems a bit more future-proof in case I want
to move this site somewhere else.

The Jekyll solution for generating static GitHub Pages seems appealing and I settled on the
[Minimal Mistakes GitHub Pages site starter ](https://github.com/mmistakes/mm-github-pages-starter).
I got the site up and running very quickly after forking it.
I tried various other approaches outlined in articles I found, but none of
the results were as good as using this starter site to get up and running
quickly.

### Updating the Site

Since it is on GitHub, the site can be updated from the GitHub
website interfaces.
I might do short postings this way,
but I will probably tend to do most of the authoring on my local machine and
push the updates up to GitHub.
I can do the writing in VS Code, using Markdown preview for a rough preview.
And by running a local server via:

```shell
bundle exec jekyll serve
```

I can preview how my changes look on the site before pushing.
The server is really nice because it detects when files change and
regenerates the site automatically.

I did have to do one interesting configuration that is a bit outside of the
normal configuration, which I'll write a post about later.

Otherwise, at this point I've just done the usual "make it mine" sorts
of changes.

I picked the _mint_ theme because it is mostly plain white with only
touches of the nice green color.
