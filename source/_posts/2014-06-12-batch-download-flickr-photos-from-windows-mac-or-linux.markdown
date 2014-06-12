---
layout: post
title: "Batch download flickr photos from Windows, Mac or Linux!"
date: 2014-06-12 00:34:00 -0400
comments: true
author: Floyd Pink
categories: [flickr downloadr, Technical, Windows, Mac, Linux, Cross Platform] 
---
The [previous entry](/blogs/blog/2014/01/20/contemplating-a-mac-port-of-flickr-downloadr/) on this blog discussed the idea of probably creating a native, Cocoa, Mac OS X port for the then, Windows-only, WPF version of the app.

Any action started much later from the date of the post; to be precise, more than two months later with the creation of a new repository with [this commit](https://github.com/flickr-downloadr/flickr-downloadr-gtk/commit/3f94a6bc13c87f905e3f5be5e9872accb6930f05). The research that happened in-between helped decide on a third alternative - that of completely porting over the .NET C#/WPF into a Mono, C#/GTK# app that could work on all of Windows, Mac and Linux.

And exactly after two months and one day [the v1.0](https://github.com/flickr-downloadr/flickr-downloadr-gtk/commit/de399a3526344ea96d1847eff2836e15674a7553) of the new GTKSharp enabled flickr downloadr was published on [this website](/).

It really was an exhilarating journey of many frustrations, joy, learnings and moments of bliss - and there are a few things that I would like to follow up in separate entries here. Like:

1. the parallel, multi-platform continuous integration/deployment that happens on [AppVeyor (Windows)](https://ci.appveyor.com/project/floydpink/flickr-downloadr-gtk), [Travis CI (Mac)](https://travis-ci.org/flickr-downloadr/flickr-downloadr-gtk) and [Wercker (Linux)](https://app.wercker.com/#applications/5363d07d2cbfc1b354003e84).
2. the building of a cross-platform installer using the free-for-open-source Enterprise edition of [Install Builder](http://installbuilder.bitrock.com/) provided generously by the awesome guys at BitRock.
3. the slight face-lift applied on the website along with the cut-over to a new, [Yeoman](http://yeoman.io/) scaffolded web-app - complete with its automated `grunt` build for minification, image compression etc. (do a quick *View Source* on this page to see the results)

...and many other such learnings and experiences.

So, please spread the word of this new, free, open-source utility for batch downloading photos from flickr that is available on Windows, Mac OS X and Linux (Ubuntu & Mint have been tested).