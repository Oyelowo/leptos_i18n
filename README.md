# Leptos i18n

This crate is made to simplify internalisation in a Leptos application, that load locales at **_compile time_** and provide compile time check for keys and selected locale.

The main focus is ease of you use with leptos, a typical component using this crate will look like this:

```rust
let i18n = get_i18n_context(cx);

let (counter, set_counter) = create_signal(cx, 0);
let inc = move |_| set_counter.update(|count| *count += 1);


view! { cx,
    {/* click_count = "You have clicked {{ count }} times" */}
    <p>{t!(i18n, click_count, count = move || counter.get())}</p>
    {/* click_to_inc = "Click to increment" */}
    <button on:click=inc>{t!(i18n, click_to_inc)}</button>
}
```

You just need a configuration file named `i18n.json` and one file per locale name `{locale}.json` in the `/locales` folder of your application.

## How to use

### Configuration files

There are files that need to exist, the first one is the `i18n.json` file that describe the default locale and supported locales, it need to be at the root of the project and look like this:

```json
{
  "default": "en",
  "locales": ["en", "fr"]
}
```

The other ones are the files containing the translation, they are key-value pairs and need to be situated in the `/locales` directory at root of the project, they should be named `{locale}.json`, one per locale defined in the `i18n.json` file.
They look like this:

```json
/locales/en.json

{
    "hello_world": "Hello World!"
}

/locales/fr.json

{
    "hello_world": "Bonjour le monde!"
}

```

All locales files need to have exactly the same keys.

### Loading the locales

you can then use the `load_locales!()` macro in a module of the project, this will load _at compile time_ the locales, and create a struct that describe your locales:

```rust
struct Locale {
    pub hello_world: &'static str
}
```

Two other helper types are created, one enum representing the locales:

```rust
enum LocaleEnum {
    en,
    fr
}
```

and an empty struct named `Locales` that serves as a link beetween the two, it is this one that is the most important, most functions of the crate need this type, not the one containing the locales nor the enum.

### I18nContext

The heart of this library is the `I18nContext`, it must be provided at the highest possible level in the application with the `provide_i18n_context` function:

```rust
// root of the application
#[component]
pub fn App(cx: Scope) -> impl IntoView {

    leptos_i18n::provide_i18n_context::<Locales>(cx);

    view! { cx,
        {/* ... */}
    }
}
```

You can then call the `get_context<T>` function to access it:

```rust
let i18n_context = leptos_i18n::get_context::<Locales>(cx);
```

It is advised to make your own function to suppress the need to pass the `Locales` type every time:

```rust
#[inline]
pub fn get_i18n_context(cx: Scope) -> I18nContext<Locales> {
    leptos_i18n::get_context(cx)
}
```

The `provide_i18n_context` function return the context, so instead of

```rust
leptos_i18n::provide_i18n_context::<Locales>(cx);

let i18n = get_i18n_context(cx);
```

You can write

```rust
let i18n = leptos_i18n::provide_i18n_context::<Locales>(cx);
```

The context implement 3 key functions: `.get_locale()`, `.get_keys()` and `.set_locale(locale)`.

### Accessing the current locale

You may need to know what locale is currenly used, for that you can call `.get_locale` on the context, it will return the `LocaleEnum` defined by the `load_locales!()` macro. This function actually call `.get` on a signal, this means you should call it in a function like any signal.

### Accessing the keys

You can access the keys by calling `.get_keys` on the context, it will return the `I18nKeys` struct defined above, build with the current locale. This is also based on the locale signal, so call it in a function too.

### Setting a locale

When the user make a request for your application, the request headers contains a weighted list of accepted locales, this library take them into account and try to match it against the loaded locales, but you probably want to give your users the possibility to manually choose there prefered locale, for that you can set the current locale with the `.set_locale` function:

```rust
let i18n = get_i18n_context(cx);

let on_click = move |_| {
    let current_locale = i18n.get_locale();
    let new_locale = match current_locale {
        LocaleEnum::en => LocaleEnum::fr,
        LocaleEnum::fr => LocaleEnum::en,
    };
    i18n.set_locale(new_locale);
};

view! { cx,
    <button on:click=on_click>
        {move || i18n.get_keys().click_to_switch_locale}
    </button>
}

```

### The `t!()` macro

As seen above, it can be pretty verbose to do `move || i18n.get_keys().$key` every time, so the crate expose a macro to help with that, the `t!()` macro.

```rust
let i18n = get_i18n_context(cx);

view! { cx,
    <p>{t!(i18n, hello_world)}</p>
}
```

