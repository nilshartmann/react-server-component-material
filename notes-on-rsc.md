# Opinionated notes on React Server components

React Server Components [are](https://twitter.com/RogersKonnor/status/1660298060352585729?s=20) [currently](https://twitter.com/CamBlackwood95/status/1628355074090061827?s=20) [discussed](https://twitter.com/passle_/status/1660297849899302921?s=20) [quite](https://twitter.com/natebirdman/status/1660895726770229248?s=20) [controversial](https://twitter.com/Swizec/status/1653605092371873792?s=20).

Personaly I think RSC can solve problems of React applications, but are not the "holy grail" or the "one fits all" solution. I tried to sump up some of my understandings and opinions in this document.

If you have any comments, corrections or additions please let me know. I'm sure the document is neither complete nor 100% correct. I have "published" this document as a pull request so you can directly comment inline [direct the PR](https://github.com/nilshartmann/react-server-component-material/pull/1) if you like.

## "Client" vs. "Server"

[Lenz Weber-Tronic](https://twitter.com/phry) wrote a very good summary of the terms client and server. Let me quote his excellent post [The Next.js "App Router", React Server Component & "SSR with Suspense" story](https://github.com/apollographql/apollo-client-nextjs/blob/pr/RFC-2/RFC.md) here:

> We should note that the terms "Client" and "Server" are a bit of a misnomer.
>
> "Server components" **aren't strictly rendered on a running server**. While this is one case, these components can also be rendered during a build (such as static generation), in a local dev environment, or in CI.
>
> "Client components" are the React components we have known for years. **They aren't strictly run in the browser**. These are also executed in an SSR pass on the server, with a subset of the capabilities they would have in the browser: Effects are not executed and there is no ability to rerender. Because of the different environments, the `window` object is not available.

(Btw the RFC is worth reading if you want to learn about RSC even if you don't want to use (Apollo) GraphQL in your application)

The fact that a _client_ component can also be rendered on the server is stritcly speaking nothing new, as it was possible before RSC when using "ordinary" SSR. But when using Server Components in your app, you 1. have components that are explicitly called _server_ components and 2. you also have (in some cases) mark your clients components with `"use client"`. For me, it's a little counter intuitive, that even in that case your _client_ component gets renderd on the _server_ too.

## Less javascript on the client

This is probably one the main arguments in favor of RSC: Any code used in a server component will not be sent to the client (browser). That means the browser has fewer code to download and fewer code to parse, interpret and execute. Also, your _Client_ components can be pre-rendered on the server.

- That might lead to better performance when opening your application (shorter time until your app is visible)
- Note that for beeing _interactive_ your Client component code still needs to be downloaded and executed.
- imho if and how much you really profit from this changes depends on your application. If you have a highly interactive application you need the javascript code anyway. If you have applications running on fast internet connections only (in-house for example) benefits might not be that much.
  - I think it still makes sense to differentiate between _web sites_ (more or less static) vs. _web applications_ (highly interactive). While the former profit from server-side React code a lot, the latter probably benefit that much. Of course there is no strict definition when "something" that runs in the web is either static site or interactive application and there a lot of grey areas. But to decide what kind of application your own app is and how much you would benefit from server-side React, it may be helpful to consider which side your own app tends to lean towards.

## Changes with data fetching

RSCs are executed on server-side only and thus can directly use your server infrastructure, like your database. As they can also be implemented as _async_ components, you can fetch required data in your component as it would be server code (what it actually is...). No need for `useEffect` or data fetching libs here.

```typescript jsx
async function fetchPosts(orderBy): Promise<BlogPost[]> {
  // read data from db, http service, whatever
}

// Server Component!
export default async function PostList({ orderBy }: PostListProps) {
  const blogPosts = await fetchPosts(orderBy);

  return (
    <div>
      {blogPosts.data.map((p) => (
        <div key={p.id}>
          <PostPreview post={p} />
        </div>
      ))}
    </div>
  );
}
```

Potential consequences:

**Your app _might_ be faster.**

In a "classic" client-side only React App, depending on the architecture of your application you might encouter a pattern called "client side waterfalls" when datafetching. That happens if your parent component A loads data then renders child component B that in turn also loads data. As B requires the loaded data from A. For example A is searching a User by its name, forwards the User (id) to B and then B loads profile data for that User. Here component B cannot fetch the user profile data in parallel with A, as the required information (user id) is available only after the fetch request from A returned).
While the problem still occurs on react server components, the app might be able to fetch the data faster, because it does not need to make client-server round-trips for each request. If the data your components need, is "nearer" to your application, requests might be faster (less delay, less ...)
If and how much you profit from this, depends on your runtime architecture and deployment I think. Remember: of course you still need your data, only the way where and how it is fetched changes.

**Your code _might_ be easier/shorter**

Using `useEffect` and `fetch` to load data might not be the easiest way of fetching data. Some people think, the code for working with data in RSC in general [is easier](https://twitter.com/shadcn/status/1660338653305114625?s=20) as in client components.
Personaly I'm not sure here. On the one hand it is true, that loading data in a server component is "easy" (or easier as with `useEffect`). On the other hand: we introduce more complexity by splitting up the application in a server- and a client part, which might lead to more complex and complicated code in total. Also there are excellent data fetching libraries as TanStack Query, Redux Toolkit Query or even GraphQL libraries available for the client side these days.

Some people also say (and welcome), that with RSC global/external statemanagement libs like Redux are not needed anymore. While that might be true in some cases, I think it hides the fact, that the required data still needs to be fetched and rendered. With RSC the logic moves to other parts of your application (and away from your browser) but of course you still need to get the data effiecently somehow.

The same is valid for local state (`useState`/`useReducer`). If you client needs interactivity (which is for me
_the_ reason to use React), you still need this kind of state in your application. An application does not become stateless when using RSC.

This is also true for **form handling**. Recently there was an [announcement or at least a hint](https://twitter.com/dan_abramov/status/1641830612955803650?s=20), that in the new React documentation uncontrolled forms are now preferred over controlled forms. The idea: with RSC you would submit your data using the form's action, as you would do with forms on traditional non-javascript websites. This code will be even [simplified](https://twitter.com/CompuIves/status/1660660884534902784?s=20) with (https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions)[server actions]. How your application benefits from this, depends on your application needs. If you have for example (complex) client side validation, that cannot be expressed with standard HTML and CSS posibilities, you still have to use client-side state and form handling. Also, if you just can submit your FormData to your backend (or use server actions) depends on your backend. If you have to send your data to a classical REST endpoint (or even GraphQL) submitting Form Data might not work in your case.

## Communication between components

In classical React app architecture you have top-level components that pass down properties to their child components. Properties can either be data or (callback-)functions. In your child component you render the data and - as result of an event - invoke the callback function. The parent component updates its state and re-renders your component tree.
In some way working with RSC is the same: the RSC renders another RSC (with props) or a child components (also with props). But on the "boundary" betweed server and client (RSC renders a client component) you can only pass props that are serializable, as they are sent over the network. There is no way to pass a callback function from a server to a client component. While that makes sense (you could not invoke a function defined on server-side anyway), this is a major change compared to classic client-side react apps. Again, I don't want to say this is good or bad, its just _different_. I'm a little more sceptical about the "API", as this is the same as in client side apps: JSX code. There is not difference in rendering a RSC or client-side component. You have to know that you are in a server component and you have to know what kind of component you're calling to know which prop types are allowed. Probably tooling will help us here in the future, but I think for reading and understanding the code, a more "explicit" boundary would have been nice.

If a server component wants to pass data down to a client component it just can pass the data as props to it (as long as they are serializable). Communicating back to the server component is not possible with callback functions. Instead, you have to somehow [make a server call](https://twitter.com/dan_abramov/status/1633574036767662080?s=20) to re-render the server component. In most case you will change the URL (path or search/query params) for doing so. While this somehow preserves the uni-directional dataflow in my opionion that is quite a big change to client-side behaviour of React, and also can lead to more complexity in your code because you have to work with the URL (not sure if complexity is the correct word here).

When you're application makes a server request to re-render a RSC, the result of your request is the [new pre-rendered UI](https://twitter.com/dan_abramov/status/1649835630711455746?s=20) of your component tree. React will receive this pre-rendered code and update the UI in your browser. Depending on your application architecture and component structure that might lead to more data that is loaded from the server compared to client side data fetching (for that single request). If you read the JSON data of a blog post via `fetch` from a REST API you only receive that _data_ of that post. When you call an RSC instead, you receive the fully rendered UI for that blog post, so that the [response might be larger](https://twitter.com/dan_abramov/status/1654478296732532737?s=20) than the raw data received from the REST request. Of course, you cannot compare that 1:1 because the whole JS code for the server code is not transferred to the browser, so that in total it's possible to save bytes with RSC also in this case. But I think that depends on some parameters. If your javascript code for example does not change that often, chances are that the JS code is already cached in your customers browser and does not need to fetched again from the server.

## Routing

While that is not exactly tied to RSC (I think), currently you can [not do client side routing](https://twitter.com/dan_abramov/status/1654950866950971392?s=20) when using react server components. You need to go with server-side routing. That means that any URL change results in a server round trip.
While this is not generally bad (on the other hand: involving the server is a desired behaviour of RSC) it might complicate migration scenarios. Also there might be cases where client side routing [would be enough or even better than doing server roundtrips](https://twitter.com/natebirdman/status/1660687874784907264?s=20) (imagine for example you have data available in the client that only needs to be rendered, but not to re-fetched from server).

## Code and concepts

Writing server components is almost the same as writing a client components: you will write in JavaScript/TypeScript and use JSX for your UI. Depending on the kind of components you can either write your components as before with Hooks API etc. (client compoents) or use features from server components (async components functions for example). And while this is on code-level not a big difference to "classic" React apps, other things drastically change imho. Of course you need a JavaScript-enabled server/runtime to run your application. A static webserver is not enough anymore. And running in production is more than just starting. You need to monitor, secure, patch etc. the server. Depending on other parts of your infrastructure that might be a massive shift (if you would migrate) or even complexity on-top of your existing infrastructure (for example when your backend is not written in JavaScript). Also using server-side React has an impact on your test strategy (which is imho even bigger if you consider Next.js various render modes...).
Also understanding what exactly happens when your code runs is harder to understand. That might not be a big deal, because many things are abstracted, but for example if anything does not work (as expected) it might be harder to trace and find the problem. Of course that is a problem you have with any new layer or abstraction in an application, so it's not specific to server components, but you should consider this before starting the "fullstack way".

## What is Next.js and what is RSC?

If you want to use RSC currently, your probably need to use Next.JS (there is an open-source [waku](https://github.com/dai-shi/waku), but I don't think is production ready).
While using Next.JS is not bad per se, Next.JS adds a lot of its own ideas and concepts to the tech stack that has consequences of behaviour, architecture und code of your application (not that this is neither "good" or "bad", it just "is"). For example, you have to use the App Router. You're forced to deal with the powerful but complex caching system of Next.js.

You have to learn and understand the [various render modes in Next.js](https://nextjs.org/docs/app/building-your-application/rendering) in Next.js, that are [quite complex](https://github.com/apollographql/apollo-client-nextjs/blob/pr/RFC-2/RFC.md#nextjs-specifics-static-and-dynamic-renders) and might change due to nuances in your code (add a [dynamic function](https://nextjs.org/docs/app/building-your-application/rendering/static-and-dynamic-rendering#using-dynamic-functions) like `useSearchParams` or `headers` in your code for example), that are not obvious to beginners I think.

## Summary

React Server Components are an interesting idea. But as with most technologies there are pros and cons and (until now) I cannot see that the advantages outweigh the disadvantages in _every_ application.

I also find it a bit annoying that the React [documentation](https://react.dev/learn/start-a-new-react-project#can-i-use-react-without-a-framework) and ([former](https://twitter.com/acdlite/status/1617611126514266112?s=20)) React [team](https://twitter.com/rickhanlonii/status/1653960176779550721?s=20) [members](https://twitter.com/dan_abramov/status/1636827365677383700?s=20) are now almost pushing you to use React with a fullstack framework, [even if they say it's a "recommendation" only](https://twitter.com/acemarke/status/1653962086068912129?s=20) because that will neither be possible nor useful for everyone (especially if you have existing applications and there is no [migration path](https://twitter.com/santiago_j_s/status/1653810360099512320?s=20)). I would have preferred a more differentiated view. But maybe that will come, since everyone is still discussing the pros and cons at the moment.

Thanks for reading so far 😊. Looking forward to your feedback, your own opionions and experiences with React Server Components.
I've "published" this document as a pull request so you can directly comment inline [in the PR](https://github.com/nilshartmann/react-server-component-material/pull/1) if you like.
