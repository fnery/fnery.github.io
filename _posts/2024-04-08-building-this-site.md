---
layout: post
title:  "Building this site"
date: 2024-04-08 09:59:54 +0100
tags: [meta]
---

## Introduction

This post summarizes the process to build this site. A few of the guiding principles were:

- Simplicity.
- Easy on the eyes (clean, uncluttered design and dark mode 🌙).
- [File over app](https://stephango.com/file-over-app) philosophy.
- Markdown rendering.
- Tags to categorize posts.
- Easy to host with [GitHub Pages](https://pages.github.com/).

After some research, I decided to go with [Jekyll](https://jekyllrb.com/) and the [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) theme which support all the above out of the box.

## Getting started with Jekyll+Chirpy

Install Jekyll by following the [docs](https://jekyllrb.com/docs/installation/). Then, create a new repository (repo) using [Chirpy starter](https://github.com/cotes2020/chirpy-starter) as a [template](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-repository-from-a-template).

> To host with GitHub Pages, ensure the repo is public _and_ named as `<github_username>.github.io`.
{: .prompt-warning }

Then, clone the new repo, install dependencies and serve it:

```bash
# Clone the repo
git clone <repo_url>

# Install dependencies
bundle

# Serve
bundle exec jekyll serve
```

The site should now be running on [http://127.0.0.1:4000/](http://127.0.0.1:4000/).

![chirpy_starter](/assets/img/chirpy_starter.png){: width="750" }
*A fresh Chirpy site running locally*

## Make it your own

Now the fun begins. Things you can do[^1]:

- Configure the site by editing [_config.yml](https://github.com/cotes2020/chirpy-starter/blob/main/_config.yml) (things like the timezone, contact info, avatar, etc...).
- Customize the Favicon (see [here](https://chirpy.cotes.page/posts/customize-the-favicon/)).
- Start [writing](https://chirpy.cotes.page/posts/write-a-new-post/) posts.
- Customize the site by adding new layouts or edit existing ones (for example, I [added](https://github.com/fnery/fnery.github.io/commit/7ccf416d46e1ac2bde79875873feeb7a496e3d39) the "last updated at" date to the archives [page](https://fnery.io/posts/)). 

## Host with GitHub pages

GitHub allows [hosting](https://pages.github.com/) one site per GitHub account for free, directly from a GitHub repo. Again Chirpy does most of the work for us by including a [GitHub Actions](https://docs.github.com/en/actions) [workflow](https://github.com/cotes2020/chirpy-starter/blob/main/.github/workflows/pages-deploy.yml) in the [starter repo](https://github.com/cotes2020/chirpy-starter) which automatically builds the site when changes get pushed to the repo (an example of [continuous delivery](https://en.wikipedia.org/wiki/Continuous_delivery)). Ensure the site is [configured](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow) to publish with GitHub Actions.

## Configure a custom domain

> At this point, we have a fully functioning website, so this section is optional.
{: .prompt-info }

Say you own a domain name and want to use it for this site. GitHub has great [docs](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site) on how to achieve this, but I'll summarize what I did.

1. [Verified](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/verifying-your-custom-domain-for-github-pages) the custom domain for GitHub Pages (GitHub recommends this).

    ![github-verified-domain-ok](/assets/img/github-verified-domain-ok.png){: width="400" }
    *Confirming domain is verified (on GitHub account Settings > Pages)*

2. On repo settings > Pages, under "Custom domain", set the custom domain (in my case, `fnery.io`).
3. Created `CNAME`, `A` and `AAAA` DNS records on my domain provider (see figure below).

![custom-site-dns-records](/assets/img/custom-site-dns-records.png){: width="650" }
*DNS records set for a custom domain (screenshot from cPanel > Zone Editor of my particular DNS provider, yours may look slightly different). The bottom row refers to the verification of the custom domain (the code can be [obtained](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/verifying-your-custom-domain-for-github-pages) from the "Add a DNS TXT record" instructions on GitHub account Settings > Pages > Add a domain)*

Once this is done, the DNS configuration changes can take up to 24h to propagate, after which the site should be available in the custom domain. Ensure you also enforce HTTPS on the repo Settings > Pages.

![github-pages-page-ok](/assets/img/github-pages-page-ok.png){: width="650" }
*Confirming site deployed using GitHub Actions with a custom domain and HTTPS enforced (on repo Settings > Pages)*

## Conclusion

We reviewed the steps to build a simple static website, host it with GitHub pages for high availability for free[^2] and configured it to a custom domain. All of this with minimal effort and almost no knowledge of web development required. This is a good example of the magic of open source[^3].

---

[^1]: For more, check the [Jekyll](https://jekyllrb.com/docs/), [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) and [Chirpy starter](https://github.com/cotes2020/chirpy-starter) docs.
[^2]: The only thing we had to pay for was the custom domain, but that's optional.
[^3]: Speaking of which, go star the [Jekyll](https://github.com/jekyll/jekyll) and [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) [repos](https://github.com/cotes2020/chirpy-starter) and, if you can, support the authors.
