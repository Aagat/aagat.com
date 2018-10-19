---
layout: post
title: Static Website On Google App Engine
description: A guide to setup static website with custom 404 on Google App Engine.
keywords: static website, custom 404, golang, jekyll, app engine, google app engine, guide
date: 2017-09-13T18:54:20+05:45
draft: false
---

*This article will describe how I hosted my static website and blog (this website) on Google App Engine (Standard). Some familiarity with *nix tools and programming is assumed.*

### Pretext

Hosting stuff with traditional shared hosts is not a pleasant experience. They're usually slow, oversold and more or less stuck in 2006 in terms of developer experience. It's 2017 and I wasn't going to subject myself to the torture that is uploading your files through FTP client. So I had two very straightforward criteria to host my website:
* __Simple__: The process to deploy new version of website should be painless. It shouldn't take more than one command.
* __Free__: I like free stuff and I can't lie.

Looking around, I found GAE's (Google App Engine) [official tutorial][1] that shows you how to host a static website. The entire process boils down to: a) install their SDK (Software Development Kit) and b) populate a file called `app.yaml` to tell appengine what to do. Deploying the application is also as simple as running `gcloud app deploy`. First requirement satisfied.

GAE also seems to include [generous free tier][2] that includes:
* 28 instance hours per day
* 5 GB Cloud Storage
* Shared memcache
* 1000 search operations per day, 10 MB search indexing
* 100 emails per day
* 1GB/day data transfer out @ maximum rate of 56 MB/minute

_The maximum rate of 56MB/minute is fine for mostly text website with low traffic but if you are hosting any sort of image, it is a good idea to configure your website behind CloudFlare._

That is good enough to run my site for free forever. Second requirement satisfied.

With the requirements sorted I followed the [official tutorial][1] and had my website up in no time. At this point, I have a single html file inside the `www` folder. This folder is going to be my web root.
```
.
├── app.yaml
└── www
    └── index.html

```

### The Roadblock

While this setup worked pretty well for a single page static website, it quickly became unusable when I tried hosting a blog. Jekyll generates pretty URL by default which means each post is inside it's own several levels deep directory (`/blog/yyyy/mm/dd/post-title/index.html`) which was difficult to configure using the configuration format for GAE. With blog included, the project has many directories than before:

```
.
├── app.yaml
├── jekyll
└── www
    ├── blog
    │   ├── 2017
    │   ├── 404.html
    │   ├── atom.xml
    │   ├── index.html
    │   ├── LICENSE.md
    │   ├── public
    │   └── styles.css
    └── index.html
```
_The directory `jekyll` has Jekyll installation with output destination configured to `www/blog`_

The configuration (`app.yaml`) quickly becomes a bowl of sphaghetti regular expressions.

```yaml
runtime: python27
api_version: 1
threadsafe: true

- url: /
  static_files: www/index.html
  upload: www/index.html
  
- url: blog/(.*\.js)
  mime_type: text/javascript
  static_files: www/blog/\1
  upload: www/blog/(.*\.js)

- url: blog/(.*\.(jpg|png|ico))
  static_files: www/blog/\1
  upload: www/blog/(.*\.img)

- url: blog/(.*\.css)
  mime_type: text/css
  static_files: www/blog/\1
  upload: www/blog/(.*\.css)

- url: blog/(.*\.(eot|svg|svgz|otf|ttf|woff|woff2))
  static_files: www/blog/\1
  upload: www/blog/(.*\.fonts)

- url: blog/
  static_files: www/blog/index.html
  upload: www/blog/index.html

- url: blog/(.+)/
  static_files: www/blog/\1/index.html
  upload: www/blog/(.+)/index.html
  expiration: "15m"

- url: blog/(.+)
  static_files: www/blog/\1/index.html
  upload: www/blog/(.+)/index.html
  expiration: "15m"

- url: blog/(.*)
  static_files: www/blog/\1
  upload: www/blog/(.*)
```
> Some people, when confronted with a problem, think "I know, I'll use regular expressions." Now they have two problems. - Jamie Zawinski


I think now I know what Jamie Zawinski was referring to.

### The Solution
Luckily Google App Engine, as name implies, is a full blown app platform. If I can't configure GAE to work exactly the way I want it to, logic dictates that I write my own.

GAE supports several programming languages including `golang`, `python` and `nodejs` among others. `golang` has the best primitives for writing a web server so it is a natural choice. Here is a minimal web server I wrote to serve static directory `www`.
```golang
package server

import (
    "net/http"
)

func init() {
    http.Handle("/", http.FileServer(http.Dir("www")))
}
```

From GAE's hello world example, a minimal configuration for golang application that leaves out `jekyll` directory is:
```yaml
runtime: go
api_version: go1

skip_files: 
- ^jekyll

handlers:
- url: /.*
  script: _go_app
```

With the new system in place, the final directory structure of the project is:
```
.
├── app.yaml
├── jekyll
├── server.go
└── www
    ├── blog
    │   ├── 2017
    │   ├── 404.html
    │   ├── atom.xml
    │   ├── index.html
    │   ├── LICENSE.md
    │   ├── public
    │   └── styles.css
    └── index.html

```

Google App Engine SDK provides a handy script to test our application locally before we publish it online. It can be started by switching to project directory and simply running:
```
$ dev_appserver.py app.yaml
```
_I assume you have already installed golang SDK component. If not run `gcloud components install app-engine-go` or consult the documentation._

If everything went well, your website should now be live at `http://localhost:8080`. What an exciting time!

This minimal golang webserver works perfectly fine but I wanted little more control over 404 pages. With some modification our webserver does exactly what I wanted: serve custom 404 for the blog.

```golang
package server

import (
    "net/http"
    "os"
    "path"
    "strings"
)

func init() {
    http.HandleFunc("/", handler)
}

func handler(w http.ResponseWriter, r *http.Request) {

    fs := http.Dir("www")
    fileServer := http.FileServer(fs)
    cleanPath := path.Clean(r.URL.Path)

    _, err := fs.Open(cleanPath)
    if os.IsNotExist(err) &&  strings.HasPrefix(cleanPath, "/blog") {
        w.Header().Set("Content-Type", "text/html; charset=utf-8")
        w.WriteHeader(http.StatusNotFound)
        http.ServeFile(w, r, "www/blog/404.html")
        return
    }

    fileServer.ServeHTTP(w,r)
}

```

Wonderful! Now all there is left to do is deploy the blog. Simply run:
```shell
$ gcloud app deploy
$ gcloud app browse
```

[1]: https://cloud.google.com/appengine/docs/standard/python/getting-started/hosting-a-static-website
[2]: https://cloud.google.com/free/
