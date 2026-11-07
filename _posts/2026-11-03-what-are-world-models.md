---
title: "What are world models"
date: 2026-11-03 17:00:00 -0800
tags: [ai, world models, robotics]
---

Six months ago, hoping to solve a problem in teleoperated robot manipulation, I unknowingly stumbled upon one of the hottest research topics in robot learning: world models. Essentially, unlike vision-language-action models (VLAs), which map what the robot sees straight to an action, world models learn the dynamics of the environment and the consequences of taking an action. If we actually want to build robots that can replace humans, then we'd want the robots to have the same physical intuition we do.

<!--more-->

## What do you think about when you're holding something or about to grab something?

If I were holding a glass of water, I'd make sure to keep it upright so the water doesn't spill. If I wanted to grab a slippery object, I'd want to make sure I get a good hold of it so that it doesn't slip out of my hand. If I lifted something heavy, I'd apply more pressure. If I had something fragile, I'd apply less pressure. These things feel so much like common sense that we rarely pause to think about how we hold something. But we weren't born with this intuition. If you gave toddlers a glass of water, they'd most certainly spill it. Yes, they have clumsy little fingers, but they also don't yet understand how the physical world behaves. It's the same story for robots learning to grab things.

## How do you teach robots how the world works?

Where an LLM predicts the next word given the words before it, a world model predicts the next observation given the observations before it *and the action taken*. Conditioning on the action is what forces it to pick up physical dynamics, 3D spatial relationships, and the link between what the robot does and what happens next. In the context of robots, world models act as a simulator linking actions with image observations. If you take this action, what will you see next? And often, since the response is determined by a series of actions, world models learn the relationship between a chunk of history frames and what happens next.

## Why is this so powerful?

Using this relationship, the world model can predict not just one step into the future but a whole chunk of future observations. For the task of grasping, this allows the robot to foresee whether a grasp would slip before it even closes its gripper.

![alt text]({{ "/assets/images/world-models/image.png" | relative_url }})
{: .figure}

This figure from the Dreamer paper shows how the world model predicts future outcomes given five history frames [1]
{: .caption}

There are many types of world models, such as renderers, simulators, and planners. I'm going to focus on the applications of world models to robotics. Of course, there are many blogs, papers, and videos about world models already, but I'd consider this post a beginner's guide. I'll make a more technical part two on how I used world models for teleoperation and the problems I encountered.

## World model for simulation

### How to use image inputs

As a physical simulator, a world model for robotics should capture the key context of the task without needing to predict every pixel perfectly like a renderer would. Pixels also hold more detail than necessary, which can hinder the model's ability to focus on the key parts. For example, if we're teaching the model what happens when you pick up a water bottle from a table, the model only needs to focus on the robot's interaction with the water bottle. To achieve this, world models often train in token space instead of pixel space. This way, instead of feeding in 150,528 raw pixel values (224 × 224 × 3 colour channels) for each image, the model takes in the image as a few hundred tokens.

![alt text]({{ "/assets/images/world-models/image-1.png" | relative_url }})
{: .figure}

[2]
{: .caption}

In this example, the 224×224 image is divided into a 16×16 grid of 14×14-pixel patches, for 256 patches in total. A pretrained image encoder (here DINOv2) turns each patch into a token: a vector of 384 numbers that summarises the information in that patch. The encoder is frozen, so the world model only has to learn the dynamics, not how to see.

### Predictor training process
Each frame is an image (as tokens) paired with the action taken at that step. Together, the chunk of history frames is the model's input. Now the question is: how do we teach the predictor to turn that input into the next frame?

There are two training methods: teacher forcing and autoregressive training.
### 1. Teacher forcing

The ground-truth trajectory acts as the teacher: at every step the predictor is fed the true history and asked, if you take this action, what will you see? For each prediction, we measure the mean squared error (MSE) between the target tokens and the predicted tokens.

![alt text]({{ "/assets/images/world-models/image-7.png" | relative_url }})
{: .figure}

Teacher forcing makes training efficient and stable, but it creates a mismatch between training and test time. During training, the predictor only ever sees ground-truth tokens as input. At test time, when we want to predict a chunk of future frames, we have to roll the model out autoregressively: the prediction ẑ(t+1) becomes the input for predicting ẑ(t+2), and so on. Each prediction carries a small error, and since the model never saw its own imperfect outputs during training, it isn't robust to them. These errors compound with every step, so the rollout drifts further from reality the longer it runs. This train–test mismatch is known as *exposure bias*.

![alt text]({{ "/assets/images/world-models/image-9.png" | relative_url }})
{: .figure}

At test time, the model feeds its own predictions back in as input. After each prediction, the input window slides forward one step: the newest prediction is added to the end, and the oldest observation drops off the front.
{: .caption}

### 2. Autoregressive training

To close this training gap, we can train the predictor the same way we use it at test time. Instead of predicting each frame from ground-truth inputs, we roll the predictor out autoregressively over a chunk of future frames, feeding each prediction ẑ(t+k) back in to predict ẑ(t+k+1). We then measure how far each predicted latent is from its ground-truth target (using mean squared error), add up these errors across the chunk, and use that total to update the model. The update traces the error back through every step of the rollout, so the model learns not just from its final mistakes but from the earlier predictions that led to them. Because gradients flow back through the model's own predictions, it learns to correct for the small errors it makes and becomes robust to them.

