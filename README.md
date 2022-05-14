# How to Create Personal Blog Using Github Pages

After doing step by step from [github's documents](https://docs.github.com/cn/pages) and [jekyll's documents](https://jekyllrb.com/docs/step-by-step/01-setup/), I still don't know how to build my personal blog, so I think neither of them is a good way to create your blog.

If you trust me, please follow this tutorial.

# Creating Github Pages

This step is really easy and you can find many articles. All you need is to:

1. register an account for github
2. create a repository named `your-name.github.io`
3. choose a theme for your repository
4. commit your initialization
5. clone it to your local machine

# Using an Existed Template

After above steps, you only got a normal website. Then you can choose a more beautiful theme from [jekyllthemes.org](http://jekyllthemes.org/), [jekyllthemes.io](https://jekyllthemes.io/), [jekyll-themes.com](https://jekyll-themes.com/) and [themes.jekyllrc.or](http://themes.jekyllrc.org/). The best way to use your chosen theme is to download, extract and copy all contexts to your repository.

Almost all themes have some built-in blogs in the `_post` directory, if you don't know how to write in markdown, they are good references.

Please noting the most important things listed here:

1. the name of your blog file must be formatted with `YYYY-MM-DD-FILE_NAME.md`
2. set the `layout`, `title`, `categories` and `tags` in the very begin of your blog file so it can be classified correctly

# Deploy jekyll environment on other computers

If you just want to write your blog on another computer with the same gems (ruby modules), after installing ruby-full, you can simply type `gem install bundler` and install bundler, then change to your blog repository and execute `bundle install`, it will automatically install all the gems in Gemfile.lock.
