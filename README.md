# node-inertia

`node-inertia` is a simple Node.js adapter/middleware for [Inertia.js](https://inertiajs.com) that uses standard Node.js APIs. It can be used with any middleware based web framework that exposes the standard Node.js [`request`](https://nodejs.org/api/http.html#http_class_http_incomingmessage) and [`response`](https://nodejs.org/api/http.html#http_class_http_serverresponse) objects such as [Express.js](http://expressjs.com) or [Polka](https://github.com/lukeed/polka).

## Install

To install `node-inertia` inside your Node.js project simply run:

```
npm install node-inertia
```

**Note:** All code examples use [Polka](https://github.com/lukeed/polka) but you can use whatever web framework you like.

## Example

```js
const polka = require("polka");
const inertia = require("node-inertia");

const ASSET_VERSION = "1";
const { PORT = 3000 } = process.env;

const html = (pageString, viewData) => `
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width,initial-scale=1" />

    <!-- Custom data -->
    <title>${viewData.title}</title>

    <!-- Your Vue, Svelte or React SPA -->
    <link rel="stylesheet" href="/build/bundle.css" />
    <script defer type="module" src="/build/main.js"></script>
  </head>

  <!-- The Inertia page object -->
  <body id="app" data-page='${pageString}'></body>
</html>
`;

polka()
  .use(inertia(html, ASSET_VERSION))
  .get("/", (req, res) => {
    req.Inertia.setViewData({ title: "Inertia Page" }).render({
      component: "Index",
      props: { username: "ironman" },
    });
  })
  .listen(PORT, (err) => {
    if (err) throw err;
    console.log(`> Running on localhost:${PORT}`);
  });
```

## Usage

`node-inertia` expects two arguments:

1. `html`: a function that recieves `pageString` - a correctly encoded string of the Inertia page object - and an optional `viewData` object that you can populate with additional data. The function should return an HTML string.
2. `version` (optional): your current asset version

It will return a standard Node.js middleware that you can use with Express.js, Polka etc. Functions can be accessed from the `Inertia` object that will be added to the `request`. You can chain functions together but you can't call another function after calling [`render`](#renderpage) or [`redirect`](#redirecturl).

**Note:** In your HTML view function make sure to always include the page string in the `data-page` attribute of the HTML node you want to render your JavaScript app in. For more information on how Inertia works read [the protocol](https://inertiajs.com/the-protocol) on the Inertia website.

## Handling Validation Errors

Server-side validation error handling in Inertia is different from what you are used to and because in Inertia you initiate a redirect after an error occurs it requires some sort of session management. `node-inertia` can handle validation errors for you if you do not want to manually setup a server-side session for this and it is very simple. It does this by essentially creating a client-only cookie session. To learn more about how to handle server-side validation errors continue reading [here](#setErrorserrors)

## API

### setViewData(data)

1. `data`, _Object_ - An Object of additional data you want to pass to your HTML view function

`setViewData` can be used to pass additional data to your [HTML view function](#usage) such as the page's title or other meta data.

```js
app.use(({ Inertia }, _, next) => {
  Inertia.setViewData({
    title: "Todo App",
    description: "A Web App to Create and Manage Todos",
  });

  next();
});

// ...

const html = (pageString, viewData) => `
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width,initial-scale=1" />

    <meta name="description" content="${viewData.description}">
    <title>${viewData.title}</title>

    <link rel="stylesheet" href="/build/bundle.css" />
    <script defer type="module" src="/build/main.js"></script>
  </head>

  <body id="app" data-page='${pageString}'></body>
</html>
`;
```

If you call `setViewData` multiple times during a single request, the data of all calls will be merged into a single object and be made available to your HTML template.

### shareProps(props)

1. `props`, _Object_ - An object of props you want to include with every `render` call

Shared props are props that will be combined with the props you pass to the [`render`](#renderpage) function. When you call `shareProps` more than once, props shared from previous calls will be merged toghether with props from any subsequent calls.

```js
app.use(({ Inertia }, _, next) => {
  Inertia.shareProps({ username: "ironman" });
  next();
});

app.get("/todos", ({ Inertia }) => {
  Inertia.render({
    component: "Todos",
    props: {
      todos: [
        { text: "Study Korean", done: false },
        { text: "Cook Arabic food", done: true },
      ],
    },
    // Vue/Svelte/React component on the client will receive
    // {
    //   username: "ironman",
    //   todos: [
    //     { "Study Korean", done: false },
    //     { "Cook Arabic food", done: true },
    //   ]
    // }
  });
});
```

### setHeaders(headers)

1. `headers`, _Object_ - An Object of custom headers you want to include in your response

Add custom headers to your response.

**Note:** Headers that are required by Inertia take precedence and cannot be overwritten.

```js
app.get("/", ({ Inertia }) => {
  Inertia.setHeaders({
    token: "7pTgHCv0JgeAyyBRDpUi",
  }).render({
    component: "Index",
    props: { username: "ironman" },
  });
});
```

### setStatusCode(statusCode)

1. `statusCode`, _number_ - The response's status code

Change the status code when sending a response. Useful for e.g. when you want to render an HTTP error.

```js
app.get("/", ({ Inertia }) => {
  Inertia.render({
    component: "Index",
    props: { username: "ironman" },
  });
});

app.use(({ Inertia }) => {
  Inertia.setStatusCode(404).render({
    component: "Error",
    props: { message: "Page not found" },
  });
});
```

### render(page)

1. `page`, _Object_ - the Inertia page object

This function will send your response as either an HTML or JSON string to the client depending on wether the client is requesting your page for the first time or is making a subsequent Inertia request.

The Inertia page object consists of the following properties. Only `component` and `props` can be specified when calling the function:

1. `component`, _string_ - The name of the JavaScript component to render on the client
2. `props`, _Object_ - The page props (data)
3. `url`, _string_ - The URL of the route. This will be automatically added by the middleware.
4. `version`, _string_ - The asset version. This will be automatically added by the middleware based on the asset [version](#usage) you specified when creating the middleware.

```js
app.get("/todos", ({ Inertia }) => {
  Inertia.render({
    component: "Todos",
    props: {
      todos: [
        { text: "Study Korean", done: false },
        { text: "Cook Arabic food", done: true },
      ],
    },
  });
});
```

On Partial Reloads only props requested by the client will be sent. To improve performance on the server you can wrap each prop inside a function so that they will only be evaluated when necessary.

```js
app.get("/todos", ({ Inertia }) => {
  const todos = async () => await db.getTodos();
  const bookmarks = async () => await db.getBookmarks();

  Inertia.render({
    component: "Todos",
    props: {
      todos,
      bookmarks,
    },
  });
});
```

Now when the client only requests `todos` using [`only`](https://inertiajs.com/partial-reloads) on the client, `bookmarks` will not be called and your database only has to query for `todos`. Note however that if you do not make a partial request using `only` but a full page request all props/functions will be evaluated.

### lazy(fn)

1. `fn`, _Function_ - A function to lazily evaluate

You can create lazy functions in which case a prop will not be sent on a full page request but only if explicitly requested with [`only`](https://inertiajs.com/partial-reloads).

```js
app.get("/todos", ({ Inertia }) => {
  const todos = async () => await db.getTodos();
  const bookmarks = Inertia.lazy(async () => await db.getBookmarks());

  Inertia.render({
    component: "Todos",
    props: {
      todos,
      bookmarks,
    },
  });
});
```

In the above example `bookmarks` is never sent on a full page request but only if a [partial visit](https://inertiajs.com/partial-reloads) happens.

### redirect(url)

1. `url`, _string_ - The URL to redirect to

Redirect the client to a different URL.

```js
app.post("/todos", (req, res) => {
  db.createTodo(req.body);
  req.Inertia.redirect("/todos");
});
```

Since redirects to the same URL are very common in Inertia, `node-inertia` provides a shortcut for this.

Calling the following code:

```js
req.Inertia.redirect(req.headers["referer"]);
```

is equivalent to just:

```js
req.Inertia.redirect();
```

**Note:** Inertia requires you to use a `303` status code when redirecting upon a `PUT`, `PATCH` or `DELETE` request and a `302` status code otherwise. The `redirect` function will automatically take care of this for you. Should you handle redirects yourself make sure to select the correct status code.

**Note:** Calling [`setStatusCode`](#setStatusCodestatusCode) before [`redirect`](#redirecturl) has no effect.

### setErrors(errors)

1. `errors`

Handle server-side validation errors in Inertia.

From the [Inertia documentation:](https://inertiajs.com/validation)

```
Handling server-side validation errors in Inertia works a little different than a classic ajax-driven form, where you catch the validation errors from 422 responses and manually update the form's error state. That's because Inertia never receives 422 responses. Rather, Inertia operates much more like a standard full page form submission. Here's how:

First, you submit your form using Inertia. In the event that there are server-side validation errors, you don't immediately return those errors as a 422 JSON response. Instead, you redirect (server-side) back to the form page you are on, flashing the validation errors in the session. Frameworks like Laravel do this automatically.

Next, any time these validation errors are present in the session, they automatically get shared with Inertia, making them available client-side as page props, which you can display in your form. Since props are reactive, they are automatically shown when the form submission completes.
```

To handle server-side validation errors in Inertia with `node-inertia` you first call `setErrors` and pass in any errors you want to be made available on the client. Then you `redirect` to the page you want the errors to be displayed on, usually to the same page the user came from.

Example:

```js
app.post("/login", async (req, res) => {
  try {
    await user = UserService.login(req.body);
  } catch (err) {
    req.Inertia.setErrors(["INVALID_CREDENTIALS"]).redirect();
  }
});
```

The `errors` parameter passed to `setErrors` can be anything as long as it is valid JSON.

In the background there are a few things happening:

1. `setInertia` will create a cookie on the client named `inertiaErrors` whose value is a URI and JSON encoded string of whatever you pass to the function.
2. `redirect` will redirect the user to the URL you specify. Since in the above example the `url` parameter has not been specified it defaults to `headers["referer"]`, the page the user came from.
3. The client receives the error cookie and redirect response and requests the page you want it to redirect to.
4. `node-inertia` checks if the new request containes an `inertiaErrors` cookie. If it does, the errors from the cookie are parsed and send back to the client again together with the page.
5. The client receives the new page and any errors you passed in to `setErrors` are made available in the `errors` prop of your Vue, Svelte or React app. Finally the `inertiaErrors` cookie is deleted.
