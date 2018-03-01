# Domino [![Build Status](https://travis-ci.org/ngauthier/domino.png?branch=master)](https://travis-ci.org/ngauthier/domino)

View abstraction for integration testing

## Usage

To create a basic Domino class, inherit from Domino and
define a selector and attributes:

```ruby
module Dom
  class Post < Domino
    selector '#posts .post'
    attribute :title # selector defaults to .title
    attribute :author_name # selector defaults to .author-name
    attribute :body, '.post-body' # example of selector override

    # pass a block if you want to convert the value
    attribute :comments do |text|
      text.to_i
    end

    attribute :posted_at do |text|
      Date.parse(text)
    end

    # Combination selector is a boolean that combines with the root node. Boolean.
    # You can find a post that also has the .active class.
    attribute :active, "&.active"


    # Combination selector for a data attribute. Finds the value of the data attribute
    # on the root node. You can convert it with a callback like any other attribute.
    attribute :rating, "&[data-rating]", &:to_i

    # Combination selector for a data attribute with no value set. Must convert to
    # boolean in the callback.
    attribute :blah, "&[data-blah]" do |a|
      !a.nil?
    end
  end
end
```

Now in your integration test you can use some of Domino's methods:

```ruby
assert_equal 4, Dom::Post.count
refute_nil Dom::Post.find_by_title('First Post')

# Multiple attributes, returns first match if any
refute_nil Dom::Post.find_by(title: 'First Post', author: 'Jane Doe')

# Multiple attributes with exception if no match is found
refute_nil Dom::Post.find_by!(title: 'First Post', author: 'Jane Doe')

# Multiple attributes, returns all matches if any
assert_equal ["12/06/2014", "12/01/2014"], Dom::Post.where(author: 'Jane Doe').map(&:posted_on)
```

What makes it really powerful is defining scoped actions:

```ruby
module Dom
  class Post < Domino
    def delete
      within(id) { click_button 'Delete' }
    end
  end
end

refute_nil Dom::Post.find_by_title('First Post')
Dom::Post.find_by_title('First Post').delete
assert_nil Dom::Post.find_by_title('First Post')
```

## Domino::Form

Domino makes it easy to model your forms for testing with `Domino::Form`.
To create a basic form, simply inherit from `Domino::Form` and define a
selector, a key (optional), and a set of fields.

```ruby
module Dom
  class PersonForm < Domino::Form
    selector 'form.person'

    # For forms with names like `person[age]`, no need to define the
    # locator on each field. Define a key to automatically generate
    # locators based on the field name.
    key 'person'

    # Define a custom selector to click to submit the form
    submit_with "input[type='submit']" # this is the default

    # locate field by label
    field :first_name, 'First Name'

    # locate field by automatically generated name (uses key, person[last_name])
    field :last_name

    # locate field by fully qualified name
    field :biography, 'person[bio]'

    # locate select field by label, acts as select
    field :favorite_color, 'Favorite Color', as: :select

    # support multiple selects for setting and reading value as an array
    field :allergies, as: :select, multiple: true

    # locate by id, convert value via callback
    field :age, 'person_age', &:to_i

    # use a custom field type for unusual or composite fields
    field :vehicles, '.input.vehicles', as: CheckBoxesField

    # locate a field with a name that doesn't use the key
    field :is_human, 'is_human', as: :boolean
  end
end
```

In the above example, you can define a field to get a reader and writer
method for the field. A form will also provide a mass-assignment writer
and a save method to submit the form.

```ruby
person = Dom::PersonForm.find!
person.age            #=> 25
person.vehicles       #=> ["Car", "Bike"]
person.is_human       #=> true
person.favorite_color #=> Blue

person.age = 35
person.age #=> 35

person.set(vehicles: ["Car", "Van"], first_name: "Jessica", last_name: "Jones")
person.attributes #=> { first_name: "Jessica", last_name: "Jones", biography: "", favorite_color: "Blue", age: 35, vehicles: ["Car", "Van"], is_human: true }
```

`Domino::Form` provides basic field types for text inputs and textareas,
single-selects, and boolean fields. You can create custom field types
for more complex form inputs by subclassing `Domino::Form::Field` and
overriding the `read` and `write` methods. For example, if you have a
collection of check boxes, this might suit your needs:

```ruby
class CheckBoxesField < Domino::Form::Field
  def read(node)
    node.find(locator).all('input[type=checkbox]').select(&:checked?).map(&:value)
  end

  def write(node, value)
    value = Array(value)
    node.find(locator).all('input[type=checkbox]').each do |box|
      box.set(value.include?(box.value))
    end
  end
end
```

Provide your custom class using the `:as` option when defining your field,
as shown in the example above.

## Integration with capybara

Domino uses capybara internally to search html for nodes and
attributes. If you need to do something special, you can have direct
access to the capybara node.

```ruby
module Dom
  class Account < Domino
    selector "#accounts li"
    # Returns this node text
    def text
      node.text
    end
  end
end
```

For more information about using Capybara nodes, check [Capybara Documentation](https://github.com/jnicklas/capybara/blob/master/README.rdoc).

## Dealing with Asynchronous Behavior

When working with Capybara drivers that support JavaScript, it may be
necessary to wait for elements to appear. Note that the following code
simply collects all `Account` dominos currently on the page and
returns the first:

```ruby
Dom::Account.first # returns nil if account is displayed asynchronously
```

When you are waiting for a unique domino to appear, you can instead
use the `find!` method:

```ruby
Dom::Account.find! # waits for matching element to appear
```

If no matching element appears, Capybara will raise an error telling
you about the expected selector.  Depending on the
[`Capybara.match` option](https://github.com/jnicklas/capybara#strategy),
this will also raise an error if the selector matches multiple nodes.

## Integration with Cucumber

Add a features/support/dominos.rb file, in which you define your dominos.

Use them in your steps.

## Integration with Test::Unit

Include "domino" in your Gemfile if using bundler, or simply

```ruby
require 'domino'
```

If you're not using Bundler.

Now, define your Dominos anywhere you want. The easiest place to start is
in your test\_helper.rb (doesn't have to be inside a Rails test class).

## Example

Check out [Domino Example](http://github.com/ngauthier/domino_example) for an
example of using Test::Unit and Cucumber with Domino.

## Copyright

Copyright (c) 2011 Nick Gauthier, released under the MIT license
