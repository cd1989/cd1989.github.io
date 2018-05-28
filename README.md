## What's It

Personal blog engined by `Jekyll`.

## Quick Start

Install Jekyll and Bundler gems through RubyGems

> gem install jekyll bundler

Clone code
> git@github.com:cd1989/cd1989.github.io.git

Build the site on the preview server

> bundle exec jekyll serve --port 8080

Now browse to http://localhost:8080

Please refer to [Jekyll Docs](https://jekyllrb.com/docs/home/) for more details.

## Theme

By default, gem-based theme `Minima` is used. If want to customize it, please override corresponding files in the theme.

To get the theme files, use `bundle show minima`

For example, if you want to change the footer layout:

- create `_includes` folder in the root directory
- copy `_incluses/footer.html` in minima to the above folder
- edit it