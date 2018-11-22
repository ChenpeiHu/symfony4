# Form Type Extension

Check this out: Symfony's form system has a feature where we can modify *any* form
or *any* field across our *entire* app! It's called a "form type extension" and
it's *so* fun.

To see how this works, let's talk about the `textarea` field. Forget about Symfony
for a moment. In HTML land, one of the features of the `textarea` element is that
you can give it a `rows` attribute. If you set `rows="10"`, it gets longer.

If we wanted to set that attribute, we could of course pass an `attr` option with
`rows` set to some value. Here's the question: could we automatically set that option
for *every* textarea across our entire app? Absolutely! We can do anything!

## Creating the Form Type Extension

In your `Form/` directory, create a new directory called `TypeExtension`, then inside
a class called `TextareaSizeExtension`. Make this implement `FormTypeExtensionInterface`.
As the name implies, this class will allow us to *extend* existing form types.

Next, go to the Code -> Generate menu, or Command+N on a Mac, and choose
"Implement Methods" to implement everything we need. Woh! We know these methods!
These are almost the *exact* same methods that we've been implementing in our form
type classes! And... that's on purpose! These methods work pretty much the same way.

## Registering the Form Type Extension

The only different method is `getExtendedType()`. To tell Symfony that this form
type extension exists *and* to tell it that we want to extend the `TextareaType`,
we need a little bit of config. This might look confusing at first. Let's code it
up, then I'll explain what's going on.

Open `config/packages/services.yaml`. And, at the bottom, we need to give our service
a "tag". First, put the form class and below, add `tags`. The syntax here is a bit
ugly: add a dash, open an array and set `name` to `form.type_extension`. Then I'll
create a new line for space and add one more option `extended_type`. We need to set
this to the form type class that we want to extend - so `TextareaType`. Let's cheat
real quick: I'll `use TextareaType`, auto-complete that, copy the class, then delete
that. Go paste it in the config. Oh, and I forgot my comma!

As soon as we do this, every time a `TextareaType` is created in the system, *every*
method on our `TextareaSizeExtension` will be called. It's almost as if each of
these methods *actually* lives *inside* of the `TextareaType` class! If we add
some code to `buildForm()`, it's pretty much identical to opening up the `TextAreaType`
class and adding code right there!

## The form.type_extension Tag & autoconfigure

Now, two important things. If you're using Symfony 4.2, then you do *not* need to
add *any* of this code in `services.yaml`. Whenever you need to "plug into" some
part of Symfony, internally, you do that by registering a service and giving it
a "tag". The `form.type_extension` tags says:

> Hey Symfony! This isn't just a normal service! It's a form type extension! So
> make sure you use it for that!

But these days, you don't see "tags" much in Symfony. The reason is simple: for
most things, Symfony looks at the interfaces that your services implement, and adds
the correct tags automatically. In Symfony 4.1 and earlier, this does *not* happen
for the `FormTypeExtensionInterface`. But in Symfony 4.2... it does! So, no config
needed... at all.

So, how does Symfony know which form type class you want to extend in Symfony 4.2?
The `getExtendedType()` method! Inside, `return TextareaType::class`. And yea,
we *also* need to fill in this method in Symfony 4.1. It's some duplication, which
is why Symfony 4.2 will be *so* much cooler.

## Filling in the Form Type Extension

Ok! Let's remove the rest of the TODOs in here so we can get to work! We can fill
in whichever methods we need. In our case, we want need to add modify the *view*
variables. That's easy for us: in `buildView()`, say `$view->vars['attr']`, and
then add a `rows` attribute equal to 10.

Done! Move over, refresh and... yea! I think it's bigger! Inspect it - yes: `rows="10"`.
*Every* `<textarea>` on our *entire* site will now have this.

## Modifying "Every" Field?

By the way, instead of modifying just one field type, *sometimes* you may want to
modify literally *every* field type. To do that, you can choose to extend
`FormType::class`. That works because of the form field inheritance system. All
field types ultimately extend `FormType::class`, except for a `ButtonType` I don't
usually use. So if you override `FormType`, you can modify everything.

## Adding a new Field Option

But wait, there's more! Instead of hardcoding 10 could we make it possible to configure
this value *each* time you use the `TextareaType`? Why, of course! In `ArticleFormType`,
pass `null` to the `content` field so it keeps guessing it. Then add a new option:
`rows` set to 15.

Try this out - refresh! Giant error!

> The option "rows" does not exist

It turns out that you can't just "invent" new options and pass them. Each field
has a concrete set of valid options. But, in `TextareaSizeExtension`, we *can*
invent new options. Do it down in `configureOptions()`: add
`$resolver->setDefaults()` and invent a new `rows` option with a default value of
10.

Now, up in `buildView()`, notice that almost every method is passed the final
array of `$options` passed to this field. Set the `rows` attribute to `$options['rows']`.

Done. The `rows` will default to 10, but we can override that via a brand, new
shiny form field option. Try it! Refresh, inspect the textarea and... yes! The
`rows` are set to 15.

*This* is the power of form type extensions. And these are *even* used in the
core of Symfony to do a few cool things. For example, remember how every form
has an `_token` CSRF token field added to it? How does Symfony magically do that?
The answer: a form type extension. Press Shirt+Shift and look for a class called
`FormTypeCsrfExtension`.

Cool! It extends an `AbstractTypeExtension` class, which implements the same
`FormTypeExtension` but  prevents you from needing to override *every* method.
We could have used this same class.

Anyways, in `buildForm()` it adds an "event listener", which activates some code
that will *validate* the `_token` field when we submit. We'll talk about events
in a little while.

In `finishView()` - which is very similar to `buildView()`, it adds a few
variables to help render that hidden field. And finally, in `configureOptions()`,
it adds some options that allow us to control things. For example, inside the
`configureOptions()` method of any form class - like `ArticleFormType` - we could
set a `csrf_protection` option to `false` to disable the CSRF token.

Next: how could we make our form look or act differently based on the *data* passed
to it? Like, how could we make the `author` field *disabled*, only on the edit
form? Let's find out!