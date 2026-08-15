---
title: "Inforill"
role: "Intern"
period: "May 2023 – Feb 2024"
location: "Chennai, India · On-site"
images:
  - "/images/work/inforill-epoch.png"
  - "/images/work/inforill-rive.png"
draft: false
---

Two separate internships at Inforill Technologies, nine months apart, in two different disciplines.

## Machine Learning Intern

**Dec 2023 – Feb 2024 · Part-time**

Reinforcement learning and deep neural networks.

PPO agents trained past a million timesteps, DQN on Atari environments, CNNs on CIFAR-10, and enough transformer reading to understand what attention is doing.

The training log above is one of those runs at the point where it's working: explained variance at 0.995, approximate KL at 0.0037, entropy loss still negative, episode reward mean climbing. Most of the job is learning to read that table and tell the difference between a run that is converging and one that has quietly collapsed into a single action.

The notebooks, models and scripts are in [SBC](/projects/sbc/).

## Graphic Design Intern

**May 2023 – Jul 2023 · Full-time**

State machines for [Rive](https://rive.app) animations, for the [Hashigo](/projects/hashigo/) Japanese e-learning app.

Rive doesn't do clips. An animation is a state machine: states, transitions, and the inputs that trigger them. The runtime plays whatever the current state says. So an interactive animation is designed as a graph before anything gets drawn.

The page of boxes and arrows above is that graph being worked out on paper: three zones (component, loading, dispatch) and the states moving between them. Idle, selected, drag, valid dropping, invalid dropping, dropped, snap back. Down the right-hand margin, a note to check whether the vibrator can be read as an input.

Working these out on paper first sounds like an odd way to make animation. It stops being odd the first time you build one straight into the tool and can't work out why it's stuck in a state you didn't know it could reach.
