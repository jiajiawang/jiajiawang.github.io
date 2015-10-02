---
layout: post
comments: true
title:  "Full text search with Elasticsearch"
date:   2015-07-24 19:32:28
categories: rails
---

[Elasticsearch](https://www.elastic.co/products/elasticsearch) is a full text
search server based on [Lucene](https://lucene.apache.org/core/).
It's easy to set up but very powerful.

Today, I'd like to show you an example of the basic usage of Elasticsearch in rails.

## Scenario
Let's say that you have a model named 'Shop' and it has attributes
'name', 'address', 'description' and 'categories'.

~~~
rails g scaffold shop name address description categories
rake db:migrate
~~~

And you want to create a search bar that can return a list of shops whose
attributes match the keywords user typed in.

How will you do it?

I assume that you won't want something like

~~~
Shop.where('name like :keywords or address like :keywords or description like :keywords or categories like :keywords', keywords: '%keywords%')')
~~~

## Elasticsearch

### Installation

Firstly, install Elasticsearch on your Mac and run it in background:

~~~
brew install elasticsearch
elasticsearch -d
~~~

Add 'elasticsearch-model' and 'elasticsearch-rails' to your Gemfile

~~~
gem 'elasticsearch-model'
gem 'elasticsearch-rails'
~~~

### Setup
Update 'app/models/shop.rb' as the following

~~~ruby
  require 'elasticsearch/model'

  class Shop < ActiveRecord::Base
    include Elasticsearch::Model
    include Elasticsearch::Model::Callbacks
  end
~~~

This will extend the model with functionality related to Elasticsearch.

### Usage

#### Importing the data

Let's create some db data first.

Add the following to db/seeds.rb:

~~~ruby
  Shop.create([
    {name: 'KFC', address: '1 King St, Sydney', description: 'Fried Chicken, Burger', categories: 'Food'},
    {name: 'Burger King', address: '1 Queen St, Sydney', description: 'Burger, Chips', categories: 'Food'},
    {name: 'JB HiFi', address: '2 Queen St, Sydney', description: 'HiFi, TV', categories: 'Electronics'}
  ])
~~~

Then execute `rake db:seed`.

To index the db data execute:

~~~
Shop.import
~~~

#### Searching

Now you can start searching shops by using `Shop.search` and here are some
simple examples.

~~~
irb(main):001:0> Shop.search('burger').records.to_a
=> [#<Shop id: 2, name: "Burger King", address: "1 Queen St, Sydney", description: "Burger, Chips", categories: "Food", created_at: "2015-10-01 14:49:10", updated_at: "2015-10-01 14:49:10">, #<Shop id: 1, name: "KFC", address: "1 King St, Sydney", description: "Fried Chicken, Burger", categories: "Food", created_at: "2015-10-01 14:49:10", updated_at: "2015-10-01 14:49:10">]
irb(main):002:0> Shop.search('king').records.to_a
=> [#<Shop id: 1, name: "KFC", address: "1 King St, Sydney", description: "Fried Chicken, Burger", categories: "Food", created_at: "2015-10-01 14:49:10", updated_at: "2015-10-01 14:49:10">, #<Shop id: 2, name: "Burger King", address: "1 Queen St, Sydney", description: "Burger, Chips", categories: "Food", created_at: "2015-10-01 14:49:10", updated_at: "2015-10-01 14:49:10">]
irb(main):003:0> Shop.search('hifi').records.to_a
=> [#<Shop id: 3, name: "JB HiFi", address: "2 Queen St, Sydney", description: "HiFi, TV", categories: "Electronics", created_at: "2015-10-01 14:49:24", updated_at: "2015-10-01 14:49:24">]
irb(main):004:0> Shop.search('sydney').records.to_a
=> [#<Shop id: 3, name: "JB HiFi", address: "2 Queen St, Sydney", description: "HiFi, TV", categories: "Electronics", created_at: "2015-10-01 14:49:24", updated_at: "2015-10-01 14:49:24">, #<Shop id: 1, name: "KFC", address: "1 King St, Sydney", description: "Fried Chicken, Burger", categories: "Food", created_at: "2015-10-01 14:49:10", updated_at: "2015-10-01 14:49:10">, #<Shop id: 2, name: "Burger King", address: "1 Queen St, Sydney", description: "Burger, Chips", categories: "Food", created_at: "2015-10-01 14:49:10", updated_at: "2015-10-01 14:49:10">]
~~~

Can you tell what will be the result of `Shop.search('sydney burger').records.to_a` ?

As you can see below, it returns all three shops and this may not be what you expected.

~~~
irb(main):005:0> Shop.search('sydney burger').records.to_a
=> [#<Shop id: 2, name: "Burger King", address: "1 Queen St, Sydney", description: "Burger, Chips", categories: "Food", created_at: "2015-10-01 14:49:10", updated_at: "2015-10-01 14:49:10">, #<Shop id: 1, name: "KFC", address: "1 King St, Sydney", description: "Fried Chicken, Burger", categories: "Food", created_at: "2015-10-01 14:49:10", updated_at: "2015-10-01 14:49:10">, #<Shop id: 3, name: "JB HiFi", address: "2 Queen St, Sydney", description: "HiFi, TV", categories: "Electronics", created_at: "2015-10-01 14:49:24", updated_at: "2015-10-01 14:49:24">]
irb(main):006:0>
~~~

Normally, in this case we would expect that it only returns 'Burger King' and
'KFC'.

The reason of getting the above result is because by default Elasticsearch uses
'OR' operator when querying multiple keywords.

To override this behaviour, you can search shops by using customized queries.

~~~
irb(main):001:0> Shop.search(query: {match: { _all: { query: 'Sydney Burger', operator: 'and' } } }).records.to_a
=> [#<Shop id: 2, name: "Burger King", address: "1 Queen St, Sydney", description: "Burger, Chips", categories: "Food", created_at: "2015-10-01 14:49:10", updated_at: "2015-10-01 14:49:10">, #<Shop id: 1, name: "KFC", address: "1 King St, Sydney", description: "Fried Chicken, Burger", categories: "Food", created_at: "2015-10-01 14:49:10", updated_at: "2015-10-01 14:49:10">]
~~~

What about fuzzy search?

As you can expected, `Shop.search('foood').records.to_a` will return 0 result.

To enable fuzzy search, you can add `fuzziness: 'AUTO'` to your query.

~~~
irb(main):007:0> Shop.search(query: {match: { _all: { query: 'Foood', fuzziness: 'AUTO'} } }).records.to_a
=> [#<Shop id: 1, name: "KFC", address: "1 King St, Sydney", description: "Fried Chicken, Burger", categories: "Food", created_at: "2015-10-01 14:49:10", updated_at: "2015-10-01 14:49:10">, #<Shop id: 2, name: "Burger King", address: "1 Queen St, Sydney", description: "Burger, Chips", categories: "Food", created_at: "2015-10-01 14:49:10", updated_at: "2015-10-01 14:49:10">]
~~~

## What's next
The query dsl of Elasticsearch is very powerful and can be very complex.

If you want to learn more advanced search queries, you can find it
[here](https://www.elastic.co/guide/en/elasticsearch/reference/1.7/query-dsl.html).
