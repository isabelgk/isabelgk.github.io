---
layout: post
title: "TIL #1: Formatting JSON in vim"
---

Today I was saving a configuration of a Chrome extension I'm enjoying ("humble" new tab page). I wanted to make it more readable and I've been adopting vim, so I learned a handy command.

A nice and simple post [here](https://blog.realnitro.be/2010/12/20/format-json-in-vim-using-pythons-jsontool-module/) taught me about the Python `json.tool`.

To use, run as you would any command in vim!

```
:%!python -m json.tool
```
