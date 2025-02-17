---
title: Routing
subtitle: Bind the page's URL to the Elmish model.
---

## Prerequisites

In order to use routing, you need to make sure that the page's base URL is set correctly. In most cases, you simply need to add the following inside the `<head>` of your page (`Pages/_Host.cshtml` for hosted apps, `wwwroot/index.html` for client-only apps):

```html
<base href="/">
```

If the root of your application is a subpath, then you need to set that subpath as the base, **including the trailing slash**. For example, if you have an application served at `https://someone.github.io/myrepo/`, then you need the following:

```html
<base href="/myrepo/">
```

## Inferred router

The easiest way to create a router is by using an inferred router. In this mode of operation, you create an endpoint type which has a 1-to-1 correspondence with your supported URLs, and store it in the Elmish model.

Here are the steps to set up an inferred router:

1. Create the endpoint type. Typically, it will be an F# union type where each case corresponds to a path prefixed by the case's name and the case's arguments are the consecutive fragments of the path. For example:

    ```fsharp
    type Page =
        | Home                                  // -> /Home
        | BlogArticle of id: int                // -> /BlogArticle/42
        | BlogList of user: string * page: int  // -> /BlogList/tarmil/1
    ```

    See [Format](#format) for all the supported types and how to customize the corresponding path.

2. Add a field in the Elmish model that stores the endpoint:

    ```fsharp
    type Model =
        {
            page: Page
            // other fields...
        }
    ```

3. Add an Elmish message that sets the endpoint:

    ```fsharp
    type Message =
        | PageChanged of Page
        // other messages...

    let update message model =
        match message with
        | PageChanged page -> { model with page = page }
        // other messages...
    ```

4. Create the router with `Router.infer`:

    ```fsharp
    let router = Router.infer PageChanged (fun m -> m.page)
    ```

5. Attach the router to the Elmish program:

    ```fsharp
    Program.mkSimple initModel update view
    |> Program.withRouter router
    ```

> Note: the message `PageChanged` is dispatched automatically when the URL has been modified (eg by clicking a link).
> It is not recommended to dispatch it directly yourself, as it will result in the same message being dispatched twice: first by you, and then by the router.
>
> Instead, if you want to trigger a page change programmatically, you can add another message for this purpose:
>
> ```fsharp
> type Message =
>     | PageChanged of Page
>     | ChangePage of Page
>
> let update message model =
>     match message with
>     | PageChanged page -> { model with page = page }
>     | ChangePage page -> { model with page = page }
> ```
>
> Alternatively, to trigger a page change programmatically from the update function, simply return a model with the desired target page:
>
> ```fsharp
> type Message =
>     | PageChnaged of Page
>     | LoginSucceeded of UserData
>
> let update message model =
>     match message with
>     | PageChanged page -> { model with page = page }
>     | LoginSucceeded userData ->
>         { model with userData = userData; page = Home }
> ```


The router has a few helpful utilities:

* `router.Link` takes an endpoint value and returns the corresponding URL.

    ```fsharp
    a { attr.href (router.Link Home); "Go to Home" }
    ```

* `router.HRef` takes an endpoint and returns an `href` attribute pointing to the corresponding URL.

    ```fsharp
    a { router.HRef Home; "Go to Home" }
    ```

### Format

`Router.infer` supports the following types:

* Standard library types:

    * `string`
    * `bool`
    * integer: `int8` (aka `sbyte`), `uint8` (aka `byte`), `int16`, `uint16`, `int32` (aka `int`), `uint32`, `int64`, `uint64`
    * `decimal`
    * float: `single`, `double` (aka `float`)

* F# union types. Each case corresponds to a path prefixed by the case's name. The case's arguments are the consecutive fragments of the path:

    ```fsharp
    type Page =
        | Home                                  // -> /Home
        | BlogArticle of id: int                // -> /BlogArticle/42
        | BlogList of user: string * page: int  // -> /BlogList/tarmil/1
    ```

    To customize the path, you can use the `EndPoint` attribute, with parameters indicated by `{curly_braces}`.

    ```fsharp
    type Page =
        | [<EndPoint "/">]
          Home                                  // -> /
        | [<EndPoint "/article/{id}">]
          BlogArticle of id: int                // -> /article/42
        | [<EndPoint "/list/{user}/{page}">]
          BlogList of user: string * page: int  // -> /list/tarmil/1
    ```
    
    An `{*asterisk}` indicates that this parameter catches the rest of the path. It must be the last parameter in the path, and correspond to a value of type `string`, `list` or `array`.
    
    ```fsharp
    type Page =
        | [<EndPoint "/list/{user}/tagged/{*tags}">]
          ListTagged of user: string * tags: list<string>

    ListTagged("tarmil", ["bolero"; "webassembly"])
    // -> /list/tarmil/tagged/bolero/webassembly
    ```
    
    Several cases can share a common prefix in their paths, as long as any parameters in this common prefix have the same type.
    
    ```fsharp
    type Page =
        | [<EndPoint "/user/{user}/favorites">]
          Favorites of user: string
        | [<EndPoint "/user/{user}/comments">]
          Comments of user: string
    ```
    
    The path must contain a `{parameter}` for each of its case's values. Alternatively, the path can consist of a single non-parameter fragment; in this case, the values are appended in order as separate fragments.
    
    ```fsharp
    type Page =
        | [<EndPoint "/list">]                  // Equivalent to /list/{user}/{page}
          BlogList of user: string * page: int  // -> /list/tarmil/1
    ```

* Tuples. The values of the tuple are the consecutive fragments of the path.

    ```fsharp
    type Page = int * string    // -> /42/abc
    ```

* F# record types. The fields of the record are the consecutive fragments of the path.

    ```fsharp
    type Page =
        {
            x: int
            y: string
        }
    // -> /42/abc
    ```

* Lists and arrays. The first fragment of the path is the number of items, and the items themselves are the consecutive fragments.

    ```fsharp
    type Page = list<string>   // -> /3/abc/def/ghi
    ```

* Any combination of the above, including recursive types.

    ```fsharp
    type Page =
        | [<EndPoint "/article">]
          Article of ArticleId              // -> /article/123/announcing-bolero
        | [<EndPoint "/list/{*tags}">]
          List of tags: list<string>        // -> /list/bolero/blazor
        | [<EndPoint "/login/{page}">]
          LoginAndRedirectTo of page: Page  // -> /login/list/bolero/blazor

    and ArticleId =
        {
            uid: int
            slug: string
        }
    ```

## Custom router

To have more control over the exact shape of your URLs, you can create a custom router like follows.

```fsharp
let customRouter : Router<Page, Model, Message> =
    {
        // getEndPoint : Model -> Page
        getEndPoint = fun m -> m.page
        // setRoute : string -> option<Message>
        setRoute = fun path ->
            match path.Trim('/').Split('/') with
            | [||] -> Some Home
            | [|"article"; id|] -> Some (BlogArticle (int id))
            | [|"list"; user; page|] -> Some (BlogList (user, int page))
            | _ -> None
            |> Option.map PageChanged
        // getRoute : Page -> string
        getRoute = function
            | Home -> "/"
            | BlogArticle(id) -> sprintf "/article/%i" id
            | BlogList(user, page) -> sprintf "/list/%s/%i" user page
    }
```

Note: if the URL is changed in such a way that `setRoute` returns `None`, then the model is not updated.

You can also create a router that maps directly to the model without an intermediary endpoint type. However this router type doesn't have some utilities such as `Link` and `HRef`.

```fsharp
let customRouter2 : Router<Model, Message> =
    {
        // setRoute : string -> option<Message>
        setRoute = fun path ->
            match path.Trim('/').Split('/') with
            | [||] -> Some Home
            | [|"article"; id|] -> Some (BlogArticle (int id))
            | [|"list"; user; page|] -> Some (BlogList (user, int page))
            | _ -> None
            |> Option.map PageChanged
        // getRoute : Model -> string
        getRoute = fun model ->
            match model.page with
            | Home -> "/"
            | BlogArticle(id) -> sprintf "/article/%i" id
            | BlogList(user, page) -> sprintf "/list/%s/%i" user page
    }
```

## Page Models

It is common to have a part of the application's model that is specific to a page. For example, a login page has the username and password that the user is typing. A "list of items" page has the items, which may have been downloaded from a [remote function](Remoting). And so on.

Let's take for example an application like the following:

```fsharp

type PersonModel = { name: string; age: int }

type LoginModel = { username: string; password: string }

type Page =
    | [<EndPoint "/people">] People  // Displays a list of Person
    | [<EndPoint "/login">] Login    // Shows a login screen with Login data
```

There are essentially two ways to handle page models:

1. Simply store the page model as a field of the application model. This is a reasonable way to go if you want to persist the state across page changes, and simply reuse the already loaded model when switching back to this page. For example, it may be a nice way to store the people in our application.

    ```fsharp
    type Model =
        {
            page: Page
            people: PersonModel list
        }
    ```

2. Store the page model in the Page union, so that it only exists when that is the current page. This is a better solution for our Login page: for security reasons, we don't want to keep the credentials in memory beyond the login page!

    For this purpose, Bolero has the type `PageModel<'T>`. When a page has an argument of type `PageModel<'T>`, it is not included in the page's URL and simply kept as internal state.

    ```fsharp
    type Page =
        | [<EndPoint "/people">] People
        | [<EndPoint "/login">] Login of PageModel<LoginModel>
    ```

    `PageModel<'T>` is simply a record with a field `Model : 'T`:

    ```fsharp
    type Message =
        | PageChanged of Page
        | SetUsername of string
        | SetPassword of string

    let updateLogin (update: LoginModel -> LoginModel) (model: Model) : Model =
        match model.page with
        | Login login -> { model with page = Login { Model = update login.Model } }
        | _ -> model

    let update (message: Message) (model: Model) =
        match message with
        | PageChanged p -> { model with page = p }
        | SetUsername username -> model |> updateLogin (fun l -> { l with username = username })
        | SetPassword password -> model |> updateLogin (fun l -> { l with password = password })
    ```

    There is one more requirement to be able to use `PageModel`: you must define the default value for the page model. Indeed, when the user clicks a link `/login`, Bolero needs to know what `LoginModel` value it needs to pass to `Login`!
    In order to define this default page model, instead of creating the router with `Router.infer`, you must use `Router.inferWithModel`. It takes an additional function of type `Page -> unit`. In this function, you must call `Router.definePageModel` with the default value of every page model in the router:

    ```fsharp
    let defaultModel = function
        | People -> ()
        | Login model -> Router.definePageModel model { username = ""; password = "" }

    let router = Router.inferWithModel PageChanged (fun m -> m.page) defaultModel
    ```

    And finally, when calling the functions `router.Link` or `router.HRef` to create a link to a page, you need to have a dummy page model value to pass to `Login`. You can use `Router.noModel` for this purpose:

    ```fsharp
    let s = router.Link (Login Router.noModel)
    // s = "/login"
    ```
