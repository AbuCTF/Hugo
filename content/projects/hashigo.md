---
title: "Hashigo"
tagline: "Japanese e-learning app"
category: "Design"
status: "archived"
type: "design"
description: "A Japanese e-learning application built at Inforill. I did the design side: interactive state machines in Rive."
language: "Rive"
tech:
  - "Rive"
  - "2D Animation"
  - "State Machines"
github: "https://github.com/AbuCTF/HashigoDesign"
images:
  - "/images/projects/hashigo/design.jpg"
draft: false
---

Hashigo is a Japanese e-learning application built at Inforill Technologies. I worked on the design side of it, between May and July 2023.

## What I did

The target tool was [Rive](https://rive.app). Rive animations aren't clips, they're state machines. You define states, the transitions between them, and the inputs that trigger those transitions, and the runtime plays whatever the current state demands. An animation that reacts to a tap or a drag is a graph, not a timeline.

So most of the work was graph design before it was animation: what states exist, what moves between them, and what the app has to send to drive it.

## What shipped

One `.riv` file, handed over on the final commit. Everything (states, transitions, inputs, artboards) compiles into that single binary asset, which is why the diff reads `+1.46 KB` and nothing else.

That's the honest scope of it. Small artefact, most of the effort upstream of it.

## Related

The state machine sketch from this period is in the [Inforill](/work/inforill/) writeup: that page of boxes and arrows is Hashigo's transition graph being worked out on paper.
