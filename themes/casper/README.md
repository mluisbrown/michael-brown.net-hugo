# CASPER theme for hugo


Casper is a single-column theme for [Hugo](https://gohugo.io/).
Ported from [Casper theme for Ghost ](https://github.com/TryGhost/Casper)

blog demo : https://vjeantet.fr
blog source : https://github.com/vjeantet/vjeantet.fr

![Hugo Casper Theme screenshot](https://raw.githubusercontent.com/vjeantet/hugo-theme-casper/master/images/screen.png)

## Features

* Google Analytics (optional)
* Disqus ( can disable comments by content)
* Share buttons on Facebook, Twitter, Google (can disable share by content)
* Big cover image (optional)
* Custom cover by content (optional)
* Tagging
* Pagination
* Menu
* Syntax Highlighting

# Theme usage and asumptions
* All blog posts are in the ```post``` folder (```content/post```)
* The homepage displays a paginated list of contents from the post Section (other contents may be added to main menu, see bellow)

# Installation

## Installing this theme

    mkdir themes
    cd themes
    git clone https://github.com/vjeantet/hugo-theme-casper casper

## Build your website with this theme

    hugo server -t casper

# Configuration

**config.toml**

``` toml
BaseUrl= "https://example.com/"
LanguageCode= "fr-FR"
Title= "My blog is awesome"
paginate = 5
DisqusShortname = "YOUR_SHORT_NAME_HERE"
Copyright = "All rights reserved - 2015"
canonifyurls = true

[params]
  description = "Welcome to my website"
  metadescription = "Used as 'description' meta tag for both home and index pages. If not set, 'description' will be used instead"
  cover = "images/cover.jpg"
  author = "Valère JEANTET"
  authorlocation = "Paris, France"
  authorwebsite = "https://vjeantet.fr"
  authorbio= "my bio"
  logo = "images/logo.png"
  googleAnalyticsUserID = "UA-79101-12"
  # Optional RSS-Link, if not provided it defaults to the standard index.xml
  RSSLink = "https://feeds.feedburner.com/..."
  githubName = "vjeantet"
  twitterName = "vjeantet"
  # facebookName = ""
  # codepenName = ""
  # linkedinName = ""
  # stackoverflowId = ""
  # keybaseName = ""
  # flickrName = ""
  # instagramName = ""
  # email = ""
  # pinterestName = ""
  # googlePlusName = ""
  # set true if you are not proud of using Hugo (true will hide the footer note "Proudly published with HUGO.....")
  hideHUGOSupport = false
  
  # Setting a value will load highlight.js and enable syntax highlighting using the style selected.
  # See https://github.com/isagalaev/highlight.js/tree/master/src/styles for available styles
  # A preview of above styles can be viewed at https://highlightjs.org/static/demo/
  hjsStyle = "default"

  [params.social]
    twitter = "your_twitter"

```

Example : [config.toml](https://github.com/vjeantet/vjeantet.fr/blob/master/config.toml)

## Overide author information per page

In addition to providing data for a single author as shown in the example above, author can be overided per page. If a author key  in page's frontmatter exists it will be used instead of the default one. 
Overrides per page can be done by adding author* = "value entry in the front matter. 
For those pages where you want to omit the author block completely, a .Params.noauthor entry is also
available.


Example override author per page file:
``` toml
+++
date = "2014-07-11T10:54:24+02:00"
title = ""
author = "NickName"
authoravatar = "https://cdn4.iconfinder.com/data/icons/gray-user-management/512/rounded-512.png"
authorbio = "IT software and Security Engineer, Open source enthusiast."
authorlocation = "Paris, France"
authorwebsite = "https://vjeantet.fr"
...
+++

Contents here

```

## Menu configuration

On top right of the screen, a "Subscribe" button is displayed with a link to the RSS feed.

When you define a menu in the main config file, Then a menu button is displayed instead of the subscribe button
When the use clicks the menu button, a sidebar appears and shows the subscribe button and all items defined in the main config file

> :information_source: If your added a metadata like ```menu="main"``` in a content file metadata, it will also be displayed in the main menu

Example of a menu definition in main config file.


``` toml
[[menu.main]]
  name = "My Blog"
  weight = -120
  identifier = "blog"
  url = "/"

[[menu.main]]
  name = "About me"
  weight = -110
  identifier = "about"
  url = "/about"

```

## Metadata on each content file, example

``` toml
+++
author = ""
date = "2014-07-11T10:54:24+02:00"
draft = false
title = "dotScale 2014 as a sketch"
slug = "dotscale-2014-as-a-sketch"
tags = ["event","dotScale","sketchnote"]
image = "images/2014/Jul/titledotscale.png"
comments = true     # set false to hide Disqus comments
share = true        # set false to share buttons
menu = ""           # set "main" to add this content to the main menu
+++

Contents here
```

# Contact me

:beetle: open an issue in github

:bird: [https://twitter.com/vjeantet](https://twitter.com/vjeantet)