It takes the context as the first parameter and the key in second.
It also help with interpolation:

### Interpolation

You may need to interpolate values in your translation, for that you can add variables by wrapping it in `{{  }}` in the locale definition:

```json
{
  "click_to_inc": "Click to increment",
  "click_count": "You have clicked {{ count }} times"
}
```

You can then do

```rust
let i18n = get_i18n_context(cx);

let (counter, set_counter) = create_signal(cx, 0);
let inc = move |_| set_counter.update(|count| *count += 1);


view! { cx,
    <p>{t!(i18n, click_count, count = move || counter.get())}</p>
    <button on:click=inc>{t!(i18n, click_to_inc)}</button>
}

```

You can pass anything that implement `leptos::IntoView + Clone + 'static` as your variable. If a variable is not supplied it will not compile, same for an unknown variable key.

You may also need to interpolate components, to highlight some part of a text, you can define them with html tags:

```json
{
  "important_text": "this text is <b>very</b> important"
}
```

You can supply them the same way as variables, just wrapped beetween `< >`, but the supplied value must be a `T: Fn(leptos::Scope, leptos::ChildrenFn) -> impl IntoView + Clone + 'static`.

```rust
let i18n = get_i18n_context(cx);

view! { cx,
    <p>
        {t!(i18n, important_text, <b> = |cx, children| view!{ cx, <b>{children(cx)}</b> })}
    </p>
}

```

The only restriction on variables/components names is that it must be a valid rust identifier. You can define variables inside components: `You have clicked <b>{{ count }}</b> times`, and you can nest components, even with the same identifier: `<b><b><i>VERY IMPORTANT</i></b></b>`.

For plain strings, `.get_keys().$key` return a `&'static str`, but for interpolated keys it return a struct that implement a builder pattern, so for the counter above but without the `t!` macro it will look like this:

```rust
let i18n = get_i18n_context(cx);

let (counter, set_counter) = create_signal(cx, 0);
let inc = move |_| set_counter.update(|count| *count += 1);


view! { cx,
    <p>{move || i18n.get_keys().click_count.count(move || counter.get())}</p>
    <button on:click=inc>{t!(i18n, click_to_inc)}</button>
}
```

If a variable or a component is only needed for one local, it is totally acceptable to do:

```json
/locales/en.json

{
    "hello_world": "Hello World!"
}

/locales/fr.json

{
    "hello_world": "Bonjour <i>le monde!</i>"
}

```

When accessing the key it will return a builder that need the total keys of variables/components of every locales, but it will fail to compile if one locale use a key for a component and another locale use the same key for a variable.

If your value as the same name as the variable/component, you can drop the assignement, this:

```rust
t!(i18n, key, count = count, <b> = b, other_key = ..)
```

can we shorten to

```rust
t!(i18n, key, count, <b>, other_key = ..)
```

### Plurals

You may need to display different messages depending on a count, for exemple one when there is 0 elements, another when there is only one, and a last one when the count is anything else. For that you can do:

```json
{
  "click_count": {
    "0": "You have not clicked yet",
    "1": "You clicked once",
    "_": "You clicked {{ count }} times"
  }
}
```

When using plurals, the key `count` variable is reserved and takes as a value `T: Fn() -> i64 + Clone + 'static`, the resulting code looks something like this:

```rust
match count() {
    0 => // render "You have not clicked yet",
    1 => // render "You clicked once",
    _ => // render "You clicked {{ count }} times"
}
```

You can also supply a range:

```json
{
  "click_count": {
    "0": "You have not clicked yet",
    "1": "You clicked once",
    "2..=10": "You clicked {{ count }} times",
    "11..": "You clicked <b>a lot</b>"
  }
}
```

But this exemple will not compile, because the resulting match statement will not cover the full `i64` range, so you will either need to introduce a fallback, or the missing range: `"..0": "You clicked a negative amount ??"`.

If one locale use plurals for a key, another locale does not need to use it, but the `count` variable will still be reserved, but it still can access it as a variable, it will just be constrained to a `T: Fn() -> i64 + Clone + 'static`.

### Examples

If examples works better for you, you can look at the different examples available on the Github.

## Features

You must enable the `hydrate` feature when building the client, and when building the server you must enable either the `actix` or `axum` feature.

The `cookie` feature enable to set a cookie when a locale is chosen by the user, this feature is enabled by default.

## Contributing

Errors are a bit clunky or obscure for now, there is a lot of edge cases and I did not had time to track every failing scenario, feel free to open an issue on github so I can improve those.

Also feel free to open PR for any improvement or new feature.
