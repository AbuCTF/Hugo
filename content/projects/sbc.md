---
title: "SBC"
tagline: "Everything I learned that winter"
category: "ML"
status: "archived"
type: "learning"
description: "The notebooks, models and scripts from my Winter 2023 internship at Inforill. Search, RL, CNNs, embeddings."
language: "Jupyter Notebook"
stars: 1
tech:
  - "PyTorch"
  - "TensorFlow"
  - "Gym"
  - "NumPy"
  - "scikit-learn"
github: "https://github.com/AbuCTF/SBC"
images:
  - "/images/projects/sbc/training.jpg"
  - "/images/projects/sbc/word-embeddings.jpg"
draft: false
---

Concepts learned as of December 2023 on the domain of artificial intelligence. Written during the winter internship at [Inforill](/work/inforill/) and kept public because some of it is useful to someone.

Not a project. A working directory.

## What's in it

Classical AI first: DFS, BFS, A\*, greedy best-first, minimax with alpha-beta pruning, Bayesian networks, hill climbing, simulated annealing, constraint satisfaction with node and arc consistency.

Then the ML: k-NN, perceptrons, regression, backprop, dropout, CNNs, RNNs, LSTMs. Then NLP: n-grams, naive Bayes with additive smoothing, Word2Vec, GloVe, transformers.

Then the reinforcement learning, which is where most of the actual time went.

## Reinforcement learning

Tabular Q-learning on FrozenLake, with epsilon decaying at 0.9993 per episode over 15,000 games. Q-learning on MountainCar with the continuous state space discretised into a 40x40 grid. The whole problem there is that the reward is one bit at the top of a hill and everything before it is silence.

Then a genetic approach for Atari, which is not Q-learning at all. A population of 24 networks, top 20% selected each generation, crossover swapping alternate rows of the weight matrix, and mutation perturbing 0.1% of weights:

```python
def mutate(self) -> tflearn.DNN:
    weights = self.weights
    dim = int(weights.shape[0] * weights.shape[1] * 0.001)
    for _ in range(dim):
        col, row = random.randint(0, weights.shape[0] - 1), random.randint(
            0, weights.shape[1] - 1)
        new_val = weights[col][row] + random.uniform(-1, 1)
        weights[col][row] = min(max(new_val, -1), 1)
        self.bias[row] += random.uniform(-1, 1)
    return weights
```

No gradients anywhere. The weight matrix is 7056x6 (84x84 grayscale pixels in, six actions out), and mutating 0.1% of it means about 42 weights change per genome. Clamping to [-1, 1] stops a lucky mutation from producing one enormous weight that dominates every decision.

Also a PPO agent on LunarLander via stable-baselines3, with the trained model committed.

## Word embeddings

A GloVe implementation from scratch: build the co-occurrence matrix over a corpus with `scipy.sparse.lil_matrix`, reduce it with truncated SVD, then query cosine similarity between two words.

498 seconds to build the embeddings. 0.0145 seconds to answer a question against them. *Fish* and *Ocean* come out at 0.8558.

That ratio is the whole point of embeddings and it's more convincing when you've waited out the eight minutes yourself.

## Vision

A CIFAR-10 CNN in an AlexNet shape: five conv layers, three max pools, dropout, three linear layers down to the class count. SGD at 0.008 for 50 epochs, checkpoint committed.

One real bug in it, still there: the validation loader is built from `inputs_train`, the same list as the training loader. So the reported validation accuracy is training accuracy wearing a different name. Worth leaving visible: it's the exact mistake that makes a model look finished when it isn't.

## Caveats

The `PyTorch/` chapter tree follows *Mastering PyTorch* and the Dockerfile pulls assets straight from that book's repo. The MountainCar script credits Moustafa Alzantot in its docstring. Those are study materials I worked through, not things I invented, and I'd rather say so than let the directory listing imply otherwise.

The Gym scripts also predate the 0.26 API change and use the old four-value `step()`, so they won't run unmodified against a current Gym.
