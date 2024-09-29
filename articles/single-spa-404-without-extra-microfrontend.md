In this article we'll explore a way to simplify your single-spa microfrontend architecture by removing the dedicated error page microfrontend

Having a project based on a microfrontends architecture has its benefits, but also some challenges.

One of the challenges is the inevitable growth of the number of microfrontends. This growth means that there will be more microfrontends to maintain, update and release. Reducing the number of microfrontends ensures a smoother development experience.

In order to reduce the number of microfrontends, we can migrate some of the existing functionality into an already existent microfrontend. We need to do this carefully, as it will cause us to break the [single responsibility principle](https://en.wikipedia.org/wiki/Single-responsibility_principle).

How great would it be to remove a microfrontend that has the sole purpose of rendering an error page?

### Removing the 404 microfrontend in single-spa

In single-spa, to add a 404 page that is written in React, you have to have a separate application microfrontend that displays the 404 error.

Usually this is done in the [layout definition](https://single-spa.js.org/docs/layout-definition/#default-routes-404-not-found):

```tsx
<route default>
    <application name="error-microfrontend"></application>
</route>
```

Getting rid of the `error-microfrontend` can be tricky.

A) Refactoring React to HTML 🙅

The single-spa routes are defined in HTML, which means that if you want to render a react component, it will be hard to include all the logic (especially if you use custom logic in your component). So this isn't a viable option.

B) Passing `customProps` 🙅

You can also try replacing it with an existing microfrontend by passing [customProps](https://single-spa.js.org/docs/building-applications/#custom-props) to it:

```tsx
// top-level layout engine API
const singleSpaRoutes = constructRoutes(template, {
    props: {
        errorPage: true
    },
    loaders: {},
});

...

// single-spa-template
<route path="/test">
    <application name="global-microfrontend"></application>
</route>

<route default>
    <application name="global-microfrontend" customProps="errorPage"></application>
</route>
```

For some reason, this doesn't work. The `customProps` aren't accessible in `global-micrfrontend`.

I've tried multiple ways of doing the same thing this, even changing the `loadRootComponent` for `global-microfrontend`:

```tsx
const lifecycles = singleSpaReact({
    React,
    ReactDOM,
    loadRootComponent: (props) => {
        console.log(props);

        return Promise.resolve(() => <Root />);
    },
});
```

Which also didn't work, which is odd, because if I look at the return for `constructApplications`:

```tsx
// usage
const applications = constructApplications({
    loadApp: ({ name }) => globalThis.System.import(name),
    routes,
});

// result
...
name: 'global-microfrontend',
app: f(),
customProps: f(e, n),
activeWhen: [
    0: f $(),
    1: f(n)

]
```

It seems that single-spa is aware that `global-microfrontend` will be mounted on 2 routes, and one of them with custom props.

I assume that this way doesn't work because of how single-spa [merges the props](https://single-spa.js.org/docs/layout-definition/#props).

C) Hacky way 🙋

Since nothing worked and I really wanted to get rid of the error microfrontend, I came up with a workaround:

```tsx
// single-spa-template
<route default>
    <div id="PAGE_NOT_FOUND"></div>
</route>
```

In the `global-microfrontend`, which loads in the top level of the application:

```tsx
import React, { useEffect, useState } from "react";

const ErrorContent = () => (
    <div>
        <h1>404 not found</h1>
        <h3>Please try visiting another page</h3>
    </div>
);

const ErrorPage = () => {
    const [visible, setVisible] = useState<boolean>(false);

    const checkForPageNotFound = () => {
        const el = document.getElementById("PAGE_NOT_FOUND");

        setVisible(!!el);
    };

    useEffect(() => {
        window.addEventListener("single-spa:routing-event", checkForPageNotFound);

        checkForPageNotFound();

        return () => window.removeEventListener("single-spa:routing-event", checkForPageNotFound);
    }, []);

    return visible ? <ErrorContent /> : null;
};

export default ErrorPage;
```

This works by listening for the `single-spa:routing-event` event, and every time it is fired we check if the div we've added in the layout definition exists. Since the div is rendered only in the default route, this means that we can confidently render the contents of the 404 page.

While this might not be the most elegant solution, it enables us to encapsulate the 404 logic inside an existing microfrontend & to get rid of another microfrontend that would need maintenance.

If you found other creative solutions for simplifying an architecture based on microfrontends, I would love to hear about your experiences!
