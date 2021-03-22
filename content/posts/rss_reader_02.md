---
title: "RSS Feed Reader with Phoenix part 2 - processing feeds"
date: 2021-03-21T14:18:51Z
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

This time we're going to implement the coordinator and worker setup we discussed in
[RSS Feed Reader with Phoenix part 1 - Design and setup](/posts/rss_reader_01/).
We will focus on the coordinator and worker setup in the post and leave the frontend and
LiveView details for the next post. If you remember from part 1 we were going to call
a function on the coordinator with a list of _n_ URLs which it will pass to _n_ workers.
Workers will be simple Erlang processes which we will communicate via messaging with.
I could use the GenServer model now, but for the our purpose we will do away with the
Genserver and stick with a simple message handler, as it will suffice for this simple
example.

## Implementing the worker

The worker's job is simple

- Fetch the contents of the URL
- Parse the RSS feed of that URL
- Send a message with either `{:ok, feed}` or `{:error, URL}` to the caller

Let's start with implementing the _worker_, specifically the `process_url\2` function.
Create a new file `lib/rss/worker.ex` and add the following lines:

```ex
defmodule Rss.Worker do
  def process_url(sender_pid, feed_url) do
    send(sender_pid, parse_feed(feed_url))
  end

  defp parse_feed(feed_url) do

  end
end
```

This defines the function that the _coordinator_ will call when it spawns the worker.
Its arguments are the `sender_pid` which is the process id of the caller, and the URL
it is going to process. As you can see, it will send a message to `sender_pid` with
the result of the `parse_feed\1` function, which is stubbed out below it.

