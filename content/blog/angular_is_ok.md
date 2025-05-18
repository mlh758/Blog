+++
title = "Modern Angular is good, actually"
date = "2025-05-12"
in_search_index = true
+++

## How I ended up on React

My first full-time job as a developer was at a healthcare IT company called Cerner.
They had a bespoke frontend framework built around jQuery and some fairly complex
prototype overrides. The backend was a proprietary reporting language called CCL. Maven was
used as the package manager and build tool.

It was quite serviceable considering the amount of custom work that went into it. Those
tools were a major investment by the company. Even still, compared to some of the open
source tools that were becoming available at the time, the developer experience was lacking.
Even basic things like building and previewing your changes could be a bit of a process.

At one point I was asked to contribute to some applications that were written by a 
different team at Cerner. Insetad of being built around this custom UI stack they were
built using AngularJS. I found the documentation abstruse, especially as a junior
developer. There were a _lot_ of features that needed to be understood from the framework
and some confusing modifications made by the codebase. Even still, when compared to what I
had been working with on other projects, it could be enticingly predictable.
Another AngularJS project I worked on later was small enough for me to upgrade it to 1.5
and start making use of components, which simplified things even more.

Not long after, my team was tasked with some greenfield work and I advocated for using AngularJS. I liked the structure it imposed and was already familiar with it.
My team wanted to use React since it seemed simpler to learn and get started. I lost that
vote and it was absolutely the right call. The Angular 2 split happened right around that
time which from the outside seemed like a Python 2 vs 3 moment that nobody on my team
wanted to deal with. React's JSX syntax was much lighter and more straightforward than
the template syntax offered by Angular at the time. The impression you got using it was
that it was _just JavaScript_. For the past ten years I've been generally enjoying it.
Especially once function components and hooks streamlined much of the development flow.

In the meantime, I've tinkered with tools like Elm and Blazor to get a feel for
other ideas people have been working on, but I've been content enough with React that I
haven't felt a compelling reason to look outside the ecosystem for serious projects.

