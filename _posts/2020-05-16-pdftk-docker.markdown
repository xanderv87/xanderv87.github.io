---
layout:     post
title:      "PDFtk magic with Docker"
description: "How to seamlessly use PDFtk with docker"
date:       2020-05-18 12:00:00
author:     "Xander Verheij"
header-img: assets/img/posts/pdftk-docker/document-icon.png

categories:
  - Random
---

PDFtk is a tool to do magic with pdf’s, some common use cases are the following:

- multiple pdf’s to one file

- one pdf to multiple files

- split odd and even pages

Till not to long ago pdftk could be easily installed with for example brew on a Mac.

Unfortunatly now a days this is nearly impossible. But do not fair, docker is here!

Using:

```sh
docker run -v ${PWD}:/work agileek/pdftk work/pdf1.pdf ....
```

we can use pdftk from the docker container, however this does mean that we miss some handy bash features as auto-completion. Making it a lot more tedious to get the correct commands.

This can be solved by adding the following to your .bashrc / .zshrc / .[insert shell]rc

After you added this (you might need to restart your shell) you can use pdftk as you where used to including all the auto-completion goodness.

```sh
function pdftk() {  docker run -w /work -v ${PWD}:/work agileek/pdftk "$@" }
```


