+++
date = "2014-08-11"
title = "Goodbye Octopress, Hello Hugo!"
description = "Why and how I abruptly decided to move my site from Octopress to Hugo and - just to be completely honest - what I miss!"
keywords = ["octopress", "hugo", "migrate", "static site", "generator"]
categories = ["octopress", "hugo"]
+++
When I first set up my little site and blog I had a look around at solutions that would meet a few simple requirements:

  * **Easy to set up.** I didn't want to dedicate a big chunk of time to get it running.
  * **Easy to keep going.** Even more than the first point, I didn't want a huge ongoing maintenance burden.
  * **Compact and fast.** There's a lot I can say about either of these points, but the basic idea was to have a snappy site with quick load times and low hosting costs.
  * A number of other things like offline writing and testing, simple uploading of new posts and site changes, and so on.

Static site generators easily ticked all of the boxes: offline writing, `rsync` for deployment, no backend to run on the server, pages load as fast as you can deliver a text file etc - sounds like a dream! [The choices are staggering, too!](http://staticsitegenerators.net/)

A bit more searching and I ended up with [Octopress](http://octopress.org/) and its many skins, plugins and helpful tutorials. A day's work or so and I was up and running. It was a smooth process, easy to deploy and get things looking as I wanted.

So far so good! So what made me switch to [Hugo](http://hugo.spf13.com) less than a year later?

---

### Unconvincing update story

Maybe it's just me, but it seems wrong for the standard site setup to involve forking the [main Octopress repo](https://github.com/imathis/octopress) and every now and again merging from upstream to stay up to date. You *should* be fine if you don't touch any of the standard files and edit only your pages - but this adds an artificial barrier to tweaking behaviour, since the only way to tie in your own `rake` tasks is to crack open the `Rakefile`.

It's also non-trivial to override theme behaviour. There's a recommended way to organise your overrides, but since the theme is cloned elsewhere, then copied to your `source` path, it's all too easy to edit those files instead. Then the built-in update process backs up your sources, gives you new ones, then tries to copy your content and overrides back in? I'm afraid they lost me partway through - maybe I went wrong somewhere, but it would be a bit more manageable if they'd given me a bit less rope, so to speak.

This is where it started failing the **easy to keep going** criteria, and I started having doubts.

### Ruby gems headache

I'm not a Ruby developer, but I've been quite impressed with how quick it is to pick up and start hacking with. I can see why a lot of really neat tools and libraries are Ruby-based, and I'd happily use them more often. But the `gem` system has let me down.

I've had to set up and work with a number of Ruby projects, and I started getting issues with gem versions being incorrect. Globally installed, *versioned* dependencies strike me as a bad idea, to be honest. What should be relatively small, single-purpose, *isolated* batch tools are now fighting to control the state of these resources in a shared space. I've had some success using `exec` but it's still an unnecessary overhead.

In a strange way this came back around to being **easy to set up**. After a different project updated some gems, Octopress was refusing to run. In fact, any Ruby-based site generator would have probably failed me here.

### Cluttered internals

Octopress isn't so much a tool as a collection of tools tied together through `rake` tasks. This works well up to a point. But then I noticed that whenever the `watch` task was stopped, it would sometimes fail to stop its child `compass` process - one example of many.

In my previous blog posts I experimented with plugins to modify behaviour at site generation time. There's nothing more frustrating than finding you can't change some specific pattern because of the way Octopress calls the process responsible for that resource.

From my own experience I've found it's usually far more manageable to have a single plugin-friendly tool doing the work, rather than coordinating a set of processes. **Easy to set up** and **easy to keep going** are both looking increasingly shaky.

---

In contrast to my points above [Hugo](http://hugo.spf13.com) is:

  * Much faster - the Go implementation has an easily visible impact.
  * No dependencies to sort out and keep in line.
  * More opinionated theme integration approach and project structure - it was very easy to set up [Hyde-X](https://github.com/zyro/hyde-x), my own theme variant.

That said, I have to admit I miss a few things.

  * The abundance of plugins and pipeline tools.
  * The variety of themes.
  * The community resources.
  * Pagination, which I find to be a strange omission.

Hugo is far from perfect, but so far I like how it's shaping up. I'll probably write a follow-up blog post at some point down the line after I've lived with it for a while, and we'll see if I'm still as enthusiastic about it.