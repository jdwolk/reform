# Reform

Decouple your models from forms. Reform gives you a form object with validations and nested setup of models. It is completely framework-agnostic and doesn't care about your database.

Although reform can be used in any Ruby framework, it comes with [Rails support](#rails-integration), works with [simple_form and other form gems](#formbuilder-support), allows nesting forms to implement [has_one](#nesting-forms-1-1-relations) and [has_many](#nesting-forms-1-n-relations) relationships, can [compose a form](#compositions) from multiple objects and gives you [coercion](#coercion).

## Installation

Add this line to your Gemfile:

```ruby
gem 'reform'
```

## Nomenclatura

Reform comes with two base classes.

* `Form` is what made you come here - it gives you a form class to handle all validations, wrap models, allow rendering with Rails form helpers, simplifies saving of models, and more.
* `Contract` gives you a sub-set of `Form`: [this class](#contracts) is meant for API validation where already populated models get validated without having to maintain validations in the model classes.


## Defining Forms

You're working at a famous record label and your job is archiving all the songs, albums and artists. You start with a form to populate your `songs` table.

```ruby
class SongForm < Reform::Form
  property :title
  property :length

  validates :title,  presence: true
  validates :length, numericality: true
end
```

To add fields to the form use the `::property` method. Also, validations no longer go into the model but sit in the form.


## The API

Forms have a ridiculously simple API with only a handful of public methods.

1. `#initialize` always requires a model that the form represents.
2. `#validate(params)` updates the form's fields with the input data (only the form, _not_ the model) and then runs all validations. The return value is the boolean result of the validations.
3. `#errors` returns validation messages in a classy ActiveModel style.
4. `#sync` writes form data back to the model. This will only use setter methods on the model(s).
5. `#save` (optional) will call `#save` on the model and nested models. Note that this implies a `#sync` call.

In addition to the main API, forms expose accessors to the defined properties. This is used for rendering or manual operations.


## Setup

In your controller you'd create a form instance and pass in the models you want to work on.

```ruby
class SongsController
  def new
    @form = SongForm.new(Song.new)
  end
```

You can also setup the form for editing existing items.

```ruby
class SongsController
  def edit
    @form = SongForm.new(Song.find(1))
  end
```

Reform will read property values from the model in setup. Given the following form class.

```ruby
class SongForm < Reform::Form
  property :title
```

Internally, this form will call `song.title` to populate the title field.

If you, for whatever reasons, want to use a different public name, use `:as`.

```ruby
class SongForm < Reform::Form
  property :name, as: :title
```

This will still call `song.title` but expose the attribute as `name`.


## Rendering Forms

Your `@form` is now ready to be rendered, either do it yourself or use something like Rails' `#form_for`, `simple_form` or `formtastic`.

```haml
= form_for @form do |f|

  = f.input :name
  = f.input :title
```

Nested forms and collections can be easily rendered with `fields_for`, etc. Just use Reform as if it would be an ActiveModel instance in the view layer.


## Validation

After a form submission, you want to validate the input.

```ruby
class SongsController
  def create
    @form = SongForm.new(Song.new)

    #=> params: {song: {title: "Rio", length: "366"}}

    if @form.validate(params[:song])
```

The `#validate` method first updates the values of the form - the underlying model is still treated as immutuable and *remains unchanged*. It then runs all validations you provided in the form.

It's the only entry point for updating the form. This is per design, as separating writing and validation doesn't make sense for a form.

This allows rendering the form after `validate` with the data that has been submitted. However, don't get confused, the model's values are still the old, original values and are only changed after a `#save` or `#sync` operation.


## Syncing Back

After validation, you have two choices: either call `#save` and let Reform sort out the rest. Or call `#sync`, which will write all the properties back to the model. In a nested form, this works recursively, of course.

It's then up to you what to do with the updated models - they're still unsaved.


## Saving Forms

The easiest way to save the data is to call `#save` on the form.

```ruby
    @form.save  #=> populates song with incoming data
                #   by calling @form.song.title= and @form.song.length=.
```

This will sync the data to the model and then call `song.save`.

Sometimes, you need to do stuff manually.


## Saving Forms Manually

Calling `#save` with a block doesn't do anything but providing you a nested hash with all the validated input. This allows you to implement the saving yourself.

The block parameter is a nested hash of the form input.

```ruby
  @form.save do |hash|
    hash      #=> {title: "Rio", length: "366"}

    Song.create(hash)
  end
```

You can always access the form's model. This is helpful when you were using populators to set up objects when validating.

```ruby
  @form.save do |nested|
    album = @form.album.model

    album.update_attributes(nested[:album])
  end
```

Note that you can call `#sync` and _then_ call `#save { |hsh| }` to save models yourself.


## Contracts

Contracts give you a sub-set of the `Form` API.

1. `#initialize` accepts an already populated model.
2. `#validate` will run defined validations (without accepting a params hash as in `Form`).

Contracts can be used to completely remove validation logic from your model classes. Validation should happen in a separate layer - a `Contract`.

### Defining Contracts

A contract looks like a form.

```ruby
class AlbumContract < Reform::Contract
  property :title
  validates :title, length: {minimum: 9}

  collection :songs do
    property :title
    validates :title, presence: true
  end
```

It defines the validations and the object graph to be inspected.

In future versions and with the upcoming [Trailblazer framework](https://github.com/apotonick/trailblazer), contracts can be inherited from forms, representers, and cells, and vice-versa. Actually this already works with representer inheritance - let me know if you need help.

### Using Contracts

Applying a contract is simple, all you need is a populated object (e.g. an album after `#update_attributes`).

```ruby
album.assign_attributes(..)

contract = AlbumContract.new(album)

if contract.validate
  album.save
else
  raise contract.errors.messages.inspect
end
```

Contracts help you to make your data layer a dumb persistance tier. My [upcoming book discusses that in detail](http://nicksda.apotomo.de).


## Nesting Forms: 1-1 Relations

Songs have artists to compose them. Let's say your `Song` model would implement that as follows.

```ruby
class Song < ActiveRecord::Base
  has_one :artist
end
```

The edit form should allow changing data for artist and song.

```ruby
class SongForm < Reform::Form
  property :title
  property :length

  property :artist do
    property :name

    validates :name, presence: true
  end

  #validates :title, ...
end
```

See how simple nesting forms is? By passing a block to `::property` you can define another form nested into your main form.


### has_one: Setup

This setup's only requirement is having a working `Song#artist` reader.

```ruby
class SongsController
  def edit
    song = Song.find(1)
    song.artist #=> <0x999#Artist title="Duran Duran">

    @form = SongForm.new(song)
  end
```

### has_one: Rendering

When rendering this form you could use the form's accessors manually.

```haml
= text_field :title,         @form.title
= text_field "artist[name]", @form.artist.name
```

Or use something like `#fields_for` in a Rails environment.

```haml
= form_for @form |f|
  = f.text_field :title
  = f.text_field :length

  = f.fields_for :artist do |a|
    = a.text_field :name
```

### has_one: Processing

The block form of `#save` would give you the following data.

```ruby
@form.save do |nested|

  nested #=> {title:  "Hungry Like The Wolf",
         #    artist: {name: "Duran Duran"}}
end
```

Supposed you use reform's automatic save without a block, the following assignments would be made.

```ruby
form.song.title       = "Hungry Like The Wolf"
form.song.artist.name = "Duran Duran"
form.song.save
```

## Nesting Forms: 1-n Relations

Reform also gives you nested collections.

Let's have Albums with songs!

```ruby
class Album < ActiveRecord::Base
  has_many :songs
end
```

The form might look like this.

```ruby
class AlbumForm < Reform::Form
  property :title

  collection :songs do
    property :title

    validates :title, presence: true
  end
end
```

This basically works like a nested `property` that iterates over a collection of songs.

### has_many: Rendering

Reform will expose the collection using the `#songs` method.

```haml
= text_field :title,         @form.title
= text_field "songs[0][title]", @form.songs[0].title
```

However, `#fields_for` works just fine, again.

```haml
= form_for @form |f|
  = f.text_field :title

  = f.fields_for :songs do |s|
    = s.text_field :title
```

### has_many: Processing

The block form of `#save` will expose the data structures already discussed.

```ruby
@form.save do |nested|
  
  nested #=> {title: "Rio"
         #   songs: [{title: "Hungry Like The Wolf"},
         #          {title: "Last Chance On The Stairways"}]
end
```


## Nesting Configuration

### Turning Off Autosave

You can assign Reform to _not_ call `save` on a particular nested model (per default, it is called automatically on all nested models).

```ruby
class AlbumForm < Reform::Form
  # ...

  collection :songs, save: false do
    # ..
  end
```

The `:save` options set to false won't save models.


### Populating Forms For Validation

With a complex nested setup it can sometimes be painful to setup the model object graph.

Let's assume you rendered the following form.

```ruby
@form = AlbumForm.new(Album.new(songs: [Song.new, Song.new]))
```

This will render two nested forms to create new songs.

When **validating**, you're supposed to setup the very same object graph, again. Reform has no way of remembering what the object setup was like a request ago.

So, the following code will fail.

```ruby
@form = AlbumForm.new(Album.new).validate(params[:album])
```

However, you can advise Reform to setup the correct objects for you.

```ruby
class AlbumForm < Reform::Form
  # ...

  collection :songs, populate_if_empty: Song do
    # ..
  end
```

This works for both `property` and `collection` and instantiates `Song` objects where they're missing when calling `#validate`.

If you want to create the objects yourself, because you're smarter than Reform, do it with a lambda.

```ruby
class AlbumForm < Reform::Form
  # ...

  collection :songs, populate_if_empty: lambda { |fragment, args| Song.new } do
    # ..
  end
```


## Compositions

Sometimes you might want to embrace two (or more) unrelated objects with a single form. While you could write a simple delegating composition yourself, reform comes with it built-in.

Say we were to edit a song and the label data the record was released from. Internally, this would imply working on the `songs` table and the `labels` table.

```ruby
class SongWithLabelForm < Reform::Form
  include Composition

  property :title, on: :song
  property :city,  on: :label

  model :song # only needed in ActiveModel context.

  validates :title, :city, presence: true
end
```

Note that reform needs to know about the owner objects of properties. You can do so by using the `on:` option.

Also, the form needs to have a main object configured. This is where ActiveModel-methods like `#persisted?` or '#id' are delegated to. Use `::model` to define the main object.


### Composition: Setup

The constructor slightly differs.

```ruby
@form = SongWithLabelForm.new(song: Song.new, label: Label.new)
```

### Composition: Rendering

After you configured your composition in the form, reform hides the fact that you're actually showing two different objects.

```haml
= form_for @form do |f|

  Song:     = f.input :title

  Label in: = f.input :city
```

### Composition: Processing

When using `#save' without a block reform will use writer methods on the different objects to push validated data to the properties.

Here's how the block parameters look like.

```ruby
@form.save do |nested|
  
  nested #=> {
         #   song:  {title: "Rio"}
         #   label: {city: "London"}
         #   }
end
```


## Forms In Modules

To maximize reusability, you can also define forms in modules and include them in other modules or classes.

```ruby
module SongsForm
  include Reform::Form::Module

  collection :songs do
    property :title
    validates :title, presence: true
  end
end
```

This can now be included into a real form.

```ruby
class AlbumForm < Reform::Form
  property :title

  include SongsForm
end
```

Note that you can also override properties [using inheritance](#inheritance) in Reform.


## Inheritance

Forms can be derived from other forms and will inherit all properties and validations.

```ruby
class AlbumForm < Reform::Form
  property :title

  collection :songs do
    property :title

    validates :title, presence: true
  end
end
```

Now, a simple inheritance can add fields.

```ruby
class CompilationForm < AlbumForm
  property :composers do
    property :name
  end
end
```

This will _add_ `composers` to the existing fields.

You can also partially override fields using `:inherit`.

```ruby
class CompilationForm < AlbumForm
  property :songs, inherit: true do
    property :band_id
    validates :band_id, presence: true
  end
end
```

Using `inherit:` here will extend the existing `songs` form with the `band_id` field. Note that this simply uses [representable's inheritance mechanism](https://github.com/apotonick/representable/#partly-overriding-properties).

## Coercion

Often you want incoming form data to be converted to a type, like timestamps. Reform uses [virtus](https://github.com/solnic/virtus) for coercion, the DSL is seamlessly integrated into Reform with the `:type` option.

### Virtus Coercion

Be sure to add `virtus` to your Gemfile.

```ruby
require 'reform/form/coercion'

class SongForm < Reform::Form
  include Coercion

  property :written_at, type: DateTime
end

form.validate("written_at" => "26 September")
```

Coercion only happens in `#validate`.

```
form.written_at #=> <DateTime "2014 September 26 00:00">
```

### Manual Coercing Values

If you need to filter values manually, you can override the setter in the form.

```ruby
class SongForm < Reform::Form
  property :title

  def title=(value)
    super sanitize(value) # value is raw form input.
  end
end
```

As with the built-in coercion, this setter is only called in `#validate`.


## Virtual Attributes

Virtual fields come in handy when there's no direct mapping to a model attribute or when you plan on displaying but not processing a value.


### Empty Fields

Often, fields like `password_confirmation` shouldn't be retrieved from the model. Reform comes with the `:empty` option for that.

```ruby
class PasswordForm < Reform::Form
  property :password
  property :password_confirmation, :empty => true
```

Here, the model won't be queried for a `password_confirmation` field when creating and rendering the form. Likewise, when saving the form, the input value is not written to the decorated model. It is only readable in validations and when saving the form.

```ruby
form.validate("password" => "123", "password_confirmation" => "321")

form.password_confirmation #=> "321"
```

The nested hash in the block-`#save` provides the same value.

```ruby
form.save do |nested|
  nested[:password_confirmation] #=> "321"
```

### Read-Only Fields

Almost identical, the `:virtual` option makes fields read-only. Say you want to show a value, but not process it after submission, this option is your friend.

```ruby
class ProfileForm < Reform::Form
  property :country, :virtual => true
```

This time reform will query the model for the value by calling `model.country`.

You want to use this to display an initial value or to further process this field with JavaScript. However, after submission, the field is no longer considered: it won't be written to the model when saving.

It is still readable in the nested hash and through the form itself.

```ruby
form.save do |nested|
  nested[:country] #=> "Australia"
```

## Validations From Models

Sometimes when you still keep validations in your models (which you shouldn't) copying them to a form might not feel right. In that case, you can let Reform automatically copy them.

```ruby
class SongForm < Reform::Form
  property :title

  extend ActiveModel::ModelValidations
  copy_validations_from Song
end
```

Note how `copy_validations_from` copies over the validations allowing you to stay DRY.

This also works with Composition.

```ruby
class SongForm < Reform::Form
  include Composition
  # ...

  extend ActiveModel::ModelValidations
  copy_validations_from song: Song, band: Band
end
```

Be warned that we _do not_ encourage copying validations. You should rather move validation code into forms and not work on your model directly anymore.

## Agnosticism: Mapping Data

Reform doesn't really know whether it's working with a PORO, an `ActiveRecord` instance or a `Sequel` row.

When rendering the form, reform calls readers on the decorated model to retrieve the field data (`Song#title`, `Song#length`).

When syncing a submitted form, the same happens using writers. Reform simply calls `Song#title=(value)`. No knowledge is required about the underlying database layer.

The same applies to saving: Reform will call `#save` on the main model and nested models.

Nesting forms only requires readers for the nested properties as `Album#songs`.


## Rails Integration

Check out [@gogogarret](https://twitter.com/GoGoGarrett/)'s [sample Rails app](https://github.com/gogogarrett/reform_example) using Reform.

Rails and Reform work together out-of-the-box.

However, you should know about two things.

1. In case you explicitely _don't_ want to have automatic support for `ActiveRecord` and form builder: `require reform/form`, only.
2. In some setups around Rails 4 the `Form::ActiveRecord` module is not loaded properly, usually triggering a `NoMethodError` saying `undefined method 'model'`. If that happened to you, `require 'reform/rails'` manually at the bottom of your `config/application.rb`.

## ActiveRecord Compatibility

Reform provides the following `ActiveRecord` specific features. They're mixed in automatically in a Rails/AR setup.

 * Uniqueness validations. Use `validates_uniqueness_of` in your form.

As mentioned in the [Rails Integration](https://github.com/apotonick/reform#rails-integration) section some Rails 4 setups do not properly load.

You may want to include the module manually then.

```ruby
class SongForm < Reform::Form
  include Reform::Form::ActiveRecord
```


## ActiveModel Compliance

Forms in Reform can easily be made ActiveModel-compliant.

Note that this step is _not_ necessary in a Rails environment.

```ruby
class SongForm < Reform::Form
  include Reform::Form::ActiveModel
end
```

If you're not happy with the `model_name` result, configure it manually.

```ruby
class CoverSongForm < Reform::Form
  include Reform::Form::ActiveModel

  model :song
end
```

This is especially helpful when your framework tries to render `cover_song_path` although you want to go with `song_path`.


## FormBuilder Support

To make your forms work with all the form gems like `simple_form` or Rails `form_for` you need to include another module.

Again, this step is implicit in Rails and you don't need to do it manually.

```ruby
class SongForm < Reform::Form
  include Reform::Form::ActiveModel
  include Reform::Form::ActiveModel::FormBuilderMethods
end
```

## Multiparameter Dates

Composed multi-parameter dates as created by the Rails date helper are processed automatically. As soon as Reform detects an incoming `release_date(i1)` or the like it is gonna be converted into a date.

Note that the date will be `nil` when one of the components (year/month/day) is missing.


## Security

By explicitely defining the form layout using `::property` there is no more need for protecting from unwanted input. `strong_parameter` or `attr_accessible` become obsolete. Reform will simply ignore undefined incoming parameters.


## Additional Features

### Nesting Without Inline Representers

When nesting form, you usually use a so-called inline form doing `property :song do .. end`.

Sometimes you want to specify an explicit form rather than using an inline form. Use the `form:` option here.

```ruby
property :song, form: SongForm
```

The nested `SongForm` is a stand-alone form class you have to provide.


### Overriding Accessors

When "real" coercion is too much and you simply want to convert incoming data yourself, override the setter.

```ruby
class SongForm < Reform::Form
  property :title

  def title=(v)
    super(v.upcase)
  end
```

This will capitalize the title _after_ calling `form.validate` but _before_ validation happens. Note that you can use `super` to call the original setter.


## Undocumented Features

_(Please don't read this section!)_


### Populator

You can run your very own populator logic if you're keen (and you know what you're doing).

```ruby
class AlbumForm < Reform::Form
  # ...

  collection :songs, populator: lambda { |fragment, args| args.binding[:form].new(Song.find fragment[:id]) } do
    # ..
  end
```


## Support

If you run into any trouble chat with us on irc.freenode.org#trailblazer.


## Maintainers

[Nick Sutterer](https://github.com/apotonick)

[Garrett Heinlen](https://github.com/gogogarrett)


### Attributions!!!

Great thanks to [Blake Education](https://github.com/blake-education) for giving us the freedom and time to develop this project in 2013 while working on their project.