`parse_feed\1` will need to fetch the URL contents and parse them out, and return either
a success or error state, depending on what happened. Let's build this up step by step,
starting with fetching the content at the URL provided. I've decided to use
[HTTPoison](https://hexdocs.pm/httpoison/HTTPoison.html) as it's a popular library and
pretty much the standard HTTP client in Elixir. As and added bonus I'm also familiar
with it already. In order to use HTTPoison we need to add it as a dependency. So in you
`mix.exs` file (root of the Phoenix project), find the section that begins with
`defp deps do` with the comment `# Specifies your project dependencies.` and at the end
of the list add

```ex
{:httpoison, "~> 1.8"}
```

not forgetting to add the comma at the end of the previous line. In a shell run
`mix deps.get` to download HTTPoison. Once done, we can return to the _worker_ file
and add the code to retrieve the URL. Luckily it's surprisingly simple. Add the following
in the `parse_feed\1` function

```ex
result =
    feed_url
    |> HTTPoison.get()
IO.inspect(result)
```

Inside a shell, change to the project directory and run `iex -S mix` to start an
iex session with the current project. Once loaded we can test our worker by calling the
`process_url\2` function and something like this should materialise.

```ex
iex(3)> RssReader.Worker.process_url(self(), "https://rss.art19.com/apology-line")
{:ok,
 %HTTPoison.Response{
   body: "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n<rss version=\"2.0\"
   xmlns:itunes=\"http://www.itunes.com/dtds/podcast-1.0.dtd\"
   xmlns:atom=\"http://www.w3.org/2005/Atom\"
   xmlns:cont...
```

That's a good result from HTTPoison in the form of `{:ok, HTTPoison.Response.t()` and
this is the RSS feed that we need to parse.

For this we're going to look at some RSS libraries as we're not going to write our own
parser this time. Heading over to [Hex Packages](https://hex.pm/packages/) we can search for
`rss`

![Hex results for RSS](/images/hex_rss_results.png)

So it's probably best to go with one that's near the top of the number of downloads. I've
taken a look at `quinn`, `fast_rss`, `feeder`, and `feeder_ex` and ultimately settled on
`feeder_ex`. I tried `quinn` at first, but couldn't easily get to grips with extracting
the individual items and data out of the results. I wasn't sure if it was expecting a
specific format, but I didn't have much luck with it in the short time I tried it.
Then I tried `fast_rss` and that worked really well. The only thing that I wasn't so
keen on is having a dependency on Rust build tools. It's fine locally, but it's another
thing to think about when you want to deploy your project on Heroku or something. `feed_ex`
provided a nice wrapper around `feeder` and it was easy to figure out how to work with it,
so I went with that. We can't pipe the result from HTTPoison directly into feeder_ex's parser,
so we need to create some function to handle the HTTPoision response cases. So let's
modify the `parse_feed\1` function by adding a call to the function that will parse the result

```ex
result =
      feed_url
      |> HTTPoison.get()
      |> parse_response
```

Then to the `worker.ex` file we can add

```ex
defp parse_response({:ok, %HTTPoison.Response{body: body, status_code: 200}}) do
    case FeederEx.parse(body) do
      {:ok, rss_map, _} ->
        {:ok, rss_map}

      _ ->
        :error
    end
  end

  defp parse_response(_) do
    :error
  end
```

Here I've used pattern matching to handle any errors from HTTPoison. If the response
is in the form `{:ok, HTTPoison.Response}` and has a `status_code` of 200 we match the `body`
and assign it to the `body` variable which we then pass to FeederEx's parser. For any other
response we just return `:error`. Let's look further into the first function, the one handling
a status*code of 200. The first line calls the `FeederEx.parse\1` function and matches the result
to `{:ok, rss_map, *}`as a good result, and anything else will be an`:error`. On the first match we return `{:ok, rss_map}`and that's it. Now we need to handle the results in the caller`parse_feed\1`
by adding this at the end

```ex
defp parse_feed(feed_url) do
  result =
    feed_url
    |> HTTPoison.get()
    |> parse_response

  case result do
    {:ok, items} -> {:ok, items}
    :error -> {:error, feed_url}
  end
end
```

You might be wondering why we can't just return either `{:ok, items}` or `{:error, feed_url}`
directly from the `parse_response\1` function. Very true. We could. We'd have to make a change
to the function definition, as an error condition would need to return the URL. We can get the
URL from the HTTPoison response, so we'd have to change the error case of `parse_response\1` to be

```ex
defp parse_response({:ok, %HTTPoison.Response{request_url: url}}) do
    {:error, url}
end
```

This will handle all responses that do not have a status code of 200, so we assume anything
other than that is an error. Great right? Well not quite. What happens if we supply something
that isn't a valid URL?

```ex
iex(8)> HTTPoison.get("ggggg")
{:error, %HTTPoison.Error{id: nil, reason: :nxdomain}}
```

We don't get the URL passed through, and are unable to return `{:error, URL}` therefore.
By leaving the response in the `parse_feed\1` function, where we have the URL, returning
`{:error, URL}` is trivial.

Shall we test our _worker_ now?

```ex
iex(9)> Rss.Worker.process_url(self(), "h.art19.com/apolo")
{:error, "h.art19.com/apolo"}

iex(10)> Rss.Worker.process_url(self(), "https://rss.art19.com/apology-line")
{:ok,
 %FeederEx.Feed{
   author: "Wondery",
   entries: [
     %FeederEx.Entry{
       author: nil,
       categories: [],
```

The results are what we expect, we're ready to move onto the coordinator.

Sorry, you have a question?

Automated testing? Yes of course we can do that. I'll go over that at the end if there's time.
Yes I know it's not TDD, and it's not a huge project either, and more a proof of concept.
Also, as we're using HTTPoison we're also going to need to mock some behaviour and we've got
exhaustively generate all manner of different feeds with different... yes, I know, and as I said,
if there's time I will cover some testing, ok?

With our _worker_ complete, we can move onto the last piece of the puzzle: the _coordinator_

## Implementing the Coordinator

As said, the _coordinator_ will spawn `n` number of _workers_ equal to the number of URLs
passed to it. Let's handle the simple case of having no URLs passed first

```ex
def rss_items([]) do
  %{ok: [], error: []}
end
```

Pretty straightforward. Now onto the actual machinery.

```ex
defmodule RssReader.Coordinator do
  defp loop(msg, results_expected) do
    receive do
      :exit -> :pizza

      _ ->
        IO.puts("TEST")
        IO.inspect(msg)
        send(self(), :exit)
        loop(msg, results_expected)
    end
  end

  def rss_items(feed_urls) do
    # Spawn RSS fetcher & parser. One for each feed
    feed_urls
      |> Enum.map(fn feed_url ->
        spawn(Rss.Worker, :process_url, [self(), feed_url])
      end)

    # Process messages from workers until all have been processed
    loop(%{ok: [], error: []}, Enum.count(feed_urls))
  end
end
```

This doesn't handle the messages correctly yet, but it's enough for us to test out
spawning the _workers_. To explain what's happening here in the `rss_items\1` function

- Use `Enum.map` to iterate over the items in the `feed_urls` list
- For each item `spawn` a worker process by calling `process_url\1` with the current pid and the feed_url
- Start the listener loop which currently does very little

If you start this now, the _coordinator_ won't stop until it received an `:exit` message. We can
try it out in an `iex` session

```ex
RssReader.Coordinator.rss_items(["url"])
TEST
%{error: [], ok: []}
:pizza
```

What happened here is that we spawn the worker and then start the _loop_ to listen for
messages. Our first message is received when we invoke the loop. This is handled the by the default case
`_`. In this block the output the string `TEST`, inspect our `msg`, send ourselves the `:exit` message,
and invoke loop again. When the loop is invoked the second time, `:exit` is sitting in the mailbox, and
this is read by `receive_do`, which handles that by returning `:pizza`. As `loop\2` is not
invoked again, we end up back at the iex prompt.

In order to make the _coordinator_ useful we need to handle the expected messages,
create our results, and handle the exit condition. Let's start with the simple ones:
exit and _any other messages_. We do this with pattern matching the `receive do` like
so

```ex
defp loop(%{ok: results, error: urls}, results_expected) do
  receive do
    :exit ->
      %{ok: results, error: urls}

  _ ->
    loop(%{ok: results, error: urls}, results_expected)
  end
end
```

When we exit we want to return the results, and for any other condition (the `_`)
we just loop again with the current information we have, waiting for further messages.
Right, so adding in the conditions for a successful parse, and one that errored is a matter
of matching in the message we expect from the _worker_

```ex
defp loop(%{ok: results, error: urls}, results_expected) do
  receive do
    {:ok, result} ->
      loop(%{ok: results, error: urls}, results_expected)

    {:error, url} ->
      loop(%{ok: results, error:urls}, results_expected)

    :exit ->
      %{ok: results, error: urls}

    _ ->
      loop(%{ok: results, error: urls}, results_expected)
  end
end
```

Pretty similar to the catch-all condition right now, but that's the basic behaviour
we want. What we need to do next is test for the exit condition: `results_expected` and the
size of the `:ok` and `:error` results are the same. So we need to add the message contents
to the correct list, check the sizes compared to `results_expected`, and if they match we send
ourselves `:exit`, otherwise we loop.

```ex
defp loop(%{ok: results, error: urls}, results_expected) do
  receive do
    {:ok, result} ->
      new_results = [result | results]

      if results_expected == Enum.count(new_results) + Enum.count(urls) do
        send(self(), :exit)
      end

      loop(%{ok: new_results, error: urls}, results_expected)

    {:error, url} ->
      new_urls = [url | urls]

      if results_expected == Enum.count(new_urls) + Enum.count(results) do
        send(self(), :exit)
      end

      loop(%{ok: results, error: new_urls}, results_expected)

    :exit ->
      %{ok: results, error: urls}

    _ ->
      loop(%{ok: results, error: urls}, results_expected)
  end
end
```

As the two new cases are pretty similar, I'll go over the `:ok:` case which should
also explain what's going on in the `:error` case. The first line concatenates the
original `results` with the new result, putting the new result at the head of the list.
This is the faster way to insert items into a list, and the order doesn't really matter
for us. Then we check the length of the two lists (ok and error) against `results_expected`
and if they match we send the `:exit` message to ourselves, otherwise we loop, waiting for
more messages to arrive.

Let's try it out in a terminal

```ex
iex(1)> RssReader.Coordinator.rss_items(["url1", "url2", "url3"])
%{error: ["url2", "url1", "url3"], ok: []}
iex(2)> RssReader.Coordinator.rss_items(["url1", "url2", "https://rss.art19.com/apology-line"])
%{
  error: ["url2", "url1"],
  ok: [
    %FeederEx.Feed{
      author: "Wondery",
      entries: [
        %FeederEx.Entry{
.
.
.
```

Results as we expected. In this project we don't care about the order of the returned feeds. The
errored feeds in this case will return quickly and will be reversed as we are putting each result
at the head of the list, and the valid URLs could come in any order, depending on how long
each takes to process. If we wanted to, we could add some extra code to maintain order, but
that's not part of the requirement, so let's not over complicate things right now.

## That's all for now

With that done we've implemented a feed parser which fetches and parses feeds in parallel,
returns the results in a map, separating them into errored and non-errored results, ready
for our frontend to display. Next time we'll start working on the LiveView side of things
and get our results back to our user. We'll add some CSS to make the feeds look nice and wrap
up the project.
