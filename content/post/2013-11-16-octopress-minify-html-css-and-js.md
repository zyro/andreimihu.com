+++
date = "2013-11-16"
title = "Octopress: Minify HTML, CSS and JS"
description = "How to improve Octopress site performance, with specific emphasis on minifying HTML, CSS and JS resources"
keywords = ["octopress", "ruby", "minify", "compress", "html", "css", "js", "javascript"]
categories = ["octopress", "ruby"]
+++
After putting my Octopress site and blog together, I obviously couldn't help tinkering even if I did promise myself not to sink too much time into it. I originally wanted to keep it mostly standard and low maintenance so I could focus on other projects, but cracking it open was just too tempting!

I am definitely very happy with how it handles most things right out of the box - it's incredibly convenient and works smoothly - but a quick inspection with [PageSpeed Insights](http://developers.google.com/speed/pagespeed/insights/) pointed out a few obvious areas where I felt I could make improvements.

Bring it on!

---

### The basics

Since the very purpose of using Octopress is to generate a static site, there's already a significant [performance advantage over dynamic engines](http://jason.pureconcepts.net/2013/01/benchmark-octopress-wordpress/). That's a big advantage, but it's only a head start and there's room for better, so I ran through the usual content-independent web optimisations.

**Optimise all images, including theme-provided ones.** A one-off trick (once per image file) that can be a very useful space saver. I used [ImageOptim](http://imageoptim.com/) but there are a good few tools that do this well. This process saves as much space as possible *without* impacting image quality or introducing compression artefacts. Same quality, smaller size - there's no downside, but many sites neglect to do this!

  * **Note:** This process usually strips image metadata, may or may not mess with colour profiles and so on - again, without impacting visual image quality. If you're serving raw graphics files, use with care!

**Enable caching on the web server.** Very basic but it's always a good idea to make sure it's done correctly - case in point, my hosting provider hadn't set up a default configuration so responses had no `Cache-Control` header at all! So, I dropped the `.htaccess` file below in my root public folder and Apache did the rest.

``` apacheconf
#### CACHING ####
# 7 DAYS DEFAULT
ExpiresActive On
ExpiresDefault A604800

# 1 MONTH
<FilesMatch "\.(ico|gif|jpe?g|png|flv|pdf|swf|mov|mp3|wmv|ppt)$">
ExpiresDefault A2419200
Header append Cache-Control "public"
</FilesMatch>

# 14 DAYS
<FilesMatch "\.(xml|txt|html|htm|js|css)$">
ExpiresDefault A1209600
Header append Cache-Control "private, must-revalidate"
</FilesMatch>
```

**Enable gzip compression.** Another relatively basic one, there are a few ways to do this. It's possible to compress all resources at generation time, as long as the web server is configured to send the correct header to the client. Additionally, it's possible to upload two version of every resource - compressed and uncompressed - and return the correct one to the client based on request headers. I chose a third option and uploaded only uncompressed resources, then let the server worry about it and decide based on request headers whether to compress or not.

  * **Note:** I may revise this decision later since having both compressed and uncompressed resources readily available eliminates the runtime overhead. That said, `runtime compression time + transfer time` still beats `uncompressed transfer time` for all but the smallest of resources.

**Use external Content Distribution Networks for common resources.** Basically, [don't host your own jQuery](http://encosia.com/3-reasons-why-you-should-let-google-host-jquery-for-you/) or similar. The gist of it is that it allows the client to download common resources from a CDN, which will almost certainly have servers that are faster or geographically closer to the client, and saves bandwidth at the same time. The resource may even already be cached on the client after visiting other sites that use it! All such script declarations in HTML files or template includes should point to externally hosted resources.

So far so good! These are all generally useful whether using a static site or not, but didn't keep me busy for long.

---

### The good part

Because Octopress is built on top of Jekyll, there are a fair few options to choose from to minify or compress resouces:

  * [Htmlcompressor](https://github.com/paolochiodi/htmlcompressor)
  * [HtmlMinifier](https://github.com/stereobooster/html_minifier)
  * [HtmlPress](https://github.com/stereobooster/html_press)
  * [CSSminify](https://github.com/matthiassiegel/cssminify)
  * [YUICSSMIN](https://github.com/matthiassiegel/yuicssmin)
  * [css_press](https://github.com/stereobooster/css_press)
  * [CssMinify](https://github.com/donaldducky/jekyll-cssminify)
  * The list goes on...

Some are designed for only one resource type, some multiple. Some build on top of others. Many have issues that have been open against them for a good few months, giving me little confidence. Two plugins deserve special mentions:

[Jekyll Asset Pipeline](https://github.com/matthodan/jekyll-asset-pipeline) is one of the most popular and works very well, but I chose not to use it because:

  * It takes some effort to integrate given the slightly different source structure used by Octopress. I did get it working, but it felt a bit cumbersome and I wasn't entirely happy with it.
  * Many theme files require some modification to take advantage of the compression and bundling provided. I wanted a solution that would still allow me to swap themes as I see fit without then going into each theme file to make changes, if at all possible.
  * Does not handle HTML minification.
  * Does not integrate well with Compass, which can also do some level of minification during processing, and is set up to do so in the default config provided by Octopress.

[jekyll-press](https://github.com/stereobooster/jekyll-press) is the closest to what I wanted, and I may actually go back to using it, but its integration was a bit rough due to some Octopress particularities. Most notably, Octopress invokes Compass as a separate process, both for initial asset generation and when running in preview mode. While this works, I noticed odd results when Jekyll would minify resources on their way to the `public` folder and Compass would believe they have been improperly altered after reaching the folder, then generate a fresh copy.

I toyed with creating my own plugin based on [jekyll-press](https://github.com/stereobooster/jekyll-press) using my preferred CSS/JS/HTML minification libraries but ended up abandoning this idea before getting very far with it, again for the reasons above.

The solution I settled on makes use of the libraries below. These could easily be substituted for pure Ruby solutions, but the underlying Java libraries of the wrappers below are very mature, stable and reliable.

  * [Ruby-YUI Compressor](https://github.com/sstephenson/ruby-yui-compressor) for CSS and JS
  * [html_compressor](https://github.com/completelynovel/html_compressor) for HTML

First, add the following gems to the base `Gemfile` Octopress provides:

``` ruby
gem 'yui-compressor', '~> 0.12.0'
gem 'html_compressor', '~> 0.0.3'
```

Include them at the top of the `Rakefile`:

``` ruby
require "yui/compressor"
require "html_compressor"
```

Then add the following as independent tasks in the `Rakefile`:

``` ruby
desc "Minify CSS"
task :minify_css do
  puts "## Minifying CSS"
  compressor = YUI::CssCompressor.new
  Dir.glob("#{public_dir}/**/*.css").each do |name|
    puts "Minifying #{name}"
    input = File.read(name)
    output = File.open("#{name}", "w")
    output << compressor.compress(input)
    output.close
  end
end

desc "Minify JS"
task :minify_js do
  puts "## Minifying JS"
  compressor = YUI::JavaScriptCompressor.new
  Dir.glob("#{public_dir}/**/*.js").each do |name|
    puts "Minifying #{name}"
    input = File.read(name)
    output = File.open("#{name}", "w")
    output << compressor.compress(input)
    output.close
  end
end

desc "Minify HTML"
task :minify_html do
  puts "## Minifying HTML"
  compressor = HtmlCompressor::HtmlCompressor.new
  Dir.glob("#{public_dir}/**/*.html").each do |name|
    puts "Minifying #{name}"
    input = File.read(name)
    output = File.open("#{name}", "w")
    output << compressor.compress(input)
    output.close
  end
end
```

These tasks can be called individually to minify either CSS, or JS, or HTML.

The last part is to decide where best to invoke them. I wanted to work with un-minified resources while writing blog posts so I could examine the source without hurting my eyes, so the natural choice seemed to be to minify at publishing time.

I added the task invocations to the existing `deploy` task, which then looked like this:

``` ruby
desc "Default deploy task"
task :deploy do
  # Check if preview posts exist, which should not be published
  if File.exists?(".preview-mode")
    puts "## Found posts in preview mode, regenerating files ..."
    File.delete(".preview-mode")
    Rake::Task[:generate].execute
  end

  # Apply minification tasks
  Rake::Task[:minify_css].execute
  Rake::Task[:minify_js].execute
  Rake::Task[:minify_html].execute

  Rake::Task[:copydot].invoke(source_dir, public_dir)
  Rake::Task["#{deploy_default}"].execute
end
```

And there it is! Every time the `deploy` task is invoked, all resources will be minified in place in the `public` folder before `rsync` takes over and pushes them to the remote server.

---

### Conclusion

Octopress is a great tool, no doubt about it. It's quick to get running, has great community support, and a bunch of great themes to get things rolling.

That said, this experiment has made me briefly consider packing up my toys and moving over to [Middleman](http://middlemanapp.com/). The customisation I attempted would likely have gone smoother and integrated more cleanly, but Octopress remains my blogging tool of choice... for now!