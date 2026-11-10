---
title: "Teaching a world model to see with one wrist camera"
description: "Three ways a wrist-only world model cheated, and what finally made it read the arm's actions."
date: 2026-11-05 17:00:00 -0800
tags: [ai, world models, robotics]
---

In robotics, world models have mostly been explored with parallel-jaw grippers, and rarely with humanoids. Gripper setups usually train on an external scene camera plus a wrist camera. The wrist camera watches the contact between gripper and object closely but says little about where the arm is in the room. The scene camera fills that gap by capturing the robot's overall interaction with the environment. Together, the two views sharply reduce hallucination in the model's predictions, especially over long horizons.[^1][^2]

<!--more-->

## What about world models for humanoids?
Humanoids move between environments to perform different tasks, so it is impractical to install scene cameras everywhere they go. They have to rely on a head camera and wrist cameras alone. The problem is that the head camera looks down the arm, so with a five-finger hand the palm hides the object at exactly the moment the grasp happens. A parallel-jaw gripper does not have this problem, because the object stays visible between the jaws.

<div class="side-by-side">
  <figure><img src="{{ "/assets/images/world-models-p2/overhead-gripper.png" | relative_url }}" alt="Head camera view with a parallel-jaw gripper: the object stays visible between the jaws"><figcaption>Head camera view with a parallel-jaw gripper: the object stays visible between the jaws</figcaption></figure>
  <figure><img src="{{ "/assets/images/world-models-p2/overhead-hand.png" | relative_url }}" alt="Head camera view with a five-finger hand: the palm hides the object"><figcaption>Head camera view with a five-finger hand: the palm hides the object</figcaption></figure>
</div>

This gets worse in cluttered or partially occluded scenes, where the head camera gives even less signal for grasp alignment. So what if we train a world model on the wrist camera alone? Once the object is in the wrist camera's frame, is that view enough to guide the hand to the object and grasp it?

These two questions led me down a rabbit hole that I've still not found the exit to. In hindsight it was ambitious. I've since read several papers that describe exactly the problems of wrist-only setups. In this post I'll go through the problems I ran into with the model's predictions and what I tried in order to fix them:
  1. object permanence
  2. object morphing
  3. movement understanding

## Model architecture

![Model architecture. Image tokens, the action and the finger force are fed to a transformer, which predicts the next state.]({{ "/assets/images/world-models-p2/image-2.png" | relative_url }})
{: .figure}

Model architecture. Image tokens, the action and the finger force are fed to a transformer, which predicts the next state.
{: .caption}

The action input is initially the normalised arm and finger joint positions, and the force input is one scalar per finger. I change the action representation later in the post.

The Inspire hand has no tactile skin. Each finger reports a single scalar from its actuator, a raw value between 0 and 2000 that rises as the finger meets resistance, so the force input at each timestep is just one number per finger rather than a pressure map.

## Object permanence
In some offline trajectories the arm overshoots slightly and the object almost leaves the wrist camera's view. Here the model likes to cheat. Instead of predicting an empty table, it invents a new object further ahead and keeps approaching it.

![Forward motion carries the object out of the bottom of the view. The expected frames end on an empty table. The predicted state invents a new object far ahead at t=3 and keeps approaching it.]({{ "/assets/images/world-models-p2/wm-object-permanence.png" | relative_url }})
{: .figure}

Forward motion carries the object out of the bottom of the view. The expected frames end on an empty table. The predicted state invents a new object far ahead at t=3 and keeps approaching it.
{: .caption}

## Object morphing
When the hand approaches off-centre, the real outcome is a miss. The model instead morphs the object into the palm so the grasp still succeeds.

![The hand approaches to the right of the object. Expected frames show the object sliding off to the left and the hand missing. The predicted state smears the object sideways until it re-forms in front of the palm and the grasp "succeeds".]({{ "/assets/images/world-models-p2/wm-object-morphing.png" | relative_url }})
{: .figure}

The hand approaches to the right of the object. Expected frames show the object sliding off to the left and the hand missing. The predicted state smears the object sideways until it re-forms in front of the palm and the grasp "succeeds".
{: .caption}

## Movement understanding
Every training trajectory starts with the hand open, and the fingers begin closing about halfway through. I wanted to know whether the model had learned that finger motion in the image follows from the finger action, or whether it was simply predicting a close at the usual point in the episode. So: if the finger action stays open for the whole trajectory, does the predicted state keep the fingers open?

<!-- TODO: add the result of the open-fingers test here -->

I wanted to test the same thing with the arm. Does the model actually read the arm action, or only the image?

