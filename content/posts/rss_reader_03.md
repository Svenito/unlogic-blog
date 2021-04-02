---
title: "RSS Feed Reader with Phoenix part 4 - LiveView"
date: 2021-03-24T10:51:50Z
draft: true
description: "Writing an RSS feed reader for codementor.io project"
tags:
  - elixir
  - phoenix
  - otp
  - liveview
keywords:
  - elixir
  - phoenix
  - concurrency
  - otp
  - codementor
---

In [Part 2](/posts/rss_reader_02/) we implemented getting and parsing the RSS
feeds. In this installment we will work on displaying the feed information using
LiveView. We really don't need LiveView as we aren't updating content manually,
but it's not a problem if we do use it regardless. It's a good learning exercise.

We set up Bulma in [Part 1](/posts/rss_reader_03/) so we are pretty much ready
to begin working on the layout and data passing. We've designed the interface
as well, so let's begin with the backend. Open the file `/lib/rss_web/live/page_live.ex`,
it's here we will add out back end logic to handle the search/submit event and
pass the data to the frontend.
