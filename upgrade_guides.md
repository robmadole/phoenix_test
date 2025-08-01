Upgrade Guides
==============

## Upgrading to 0.7.0

Version 0.7.0 has a potentially breaking changes for those using the `upload`
helpers.

PhoenixTest will now automatically trigger `phx-change` on `upload`.

If you were previously triggering `phx-change` after your `upload`, you might
now get duplicate `phx-change` events. Ideally, you simply no longer have to do
that.

For more information, see the discussion in PR [#162] or commit [8edd7b4].

[#162]: https://github.com/germsvel/phoenix_test/pull/162
[8edd7b4]: https://github.com/germsvel/phoenix_test/commit/8edd7b4

### Why the change?

We always want PhoenixTest to behave as closely as possible like real Phoenix
does. When you have an upload, it triggers your `phx-change` event. So, we want
to emulate that. All other form helpers already do that. But `upload` didn't do
it until now.

## Upgrading to 0.6.0

Version 0.6.0 has one deprecation warning and one (potentially) breaking change.

### Deprecates `select/3` and `select/4` using `:from` (use `:option` instead)

Deprecates `select/3` and `select/4` using `:from` to denote the label. Instead,
it expects the label text to be passed as a positional argument and an `:option`
keyword argument to pass the option's text.

Thus, you'll need to make this change:

```diff
- |> select("Option 1", from: "Select Label")
+ |> select("Select Label", option: "Option 1")
```

And if you're using the version that provides a CSS selector:

```diff
- |> select("#super-select", "Option 1", from: "Select Label")
+ |> select("#super-select", "Select Label", option: "Option 1")
```

#### Why the change?

It may seem like a silly change (basically swapping positions of label and
option arguments), and in some ways it is. There's no real change in
functionality. So I've been very hesitant to make this change for a while.

The problem is that `select` is a bit surprising and confusing to use!

All other form helpers take in the label first as a positional argument, and
then any additional arguments (when they have them) go into a keyword list.

But `select` breaks that convention. It causes people to have to do mental
gymnastics to switch the order of arguments.

Rather than live with confusion for the rest of our lives, it seems better to
incur the cost right now, and then we can move on with all of our form helpers
being consistent.

### Raising when a route isn't found

If you have any tests that navigate to a route that isn't defined in your
router. Version 0.6.0 will raise an error.

If you visit a path that isn't defined, it's possible you were already getting
an error -- in which case this isn't a breaking change for you.

But in some cases, people might've been landing on a page that wasn't defined
(perhaps getting their 404 page), but their assertions still passed for other
reasons (e.g. they were just asserting the path name)

In those cases, the change we introduced would be a breaking change.

#### Why the change?

We want PhoenixTest to be as helpful as possible when you're test-driving the
your implementation. And we don't want it to provide false positives (meaning
your test passes, but it shouldn't pass).

Let me give you two examples:

1. Your test fails because the `assert_has` doesn't find the text on the page.
   You know that text is on the page, so you're confused as to why that would
   be. Only after you add an `open_browser` do you realize you had landed on a
   404 or 500 page because there was a typo on a route somewhere. PhoenixTest
   could just have raised an error that the route wasn't defined instead!

2. You have a test that has a `refute_has` with some text. Your test passes, so
   you think everything is good. Much later someone stumbles upon that test, but
   when they use `open_browser`, they realize the `refute_has` was passing only
   because you were on a 404 or 500 -- a completely different page from what you
   thought! It turned out that you navigated to a path that didn't exist. You
   thought your test was asserting that, for example, a user had been deleted,
   but instead you had gone to a non-existent path.

In both of those cases, PhoenixTest could have saved you time and pain by simply
ensuring that the path you're trying to visit is a real one. And the sooner we
can give you feedback about that, the better.

## Upgrading to 0.2.13

Version 0.2.13 deprecates `fill_form/3` and `submit_form/3`.

🥺 I know it's a pain. I'm sorry about that.

I don't take changing APIs lightly (even pre 1.0)... but I think you'll like
these changes.

### New form helpers

Let me introduce you to our new form helpers:

- `fill_in/3`
- `select/3`
- `choose/3`
- `check/3`
- `uncheck/3`

These new form helpers target elements by labels! 🥳

Instead of relying on the underlying data structures generated by Phoenix
forms and changesets, you can now specify which label you're targeting.

Change this: 👇

```elixir
session
|> fill_form("form", user: %{
  name: "Aragorn",
  admin: true,
  country: "Arnor"
})
```

To this: 👇

```elixir
session
|> fill_in("Name", with: "Aragorn")
|> check("Admin")
|> select("Arnor", from: "Countries")
```

The new format:

- encourages us (me included!) to use labels in forms,
- decouples the testing of our forms from the underlying shape of a changeset or
  Phoenix form -- something that's a mere implementation detail, and
- allows us to express our tests closer to the language a user would use when
  seeing a page.

**But what if I don't want the label to show?**

It's a good idea to have labels for accessibility -- even if they're not visible
on the page. In those cases, you should hide them with CSS.

For example, if you use Tailwind, you can add a `sr-only` class to your label.
That will mark it as "screen-reader only" and hide it.

### Targeting a form to fill out

Since `fill_form/3` used to allow targeting a form by CSS selector, you may want
to target a form via CSS selector with the new format. To do that, you can scope
all of the form helpers using `within/3`:

```elixir
session
|> within("#user-form", fn session ->
  session
  |> fill_in("Name", with: "Aragorn")
  |> check("Admin")
  |> select("Arnor", from: "Countries")
end)
```

NOTE: you may no longer _need_ to target your form via CSS selector. The new
helpers are a lot smarter since they're looking for the labels and their
associated inputs or options.

But if you have multiple forms with the same labels (even when those labels
point to different inputs), then you might have to scope your form-filling. And
that's where `within/3` can be handy.

### Submitting forms without clicking a button

Once we've filled out a form, we typically click a button with `click_button/2`
to submit the form. But sometimes you want to emulate what would happen by just
pressing <Enter> (or do what `submit_form/3` used to do).

For that case, you can use `submit/1` to submit the form you just filled out.

```elixir
session
|> fill_in("Name", with: "Aragorn")
|> check("Admin")
|> select("Arnor", from: "Countries")
|> submit()
```
