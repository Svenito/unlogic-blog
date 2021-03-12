---
title: "RSS Feed Reader with Phoenix"
date: 2021-03-11T10:15:14Z
draft: false
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

I've been learning and using Elixir for some time now, and aside from some small
projects created by following a tutorial, or reading something in a book, I've always
struggled with coming up with a project to work on from scratch. Something interesting
that also makes used of Elixir's strengths. I happened upon [Codementor.io's DevProjects](https://www.codementor.io/projects)
which has the tag line _Still writing “Hello World”? Build real-world projects_

What better way to get aquainted with a language and its ecosystem than to write a project from
scratch: design, implement, release. So I decided to implement a solution
for the [RSS Feed Reader project](https://www.codementor.io/projects/rss-feed-reader-website-atx32j280x).
In order to share some of what I've learned, I decided to write it up, detailing design, including
some of the descisions I made, things I'm not entirely sure about, implementation, and deployment.
In theory, if you were to follow along, you should be able to implement it also.

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

Seems clear and concise. I'm a big fan of RSS for keeping up to date with interesing sites,
so it'll also tie in with an interest of mine. I'm not planning on writing a replacement for
GoogleReader or anything, just something to parse and display some RSS feeds on a page.

I've already decided to use Phoenix as our web framework, and as it's all the rage, let's also
leverage [LiveView](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html)
while we are at it. I'm going to use [Bulma](https://bulma.io) for CSS styling
because I'm familiar with it and will provide everything we need. If it were a bigger project, perhaps
Tailwind CSS might be more suitable. Feel free to use any CSS framework of your choosing.

# Design

## Frontend

First let's think about the input and output. Users will input one or more URLs to RSS feeds
which are then fetched, parsed, and their content displayed on the web page. Each feed has
a title, and 1 or more items which will have a header, link, date, and body/summary, and
other features. The main possible errors I can initially think of are

1. Fetching a URL may timeout/error
2. Parsing a feed may cause an error in the parser
3. Feed content differs across different sources

So now we know some things to bear in mind when thinking about how the tasks are going to be coded. By not relying
on too many data pieces in the RSS feed, we hope to minimise having to deal with different items being
available or named different across feeds. We're going to display the feed title and image (if there is one),
and for each item we will show its title as a link to the full article, its date, and summary content.

So here's how I imagine the site to look, along with how a feed item might be laid out

![Frontend layout](/images/frontend.jpg "Frontend layout")
![Feed Item layout](/images/feed_item.jpg "Feed Item Layout")

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
loop. This can be done by it sending itself an `:exit` message.

![Worker and Coordinator comms](/images/coord_worker_details.jpg)

Ok, so that will handle our requirements.
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

![Default Phoenix landing page](/images/phoenix_default_page.png)

Right, so next up is to add Bulma for our styling. Change to the `assets` directory and
run

```bash
npm install --save-dev bulma
```

Once complete we can change into the `css` directory and remove the `phoenix.css`
and create a `_custom.css` file where any style overrides we might need to make will
be added.

```bash
cd css
rm phoenix.css
touch _custom.css
```

Now edit the `app.scss` file and add these two lines to the top

```css
@import "custom";
@import "bulma";
```

and also remove the `@import "phoenix.css";` line as it's no longer needed.

This will import our custom styles and the bulma styles into the application CSS. If you
start the Phoenix server now, or reload the page if it's already running, our default page should
now look like this

![Bulma styled default Phoenix page](/images/phoenix_bulma.png)

If you see the above you're in a good place, so let's remove the default content and
start adding our own. Inside the Phoenix project, edit `lib/rss_reader_web/live/page_live.html.leex`.
This is the LiveView template page that will get rendered by default. We can delete everything in here
and replace it with whatever we like, for example some text

```html
HELLO FROM LIVEVIEW
```

Reload and we see

![Hello from LiveView](/images/hello_from_liveview.png)

So let us remove the header and links at the top so we start with a clean slate. Open
`lib/rss_reader_web/templates/layout/root.html.leex` and remove everything between the
`<header></header>` tags which reside inside the `<body>` tags to end up with this.

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8"/>
    <meta http-equiv="X-UA-Compatible" content="IE=edge"/>
    <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
    <%= csrf_meta_tag() %>
    <%= live_title_tag assigns[:page_title] || "RssReader", suffix: " · Phoenix Framework" %>
    <link phx-track-static rel="stylesheet" href="<%= Routes.static_path(@conn, "/css/app.css") %>"/>
    <script defer phx-track-static type="text/javascript" src="<%= Routes.static_path(@conn, "/js/app.js") %>"></script>
  </head>
  <body>
    <%= @inner_content %>
  </body>
</html>
```

Finally to make Bulma work nicely, we can wrap the `@inner_content` in a `main` block

```html
<body>
  <main role="main" class="section"><%= @inner_content %></main>
</body>
```

Now everything we will render (the RSS feeds and their items) will be rendered through the `@inner_content` which
comes from the `lib/rss_reader_web/live/page_live.html.leex` template. This template will receive its data from
`lib/rss_reader_web/live/page_live.ex` which we will work on in the next section. This file will be what
I've referred to as the _caller_ previously. It will invoke the _coordinator_ with a list of URLs it will
receive from the frontend.

A lot of ground has been covered and the project framework is in place now, ready for us to add
the plumbing and the rest of the frontend.