![alt text]({{ "/assets/images/world-models/image-8.png" | relative_url }})
{: .figure}

Autoregressive training process
{: .caption}

Input -> Output

![alt text]({{ "/assets/images/world-models/image-5.png" | relative_url }})
{: .figure}

### Test-time inference
Once trained, the world model acts as a simulator we can use to plan a trajectory. Many papers use model predictive control (MPC) with the cross-entropy method (CEM). We start by encoding a goal image, the state we want to reach, into tokens. CEM then samples many candidate action sequences and rolls each one out through the world model simulator. Each candidate is scored by how close its final predicted image tokens are to the goal image tokens. The best-scoring sequences are used to refine the sampling distribution, and the process repeats for a few iterations. Finally, the agent executes only the first action(s) of the best plan, observes the new state, and replans from there.

![alt text]({{ "/assets/images/world-models/image-10.png" | relative_url }})
{: .figure}

Replanning on hardware is necessary because small modeling errors and physical disturbances can quickly push the actual state away from the plan.

## Dreamer
There are many other variants of world models of course, each suited to different tasks, environments and requirements. For example, instead of searching for actions with MPC at test time, Dreamer learns a policy directly inside its world model. Alongside the dynamics, it learns a reward predictor that estimates the reward of each latent state, and it uses those predicted rewards to train the policy entirely in imagination.

![alt text]({{ "/assets/images/world-models/image6.gif" | relative_url }})
{: .figure}

Dreamer learns a world model from experience. Using past images (o1–o3) and actions (a1–a2), it computes a sequence of compact model states (green circles) from which it reconstructs the images (ô1–ô3) and predicts the rewards (r̂1–r̂3).  [1]
{: .caption}

Holding the world model and reward predictor fixed, Dreamer then trains an actor to propose actions entirely on imagined rollouts: starting from states the robot has actually visited, the actor proposes actions, the world model predicts the resulting states and rewards, and the critic estimates the return beyond the imagined horizon. Using the feedback from the world model, the actor learns to choose actions that maximize the predicted total return and the critic is updated to predict the return more accurately.

These components improve each other in a loop: the imperfect world and reward models train the actor, and the actor gathers real experience that improves the world and reward models. The figures below walk through a simplified example. In step 1, the actor is trained entirely in imagination: starting from a real state zₜ, it tries actions, the world model predicts the next state ẑₜ₊₁ and its reward r̂, and the actor is updated to favor actions with higher predicted reward. In step 2, the updated actor chooses action a₂ on the real robot. In step 3, the actual outcome is used to correct the world and reward models' predictions.

![alt text]({{ "/assets/images/world-models/image-17.png" | relative_url }})
{: .figure}

The world model predicted a reward of +5 for a₂, but the robot actually received −1. Training on this real transition updates the world model and reward predictor (v1 → v2) to reduce the gap between the imagined state ẑₜ₊₁ and the actual state zₜ₊₁. The actor is then retrained in imagination using the corrected model, which steers it away from a₂ toward better actions. Its new choice, a₃, is again only a prediction, so the cycle repeats: the robot tests it, and the world model is corrected again.

![alt text]({{ "/assets/images/world-models/image-18.png" | relative_url }})
{: .figure}

For clarity, this example shows a single start state and one-step predictions. In practice, Dreamer imagines rollouts of about 15 steps from many start states at once, and the actor and critic are updated on all of them together.

## Simulator vs Dreamer

Both approaches rest on the same idea: learn how the world responds to actions, then use that knowledge to decide what to do. They differ in two places.

The first is the visual representation. The simulator-style model above reuses a frozen pretrained encoder like DINOv2 to turn images into tokens, so it inherits general-purpose visual features and only has to learn the dynamics. Dreamer learns its encoder from scratch alongside the dynamics. That lets the representation specialise to the task, but it needs more environment data before the visual features are any good.

The second, bigger difference is how each decides what to do. The simulator has no notion of reward. You hand it a goal image, and a planner like CEM searches for actions whose predicted outcome matches that goal. This is flexible, since any task you can describe with a goal image is fair game, and the same model can be reused across tasks. But planning is expensive at test time, and the plan is only as good as the imagined rollout. Dreamer instead learns a reward predictor and trains an actor on imagined rollouts, so at test time it simply runs the policy, with no search at all. The cost is that it depends on a reward signal. In manipulation, rewards are often sparse or hard to define: if the robot rarely receives a nonzero reward, the reward predictor has little to learn from, and the actor gets almost no signal in imagination.

Neither is strictly better. If you can write down a reward and afford the environment interaction, Dreamer gives you a fast, reactive policy. If your task is naturally described by a goal state, or you want one model to serve many tasks, planning through a simulator is the more natural fit.

Either way, what makes world models worth the trouble is the same thing that lets you carry a glass of water without thinking about it: a sense of what will happen before you do it. In part two, I'll get into how I used a simulator-style world model for teleoperation, and the places where this tidy picture breaks down.

[1] Danijar Hafner. “Introducing Dreamer: Scalable Reinforcement Learning Using World Models.” Google Research Blog, March 18, 2020. https://research.google/blog/introducing-dreamer-scalable-reinforcement-learning-using-world-models/

[2] Vizuara. “Build a World Model from Scratch — New Series Launch | Episode 0.” YouTube video, August 4, 2026. https://www.youtube.com/watch?v=7l7rBM784pg
