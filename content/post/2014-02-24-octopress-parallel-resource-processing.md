+++
date = "2014-02-24"
title = "Octopress: Parallel resource processing"
description = "How to improve Octopress site generation performance, with specific emphasis on parallelising the minification of HTML, CSS and JS resources"
keywords = ["octopress", "ruby", "parallel", "multithread", "speed", "performance", "minify", "compress", "html", "css", "js", "javascript"]
categories = ["octopress", "ruby"]
+++
A while back I wrote about [minifying resources in Octopress](/blog/2013/11/16/octopress-minify-html-css-and-js/) to improve page weight and load speed. A few extra dependencies and a little bit of additional Ruby code allowed the standard deployment task to minify HTML, CSS and JavaScript files before uploading them to the target server.

That's a great place to start making an already speedy static site even faster and lighter, but there was no special attention paid to what would happen as the number of pages grew with time. Octopress does allow working on blog posts in isolation, to avoid waiting to long when working in local preview mode, but at some point all of that content must be published.

Enter parallelisation!

---

### The problem

Left as they are, the built-in `rake` tasks will dutifully process all of the resources in the `source` folder one at a time. This is likely to keep the average blogger in business for a decent amount of time, but what happens when this starts taking too long?

Obviously there are limits to how much is reasonable to do on a single thread before the site generation or deployment time reaches 'inconvenient' and keeps right on climbing!

[Nathan LeClaire](http://nathanleclaire.com/) got in touch about this very issue and suggested the nifty [Parallel](https://github.com/grosser/parallel) library, so I took it for a spin.

### A solution appears!

Parallel makes it very easy to gracefully spin out any given bit of Ruby into its own thread or process. For the purposes of this post, I'll look at using it in the context of the resource minification I detailed previously.

The initial setup is straightforward as usual. Add the new dependency to the `Gemfile`, using the most up-to-date version available:

``` ruby
gem 'parallel', '~> 0.9.2'
```

And ensure it's included at the top of the `Rakefile`:

``` ruby
require "parallel"
```

After that a very small code change transforms the additional tasks in the `Rakefile` into:

``` ruby
desc "Minify CSS"
task :minify_css do
  compressor = YUI::CssCompressor.new
  Parallel.map(Dir.glob("#{public_dir}/**/*.css"), :in_threads=>3) do |name|
    puts "Minifying #{name}"
    input = File.read(name)
    output = File.open("#{name}", "w")
    output << compressor.compress(input)
    output.close
  end
end

desc "Minify JS"
task :minify_js do
  compressor = YUI::JavaScriptCompressor.new
  Parallel.map(Dir.glob("#{public_dir}/**/*.js"), :in_threads=>3) do |name|
    puts "Minifying #{name}"
    input = File.read(name)
    output = File.open("#{name}", "w")
    output << compressor.compress(input)
    output.close
  end
end

desc "Minify HTML"
task :minify_html do
  compressor = HtmlCompressor::HtmlCompressor.new
  Parallel.map(Dir.glob("#{public_dir}/**/*.html"), :in_threads=>3) do |name|
    puts "Minifying #{name}"
    input = File.read(name)
    output = File.open("#{name}", "w")
    output << compressor.compress(input)
    output.close
  end
end
```

Translation: for every `.css` or `.js` or `.html` file in the target `public` directory, or any of its sub-directories, execute the following block of code using 3 threads. Easy!

Feel free to experiment with the number of threads, or even use processes, but be aware results may very based on the hardware and system particulars this will run on.

Keeping the tasks separate still allows the level of flexibility enjoyed previously, but is it really necessary to run them one at a time? Let's see what happens if we change the `deploy` task to:

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
  Parallel.map([:minify_css, :minify_js, :minify_html],
               :in_threads => 3) do |parallel_task|
    Rake::Task[parallel_task].execute
  end

  Rake::Task[:copydot].invoke(source_dir, public_dir)
  Rake::Task["#{deploy_default}"].execute
end
```

Success! Same reasoning as before, but rather than distributing a set of files to be processed across 3 threads, this is telling doing the same for 3 separate `rake` tasks.

---

### Conclusion

As mentioned before, do experiment with the number of threads, try processes, see what works best for your hardware. I personally ran into trouble if I tried to spin up too many threads, and switching to processes was even less forgiving.

Why not take it a step further and see what other standard Octopress `rake` tasks are good candidates for parallelisation?

And if anyone cares to use this (or something similar) on a larger blog, I'd love to see benchmarks!