If the arm pauses, does the predicted state reflect that? No. The predicted state pretends that the arm is still moving forward towards the goal.

![The action switches to hold still at t=2. Expected frames freeze. The predicted state keeps the object growing and arrives anyway.]({{ "/assets/images/world-models-p2/wm-paused-action.png" | relative_url }})
{: .figure}

The action switches to hold still at t=2. Expected frames freeze. The predicted state keeps the object growing and arrives anyway.
{: .caption}

If the arm moves to the left of the object, does the predicted state reflect that? No. The predicted state pretends that the arm is moving straight.

![The action becomes forward plus left from t=1. The object should drift right and leave the frame. The predicted state keeps it centred as if the arm went straight.]({{ "/assets/images/world-models-p2/wm-left-action.png" | relative_url }})
{: .figure}

The action becomes forward plus left from t=1. The object should drift right and leave the frame. The predicted state keeps it centred as if the arm went straight.
{: .caption}

## Making the object count in the loss
My hypothesis for the first two problems, object permanence and object morphing, was that the object simply occupies too few patches to matter to the loss. A plain token MSE averages over the whole frame, so the model can hallucinate a new object or let the real one smear into the hand while paying almost no penalty. I wanted to weight exactly those patches so that getting the object wrong costs something.

Following DexWM,[^3] I added a small auxiliary transformer that takes the predicted state from the main transformer and tries to reconstruct a heatmap of where the object is. Its output is compared against a ground-truth heatmap built from the object mask of the real next frame, and the error is added to the training loss. The gradient only reaches the main transformer through the predicted state, so the auxiliary branch acts as a check: if the heatmap cannot be recovered from the state, the state did not encode the object's position and shape, and the main model is pushed to fix that.

![The auxiliary heatmap head. It reads the predicted state and tries to reconstruct where the object is. The reconstruction error is added to the training loss.]({{ "/assets/images/world-models-p2/image-3.png" | relative_url }})
{: .figure}

The auxiliary heatmap head. It reads the predicted state and tries to reconstruct where the object is. The reconstruction error is added to the training loss.
{: .caption}

The heatmap loss visibly lowered the training loss, but on held-out trajectories the object was still reconstructed in the wrong place. The auxiliary loss sharpened the representation of the object, but it did not teach the model how a given action moves the object in the wrist view, so it failed to generalise to held-out trajectories with new actions.

## Is the model overfit to the training goal?
The model behaved as if every rollout were on a fixed schedule. No matter what actions I fed it, the predicted states marched toward the final grasp at the same pace, which is exactly the behaviour behind the paused-action and sideways-motion failures above. Even with the action held constant, each new step brought the object closer, as if the model were counting frames rather than reading actions.

My guess was that the training data invited this shortcut. The trajectories differ in how they approach the object, but they all end the same way, with the hand closed around it at roughly the same timestep. The cheapest way to score well on those endings is to learn "at step N, show a grasp", regardless of what happened in between. I tried two things to break that association.

First, I downweighted the loss on the final grasping frames so the shared ending counted for less. Second, I removed the temporal embedding attached to each frame, so the model could no longer condition on how far along in the episode it was and had to infer progress from the images and actions alone.

This fixed most of the paused-action failures, but the sideways-motion predictions were still wrong. The timing problem was gone; the model still had no sense of where the arm was.

## Teaching spatial understanding from a wrist camera alone
My first action representation was the raw joint targets, each normalised to [0, 1]. For the model to connect those numbers to what changes in the wrist image, it would have had to learn forward kinematics implicitly: how the arm's seven joint values combine into a position and orientation of the hand. That felt like too much to ask of a model that was already struggling with the image side, so I switched the action to the end-effector pose, a 3D position plus a quaternion in the robot's base frame. It is the same number of inputs, but in this space the relationship is at least simple and monotonic: x increasing means the arm moves forward, y increasing means it moves left, and the camera view should shift accordingly.

To make the relationship even more explicit, I added a second, relative input: the end-effector position expressed in a frame centred on the object. This tells the model not just that the arm moved left, but how far it now sits from the object in each axis, which is exactly the quantity that determines where the object appears in the wrist view. The hope was that with both the absolute pose and the object-relative offset available, the model would stop treating the object as something that drifts on its own and start treating it as something whose position in the frame follows from where the hand is.

The object position itself came from the wrist camera: an object detector on the RGB image gave the object's pixel location, and the RealSense D455 depth at that pixel turned it into a 3D point in the camera frame.

