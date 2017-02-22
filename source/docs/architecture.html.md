---
title: Architecture
---
# Hyperloop Architecture

Hyperloop's primary goal is to allow you to enjoy quickly building modern interactive web applications.

Hyperloop includes full access to the **entire Rails ecosystem** and **universe of front-end JavaScript libraries** like React and jQuery - all using one great language - Ruby.

Hyperloop lets you write code that is directed toward solving the user's needs in the most straightforward manner, without redundant code, unnecessary APIs, or artificial separation between client and server.

Hyperloop is an Isomorphic Web Application Framework consisting of Components, Operations, Models, Policies, and Stores. This structure is analogous to and replaces the older MVC architecture, but with a more logical and finer grained division of labor.  

![](/images/HSCOMP-schematic.png)

+ **Components** describe how the UI will display the *current* application state and how it will handle user actions.  Using React Components automatically rerender parts of the display as state changes due to local or remote activities.
+ **Operations** encapsulate activities. In an MVC architecture operations end up either in controllers, models or some other secondary construct such as service objects, helpers, or concerns. Here they are first class objects.
+ **Models** can now focus on one thing and that is the structure of the data as it is persisted.  Any business logic is moved to Operations.
+ **Policies** keep authorization logic out of models, and operations, and also allows the isomorphic transport mechanism to know what and when to communicate between client and server.
+ **Stores** hold application state. Stores are Ruby classes that keep the dynamic parts of the state in special state variables.

The only remaining function of controllers - acting as the HTTP end point - is automatically handled by the Hyperloop framework.  Both Models and Operations will automatically *work across the wire* using the Policies to determine access rights, and reactive Components to update the display as data changes.

This overview will take you through all the main Hyperloop architectural layers.

## Components

You build your UI using React Components described as Ruby classes.  Within your Components, you can display other components, change state, access models, or communicate with third party APIs.  Typically you will want to use Operations to encapsulate these activities.  Here is a simple example using our `AddBookToBasket` operation.

```ruby
class BookList < React::Component::Base
  # Display each book in our catalog unless it's already in the cart basket.
  # When the user clicks on a book, add it to the Basket.
  render(UL) do
    Book.all.each do |book|
      LI { "Add #{book.name}" }.on(:click) do
        AddBookToBasket(book: book)
      end unless acting_user.basket.include? book
    end
  end
end
```

Notice how our component directly scopes the `Book` model and reads the `name` attribute.  Models are dynamically synchronized to all connected and authorized clients using ActionCable, pusher.com or polling.  The synchronization is completely automatic and magical to behold.

## Operations (coming soon)

Hyperloop *recommends* that only scopes, relations, and validations are described in model classes. All business logic can be encapsulated in reusable *isomorphic* Operations that do not complicate your Models or Components.  Each Operation is a self-contained piece of logic that does one simple thing.  

```ruby
class AddToActingUsersWatchList < Hyperloop::Operation
  # Add a book to the current acting_user's watch list, and
  # send an initial email about the book.

  param :book, type: Book
  # Operations have access to the current 'acting_user'
  # so we do not need to pass it as a parameter.

  def execute
    return if acting_user.watch_list.include? params.book
    WatchListMailer.new_book_email\
      WatchList.create(user: acting_user,  book: params.book)
  end
end
```

Pretty simple.  Typically code like this might be found in a controller which makes it hard to reuse or in a model which makes maintenance difficult when business logic changes.  Placing it in its own `Operation` makes it easy to maintain, reuse and test.

Of course, Operations can invoke other Operations:

```ruby
class AddBookToBasket < Hyperloop::Operation
  # Add a book to the basket and add to users watchlist
  param :book, type: Book

  def execute
    acting_user.basket << book
    AddToActingUsersWatchList(book: params.book)
  end
end
```

## Models

Hyperloop uses Rails ActiveRecord for data persistence.  This allows easy integration with existing Rails apps.  Hyperloop Models are implemented in the HyperMesh Gem.

HyperMesh gives you full access to the ActiveRecord models on the client or the server which means we can use the models directly within our Components without needing the abstraction of an API:

```ruby
class BookList < React::Component::Base
  # Display each book in the catalog
  render(UL) do
    Book.in_catalog.each do |book|
      LI { book.name }
    end
  end
end
```

Changes made to Models on a client or server are **automatically synchronized** to all other authorized connected clients using ActionCable, pusher.com or polling. The synchronization is completely automatic and magical to behold.


## Policies

While communication between the client and server is automatic it does need to be authorized.  This is accomplished by *regulations* which can be grouped into pundit style Policy classes.  This allows your access rules to be described separately from your Models and Operations.

```ruby
class BookPolicy
  regulate_broadcast do |policy|
    # allow the entire application to see all book attributes
    # except the 'unit_cost'.
    policy.send_all_but(:unit_cost).to(Application)
  end
  # but only acting_user's who are admins can make changes to Books
  allow_change(on: [:create, :update]) { acting_user.admin? }
end

class OperationPolicy
  # We need AddToActingUsersWatchList to execute on the server but
  # it can be invoked from the client if there is a logged in user.
  AddToActingUsersWatchList.execute_on_server { acting_user }
end
```

## Stores

Stores are where the state of your Application lives.

Anything but a completely static web page will have dynamic states that change because of user inputs, the passage of time, or other external events.

**Stores are Ruby classes that keep the dynamic parts of the state in special state variables**

For example here is Store that keeps track of time at a given location:

```ruby
class WorldClock < HyperStore
  # Keep track of the time at multiple locations
  attr_reader :name
  attr_reader :lattitude
  attr_reader :longitude
  attr_reader :time_zone_offset

  def current_time
    WorldClock.gmt+time_zone_offset
  end

  def initialize(name, lattitude, longitude, time_zone_offset)
    @name, @lattitude, @longitude, @time_zone_offset =
      [name, lattitude, longitude, time_zone_offset]
  end

  def WorldClock.gmt
    unless state.gmt
      every(1) { state.gmt! = Time.now.gmt }
      state.gmt! = Time.now
    end
    state.gmt
  end
end  
```  

Now we can create a clock and post the time to the console every minute like this:

```ruby
new_york = WorldClock.new('New York', 40.7128, -74.0059, 5.hours)
every(1.minute) { puts new_york.current_time }  
```

But because it is a Reactive `Store` we can also say this:

```ruby
# assume we have a div with id='new-york' some place in our code
Element['div#new-york'].render do
  "The time in #{new_york.name} is #{new_york.current_time}"
end
```

This will automatically rerender the contents of the 'new-york' DIV **whenever the store changes**


## Benefits of COMPS Architecture

+ The COMP architecture encapsulates functionality for clean, predictable, testable code
+ One language - Ruby everywhere - reduces complexity and lets developers build solutions quickly
+ Encourages client-side execution for distributed processing and a rich interactive user experience
+ Full power of Rails, React and the entire JavaScript universe
+ Transparent, automatic and secure client-server communication built into Models and Operations