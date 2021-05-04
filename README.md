# Elaborative Encoding

[![GitHub Pages](https://github.com/albertquiroga/albertquiroga.github.io/actions/workflows/gh-pages.yml/badge.svg)](https://github.com/albertquiroga/albertquiroga.github.io/actions/workflows/gh-pages.yml)

Albert Quiroga's personal page, built using GitHub pages.

## How this works

The site is built using [Hugo](https://gohugo.io/) and the [Cactus theme](https://github.com/monkeyWzr/hugo-theme-cactus) by [Zeran Wu](https://github.com/monkeyWzr). 

Steps to reproduce:

1. Install Hugo
2. Create a new Hugo site: `hugo new site elaborative-encoding`
3. Initialize as a git repo: `cd elaborative-encoding && git init`
4. Install cactus theme: `git submodule add https://github.com/monkeyWzr/hugo-theme-cactus.git themes/cactus`
5. Head to the repo of the theme and find the provided sample config.toml file. Copy&paste onto the local one.
6. Add posts to `content/posts` in Markdown format
7. Add about page as a Markdown file in `content/about/index.md`
8. Add favicon as `static/images/favicon.ico`, add logo as `static/images/logo.png`

Once all is done, try with `hugo server` or `hugo server --bind=<your_ip_address>` if serving from a host in your local network.

If you like the results, build the static site with `hugo --minify`. This will produce a `public` directory with the results, which can then be uploaded to S3 for instance for static website hosting.

## Deploying to GitHub

Create a `.github/workflows/gh-pages.yml` file by copying [the one provided by Hugo](https://gohugo.io/hosting-and-deployment/hosting-on-github/). After that, simply add a GitHub repo as a remote, and every time the repo is pushed the action will run, building the site on the `gh-pages` branch. 

Head to the pages section of the configuration of the GitHub repo, and configure it to publish the root of the `gh-pages` branch. The site should be available shortly after that. If it's not, change the settings to a different branch/directory, save, then change back to the root of `gh-pages`. This should force a re-build and make things work.

## Understanding how this works

GitHub supports hosting static websites through what they call [GitHub pages](https://pages.github.com/). This is a bit buggy in my experience, and was designed to work with [Jekyll](https://jekyllrb.com/) rather than Hugo, but is a good option for free static website hosting with an easy-to-remember hostname.

If enabled, GitHub pages will serve an index.html file present in the main branch (default config). This means that with Hugo, you will need one repository for your source files (the root of the `elaborative-encoding` directory created above) and another for the site itself (the contents of the `public` directory created after running `hugo --minify`). This is a [PITA](https://www.netlingo.com/word/pita.php) to maintain, manage and control when you just want to write something from time to time and push it.

Luckily, GitHub pages can also be configured to publish the contents of a different branch as a site, which allows you to have a main branch with your site's source files, and a publishing branch with your site's static files. This branch is typically named `gh-pages`, and with this you can have a single-repo site which makes things way easier to work with.

In order to do this, one would have to make a complex git structure where the `gh-pages` branch is empty, then every time the site is built the contents of `public` are copied onto the branch and everything is properly committed. We would also have to automate this for every push we do on the repo.

Luckily for us, our friends at Hugo have completely automated this with a [GitHub actions workflow](https://docs.github.com/en/actions) available [here](https://gohugo.io/hosting-and-deployment/hosting-on-github/).

