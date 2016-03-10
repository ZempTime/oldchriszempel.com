---
layout: post
title:  "Filtering by Date Ranges with SearchKick & ElasticSearch"
date:   2016-3-10 12:22:25
---

In this post you'll find out how to filter & sort records by date ranges. And also one of most insidious mistakes I make as a programmer.

(I hope you arrived at this page because you know what this means, but just in case: [SearchKick](https://github.com/ankane/searchkick) is an absolutely wonderful project that lets you easily utilize [ElasticSearch](https://www.elastic.co/products/elasticsearch) in your Rails Apps.)

## Filtering by Date Range

Let's set the stage:

```ruby
# We've got our model that we want to filter:
class Lemon < ActiveModel::Base
  searchkick

  def search_data
    {
      name: name,
      good_until: good_until,
      picked_at: picked_at,
      description: description
    }
end

# And our controller where the magic happens
class LemonsController < ApplicationController
  def search
    @lemons = Lemon.search("*").results
  end
end
```

We want to know all the lemons that will be good until next week! To do that, we need to hit:

`localhost:3000/lemons/search?good_until=2016-03-17`

```ruby
class LemonsController < ApplicationController
  def search
    @lemons = Lemon.search("*", { where: { picked_at: {lte: params[:good_until]} }).results
  end
end
```

But what if we want the lemons good until next, but picked within the last week, too?

`localhost:3000/lemons/search?good_until=2016-03-17&picked_at=2016-03-03`

```ruby
class LemonsController < ApplicationController
  def search
    @lemons = Lemon.search("*", { where: { good_until: {lte: params[:good_until], gte: params[:picked_at]} }).results
  end
end
```

Scratch both those - we want to show the results for any lemons good until next week, OR picked within the last two weeks (even if they've gone bad)

`localhost:3000/lemons/search?good_until=2016-03-17&picked_at=2016-02-25`

```ruby
class LemonsController < ApplicationController
  def search
    @lemons = Lemon.search("*", { where: { good_until: {lte: params[:good_until]}, or: [ picked_at: {gte: params[:picked_at]}]} }).results
  end
end
```

There are two things to pay attention to:

1. Look at the [queries](https://github.com/ankane/searchkick#queries) section closely, specifically the first example in the 'where' section.
2. Now glance over the [date logic](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-range-query.html) here.

## The Insidious Mistake

Here's exactly what I did:

I went to the [SearchKick](https://github.com/ankane/searchkick) readme and searched for 'date_range.' That shot me to the bottom section on facets/aggregations.

I put an aggregation in my controller with the same syntax as the readme:

```ruby
class LemonsController < ApplicationController
  def search
    date_ranges = [{from: params[:picked_at], to: params[:good_until]}]
    @lemons = Lemon.search("*", aggs: {date_field: {date_ranges: date_ranges}}).results
  end
end
```

It didn't work - life was giving me all the lemons - I got frustrated, and allowed my brain to shut off. I spent way more time on this than I needed to, and that's why I wrote this post.

There is a basic truth I consistently ignore from time to time, and it's this: technology will always do exactly what you tell it. Technology is a wonderful tool, but a poor master. Rather than flipping on my brain, and taking that extra 20 minutes to just really read through the documentation, I tossed 2-3 hours in flippant violation of any sort of reasoned approach to figuring out the problem. Why?

The underlying emotion: "This isn't my fault. This should just work. Why doesn't someone already have the answer for me?"

What you choose from the world and how you choose to reflect that in your code are the most important decisions you will make as a programmer. Of course, these will be modulated by several areas:

* like your experience level,
* the policies of your company,
* your specific technical proficiency,
* and what kind of performance you need - for 4 users or highly available & eventually consistent,
* etc, etc, *etc!*

It's extremely easy to overcomplicate this. What's hard is to simplify.

All of those reasons above (even the 'etc') have one thing in common between them: it depends. So much of any discussion depends on context and nuance, in programming almost more than any other due to the nature of needing to break things down so explicitly you can tell a computer to do it. So the simple truth worth looking for underneath this ultimately miniscule fiasco *won't* depend on any of that.

And the truth is, when something blows up and doesn't behave the way you intended it to, you can respond in two ways: you can own it, own the fact you messed up, critically examine it, remain in the right professional mindset of pleasant curiosity and healthy emotional detachment and grow, or you can get lost in the story of who you tell yourself you are and blame someone or something else.

It's so silly when I write it out like this, so obvious the right way to be, but efforts still unfurl the wrong way even after all these years.

> “Waste no more time arguing about what a good man should be. Be one.” - Marcus Aurelius

The hidden requirement here is that to be a good man depends upon the habit of presence.