![The two new inputs: the absolute end-effector pose in the robot's base frame, and the end-effector position relative to the detected object.]({{ "/assets/images/world-models-p2/wm-relative-input.png" | relative_url }})
{: .figure}

The two new inputs: the absolute end-effector pose in the robot's base frame, and the end-effector position relative to the detected object.
{: .caption}

You could argue that if depth is available, the model should just take the depth image as an input too. But I wanted to keep the model as simple as possible, and feeding depth would have meant a second encoder and a second stream of tokens to attend over.

### Results
I was sceptical that these ten extra numbers would change much. Ten numbers next to hundreds of image tokens seemed easy to ignore, and I expected them to be drowned out. However, after some tuning of the loss weights, the combination of absolute end-effector pose and object-relative offset worked far better than I expected, both on the training set and on held-out trajectories. The object stayed where the arm's motion said it should be.

More surprising still, once these inputs were in place I could remove the heatmap loss entirely and the object was still predicted correctly. The information that the heatmap had been forcing into the state was now arriving directly through the conditioning.

World Labs report something similar for their Atlas world model: they feed in camera pose as a direct input and credit it with giving the model spatial memory.[^4]

## Open question
After these changes, the predicted states look good on held-out offline trajectories. On hardware though, with the world model driving a CEM planner, I still see hallucinations and planner behaviour I can't yet explain.

My main suspect is the data. I set out to collect expert demonstrations, but they may be too clean compared with what the CEM planner actually samples.

![Data collection. The arm trajectory is recorded in guide mode, then replayed while the hand is teleoperated with a finger-tracking glove.]({{ "/assets/images/world-models-p2/wm-data-collection.png" | relative_url }})
{: .figure}

Data collection. The arm trajectory is recorded in guide mode, then replayed while the hand is teleoperated with a finger-tracking glove.
{: .caption}

Each training trajectory was recorded in two passes. First I puppeteered the Franka arm in guide mode, physically moving the end effector to the grasp position while dense waypoints were logged. Then I replayed that trajectory on the arm and teleoperated the Inspire hand with a finger-tracking glove to close the grasp. The result is a dataset of perfectly smooth arm motion. A CEM planner does not produce anything like that. It samples noisy action sequences and scores them, so the model is being asked to predict the consequences of action patterns it never saw during training, and it may simply be out of distribution.

![Training actions are smooth demonstrations. The CEM planner samples noisy action sequences the model never saw during training.]({{ "/assets/images/world-models-p2/wm-action-mismatch.png" | relative_url }})
{: .figure}

Training actions are smooth demonstrations. The CEM planner samples noisy action sequences the model never saw during training.
{: .caption}

## Conclusion
Looking back, the three problems I started with were not really three problems. Object permanence, object morphing and the ignored actions were all symptoms of the same thing: the model had never been given a reason to connect what the arm did to what the camera saw. With a single wrist camera, that connection is the whole task, and nothing in a plain pixel loss or a hand-tuned frame weighting forces the model to learn it. It found cheaper explanations instead, counting frames and redrawing the object where the grasp needed it to be.

What finally worked was not a cleverer loss but a better description of the state. Once the model was told where the hand was and where the hand was relative to the object, the geometry stopped being something it had to reverse-engineer from pixels and became something it could read off directly. The heatmap loss, which I had spent the most time on, turned out to be a workaround for information that should have been in the input all along. I suspect this is a general lesson for wrist-only setups: the camera cannot see the arm, so the arm has to be described some other way.

That said, a model that predicts offline trajectories well is not yet a model that can drive a robot. The hardware results are still messy, and my best guess is the gap between the smooth demonstrations the model was trained on and the noisy action sequences a planner actually proposes. In part three, I'll get into the planning side: why a goal-image CEM did not behave the way I expected, how the learned representation collapsed under it, and what I tried to do about that.

## References

[^1]: Yanjiang Guo et al. "Ctrl-World: A Controllable Generative World Model for Robot Manipulation." arXiv preprint, October 2025. [arxiv.org/abs/2510.10125](https://arxiv.org/abs/2510.10125)

[^2]: Arnav Kumar Jain et al. "WEAVER, Better, Faster, Longer: An Effective World Model for Robotic Manipulation." arXiv preprint, June 2026. [arxiv.org/abs/2606.13672](https://arxiv.org/abs/2606.13672)

[^3]: Raktim Gautam Goswami et al. "World Models for Learning Dexterous Hand-Object Interactions from Human Videos." arXiv preprint, December 2025. [arxiv.org/abs/2512.13644](https://arxiv.org/abs/2512.13644)

[^4]: World Labs. "Atlas: A World Model for Spatial Intelligence." *World Labs Blog*, September 1, 2026. [worldlabs.ai/blog/atlas](https://worldlabs.ai/blog/atlas)
