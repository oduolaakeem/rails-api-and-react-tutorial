
# Data modeling

### Setup

- `git clone` this repo (`git clone git@github.com:blueapron/Rails-api-and-react-tutorial.git`)
and switch to branch: `git checkout 2_data_setup`

The state of the application is a newly created Rails API

### What are we going to build?

Say our product manager and tech lead have determined that we need to build a new
Rails API that can store information about a user's orders and an order's
products. These are the requirements:

- Create a new user
- Create new orders for a user
- See a user's orders
- See an order's products

### General overview

1. Plan our data model (i.e., how our data is going to be stored and
  associated with each other). This will involve creating database migrations
  and Rails models to interact with the database. This is the 'M' in MVC.
2. Create Routes and Controllers to access the data via HTTP. We will create
an endpoint to create a user, create a new order for a user, see a user's orders,
and see an order's products. This is the 'C' in MVC.
3. Create serializers to serialize the data into JSON so that it can be
transmitted over the network to a consuming client side application. This enables
the 'C' in MVC.
4. We have no 'V' or views in the traditional MVC, at least not on the Rails
server side of things. Since we're building a Rails API, our application does
not need to render Views, just the JSON for a client-side application to use.

### Data modeling, ERDs, and You!

So, what will our data model look like? Let's conceptualize it first in an
[ERD](https://www.techopedia.com/definition/1200/entity-relationship-diagram-erd)
before actually creating tables in our database.

We first need to create a `User` and let's say this user has a `first_name` and
`last_name`. Our `User` table will look like this:

![User Table](user_table.png)

But our user also needs to have many orders. Let's adapt our schema concept:

![User And Orders](user_and_orders.png)

So, we conceptualized the relationship between users and orders as follows: A
user can have many orders and an order can belong to a single user. The `user_id`
field on the orders table is a [foreign key](https://en.wikipedia.org/wiki/Foreign_key)
which corresponds to a primary key on the `User` table. If `user_id` on the Orders
table is equal to 1, that means that the order is associated to a user in the
`User` table with a primary key of 1.

Finally, let's add the relationship between orders and products:

![Full ERD](full_erd.png)

We created a joined table between the `Order` and `Product` table. Why? That's
the only way we can express a many-to-many association with another model in a
relational database. We want an order to have many products and for products
to have many orders (i.e., a ramen Product can be in many orders). Since the
association is created using a foreign key, we need another table to store the
foreign keys of both tables. If we simply put an `order_id` on the `Product`
table, it wouldn't be possible for a Product (represented by a row of data in the
`Product` table) to be associated with several orders; it could only be associated
with the single order referenced by the row's `order_id` foreign key.


### Setup Rails. Add a couple gems:

Before we create our database tables and Rails models, let's configure Rails to
use a few external libraries.

- [rspec-rails](https://github.com/rspec/rspec-rails) - Testing framework.
- [factory_girl_rails](https://github.com/thoughtbot/factory_girl_rails) - A fixtures
replacement with a more straightforward syntax. You'll see.
- [database_cleaner](https://github.com/DatabaseCleaner/database_cleaner) - You
guessed it! It literally cleans our test database to ensure a clean state in
each test suite.
- [faker](https://github.com/stympy/faker) - A library for generating fake data.
We'll use this to generate test data.
- [pry-rails](https://github.com/rweng/pry-rails) - Debugger. We're going to replace
`byebug` which comes standard with new Rails applications.

We're going to add these gems to our folder's [`Gemfile`](http://tosbourn.com/what-is-the-gemfile/)

Add `factory_girl_rails`, `pry-rails`, `rspec-rails, '~> 3.5'` and `faker` to the `group :development, :test` section
of your Gemfile while adding `database_cleaner` to the `:test` group
section.

We'll also add the `ActiveModelSerializer` and `JSON-api` gems so that we can
serialize our model data into JSON later on. We don't need them right now when
generating models, but we'll add these gems to the top-level of our `Gemfile`
for later use.

So our `Gemfile` will look like this:

```rails
gem 'rails', '~> 5.0.3'
gem 'pg', '~> 0.18'
gem 'puma', '~> 3.0'
gem 'active_model_serializers', '~> 0.10.0'
gem 'jsonapi-resources'

group :development, :test do
  gem 'pry-rails'
  gem 'factory_girl_rails'
  gem 'faker'
  gem 'rspec-rails', '~> 3.5'
end

group :test do
   gem 'database_cleaner'
end

group :development do
  gem 'listen', '~> 3.0.5'
  # Spring speeds up development by keeping your application running in the background. Read more: https://github.com/rails/spring
  gem 'spring'
  gem 'spring-watcher-listen', '~> 2.0.0'
end

# Windows does not include zoneinfo files, so bundle the tzinfo-data gem
gem 'tzinfo-data', platforms: [:mingw, :mswin, :x64_mingw, :jruby]
```

Run `bundle install` to install the gems.

A few more commands to run to configure our Rails application. Run these in terminal:

```ruby
rails generate rspec:install
```

Look in terminal. By running the above command, we added the following files to
our Rails app which are used for configuration our tests.

```rails
.rspec
spec
spec/spec_helper.rb
spec/rails_helper.rb
```

A lot of these gems and configuration hook into the Rails generators which
we will use to generate our models and migrations in the next section.

### Create database tables and Rails models.
Now that we have our ERD, let's create our actual tables. We'll be using the
build-in Rails generators. Rails comes with a number of scripts called generators
that are designed to make your development life easier by creating everything
that's necessary to start working on a particular task. One of these is the `new`
generator that we used already when creating the new Rails application. It created
a bunch of files that saved us from having to create them by hand:

![new generator](rails_new.png)

Rails provides a generator for creating models, which most Rails developers tend
to use when creating new models.

```bash
rails generate model User name:string
```

And we can see our terminal output:

```bash
Running via Spring preloader in process 34457
      invoke  active_record
      create    db/migrate/20170615205116_create_users.rb
      create    app/models/user.rb
      invoke    rspec
      create      spec/models/user_spec.rb
      invoke      factory_girl
      create        spec/factories/users.rb
```

As a result of our gems and configuration, `factory_girl` files and an `rspec`
spec (test) were generated along with our model (code that sits on top of and
accesses the database) and a migration (code that will generate our database table).

Let's inspect this newly created migration file. You should see a migration file inside the `db/migrate` folder.
It should look like this:

```rails
class CreateUsers < ActiveRecord::Migration[5.0]
  def change
    create_table :users do |t|
      t.string :name

      t.timestamps
    end
  end
end
```

This migration file will be used to create an actual table in our database.

Even though this migration file does interact with your ORM to create a database
table, it's written in plain Ruby. It calls a class method called `create_table`
which creates a table called `users`. The method also takes a block which gets
yielded to by the `create_table` method which creates a column called `name`.
The table will also create columns called `create_at` and `updated_at` which will
be created via the `timestamps` method call on `t` (some kind of table object that
gets passed via the yield method). More on yield [here](https://rubymonk.com/learning/books/4-ruby-primer-ascent/chapters/18-blocks/lessons/54-yield)

Now let's create an actual table in our database using this migration file.

First add this to your `application.rb` inside the `Application` class:
```ruby
config.active_record.schema_format = :sql
```

Now run:

```rails
rake db:create
bundle exec rake db:migrate
```

The above command will run all of the unrun migrations in your migrations folder.
Every time you run a migration, a migration version (written in the beginning
of your migration file) will be added to your database's `schema_migrations`
table. If the version number of a migration is not inside this table, the
migration will run. If it is inside this table, it will not run. This is to
ensure that no conflicting or duplicate migrations are run.

Before checking our actual database, let's take a quick look at our `structure.sql`
file which is a representation of the current state of our database. It should include
this sql:

```sql
CREATE TABLE users (
    id integer NOT NULL,
    name character varying,
    created_at timestamp without time zone NOT NULL,
    updated_at timestamp without time zone NOT NULL
);
...
CREATE SEQUENCE users_id_seq
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1;
...
ALTER TABLE ONLY users ALTER COLUMN id SET DEFAULT nextval('users_id_seq'::regclass);
```

(If you're unfamiliar with sql, I recommend this [site](http://www.sqlcourse.com/). In
short, the sql statements are creating a users table, creating a sequence function
and setting the id of the users table to use the sequence function via the
[`nextval`](https://www.postgresql.org/docs/9.1/static/functions-sequence.html) function.)

This file is a good way to quickly check what kind of attributes our `User` model
actually has. More on structure.sql and schema files [here](http://guides.rubyonrails.org/v3.2.8/migrations.html#what-are-schema-files-for)

Now let's actually look at the state of our database. If you look up our database name inside our `database.yml` file, it should be
named similarly to the name of our app when we ran `rails new <some name>`. My development
database is named `rails_5_ba_tutorial_development` and you see it under the
development key of the `database.yml` file.

Now if we check the state of your actual database, we will see a users table
and a version added to our `schema_migrations` table:

Run these commands in terminal:

```bash
$ psql rails_5_ba_tutorial_development
```

Inside the psql session, type this:

```bash
\d users
```

and you should see:

```bash
rails_5_ba_tutorial_development=# \d users
                                     Table "public.users"
   Column   |            Type             |                     Modifiers
------------+-----------------------------+----------------------------------------------------
 id         | integer                     | not null default nextval('users_id_seq'::regclass)
 name       | character varying           |
 created_at | timestamp without time zone | not null
 updated_at | timestamp without time zone | not null
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
```

That's our users table. Notice the balance tree index on the primary key (id). We'll
explain this a bit more later.

Also run this command while inside 'psql':


```ruby
select * from schema_migrations
```

Your output should look something like this:

```bash
    version
----------------
 20170615205116
(1 row)
```

Your version number will be different since it's based on timestamps... but you
should see 1 row. That row represents the migrations that you just ran.

You can exit your database by typing `\q`.

If you want to read the Rails docs about migrations, go [here](http://edgeguides.rubyonrails.org/active_record_migrations.html)

Anyway, let's create the rest of our tables:

```bash
rails generate model Order date:datetime user:references:index
rails generate model Product cost:float name:string
rails generate model OrderProduct order:references product:references
rake db:migrate
```

*****What is `references` you ask?*****
We're simply saying that the table being created has a reference to another table.
For example, the `Order` table references the `User` table. How does it achieve this? By creating
a foreign key to the `users` table.

Let's look at our migrations for the orders table:

```
class CreateOrders < ActiveRecord::Migration[5.0]
  def change
    create_table :orders do |t|
      t.datetime :date
      t.references :user, foreign_key: true

      t.timestamps
    end
  end
end
```

By the way, using the built-in generator commands isn't necessary, but they do make
our lives easier. If you just ran `rails g migration CreateOrderTable` and filled in
the migration file with the `def change` method and then ran `rake db:migrate`,
it would have created this same table in the actual database.

Let's look at our `structure.sql` (again, a SQL representation of our actual database):

```
CREATE TABLE orders (
    id integer NOT NULL,
    date timestamp without time zone,
    user_id integer,
    created_at timestamp without time zone NOT NULL,
    updated_at timestamp without time zone NOT NULL
);
...
(somewhere lower in the `structure.sql` file)

CREATE INDEX index_orders_on_user_id ON orders USING btree (user_id);
```

The `CREATE INDEX` line enhance database performance by creating a balanced
tree (not a binary search tree, but similar) where data can be accessed using
the same number of steps because tree depth increases slower than the tree nodes.
The index is named `index_orders_on_user_id` and we're creating the index on the
`user_id` since we're assuming that lookups for the associated user will frequently
occur and so we index this column in order to speed up lookup times. Remember that
the users table also has a balanced tree index on the primary key (id). This makes
sense since we're constantly looking up rows in a table via the primary key. If we
were, for some reason, doing lookups on another kind of column (perhaps a `uuid`),
then a balanced tree on this column would also make sense.

The products table is pretty straightforward. It has 2 columns: `name` and `cost`.
The type of the cost column is a float.

You might ask: how are products and orders connected?

****Our joined table*****

So there is a joined table between orders and products that stores the association
between the tables. Before we continue, let's add these lines to the `order.rb` file.

Our class should now look like this:

```ruby
class Order < ApplicationRecord
  belongs_to :user
  has_many :order_products
  has_many :products, through: :order_products
end
```

We're telling Rails that an order is associated to products through the `order_products` table.
The `has_many` class method gives an order instance new methods. Now `order_products`
and `products` can be called on an instance of `order.` (If this doesn't make sense,
review what a class method and instance method are in Ruby).

Let's take a quick look at our joined table class (auto-generated when we ran the generator
command):

```ruby
class OrderProduct < ApplicationRecord
  belongs_to :order
  belongs_to :product
end
```

As you can see, two `belongs_to` method calls are provided which signify that an
instance of an `order_product` belongs to both an order and a product. The table itself
consists of only two foreign key columns. In our `structure.sql` :


```sql
CREATE TABLE order_products (
    id integer NOT NULL,
    order_id integer,
    product_id integer,
    created_at timestamp without time zone NOT NULL,
    updated_at timestamp without time zone NOT NULL
);
```

Let's actually use our joined table. Run the following commands in terminal:

```ruby
rails console
```

The console command lets you interact with your Rails application from the command line.
On the underside, rails console uses IRB, so if you've ever used it for Ruby, you'll be
right at home. This is useful for testing out quick ideas with code and
changing data server-side without touching the website. Then run:

```ruby
user = User.create #create a user so that we can create an order which has a foreign key
# now let's create two orders
Order.create(date: Date.new, user: user)
order2 = Order.create(date: Date.new, user: user)
order2.products.create # We will create a product off only the second order
OrderProduct.first # Let's find our association between the second order and its product
```

After typing in the last command, you should see:

```sql
OrderProduct Load (0.4ms)  SELECT  "order_products".* FROM "order_products" ORDER BY "order_products"."id" ASC LIMIT $1  [["LIMIT", 1]]
=> #<OrderProduct:0x007fb7e2bbda40 id: 1, order_id: 2, product_id: 1, created_at: Mon, 19 Jun 2017 14:08:49 UTC +00:00, updated_at: Mon, 19 Jun 2017 14:08:49 UTC +00:00>
```

The SQL command that fired does the following:
"Select every row (*) from the order_products table, order the returned rows by
id in ascending order, take the first one (it would be the lowest id)"

As you can see, the returned order product has an `order_id` of 2, and a `product_id` of 1 which
is the stored association between the second order and its product.

The joined table isn't only a place to store associations. If there was an attribute that
needed to be stored on the joined table that didn't belong on either the `orders`
or `products` table, we can store it on the joined table. Imagine that an order's
products had to be tracked as they were packed. Since this attribute (let's called it
`packed_or_not`) tracks whether an individual product has been packed in the order...
it doesn't really belong on the `orders` table (how could an attribute on the entire
order keep track of whether an individual product has been packed or not?) or on the
universal `products` table (if we set the attribute on the `products` table, that would
mean that the specific product has been packed universally, which isn't what we
want either.)



### Recap

What did we learn?

- How to create an ERD at the beginning of building a new feature. (the site used was [this](http://ondras.zarovi.cz/sql/demo/))
- How to generate a new Rails API-only application and configure it to use third-party gems.
- How to use Rails generators to create tables and models. We also learned a bit about
SQL tables and foreign keys.
- How to read the structure.sql file and basic SQL.
- How to interpret a joined table.

Next, we'll flesh out our tables with some validations and write some basic tests in RSpec.
