---
layout: post
title:  "Jekyll GitHub Actions"
date: 2021-02-27 14:08:00 GMT-7
categories: blog jekyll github workflow
description: Using GitHub Actions to build this site with custom Jekyll plugins
---

I had wanted to add a new collection to this site with a categorized view of various
code snippets to collect for posterity. I was inspired by Alexandru Nedelcu's
[snippet section][1] where he has a similar collection of unstructured code dumps.

Previously, this site leveraged the simple setup provided by [GitHub][2], which makes
setting up a GitHub Pages site with Jekyll, hosted directly from a GitHub repository,
quick and painless. The default setup gives you quite a bit in terms of low barrier
blogging. For example, GitHub builds the site automatically on push to your repo and
enables some Jekyll [plugins][3] by default. However, if your needs extends beyond
this set of plugins, GitHub Pages cannot build sites using unsupported plugins. If you
want to use unsupported plugins, you need to generate the site locally and push the
static files to GitHub.

When I program, I don't typically store machine generated source or compiled binaries in
my version control system. I see the statically generated site through a similar lens and
felt it wasn't something I wanted to manage myself. So I did some digging and found a
[page][4] on the Jekyll documentation site for setting up [jekyll-actions][5], which
should be able to do the work for me instead. The generated site will still be stored in
my repo in order for GitHub to be able to serve the site, however, it'll be sequestered
to the `gh-pages` branch and completely managed by the workflow. A compromise I can happily
live with. In order to get the new workflow to function, I needed to make some configuration
changes to my repo. First, as the documentation site states, I needed to create a Personal
Access Token. After that was configured as a secret for my repo, I was able to push the
first [commit][6] to test the action and make sure the pipeline worked as expected. The
action can be configured with the following snippet:

{% highlight yaml %}{% raw %}
name: Deploy jekyll site to GH Pages

on:
  push:
    branches:
      - master

jobs:
  jekyll:
    runs-on: ubuntu-16.04
    steps:
      - uses: actions/checkout@v2

      # Use GitHub Actions' cache to shorten build times
      # and decrease server load
      - uses: actions/cache@v2
        with:
          path: vendor/bundle
          key: ${{ runner.os }}-gems=${{ hashFiles('**/Gemfile') }}
          restore-keys: |
            ${{ runner.os }}-gems-

      - uses: helaili/jekyll-action@v2
        with:
          token: ${{ secrets.JEKYLL_PAT }}
          target_branch: 'gh-pages'
{% endraw %}{% endhighlight %}

where `JEKYLL_PAT` represents the secret that was created for my repo. After this commit was
built, I saw that the action created a new `gh-pages` branch in my repository with the statically
generated site contents. I then changed the source branch in the Setting tab under GitHub Pages
to reflect the new branch that GitHub should use to publish my site contents. After this was
complete, I was able to easily add the new [snippets][7] section to my blog.

Here's to more forthcoming contributions!

[1]: https://alexn.org/snippets
[2]: https://pages.github.com
[3]: https://docs.github.com/en/github/working-with-github-pages/about-github-pages-and-jekyll#plugins
[4]: https://jekyllrb.com/docs/continuous-integration/github-actions
[5]: https://github.com/marketplace/actions/jekyll-actions
[6]: https://github.com/jopecko/jopecko.github.io/commit/6c03a4c074b91db4cf2b74d00b806a93d8c25206
[7]: https://github.com/jopecko/jopecko.github.io/commit/8a0cd8d5f214503dafce352450c6cad80ca938cf