# Motivation

The motivation for Elementary primarily rests on two ideas. The first is that a
functional, declarative style of writing our audio processing algorithms offers
a more intuitive way of reasoning about our applications, facilitating a faster
development timeline and delivering more resilient software. In the functional,
declarative style, we can focus on *what* our application should sound like, as
a function of our application state, while largely delegating the process of
*how* to get it done to the framework. The second idea follows from an
observation in the audio software industry. The conventional process by which
we experiment, prototype, and iterate often differs enough from the process by
which we develop a production version of the same app that we end up with
nearly twice the work. This redundancy complicates the process from
ideation to ship, costs expensive time, and is an issue that I think we can
address with an updated approach to our problem domain.

## Functional, Declarative Style

Consider an audio application where the user interacts with a dynamic set of
processors, whether that’s assembling a unique effects chain, reordering a
fixed chain, or routing modulators to various parameters in the system. As the
user interacts with the system, there are state changes that we need to address
at each step. Our code might frequently consider: what is the current state of
our audio graph? Which nodes should I keep? Which can I reuse? Which should I
delete? Which edges should I break, and which should I create? And how do I
handle all of this while being thread safe, being memory safe, and preserving
the continuity of the output signal?

This same problem rears its head in many different contexts, even inside a
fixed audio graph if any of our state is external. Consider sequencing a
synthesizer while respecting the playhead position of the DAW hosting our audio
process. Our code constantly must ask, where is the playhead now? Does it match
what I expected? Should my synth voices be playing? Which ones? Which ones
should I engage now? Or disengage now?

The functional style that Elementary adopts asks instead that you consider one
simpler question: given your current application state as plain old data, what
is your expected audio process?

```jsx
function synthVoice(voice) {
  let env = el.adsr(4.0, 1.0, 0.4, 2.0, voice.gateRef);

  return el.mul(
    env,
    el.add(
      el.blepsaw(el.sm(voice.freqRef)),
      el.blepsaw(el.sm(el.mul(voice.detuneMultiplier, voice.freqRef))),
    ),
  );
}

function describe(appState) {
  let drySynth = el.add(...appState.voices.map(synthVoice));
  let wetSynth = el.lowpass(appState.cutoff, appState.reso, drySynth);

  return wetSynth;
}
```

In this model, we’re not asking ourselves *how* to reconcile any differences in
a running audio graph, we’re asking ourselves simply, what should I expect to
hear, right now? As the user continues to interact with the application, our
mental model doesn’t change; we just ask the same question again. Elementary’s
central design goal is to provide a strong, generic solution to the state
transition matrix that lies between your description of your expected audio
process at a given point in time, and all necessary manipulation of the
underlying running audio graph. That means that as an application developer,
you no longer concern yourself with the complexity that arrises from these
state transitions, you can simply focus on *what* you expect to hear.

<center>

![](/img/motivation/ElemFlux.svg)

</center>

As a result, our application lifecycle breaks down into a simple, high-level
flow. With each input event, we resolve our new app state. Typically this is a
function that accepts the current app state and some payload representing the
input event, and produces the new app state. Next, we take the app state as
input to a second function which describes what we expect to be hearing.
Finally, we hand the description to Elementary’s `render` function to do the
rest of the work, and then we wait for the next input event and the cycle
repeats.

There is a sharp contrast between this style of writing and the conventional
style of writing audio processes, and while it may take some getting used to,
the functional, declarative model delivers a faster development process that
yields code that’s simple to reason about, and easy to change.

## From Prototype to Shipping

The second motivating idea behind Elementary comes from first-hand experience
with and within teams that effectively build their audio applications twice:
once in the prototyping phase, and then again from the ground up for shipping a
final product.

From one perspective, this makes sense. Digital audio is a complicated domain
even if we consider just the math and logic involved in designing signal
chains. Then of course we have to consider all of the complexity we take on to
realize these processes in modern software: multi-threaded concurrency,
lock-free data structures, careful memory management, strictly deterministic
behavior on real-time threads, audio signal continuity, etc. Tools like
Max/MSP, PureData, Reaktor, ChucK, SuperCollider, and others already offer
environments that allow us to work with signals *without* having to think about
that additional software complexity. Perfectly sound reasoning would suggest
using such a tool to prototype. Historically, the consequence of prototyping
this way is that no part of the prototype can be reused in the final product
because these various environments don’t easily embed and don’t easily export.
So, teams rewrite from scratch to build a production version of the prototype
in the target environment– an audio plugin, embedded device, web browser, etc.

In recent years the landscape has evolved, and continues to do so: libpd showed
up as a way of embedding PureData patches in target applications, Max/MSP
introduced gen~ for exporting code and RNBO for exporting complete apps, and
new projects like CMajor have come into play to offer targeted, embeddable DSLs
(domain specific languages) for audio processing. These are all obvious
improvements but imperfect solutions. Integration complexity still remains with
clunky state management between the app and the embedded component, a lack of
portability, and a different toolset for the different parts of the app.

Elementary takes the stance that there’s still room to improve, and it aims to
do exactly that by providing a model for writing audio software that feels as
fast and intuitive as any prototyping environment, yet produces
production-ready code. Further, Elementary leans on JavaScript because of its
ubiquity and ease of integration in various contexts, along with its popularity
as a choice for writing user interfaces. That means that in an Elementary
application, all of your state management, your user interface, and your audio
processing all share the same language, same environment, and same mental
model.

Ultimately, these two motivating factors lead to the same end. Elementary aims
to make the process of writing audio software faster and more intuitive while
producing more resilient code.
