+++
title = "Covering your components with P"
date = "2022-10-02"
in_search_index = true
+++
P is a [programming language](https://www.microsoft.com/en-us/research/blog/p-programming-language-asynchrony/) designed to model other computer programs. 
It's similar to TLA+ in that you are working at a much higher level of abstraction than the final program. P provides a formal specification of the program
or protocol that is helpful both as a thinking aid and a mechanism for checking correctness.
This helps you find bugs _very_ early, when you're still thinking about how you are going to design something.
It can also serve as a description of what the code you are modeling is _supposed to be doing_ when you or someone else comes back to it later. P diverges from TLA+ in two important ways:

1. TLA+ is more about proving correctness and will search the entire possible state space if you let it. P will search a limited
subspace of possible states.
2. I can actually understand what's going on in a P program.

No offense to Leslie Lamport on this one, but TLA+'s way of describing a system is less approachable to me since I don't
usually think about the code I'm writing in mathematical terms. I don't think I'm alone in finding the syntax and tools to be abstruse. TLA+ added PlusCal to give a different syntax
that might be more approachable but having to essentially write it as a giant code comment in a TLA file makes it feel very second-class. I also picked up _Practical TLA+: Planning Driven Development_
as an alternative source for learning TLA+ and found that the book had to include a substantial module of their own to make working with the built-in data types easier. I'm on board with
wanting a way to describe my programs that can help me find design bugs but TLA+ has been an uphill battle. I still highly recommend you check out the [video series](https://www.youtube.com/channel/UCajiu4Cj_GHOX0if3Up-eRA)
on TLA+ at some point, it might spark some interest for you. It's been hugely influential in proving the correctness of distributed protocols like Paxos.

P takes a more familiar approach to modeling that is based around state machines communicating with events. You declare a set of machines, their states, the events in the system, and start filling in the
state machine definitions. Events are asynchronous and the mailboxes of machines are unbounded. You can declare new instances of machines at runtime. The model checker will explore many combinations of
states and events that you describe as valid while looking for issues. You provide test machines that assert correctness and liveness properties that the checker uses to check the validity of your spec as it runs.

When I'm thinking about my programs at a high level, actors passing messages is a natural abstraction for me.
In many ways that's object oriented code described in the simplest terms, hence Smalltalk. If any of this sounds a bit like Erlang/Elixir to you, we'd probably be friends.

You can find all the spec code for this post [here](https://github.com/mlh758/blog_p_spec) if you want to follow along with the final version.

## Boring, workaday uses

As you might have assumed from the name of this website, I end up writing a lot of React components for my day job. In many ways, a React component functions as a tiny distributed system. If you call out to any services in your
component you have at least 3 actors in your system:

1. A user who can decide to do just about anything at any given moment
2. Your JavaScript which has to respond to user events and communicate with a remote service
3. Whatever that remote service is which hopefully gives you the data you want, might return errors, and might be behind a firewall that decides to quietly
drop all the requests you're sending without providing a response.

Taking a little time to think through how those actors communicate, what states your code might pass through to provide the user meaningful feedback, and how you handle the various responses
from the remote service, can save you a lot of headache later. Starting with an awareness of most of those issues will give you more legible code in the end and your future self or coworkers
will probably be grateful.

It might seem like a lot of work to start with a P model before writing code but it can give you quick feedback on if you've accounted for all the scenarios you care about and the states you
end up with in a P program are likely to correspond nicely with the states of the final component. This is especially true if you use TypeScript and use union types in your component state.

## How to P

The [official docs](https://p-org.github.io/P/tutsoutline/) are of course your best resource but here's a crash course so you can follow along.

* P programs are collections of P source files like `whatever.p`.
* Your life will be easier if you create a `something.pproj` file that defines the directories that contain the source files. [example](https://github.com/p-org/P/blob/438182d2689c608b961594c862e7f7874382ab07/Tutorial/1_ClientServer/ClientServer.pproj)
* You build a P program with `pc -proj:something.pproj` which will dump a bunch of stuff in a POutput directory.
* Run the spec with `coyote test ./POutput/dotnetnonsense/something.dll` where `dotnetnonsense`
is the dotnet version it's working with (probably netcoreapp3.1) and `something.dll` is your project name. It's `.dll` even on Mac and Linux. Microsoft don't care.
* Most programs will at least have a ./PSrc directory with the definitions of your state machines and a ./PTst file that defines the different test cases.
* The simplest test case that works looks is basically what this blog post is going to be running on. It'll look something like this:
`test ReactComponent [main=User]: {User,Component};`. It's the `test` keyword followed by a name for the test that you can provide.
`[main=<...>]` is saying "the main state machine that kicks everything off is ...". Then you provide the list of definitions needed. This is likely to just be the list of machines used
in the test but there is also a module system to substitute complex machines for simple substitutes if you need to limit the state search space.

In the official docs you can find a more complex test set definition [here](https://github.com/p-org/P/blob/438182d2689c608b961594c862e7f7874382ab07/Tutorial/1_ClientServer/PTst/Testscript.p) where
test machines are defined that provide different scenarios to run through. This also shows the module/substitution system in action.

Machines communicate by passing events asynchronously. There are mechanisms for non-determinism which provide branching points in the state space to be searched. These arbitrary decision points are like a user
rampaging through all the weird edge cases in your application as fast as they can and are really helpful for uncovering subtle concurrency issues. Where a unit or integration test defines a specific scenario
and makes assertions about it, a model defines the scope of possible actions. The spec then defines properties that must hold while the system executes. Tests in this way are much more exploratory.

Correctness can be enforced _within_ your state machines with the `assert` keyword.
You can also provide monitor machines defined as `spec SomeMachineName observes e1, e2, e3... {}`. These are also just state machines but they can see events other machines are sending and assert errors too.
These monitor machines assert liveness by entering a state annotated with `hot`. If a monitor stays in a hot state for too long, the test will fail with a liveness error. Hot states ensure your program is making
progress rather than just passing by ignoring all the events.

### The most basic example

At my day job there was recently a project to create a React component that accepts a user's IBAN (International Bank Account Number) and extract the BIC (Business Identifier Code) while
validating that the IBAN is structurally correct. There is an IBAN API that provides this information so when the user enters an IBAN into the input, send that to the API and don't let them submit the form
with a bad IBAN. If the IBAN API is misbehaving, we should fail open and just use a basic format validation while letting the user enter their BIC manually. I didn't implement this component,
and this post is in no way a criticism of the implementation. This simplified description happens to be _just complicated enough_ to make an informative P program without being _so complicated_
that I give up on writing this blog post.

The user is basically the driver of our fledgling test system. They interact with the system by typing into the input. We're assigning an ID to their messages so we can assert on the order later. Instead of actually sending some text, let's just say whether it's long enough to validate or not. Here's the User machine:

```
machine User {
  var component: Component;
  var reqId: int;
  start state Init {
    entry {
      component = new Component();
      reqId = 0;
      goto Typing;
    }
  }

  state Typing {
    on null do {
      var longEnough: bool;
      // $$ means arbitrarily choose one or the other but eventually try both branches
      if($$) {
        longEnough = true;
      } else {
        longEnough = false;
      }
      send component, eOnInput, (id = reqId, longEnough = longEnough);
      reqId = reqId + 1;
    }
  }
}
```

The `start` state here initializes the user with a component to interact with and then sets them off to type into it. The typing state is randomly choosing whether to send a plausible IBAN which should trigger validation.

Next up let's define the server the component will talk to. It receives a validation request and does one of 4 things:

1. Returns a status indicating the IBAN is valid
2. Returns a status indicating the IBAN is _not_ valid
3. Returns an error
4. Drops the request on the floor and never responds (skipped for brevity)

Here's the definition:

```
machine Server {
  start state Serving {
    on eValidationRequest do (req: ValidationRequest) {
      if ($) {
        if ($) {
          send req.comp, eValidationResult, (id = req.id, valid = true);
        } else {
          send req.comp, eValidationResult, (id = req.id, valid = false);
        }
      } else {
        send req.comp, eValidationError, req.id;
      }
    }
  }
}
```

Notice this uses `$` instead of `$$`. This is _unfair_ indeterminism which means it might _never_ return a valid response or _always_ return an error in response to requests.

Alright, it's time to start iterating on the actual component! We're going to start with something that's obviously wrong and build it up. I want to illustrate some of the errors the model checker can show you.


```
machine Component {
  var server: Server;
  start state Init {
    entry  {
      server = new Server();
      goto Waiting;
    }
  }
  state Waiting {
    on eInput do (input: InputMessage) {
      if (input.longEnough) {
        send server, eValidationRequest, (comp = this, id = input.id);
        goto Validating;
      }
    }
  }
  state Validating {
    on eValidationResult do (result: ValidationResult) {
      if (result.valid) {
        goto Valid;
      } else {
        goto Invalid;
      }
    }
  }
  state Valid {
    ignore eInput;
  }
  state Invalid {
    ignore eInput;
  }
}
```

When `Component` initializes it creates a server to talk to then transitions to waiting. If it receives a message that could plausibly be an IBAN it calls out to the server by requesting validation.
If it's valid, go to the valid state and likewise with invalid. For now those states ignore all further input which represents the user being stuck. What happens when we run this?

Running with: `coyote test ./POutput/netcoreapp3.1/iban.dll`

We get:

```
...
<SendLog> 'PImplementation.User(2)' in state 'Typing' sent event 'eInput with payload (<id:0, longEnough:True, >)' to 'Component(3)'.
<DequeueLog> 'Server(4)' dequeued event 'eValidationRequest with payload (<comp:Component(3), id:0, >)' in state 'Serving'.
<DequeueLog> 'Component(3)' dequeued event 'eInput with payload (<id:0, longEnough:True, >)' in state 'Validating'.
<StateLog> Component(3) exits state 'Validating'.
<PopLog> 'Component(3)' popped with unhandled event 'eInput with payload (<id:0, longEnough:True, >)' and reentered state 'Validating.
<ExceptionLog> Component(3) running action '' in state 'Validating' threw exception 'UnhandledEventException'.
<ErrorLog> Component(3) received event 'PImplementation.eInput' that cannot be handled.
```

So we were in the `Validating` state and the user began typing again. Yup, we should probably account for that. Let's try running it again and see what we get.

```
...
<ExceptionLog> Component(3) running action '' in state 'Validating' threw exception 'UnhandledEventException'.
<ErrorLog> Component(3) received event 'PImplementation.eValidationError' that cannot be handled.
```

Ah right, we also weren't handling the error case. (I actually had to run it two more times for this to pop up)

These may have been obvious to you given how brief the spec was, but that's kind of the point. Without the clutter of implementation it's pretty clear that there are still events we need to handle.
Starting with this, you have a better baseline for what to write. Let's keep going.

### Handling the first errors

When the more comprehensive remote validation fails, we should fall back to the less reliable local validation. Let's add states for that now.
I'm going to rename the `longEnough` field in the `eInput` event to `maybeValid` so I can be lazy about the `User` machine. In a more comprehensive spec you
might want to send an event to the user telling them to do manual BIC entry and change from the basic `Typing` state to a new `EnteringBoth` state.
In this case I'm just going to have the component change over to a `WaitingManualBic` state and have `maybeValid` represent receiving a valid IBAN and BIC in that state.
It's not quite as descriptive but it keeps the post focused on evolving the component spec. Let's make those updates now.

```
state Validating {
  // ... existing eValidationResult handler was here
  on eValidationError do {
    goto WaitingManualBic;
  }
}

state WaitingManualBic {
  on eInput do (input: InputMessage) {
    if (input.maybeValid) {
      goto LocalValidationSuccess;
    }
  }
}
state LocalValidationSuccess {
  ignore eInput;
}
```

The validating state now picks up the `eValidationError` and transitions to the state to handle manual BIC entry.
If we get an input that is validated locally, transition to a new termination state showing that we have validated the input locally.

Let's try dealing with the error where we received input during the validation process.

```
state Validating {
  // ... existing eValidationResult and eValidationError handler was here
  on eInput do {
   goto Waiting;
  }
}
```

Simple enough, right? Just go back to listening for input if the user starts typing again. This error gets a little more complicated since we've got our first race condition:

```
<SendLog> 'PImplementation.User(2)' in state 'Typing' sent event 'eInput with payload (<id:0, maybeValid:False, >)' to 'Component(3)'.
<DequeueLog> 'Component(3)' dequeued event 'eInput with payload (<id:0, maybeValid:False, >)' in state 'Waiting'.
<StateLog> Server(4) enters state 'Serving'.
<SendLog> 'PImplementation.User(2)' in state 'Typing' sent event 'eInput with payload (<id:1, maybeValid:True, >)' to 'Component(3)'.
<DequeueLog> 'Component(3)' dequeued event 'eInput with payload (<id:1, maybeValid:True, >)' in state 'Waiting'.
<SendLog> 'Component(3)' in state 'Waiting' sent event 'eValidationRequest with payload (<comp:Component(3), id:1, >)' to 'Server(4)'.
<GotoLog> Component(3) is transitioning from state 'Waiting' to state 'Validating'.
<StateLog> Component(3) exits state 'Waiting'.
<StateLog> Component(3) enters state 'Validating'.
<SendLog> 'PImplementation.User(2)' in state 'Typing' sent event 'eInput with payload (<id:3, maybeValid:True, >)' to 'Component(3)'.
<DequeueLog> 'Component(3)' dequeued event 'eInput with payload (<id:2, maybeValid:True, >)' in state 'Validating'.
<GotoLog> Component(3) is transitioning from state 'Validating' to state 'Waiting'.
<StateLog> Component(3) exits state 'Validating'.
<StateLog> Component(3) enters state 'Waiting'.
<DequeueLog> 'Component(3)' dequeued event 'eInput with payload (<id:3, maybeValid:True, >)' in state 'Waiting'.
<DequeueLog> 'Server(4)' dequeued event 'eValidationRequest with payload (<comp:Component(3), id:1, >)' in state 'Serving'.
<SendLog> 'Server(4)' in state 'Serving' sent event 'eValidationError' to 'Component(3)'.
<SendLog> 'Component(3)' in state 'Waiting' sent event 'eValidationRequest with payload (<comp:Component(3), id:3, >)' to 'Server(4)'.
<GotoLog> Component(3) is transitioning from state 'Waiting' to state 'Validating'.
<StateLog> Component(3) exits state 'Waiting'.
<StateLog> Component(3) enters state 'Validating'.
<DequeueLog> 'Component(3)' dequeued event 'eValidationError' in state 'Validating'.
<GotoLog> Component(3) is transitioning from state 'Validating' to state 'WaitingManualBic'.
<StateLog> Component(3) exits state 'Validating'.
<StateLog> Component(3) enters state 'WaitingManualBic'.
<DequeueLog> 'Server(4)' dequeued event 'eValidationRequest with payload (<comp:Component(3), id:3, >)' in state 'Serving'.
<SendLog> 'PImplementation.User(2)' in state 'Typing' sent event 'eInput with payload (<id:4, maybeValid:True, >)' to 'Component(3)'.
<SendLog> 'Server(4)' in state 'Serving' sent event 'eValidationError' to 'Component(3)'.
<DequeueLog> 'Component(3)' dequeued event 'eInput with payload (<id:4, maybeValid:True, >)' in state 'WaitingManualBic'.
<GotoLog> Component(3) is transitioning from state 'WaitingManualBic' to state 'LocalValidationSuccess'.
<StateLog> Component(3) exits state 'WaitingManualBic'.
<StateLog> Component(3) enters state 'LocalValidationSuccess'.
<DequeueLog> 'Component(3)' dequeued event 'eValidationError' in state 'LocalValidationSuccess'.
<StateLog> Component(3) exits state 'LocalValidationSuccess'.
<PopLog> 'Component(3)' popped with unhandled event 'eValidationError' and reentered state 'LocalValidationSuccess.
<ExceptionLog> Component(3) running action '' in state 'LocalValidationSuccess' threw exception 'UnhandledEventException'.
<ErrorLog> Component(3) received event 'PImplementation.eValidationError' that cannot be handled.
```

Here's a summary of the error:

1. User sends a maybe valid response
2. Component requests validation from the server and transitions to validating
3. User types some more, we go back to waiting.
4. User sends some potentially valid input, component requests validation and goes back to validating
5. Component get the _first_ response for the validation request from the server and transition to manual BIC entry.
6. User sends valid input, component goes to the local validation success state.
7. Component receives the _second_ response from the server and doesn't know what to do with it.

This is basically what happens when you have `fetch` in a `useEffect` and you don't abort the requests when they're
obsolete. You could get out of order responses, or receive your response after the component has unmounted yielding the classic
error about setting state on an unmounted component. Let's solve this the wrong way first and introduce spec machines. The easiest way
to not have an unhandled message is to ignore the message!


```
state Valid {
  ignore eInput, eValidationError, eValidationResult;
}
```

Invalid, LocalValidationSuccess, WaitingManualBic are handled similarly. Let's run the test again. We should see something like this:

```
... Testing statistics:
..... Found 0 bugs.
```

### Testing Liveness

We know this isn't right though. Let's add a spec machine that asserts whatever response we show the user corresponds to their most recent input. In TestScript.p:

```
spec Liveness observes  eInput, eDone {
    var lastRequest: int;
    start cold state Init {
        on eInput do (input: InputMessage) {
            HandleInput(input);
            goto AwaitingValidation;
        }
    }

    hot state AwaitingValidation {
        on eInput do HandleInput;
        on eDone do (id: int) {
            if (id == lastRequest) {
                goto Init;
            }
        }
    }

    fun HandleInput(input: InputMessage) {
        lastRequest = input.id;
    }
}
```

A spec machine states what events it observes. Here we transition to a `hot` state when an input event is sent while we're in the `Init` state.
If we stay in a hot state too long, we get a liveness bug. If our most recent request gets validated (`eDone` sent with our latest ID) then we go back to the cold state.
There's another new concept in this block of code. You can define functions and pass them directly as handlers for events.
Update our `test` as well:

```
test ReactComponent [main=User]: assert Liveness in {User,Component,Server};
```

Finally we'll need to update `Component` to `announce` when it has finished validating a request.

```
state Validating {
  on eValidationResult do (result: ValidationResult) {
    announce eDone, result.id;
    ...
  }
  on eValidationError do (id: int) {
    announce eDone, id; 
    goto WaitingManualBic;
  }
}

...

state WaitingManualBic {
  on eInput do (input: InputMessage) {
    if (input.maybeValid) {
      announce eDone, input.id;
      ...
    }
  }
}
```

If we run the tests now, we can see that we're still doing the wrong thing:

```
<MonitorLog> PImplementation.Liveness is processing event 'PImplementation.eInput' in state 'AwaitingValidation'.
<ErrorLog> PImplementation.Liveness detected potential liveness bug in hot state 'AwaitingValidation'.
```

The monitor is stuck in `AwaitingValidation` because the latest request from the user never gets validated. Our termination states
are also now clearly problematic. Our component will ignore any further input from the user if it's reached a decision about what they typed.
It's clearly not helpful to only give them one shot at typing something in. Time for the final round of updates. There's a few things we'll be doing:

1. When we complete local or server validation, announce to the monitor that we finished looking at that input.
2. Only deal with the server's message if it's for our latest request.
3. The Valid, Invalid, and LocalValidationSuccess states need to respond to input and start the validation cycle over again.

First off, let's define a couple functions to make this easier:

```
fun ServerValidate(input: InputMessage) {
  currentInput = input;
  send server, eValidationRequest, (comp = this, id = input.id);
}

fun HandleInput(input: InputMessage) {
  if (input.maybeValid) {
    ServerValidate(input);
    goto Validating;
  } else {
    currentInput = input;
    goto Waiting;
  }
}
```

HandleInput kicks off server validation if necessary or drops us back into the waiting state. Both branches update our local request ID with the latest from the user.
This would be aborting your `AbortController` for `fetch` or otherwise canceling a pending async task in the React implementation. Valid and Invalid are trivial now:

```
state Valid {
  on eInput do HandleInput;
}
state Invalid {
  on eInput do HandleInput;
}
```

After a server error we enter local validation. Since it happens locally we don't have to worry about race conditions. `WaitingManualBic` just needs to announce when it's valid. `LocalValidationSuccess`
needs to go back to the waiting state if it receives invalid input.

```
state LocalValidationSuccess {
  on eInput do (input: InputMessage) {
    if (input.maybeValid) {
        announce eDone, input.id;
    } else {
      goto WaitingManualBic;
    }
  }
}
```

At this point, the code in GitHub for the spec matches this state. It might be clearer if you look at the `iban.p` file to see the final state if you're wondering about any of the individual pieces.
Running the tests shows our spec passing:

```
Starting TestingProcessScheduler in process 7592
... Created '1' testing task.
... Task 0 is using 'random' strategy (seed:654229532).
..... Iteration #1
... Testing statistics:
..... Found 0 bugs.
... Scheduling statistics:
..... Explored 1 schedule: 1 fair and 0 unfair.
..... Number of scheduling points in fair terminating schedules: 100000 (min), 100000 (avg), 100000 (max).
..... Exceeded the max-steps bound of '10000' in 100.00% of the fair schedules.
... Elapsed 2.1274284 sec.
```

At this point we're handling Liveness in that if the user types something they'll eventually get a response from the component. We're also handling correctness in that we're asserting the response they get is for their most recent input.

### What about the other server failure case?

The Server implementation is only sending Valid, Invalid, and Error. I haven't written a condition for not sending any response at all or for out of order responses.
To do that we could store a set of requests in the Server as they come in and use the `on null` pattern to choose a request and either respond to it or drop it entirely.

To handle that, we would need a new `Timer` machine. When `Component` sends a request to the server it starts the timer. The timer would then send an `eTimeout` message back to
the component and turn itself off. The official docs have an example for this. On the component side, we'd consider the timeout to be an error and abandon that request.

## Wrapping Up

That covers a lot of the features of P! I've always found formal verification to be a bit intimidating but I was able to learn P while writing this post and I'm happy with how it went.
There's definitely some room for improvement in the spec but I'm even more convinced now that at a minimum this can be a useful thinking aid while working over a problem. Here are some additional resources:

* [The code on GitHub](https://github.com/mlh758/blog_p_spec)
* [The official docs](https://p-org.github.io/P/)
* [Microsoft's P announcement](https://www.microsoft.com/en-us/research/blog/p-programming-language-asynchrony/)
* [Blog summarizing P that I found helpful for referencing keywords](https://www.mydistributed.systems/2021/06/p-language.html)