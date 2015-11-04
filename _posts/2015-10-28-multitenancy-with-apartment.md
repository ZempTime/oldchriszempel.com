---
layout: post
title:  "The TL;DR of Multitenancy in Rails with Apartment"
date:   2015-10-28 12:22:25
---

This post will overview some choices you can make surrounding multitenancy, then dive into the how and why behind some design decisions in some different situations. Multitenancy might scare you because it sounds like this large and complicated thing. It's not, it's relatively straightforward and fun to do. If you've never implemented multitenancy in a Rails app, you probably have this question:

## When and why do you need multitenancy?

Multitenancy is when you want to provide the same application to a bunch of different groups of users in such a way that they're fundamentally segregated. Say you're writing an inventory management system - the companies you're building this for definitely don't want their competition to see everything they have!

So, before diving into exactly what you can do, it makes more sense to ask yourself: what do I really need this to do?

* Securely segregate and separate user data
* Provide a way for a bunch of users and organizations to run out of the same app.

So really, the only thing you need to worry about is, when you call data up from the database...how do you make sure you're calling the right information for the right person? Maybe by the subdomain, maybe by other information you can ask about a user that's already in the app.

## What are the approaches?

With Rails, there are three:

1. Handrolling it
2. Multi-Schema with Apartment
3. Single-schema with acts_as_tenant

### Handrolling it

  You can always do this yourself. In fact, if it's your own app, I [advocate this approach](https://gorails.com/episodes/subdomains-and-multi-tenancy-from-scratch?autoplay=1) for your own learning. You could even theoretically spin up an entirely new database for each new tenant...but unless you have some pretty foolproof ways of maintaining migrations across all of the databases, this is probably a horrible idea in practice.

However, when you're writing code for other people, your time is their money and and those people need to go tell their eventual users "your data is completely safe - in fact, it's completely separate from everyone else's" then I'd recommend leaning on something existing unless you've already implemented and maintained something along these lines on your own.

### Multi-Schema with Apartment

Postgres provides a mechanism by which a single databse can have multiple schemas, so you can go ahead and segregate everything by schema. This is largely my preferred approach, and the one I'll be diving into more, so let's glance at the next one.

### Single-Schema with acts_as_tenant

The difference between single-schema and multi-schema is this - when we call up data from the database, in one call, can we potentially access everything?

I've written up an expert scenario demonstrating the difference. A hacker has compromised your app, although you're fast closing in on them, splinters of wood are flying off their door as you maul it down with a metal bat. They only have time to run this code:

{% highlight ruby %}
@confidential = SensitiveInfo.all
@confidential.each do |info|
  BlackMarket.sell(info)
end
{% endhighlight %}

Did they just sell all the info in your application on the black market, or only one of the tenant's information?

In a multi-schema setup, you can only grab the information residing in that one schema. Single-schema, you can theoretically grab everything, although the point is not to do that. So it depends on how you scope stuff:

