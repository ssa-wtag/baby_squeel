# BabySqueel

[![Build Status](https://travis-ci.org/rzane/baby_squeel.svg?branch=master)](https://travis-ci.org/rzane/baby_squeel)
[![Code Climate](https://codeclimate.com/github/rzane/baby_squeel/badges/gpa.svg)](https://codeclimate.com/github/rzane/baby_squeel)
[![Coverage Status](https://coveralls.io/repos/github/rzane/baby_squeel/badge.svg?branch=master)](https://coveralls.io/github/rzane/baby_squeel?branch=master)

<img align="right" src="http://static.thefrisky.com/uploads/2010/07/01/pig_in_boots_070110_m.jpg" alt="biddy piggy">

Have you ever used the [squeel](https://github.com/activerecord-hackery/squeel) gem? It's a really nice way to build complex queries. However, squeel monkeypatches ActiveRecord internals, so it has a tendency to break every time a new ActiveRecord version comes out.

For me, that's a deal breaker. BabySqueel provides a query DSL for ActiveRecord without all of the evil. :heart:

It's really just a layer of sugar on top of Arel.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'baby_squeel'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install baby_squeel

## Introduction

With ActiveRecord, you might write something like this:

```ruby
Post.where('created_at >= ?', 2.weeks.ago)
```

But then someone tells you, "Hey, you should use Arel!". So you convert your query to use Arel:

```ruby
Post.where(Post.arel_table[:created_at].gteq(2.weeks.ago))
```

Well, that's great, but it's also pretty verbose. Why don't you give BabySqueel a try:

```ruby
Post.where.has { created_at >= 2.weeks.ago }
```

#### Quick note

BabySqueel's blocks use `instance_eval`, which means you won't have access to your instance variables or methods. Don't worry, there's a really easy solution. Just give arity to the block:

```ruby
Post.where.has { |post| post.created_at >= 2.weeks.ago }
```

## Usage

Okay, so we have some models:

```ruby
class Post < ActiveRecord::Base
  belongs_to :author
  has_many :comments
end

class Author < ActiveRecord::Base
  has_many :posts
  has_many :comments, through: :posts
end

class Comment < ActiveRecord::Base
  belongs_to :post
end
```

##### Selects

```ruby
Post.selecting { (id + 5).as('id_plus_five') }
# SELECT ("posts"."id" + 5) AS id_plus_five FROM "posts"

Post.selecting { id.sum }
# SELECT SUM("posts"."id") FROM "posts"

Post.joins(:author).selecting { [id, author.id] }
# SELECT "posts"."id", "authors"."id" FROM "posts"
# INNER JOIN "authors" ON "authors"."id" = "posts"."author_id"
```

##### Wheres

```ruby
Post.where.has { title == 'My Post' }
# SELECT "posts".* FROM "posts"
# WHERE "posts"."title" = 'My Post'

Post.where.has { title =~ 'My P%' }
# SELECT "posts".* FROM "posts"
# WHERE ("posts"."title" LIKE 'My P%')

Author.where.has { (name =~ 'Ray%') & (id < 5) | (name.lower =~ 'zane%') & (id > 100) }
# SELECT "authors".* FROM "authors"
# WHERE ("authors"."name" LIKE 'Ray%' AND "authors"."id" < 5 OR LOWER("authors"."name") LIKE 'zane%' AND "authors"."id" > 100)

Post.joins(:author).where.has { author.name == 'Ray' }
# SELECT "posts".* FROM "posts"
# INNER JOIN "authors" ON "authors"."id" = "posts"."author_id"
# WHERE "authors"."name" = 'Ray'

Post.joins(author: :posts).where.has { author.posts.title =~ '%fun%' }
# SELECT "posts".* FROM "posts"
# INNER JOIN "authors" ON "authors"."id" = "posts"."author_id"
# INNER JOIN "posts" "posts_authors" ON "posts_authors"."author_id" = "authors"."id"
# WHERE ("posts_authors"."title" LIKE '%fun%')
```

##### Orders

```ruby
Post.ordering { [id.desc, title.asc] }
# SELECT "posts".* FROM "posts"
# ORDER BY "posts"."id" DESC, "posts"."title" ASC

Post.ordering { (id * 5).desc }
# SELECT "posts".* FROM "posts"
# ORDER BY "posts"."id" * 5 DESC

Post.select(:author_id).group(:author_id).ordering { id.count.desc }
# SELECT "posts"."author_id" FROM "posts"
# GROUP BY "posts"."author_id"
# ORDER BY COUNT("posts"."id") DESC

Post.joins(:author).ordering { author.id.desc }
# SELECT "posts".* FROM "posts"
# INNER JOIN "authors" ON "authors"."id" = "posts"."author_id"
# ORDER BY "authors"."id" DESC
```

##### Joins

```ruby
Post.joining { author }
# SELECT "posts".* FROM "posts"
# INNER JOIN "authors" ON "authors"."id" = "posts"."author_id"

Post.joining { [author.outer, comments] }
# SELECT "posts".* FROM "posts"
# LEFT OUTER JOIN "authors" ON "authors"."id" = "posts"."author_id"
# INNER JOIN "comments" ON "comments"."post_id" = "posts"."id"

Post.joining { author.comments }
# SELECT "posts".* FROM "posts"
# INNER JOIN "authors" ON "authors"."id" = "posts"."author_id"
# INNER JOIN "posts" "posts_authors_join" ON "posts_authors_join"."author_id" = "authors"."id"
# INNER JOIN "comments" ON "comments"."post_id" = "posts_authors_join"."id"

Post.joining { author.outer.comments.outer }
# SELECT "posts".* FROM "posts"
# LEFT OUTER JOIN "authors" ON "authors"."id" = "posts"."author_id"
# LEFT OUTER JOIN "posts" "posts_authors_join" ON "posts_authors_join"."author_id" = "authors"."id"
# LEFT OUTER JOIN "comments" ON "comments"."post_id" = "posts_authors_join"."id"

Post.joining { author.comments.outer }
# SELECT "posts".* FROM "posts"
# INNER JOIN "authors" ON "authors"."id" = "posts"."author_id"
# LEFT OUTER JOIN "posts" "posts_authors_join" ON "posts_authors_join"."author_id" = "authors"."id"
# LEFT OUTER JOIN "comments" ON "comments"."post_id" = "posts_authors_join"."id"

Post.joining { author.outer.posts }
# SELECT "posts".* FROM "posts"
# LEFT OUTER JOIN "authors" ON "authors"."id" = "posts"."author_id"
# INNER JOIN "posts" "posts_authors" ON "posts_authors"."author_id" = "authors"."id"

Post.joining { author.alias('a').on((author.id == author_id) | (author.name == title)) }
# SELECT "posts".* FROM "posts"
# INNER JOIN "authors" "a" ON ("authors"."id" = "posts"."author_id" OR "authors"."name" = "posts"."title")
```

##### Grouping

```ruby
Post.selecting { id.count }.grouping { author_id }.when_having { id.count > 5 }
# SELECT COUNT("posts"."id") FROM "posts"
# GROUP BY "posts"."author_id"
# HAVING (COUNT("posts"."id") > 5)
```

##### Functions

```ruby
Post.selecting { coalesce(author_id, 5).as('author_id_with_default') }
# SELECT coalesce("posts"."author_id", 5) AS author_id_with_default FROM "posts"
```

##### Subqueries

```ruby
Post.joins(:author).where.has {
  author.id.in Author.select(:id).where(name: 'Ray')
}
# SELECT "posts".* FROM "posts"
# INNER JOIN "authors" ON "authors"."id" = "posts"."author_id"
# WHERE "authors"."id" IN (
#   SELECT "authors"."id" FROM "authors"
#   WHERE "authors"."name" = 'Ray'
# )
```

##### Custom SQL Operators

```ruby
authors = Author.selecting { name.op('||', quoted('-dizzle')).as('swag') }
# SELECT "authors"."name" || '-dizzle' AS swag FROM "authors"

authors.first.swag #=> 'Ray Zane-dizzle'
```

## Sifters

Sifters are like little snippets of conditions that take parameters.

```ruby
class Post < ActiveRecord::Base
  sifter :funny do |str|
    title == 'rabies'
  end
end

class Author < ActiveRecord::Base
  sifter :name_contains do |string|
    name =~ "%#{string}%"
  end
end

Post.joins(:author).where.has {
  sift(:funny) | author.sift(:name_contains, 'blergh')
}
# SELECT "posts".* FROM "posts"
# INNER JOIN "authors" ON "authors"."id" = "posts"."author_id"
# WHERE ("posts"."title" = 'rabies' OR "authors"."name" LIKE '%blergh%')
```

## What's what?

The following methods give you access to BabySqueel's DSL:

| BabySqueel    | ActiveRecord Equivalent |
|---------------|-------------------------|
| `selecting`   | `select`                |
| `ordering`    | `order`                 |
| `joining`     | `joins`                 |
| `grouping`    | `group`                 |
| `where.has`   | `where`                 |
| `when_having` | `having`                |

## Development

1. Pick an ActiveRecord version to develop against, then export it: `export AR=4.2.6`.
2. Run `bin/setup` to install dependencies.
3. Run `rake` to run the specs.

You can also run `bin/console` to open up a prompt where you'll have access to some models to experiment with.

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/rzane/baby_squeel.


## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).
