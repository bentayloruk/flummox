Why FluxComponent > FluxMixin
=============================

In the [React integration guide](react-integration.md), I suggest that using [FluxComponent](api/FluxComponent.md) is better than using [FluxMixin](api/FluxMixin.md), even though they do essentially the same thing. A few people have told me they like the mixin form more, so allow me to explain.

My argument can be broken down into three basic points. Note that these aren't my original ideas, nor are they unique to Flummox — they are the "React Way":

- Declarative > imperative
- Composition > inheritance
- State is evil

Declarative > imperative
------------------------

This is a no brainer. Remember the days before React, when you had to write one piece of code to render your application and another to update it? HAHAHAHA. That was awful. React showed us that expressing our views declaratively leads to clearer, more predictable, and less error-prone applications.

You might feel like FluxMixin and FluxComponent are equally declarative. They do have similar interfaces: a single argument/prop, that does (almost) the same thing. Still, as nice as FluxMixin's interface is, there's no beating a component in terms of clarity. A good rule of thumb in React is that everything that can be expressed as a component, should be.


Composition > inheritance
-------------------------

React has a very clear opinion on composition vs. inheritance: composition wins. Pretty much everything awesome about React — components, unidirectional data flow, the reconciliation process — derives from the fact that a React app is just a giant tree of components composed of other components.

Components make your code easy to reason about. If you stick to the basics of using components and props in your React app, you don't have to guess where data is coming from. The answer is always *from the owner*.

However, when you use FluxMixin, you're introducing data into your component that comes not from the owner, but from an external source — your stores. (This is also true of FluxComponent, but to a lesser extent, as we'll see later.) This can easily lead to trouble.

For instance, here's a component that renders a single blog post, based on the id of the post.

```js
let BlogPost = React.createClass({
  mixins: [FluxMixin({
    posts: function(store) ({
      post: store.getPost(this.props.id),
    })
  })],

  render() {
    <article>
      <h1>{this.state.post.title}</h1>
      <div>{this.state.post.content}</div>
    </article>
  }
});
```

Can you spot the problem? What happens when you want to re-use this same component to display a list of blog posts on your home page? Does it really make sense for each BlogPost component to separately retrieve its own post data from the store? Nope, not really.

Consider that the owner component (BlogRoll, let's say) has to pass down an `id` prop in order for BlogPost to work properly. Where do you think BlogRoll is going to get that id from? The store, of course. Now you have BlogRoll *and* each of its children getting data from the store, each with their own event listeners, and each calling `setState()` every time the store changes. D'oh!

A better approach is to separate the data fetching logic from the logic of rendering the post. Instead of having a prop `id`, BlogPost should have a prop `post`. It shouldn't concern itself with how the data is retrieved — that's the concern of the owner.

After we rewrite BlogPost, it looks something like this:

```js
let BlogPost = React.createClass({
  render() {
    <article>
      <h1>{this.props.post.title}</h1>
      <div>{this.props.post.content}</div>
    </article>
  }
});
```

And its owner looks something like this:

```js
let BlogPostPage = React.createClass({
  mixins: [FluxMixin({
    posts: function(store) ({
      post: store.getPost(this.props.id),
    })
  })],

  render() {
    <div>
      <SiteNavigation />
      <MainContentArea>
        <BlogPost post={this.state.post />
      </MainContentArea>
      <SiteSidebar />
      <SiteFooter />
    </div>
  }
})
```

*For the sake of this example, let's just assume the `id` prop magically exists and is derived from the URL. In reality, we'd use something like React Router's [State mixin]( https://github.com/rackt/react-router/blob/master/docs/api/mixins/State.md).*

There's another problem, though. Every time the store changes, FluxMixin calls `setState()` on BlogPostPage, triggering a re-render of the *entire* component.

Which brings us to the final point...

State is evil
-------------

Once you've grokked the basics, this is perhaps the most important thing to know about React. To [think in React](http://facebook.github.io/react/blog/2013/11/05/thinking-in-react.html) is to find the minimal amount of state necessary to represent your app, and calculate everything based on that. This is because state is unpredictable. Props are, for the most part, derived from other props and state, but state can be anything. The more state in your application, the harder it is to reason about it. As much as possible, state in React should be an implementation detail — a necessary evil, not a crutch.

On an even more practical level, every time the state of a component changes, the entire component sub-tree is re-rendered. In our example from the previous section, BlogPostPage updates every time the `posts` store changes — including SiteNavigation, SiteSidebar, and SiteFooter, which don't need to re-render. Only BlogPost does. Imagine if you're listening to more than just one store. The problem is compounded.

Alright, so we need to refactor once again so that FluxMixin is only updating what needs to be updated. We already learned that we shouldn't put the mixin inside BlogPost itself, because that makes the component less reusable. Our remaining option is to create a new component that wraps around BlogPost:

```js
let BlogPostWrapper = React.createClass({
  mixins: [FluxMixin({
    posts: function(store) ({
      post: store.getPost(this.props.id),
    })
  ]

  render() {
    <BlogPost post={this.state.post} />
  }
});
```

This works. But it's kind of tedious, right? Imagine creating a wrapper like this for every single component that requires data from a store.

Wouldn't it be great if there were a shortcut for this pattern — a convenient way to update specific parts of your app, without triggering unnecessary renders?

Yep! It's called FluxComponent.

```js
let BlogPostPage = React.createClass({
  render() {
    <div>
      <SiteNavigation />
      <MainContentArea>
        <FluxComponent key={this.props.postId} connectToStores={{
          posts: store => ({
            post: store.getPost(this.props.postId),
          })
        }}>
          <BlogPost />
        </FluxComponent>
      </MainContentArea>
      <SiteSidebar />
      <SiteFooter />
    </div>
  }
});
```

Do what's right
---------------

If I'm leaving you unconvinced, just do what you feel is right. I think components are generally preferable to mixins, but as with any rule, there are exceptions. For instance, [React Tween State](https://github.com/chenglou/react-tween-state) is a great project that wouldn't make sense as a component.

Either way, both FluxMixin and FluxComponent are available for you to use, and both are pretty great :)

If you have any suggestions for how they could be improved, please let me know by submitting an issue.
