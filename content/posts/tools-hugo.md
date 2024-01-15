---
title: "Hugo & GitHub - Setup Static Website on GitHub"
date: 2020-06-12T22:32:01+04:00
draft: false
toc: false
images:
tags:
  - hugo
  - github
---
![Hub](/hugo-logo-wide.svg)

## Hugo and Github Account
A small write-up on how [this site](https://rizwan-kh.github.io) was setup. There are plenty of materials available all over the internet, but this is more of a self-help document for myself to refer back in future to understand how this was all setup.

- Hugo can be downloaded from their official [release page](https://github.com/gohugoio/hugo/releases)
- A GitHub account needs to be created from [this link](https://github.com/join) and a public repo is required to be setup [here](https://github.com/new)

After installing **Hugo**, to generate a new site, run the below command:
```sh
hugo new site rizwan-blogs
```
This will automatically create a directory named *rizwan-blogs* which will contain the necessary sub-directories and files required for our new hugo-based static website.

The `config.toml` file is the main configuration file where we can add/change the theme or add additional settings like title, author, etc. Sample **config.toml** looks like below
```sh
baseURL = "https://rizwan-kh.github.io"
title = "Rizwan Khan👨🏻‍💻"
languageCode = "en-us"
theme = "hello-friend-ng"

```
Replace the above 4 values as you want to in the config.toml and you are good to go.

To add content/pages, we run the below command, this will create a markdown page with the post marked as `draft: true` and we can keep it as true until we want to publish the post.
```sh
huge new posts/my-post.md
```
Fire up your favorite editor and it's all [markdown](https://www.markdownguide.org/cheat-sheet/) from there.
Then we can start the hugo server in build drafts enabled mode (or as I think of it as development mode) with the below command and it will let you browse the site on address [localhost:1313](http://localhost:1313/) with auto-refresh mode on.
```sh
hugo server -D
```

When the pages/posts are ready to be published, simply run the server in publish mode and hugo will take care of generating the required files in `./public` directory, 
*Note: Make sure to clean up the public directory, as the directory will contain drafts pages as well from the previous run*
```sh
hugo
```

#### Deploy your website
The final production content in stored in the public directory and we just need to copy the directory in the production server either on **AWS S3**, or on **GitHub Pages**

----
Now for the publishing the website, Go to [GitHub](https://github.com) and create a [new public repository](https://github.com/new) named ***username.github.io***, where ***username*** is your username (or organization name) on GitHub.

Use any **git** client to clone the repo, copy over the entire _public_ directory

```sh
git clone https://github.com/username/username.github.io
cd username.github.io
```
copy over the entire content of the `public` directory `username.github.io` and use git to commit and push the content over to *GitHub*
```sh
git add --all
git commit -m "initial commit"
git push origin main
```

open your browser and type up the URL [username.github.io](https://username.github.io) and voila your static website content is up and ready for free.