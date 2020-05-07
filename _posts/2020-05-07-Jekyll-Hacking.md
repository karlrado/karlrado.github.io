---
title: "Jekyll Hacking"
date: 2020-05-07
categories:
  - blog
tags:
  - Jekyll
  - update
---
Well, that didn't work.  Haven't blogged much at all.  Maybe that will change.

### A Misconception
When I pulled up this blog's repo on my machine, I found some uncommitted changes.
I was trying to add a "sub-section" of sorts by adding a "topic" item to the top
horizontal menu.

I did this because I was planning to scrape some postings from a very old blog
and put them here.
There would be a couple of dozen posts and I wanted to group them together into
their own logical space.
I then realized that doing this would conflict with the intent of the top
menu bar.

The top menu bar is meant to select particular grouping _mechanisms_ in the
blog and not to select a particular _group_.
That is, you can choose to view the content by _tag_ or _category_, but not by some instance of a category, such as "hacking".

This topic I'm thinking of could be a Category and I would set it as such.
A tag would be another option.
Perhaps the category would be "Hobbies" and the individual hobby project would be a tag such as "Spirit-100 glider".
If a person then wanted to see the entire series of posts for that topic,
they would then just select the category.

I'm just trying to understand the intent of these groupings.
I am new here.

This would then let me create that blog series as a set of individual posts
instead of one long one.
And this would preserve the old blog's original presentation.

Apparently, I did some investigation on how to modify the top menu.

Here is what I made `navigation.yml` look like:
```yml
main:
  - title: "Posts"
    url: /posts/
  - title: "Categories"
    url: /categories/
  - title: "Tags"
    url: /tags/
  - title: "About"
    url: /about/
  - title: "Spirit-100"
    url: /hobbies/spirit-100/

toc:
  - title: Hobbies
    subfolderitems:
      - page: Spirit 100 Glider
        url: /hobbies/spirit-100.md
```

And then `_pages/hobbies/spirit-100/spirit-100.md` looks like:
```yml
---
permalink: /hobbies/spirit-100/
title: "Spirit 100 Electric Glider"
layout: default
---

# Coming Soon
```

This all sort of works, but I'm not sure _everything_ works, such as the
generator finding all the tags, etc.
I _am_ working outside the box here.

So, no, I'm not going to do it this way and just stick with making a
new category/tag for importing those old postings.

A quick experiment suggests that this will work OK.
I'm just recording how to modify that top menu bar in case I need to
do it for the right reasons someday.
I'll toss the changes in my repo now.
