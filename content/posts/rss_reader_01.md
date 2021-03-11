---
title: "RSS Feed Reader with Phoenix"
date: 2021-03-11T10:15:14Z
draft: true
description: "Writing an RSS feed reader for codementor.io project"
tags:
  - elixir
  - phoenix
  - otp
keywords:
  - elixir
  - phoenix
  - concurrency
  - otp
  - codementor
---

I've been learning and using Elixir for some time now, and aside from some small
projects created by following a tutorial, or reading something in a book, I've always
struggled with coming up with a project to work on from scratch. Something interesting
that also makes used of Elixir's strengths. I happened upon [Codementor.io's DevProjects](https://www.codementor.io/projects)
which has the tag line _Still writing “Hello World”? Build real-world projects_

What better way to get aquainted with a language and its ecosystem than to write a project from
scratch: design, implements, release. So that's what I did by implementing a solution
for the [RSS Feed Reader project](https://www.codementor.io/projects/rss-feed-reader-website-atx32j280x)
In order to share some of what I've learned, I decided to write it up, detailing design, including
some of the descisions I made, things I'm not entirely sure about, and deployment. In theory, if
you were to follow along, you should be able to implement it also.

# Requirements

We are to implement a basic RSS Feed Reader website. The requirements stated are:

```
Requirements

    * The user can input a RSS feed URL.
    * The reader will display the title, description, and link of the original content.

For an extra challenge

    * The user can add more than one RSS feed URL.
    * Instead of using a RSS parser library, build the parser yourself!
```

Seems clear and concise. I'm a big fan of RSS, so it'll also tie in with an interest. I'm not
planning on writing a replacement for GoogleReader or anything, just something to parse and display
some RSS feeds on a page.

I've already decided to use Phoenix as our web framework, and as it's all the rage, let's also
leverage LiveView while we are at it. I'm going to use [Bulma](https://bulma.io) for CSS styling
because I'm familiar with it and will provide everything we need.

# Design

## Frontend

First let's think about the input and output. Users will input one or more URLs to RSS feeds
which are then fetched, parsed, and their content displayed on the web page. Each feed has 1 or more
items which will have a header, link, date, and body/summary, and other features.
The main possible errors that could arise are

1. Fetching a URL may timeout/error
2. Parsing a feed may cause an error in the parser
3. Feed content differs across different sources

So now we know some things to bear in mind when thinking about how the tasks are going to be coded. By not relying
on too many data pieces in the RSS feed, we hope to minimise having to deal with different items being
available or named different across feeds.

So here's how I imagine the site to look, long with how a feed item might be laid out

![Frontend layout](/images/frontend.jpg "Frontend layout")

## Backend

With the frontend sketched out, let's think about the backend. As we will be fetching and parsing
one or more URLs, we might aswell leverage some of Elixir's great concurrency and bear this in mind
when thinking about how we're going to fetch and parse the feeds.

In essence a list of URLs will be passed to a _coordinator_ which will pass each URL
to a dedicated _worker_ which will fetch the feed content and parse it. The _coordinator_
will gather the results of the _workers_ and return those to the caller. Seems like a
job for some sort of GenServer type setup. _Coordinator_ will spawn 1 or more _workers_ and then
listen for messages from the _workers_, collecting the results until all URLs have been processed.
Once done, it will exit its listening loop and return the feeds. The _workers_ will do what they
do and each time they complete processing a URL will pass a message to the _coordinator_ with the
relevant information. Something like this

![Coordinator and workers](/images/workers.jpg "Coordinator and workers")

The _coordinator_ returns the results to the caller which will then pass the data
to the front end.

The URLs will be in the form of a list of strings. The results however will need a little
more thought as we may end up with error conditions, which we'd like to feed back to the user.
As the URLs will either be successful or not, a map with two keys might be suitable. Something like
this perhaps?

```ex
%{ok: [], error: []}
```

It should clear that the `:ok` key will index into a list of successfully parsed feeds and
the `:error` key will index to a list of errored (for whatever reason, we won't care why it errored)
feeds. We will display a feed for all entries under `:ok` and something like
`Error processing https://invalidfeed.com/` for the errored ones. Which means the list
for `:error` will be a list of URL strings.

So the _workers_ must respond with either a `{:ok, rssfeed}` or a `{:error, url}` for the
particular URL they are working on. The _coordinator_ will receive these messages and keep
track of how many feed URLs have been processed and once complete will need to exit its message
loop. This can be done by it sending itself an `:exit` message. Ok, so that will handle our requirements.
It should be noted that there's no handling for timeouts. Should a _worker_ get stuck in a loop
or somehow never complete its work, there's no mechanism at this point to handle that. I'm going to
rely on the HTTP client to timeout on invalid requests, and for any other case just cross my fingers.

# Setup

So with our design out of the way, let's move onto getting a basic Phoenix application setup.
We won't need Ecto as there's no database, but we will be using LiveView, so let's
create our project. Make sure you have npm installed.

```bash
mix phx.new rss_reader --no-ecto --live
```

And with that we have our basic application ready. If you follow the onscreen instructions
you can check everything went ok

```bash
cd rss_reader
mix phx.server
```

and visit `localhost:4000` you should see the Phoenix page.

Right, so next up is to add Bulma for our styling