Recently, however, I've been frustrated with the direction of React and the ecosystem.
There's been a lot of push towards NextJS in an effort to optimize specific metrics in a
way [that is likely actually harming the user experience.](https://danluu.com/slow-device/).
Many people also interpreted the post about not needing [effects](https://react.dev/learn/you-might-not-need-an-effect) as the hook being too dangerous to use.
Much of the current direction of React feels like an on-ramp to hosted services that 
benefit Vercel.

Lately this has coalesced into a trend of gluing ever more tools together into a variety of
ad hoc frameworks. Even basic operations like calling `fetch` had become taboo to do
directly. Everything needed to be inside of a custom hook, preferably from a third party.
Performance issues and weird bugs related to that have piled up as people forget to
stabilize the outputs of their hooks, neglect cancellation, omit dependencies, and
muddle the ownership of state trying to appease form libraries.

## If we need a framework anyway...

The awkwardness of fitting so many tools together, papering over their somewhat incompatible
APIs, upgrade lifecycles, and various other idiosyncrasies had me thinking back to my Angular
days and if it may have improved somewhat since the comparatively awkward experience I had
with 1.5.

Here are some highlights of tools that come with Angular:

* Server-side and hybrid rendering
* Translation and localization tools
* HTTP client that meshes well with the framework
* Test suite
* Routing
* Forms
* Authorization via route guards and resolvers
* HTML/CSS/URL sanitization tools beyond just template safety
* CLI generators for components, services, etc.
* Services to manipulate title/meta tags similar to React Helmet
* Lazy loading components and specific elements similar to Suspense and `lazy`.

Some of the old criticisms of Angular still ring true today. There is a _lot_ to learn in 
the Angular framework. The documentation has improved immensely over the years and [angular.dev](https://angular.dev) is a great resource. I still think it would benefit from more
complete examples. Many examples that do exist reference older patterns that are
being phased out in favor of the newer signals-based APIs.

I felt strongly enough about this that I ended up building a [sample app](https://github.com/mlh758/angular-todo) around as many of the new concepts as possible. If I get
stuck on a concept at work, I have been coming back to that sample to implement some
version of it to supplement the documentation. Please feel free to open issues or otherwise
do what you wish with it if you find it useful.

## Angular from a React Dev's Perspective

Angular's component model should be immediately familiar to anyone who has used React. Most
of the same concepts still apply. The idea of "smart components" and "dumb components",
choosing which component should own that state, and even some of the same ideas about
what sort of values will be computed on every render still apply.

[Here](https://github.com/mlh758/angular-todo/blob/d69d670b7639e888d35c2a1829495bfa6090a661/src/app/components/button/button.component.ts) is a basic button component:

```
@Component({
  selector: 'app-button',
  imports: [],
  templateUrl: './button.component.html',
  styleUrl: './button.component.css',
})
export class ButtonComponent {
  disabled = input(false, { transform: booleanAttribute });
  variant = input<Variant>('primary');
  type = input<ButtonType>('button');
  onClick = output<MouseEvent>();

  handleClick(event: MouseEvent): void {
    if (!this.disabled) {
      this.onClick.emit(event);
    }
  }
}
```

and here is the template:

```
<button 
  class="button"
  [class.secondary]="variant() === 'secondary'"
  [disabled]="disabled()"
  [type]="type()" 
  (click)="handleClick($event)"
>
  <ng-content />
</button>
```

`input` is analogous to React's props. You can also do `input.required` to ensure
it isn't nullable. Additionally, you can pass them transformer functions that will loosen
the input types but ensure you still get the desired type within the component. For example,
parsing a string to a date.

`output` is how a component communicates to the parent component. Think of it as a slightly
more ergonomic callback function. You can have an `output` of any type and pass that data
via `emit`. The parent would have something like `(onClick)=someHandler($event)`. `$event`
is the magic keyword to bind the output to your handler in the template.

`[class.secondary]` on the template is part of the conditional CSS functionality. [classnames](https://www.npmjs.com/package/classnames) is more or less built into the
template language.

`input`, `output`, `signal`, and `computed` are all part of the [signals](https://angular.dev/guide/signals) mechanism.
Angular still primarily uses zone.js for automatic change detection in components. There
are ways to opt into manual change detection as well, but this isn't something you'll likely
need to do often. Signals offer another way to perform change detection. By accessing a
signal it creates a link to where it was called from. If a signal changes, those links
can then also be notified. This creates more targeted updates as well as handy ways
to integrate with other reactivity tools in the Angular ecosystem. I'll cover them in
more detail some other time.

`<ng-content />` is the Angular version of `children`. You can have several content outlets
however, and use selectors to match content provided in the parent to their respective
outlets in the template.

The `@Component` decorator marks the class as being an Angular component and provides
a way to customize some of its attributes. `imports` is where you put external functions
you'll be using in your template. You could declare our template inline here if you wanted
and there are a bunch of other things you can do in the decorator that I'll save for a
more detailed post.

### Dependency Injection

This is an area where Angular and React diverge quite a lot and it was something I struggled
to get my head around when I initially encountered Angular years ago. Having more experience
with the concept now from other systems, I am quite a fan of its use here in Angular.

The component constructor or the `inject` function can both provide your component with
dependencies. For example, you might inject the `ActivatedRoute` to get access to
current routing information. Often in Angular you would define API clients as services
and then inject those into components where you need them. 

Between DI and the `imports` on your component declaration, there is a little bit of
extra ceremony putting your component together. The main trade-off you get in return is
that unit testing is _much_ more straightforward. You don't have to mock imports to get
control over dependencies, you can just swap them out in the test setup. The same goes
for anything you imported for your template code.

This also means it's easier to flex implementations for different deployment environments.
Monitoring tools can be configured once in a service and injected wherever you need them.
In development you can inject a stub implementation.
The system plays well with tree shaking and injected items are created lazily to
improve startup time.

### Observables and RxJS

Most asynchronous behavior in Angular is done through observables and RxJS which come
bundled with the framework. Observables bear some surface resemblance to promises in
that they both represent a computation that will complete at a later time. However,
observables can emit many values and may never actually complete. You could represent
subscribing to events from a server as an observable, but not as a single promise.

Observables are lazy and don't compute anything until some code subscribes to it
to get the result. Subscribing and unsubscribing represent setup and teardown for
the operation. If you use the `fromEvent` operator to create an observable on
`click` events on some HTML element then subscribing will attach the listener
and unsubscribing will remove it. Similarly, with an HTTP request subscribing
submits the request and unsubscribing will abort if it is still in progress.

The RxJS library provides a suite of operators for creating, altering behavior,
and connecting observables. The facilities for composition remind me of Redux Sagas.
I've found it very straightforward to handle timeout, retry, and memoization with them.

There are also operators to turn an observable into a Promise so that you can `await` it.
I find this convenient in tests sometimes, but I'm sure there are other reasons to do so.
Because signals also represent a series of values over time, Angular provides functions
to go between observables and signals depending on what is more useful in the current context.

To use the value of an observable in a template you have a few options:

* `pipe` operators together to massage the observable into a convenient display value and
  turn it into a signal. You can just access the signal in the template. You probably want
  to include `takeUntilDestroyed` to make sure it cleans up when the component is destroyed.
  * Use the [AsyncPipe](https://angular.dev/api/common/AsyncPipe) which will handle
  subscribing and teardown for you within the template.
* [rxResource](https://angular.dev/api/core/rxjs-interop/rxResource) elegantly handles the
  scenario where you need to run an async function every time an input value changes. It
  exposes signals you can use in your templates.

My impression is that `AsyncPipe` was the default choice for quite a while. I most often
find myself using `rxResource` since it mixes well with other signals.

As a note, I wish async abstractions like `rxResource`, and [similar ones from the React ecosystem](https://tanstack.com/query/latest/docs/framework/react/guides/queries), would
better leverage the type system and use union types instead of a bunch of boolean flags. The
flags are easier in the happy path but because of that in practice I notice a lot of people
don't bother to check all the flags and instead just `?.` the value and hope for the best.
As the user of an application, I'd prefer a half-baked error state you never thought anyone
would see to an infinite loading indicator or suddenly having a bunch of interactions
inexplicably not work anymore.

### Forms

The built-in form tools remind me of [React Hook Form](https://react-hook-form.com/). So far,
they've worked well for basic forms up to some semi-complex dynamic ones. I haven't had to
build a _really_ complex form mechanism yet like I have in React so I won't speak to it too
much. I get nervous using form libraries in the React ecosystem since controlled inputs
can easily lead you to some nasty input lag if you aren't careful. I usually prefer
uncontrolled inputs in React, _especially_ as the forms get larger and more dynamic.

I haven't run into similar bottlenecks with the Angular form tools yet. I have no reason
to go against the grain of the framework, and so far it seems to be serving me well in this
regard. The API is sensible and it integrates well with your templates.

## It's come a long way

Revisiting Angular with version 19, I'm impressed with how much they have managed to take
away. The simplifications remind me of how the C# language has evolved since the
release of .NET Core around 2014. A blank slate project is much less intimidating to a
newcomer than it used to be. You don't need to know as many concepts to get off the ground,
the syntax is less cluttered, and you get a mature ecosystem of tools you can grow into
as you make progress.

Assuming Angular doesn't get buried by React out of sheer momentum, I will probably choose
to stick around in future projects.