Did that hacker execute the code straight up on the model, or just in a controller somewhere? If it was in a controller, then the info will likely have been scoped by some code resembling this ([source](https://github.com/ErwinM/acts_as_tenant)):

{% highlight ruby %}
class ApplicationController < ActionController::Base
  set_current_tenant_through_filter
  before_filter :your_method_that_finds_the_current_tenant

  def your_method_that_finds_the_current_tenant
    current_account = Account.find_it
    set_current_tenant(current_account)
  end
end
{% endhighlight %}

In a single-schema setup, below this kind of scoping that happens at the controller level, everything lives in the same place. In a multi-schema one, you'll only ever be able to access one tenant at a time.

One of the notable differences between approaches is that if you host on Heroku, they recommend against multi-schema setups because it makes backups really hard. It also gives you some extra hoops to jump through if you ever want to ask questions about everything in the database. However, if you DO go with mult-tenant...

## Apartment: Public vs Private

The main concept to wrap your head around in Apartment are actually Postgres schemas, because Apartment is an extremely convenient wrappeer around those. Apartment will manage all the migrations and provide some easy ways to make sure you're choosing the right place.

Here's a description straight from the elephant's mouth:

> A PostgreSQL database cluster contains one or more named databases. Users and groups of users are shared across the entire cluster, but no other data is shared across databases. Any given client connection to the server can access only the data in a single database, the one specified in the connection request.

Each "schema" is like a namespace in the database, and you can categorize every query by whether or not it is this: public, or not public.

"Public" schemas are where you put everything that needs to be universally available. So pretty much will be the stuff that tells you what part of the database you actually want to be talking with.


{% highlight ruby %}
User.first -> "SELECT  "public"."users".* FROM "public"."users"  ORDER BY "public"."users"."id" ASC LIMIT 1"
{% endhighlight %}

You can see that it grabs the information out of public.users.

Thing is, for the rest of the data, you can only access one tenant at a time. This is the second kind of "not public" - meaning, you'll only access the data available in the "tenant" specified by the request.

So if you've got three major clients, and that hacker ran

{% highlight ruby %}
SensitiveInfo.all
{% endhighlight %}

They'd only be stealing one of your client's data. Not all three. So when you finally break into the room where the hacker is, they'll swiftly say, "If it isn't Leopard Boy and the Decepticons." Then you take them out and 2/3 of your client's data is safe.

Let's dig into the things you'll actually need to decide in order to build in multitenancy with Apartment.

## Setting up your Signposts

So typically, you determine what data somebody wants to access by one of a couple things: the subdomain, the user, the company or account that user is associated with, etc.

I'm going to go ahead and call this the "signpost" - the piece information you look to that tells your requests where to go in the database.

Now, all of those tutorials cover basic scenarios, so let's spice it up:

* Each one of your tenants (separate companies) will have many users
* Those users will need to be able to switch between the various companies they're a part of on the fly - on the same account

So in this case, where's your signpost? Well, it sounds like you need a new idea to represent this relationship: a Membership.

Hey, so if you hate ActiveRecord, think it's a horrible ORM, and want to marry your database, then why are you even reading this? Anyway don't read any further. Trigger warning. What I'm going to detail is an approach that's incredibly user-friendly and a fantastic setup for the kinds of usage most people will get.

It probably looks something like this:

{% highlight ruby %}
class User < ActiveRecord::Base
  has_many :memberships
  has_many :companies, through: :memberships
  has_many :sensitive_infos
end

class Membership < ActiveRecord::Base
  belongs_to :users
  belongs_to :companies
end

class Company < ActiveRecord::Base
  has_many :memberships
  has_many :users, through: :memberships
end
{% endhighlight %}

In this case, your signpost is actually the Membership class. That'll tell you what company the user currently wants to access the information of.

Hold on though - what's actually going on here? So you know what company who's sensitive information you want to access, but how does that actually work with Apartment?


{% highlight ruby %}
  Apartment::Tenant.switch!(current_membership.company.tenant_name)
{% endhighlight %}

The answers, as with most things in life, are actually somewhere in the configuration files.

{% highlight ruby %}
# config/initializers/apartment.rb
Apartment.configure do |config|

  # Add any models that you do not want to be multi-tenanted, but remain in the global (public) namespace.
  # A typical example would be a Customer or Tenant model that stores each Tenant's information.
  #
  config.excluded_models = %w{ User Company Membership}

  config.tenant_names = lambda { Company.pluck :tenant_name}
{% endhighlight %}

Each company has a tenant_name, or a shortened, unique rendition of the company's full name, which you feed into the Apartment::Tenant.switch! method. This will make queries run in the schema associated with that company.

Now you can actually make this whatever you want, so long as you don't have any strange punctuation and generally stick to lower case versions of things. In this case, @company.tenant_name obviously meant to be used to indicate what schema to be accessed via Apartment.

Your heart starts racing. Wait. If you have to define a tenant before a request...

{% highlight ruby %}
`rails c`
SensitiveInfo.first

#=>   SensitiveInfo Load (0.9ms)  SELECT  "sensitive_info".* FROM "sensitive_info"  ORDER BY "sensitive_info"."id" ASC LIMIT 1
=> nil
{% endhighlight %}

This plague of a hacker didn't actually sell any of your user's information on the black market, because he forgot to set a tenant beforehand! You rejoice, noting the growing pool of blood around his head where you incapacitated him on the floor. You should probably call an ambulance. I mean, this isn't Texas.

Your new tenants will be created at two times:

* Whenever you make a new one
{% highlight ruby %}
class Company < ActiveRecord::Base
  after_create :create_tenant

  private
    def create_tenant
      Apartment::Tenant.create(tenant_name)
    end
end
{% endhighlight %}

* Or whenever you run rake db:migrate, that block of code in Apartment.rb will run and add any tenants that aren't currently there.
{% highlight ruby %}
lambda { Company.pluck :tenant_name}
{% endhighlight %}


## Background Jobs

Almost all applications are going to want to do some sort of background processing at some point. How does this work with Apartment?

You just need to worry about two things:

1. You record what tenant the job is supposed to run inside
2. You make sure to switch to that tenant before running the job

This simply means, you'll need to pass in an extra argument to each job about what tenant this job is supposed to operate on. Presumably, this will already be set in the request that creates the job.

{% highlight ruby %}
MysteriousJob.perform_later(real_args, tenant)


class MysteriousJob < ActiveJob::Base

  def perform(real_args, tenant)
    Apartment::Tenant.switch!(tenant)

    over_here = real_args.send_over_there
    over_here.look_over_and_grimly_disapprove!
    over_here.send_back_over_there
  end
{% endhighlight %}

What you do and how you name your methods in your background jobs is your own business. I won't judge.

Now the thing is, if you use Sidekiq, you won't need to worry about this - check out [apartment-sidekiq](https://github.com/influitive/apartment-sidekiq). All you have to do is install the gem.

> That's it. There's nothing to do. Each job that is queued will get an additional entry storing Apartment::Tenant.current when it is queued. Then when the server pops it, it will run job within an Apartment::Tenant.switch block.

You look down at the unconscious hacker, and look away in disgust while strongly exhale an "Ugh!" Spandex is a privelege, not a right.
