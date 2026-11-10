---
title: "Problems I encountered with world models"
date: 2026-11-05 17:00:00 -0800
tags: [ai, world models, robotics]
---

## Teaching a world model to see with one wrist camera

In robotic settings, world models have been largely explored with parallel jaw grippers but little work has been done with humanoids.
With parallel jaw grippers, world models are often trained using external scene camera in addition to wrist camera viewpoints.
These two cameras complement each other to provide the best learning signal possible. Wrist camera is able to closely monitor the contact between gripper and object,
but it provides a limited understanding of how the arm is moving in the environment. Hence, the external camera helps by capturing of the overall robot interactions with the environment.
This combination has proven to greatly reduce hallucination in world model predictions especially for long-horizon. [0] [3]

<!--more-->

## What about world models for humanoids?
Since humanoids need to move around to perform different tasks, it is impractical to install scene cameras in every environment. And so humanoids have to rely on overhead and wrist cameras only.
The problem is that overhead cameras are not as good as scene cameras especially when the humanoid has a 5-fingered hand instead of a parallel jaw gripper.

<div class="side-by-side">
  <figure><img src="{{ "/assets/images/world-models-p2/overhead-gripper.png" | relative_url }}" alt="Head camera view with a parallel-jaw gripper: the object stays visible between the jaws"><figcaption>Head camera view with a parallel-jaw gripper: the object stays visible between the jaws</figcaption></figure>
  <figure><img src="{{ "/assets/images/world-models-p2/overhead-hand.png" | relative_url }}" alt="Head camera view with a five-finger hand: the palm hides the object"><figcaption>Head camera view with a five-finger hand: the palm hides the object</figcaption></figure>
</div>

This problem is exacerbated in partially occluded environments where overhead cameras provide even less signal for grasp alignment. So what if we only use wrist camera to train a world model? Once the object is in the wrist camera frame, is the wrist camera signal enough to guide the hand to the object and grasp it?

These two questions led me down a rabbit hole that I've still not found the exit to. In hindsight, it was probably too ambitious of me since I've read a lot of papers since then outlining the issue of only using wrist cameras. In this blog, I will outline some problems I've encountered so far with the world model predictions and what I've tried to resolve them:
  1. object permanence
  2. object morphing shapes
  3. movement understanding

## Model architecture

![figure]({{ "/assets/images/world-models-p2/image-2.png" | relative_url }})
{: .figure}

action = normalized arm and finger joint positions  
force = finger force

The Inspire hand has no tactile skin. Each finger reports a single scalar from its actuator, a raw value between 0 and 2000 that rises as the finger meets resistance, so the force input at each timestep is just one number per finger rather than a pressure map.

## Object permanence
In some offline trajectories, the arm overshoots the object a little such that the object almost disappears from the wrist camera view. In these cases, the model likes to cheat. Instead of knowing to turn around, the model just invents a new object and keeps going forward towards it.

![forward motion carries the object out of the bottom of the view. The expected frames end on an empty table. The predicted state invents a new object far ahead at t=3 and keeps approaching it.]({{ "/assets/images/world-models-p2/wm-object-permanence.png" | relative_url }})
{: .figure}

Forward motion carries the object out of the bottom of the view. The expected frames end on an empty table. The predicted state invents a new object far ahead at t=3 and keeps approaching it.
{: .caption}

## Object morphing
When the arm moves towards the object, if the hand is not in the correct position, instead of the arm adjusting, the object morphs into place.

![the hand approaches to the right of the object. Expected frames show the object sliding off to the left and the hand missing. The predicted state smears the object sideways until it re-forms in front of the palm and the grasp "succeeds".]({{ "/assets/images/world-models-p2/wm-object-morphing.png" | relative_url }})
{: .figure}

The hand approaches to the right of the object. Expected frames show the object sliding off to the left and the hand missing. The predicted state smears the object sideways until it re-forms in front of the palm and the grasp "succeeds".
{: .caption}

## Movement understanding
The dataset I used for training the model only contained trajectories where the hand starts open and fingers begin around halfway through the recording to grasp the object. I wanted to see if the model learned to relate the finger movement in the image with the action input or only relative to the context of the image. That way I can check if the model can really understand/generalize what closing fingers meant. Would the predicted state still show the fingers open if the finger action input remained open for the entire trajectory?

Similarly, I wanted to test the same thing with the arm. Does the model understand the correlation between the image and action input?

If the arm pauses, does the predicted state reflect that? No. The predicted state pretends that the arm is still moving forward towards the goal.

![the action switches to hold still at t=2. Expected frames freeze. The predicted state keeps the object growing and arrives anyway.]({{ "/assets/images/world-models-p2/wm-paused-action.png" | relative_url }})
{: .figure}

The action switches to hold still at t=2. Expected frames freeze. The predicted state keeps the object growing and arrives anyway.
{: .caption}

If the arm moves to the left of the object, does the predicted state reflect that? No. The predicted state pretends that the arm is moving straight.

![the action becomes forward plus left from t=1. The object should drift right and leave the frame. The predicted state keeps it centered as if the arm went straight.]({{ "/assets/images/world-models-p2/wm-left-action.png" | relative_url }})
{: .figure}

The action becomes forward plus left from t=1. The object should drift right and leave the frame. The predicted state keeps it centered as if the arm went straight.
{: .caption}

## Targeted object loss within image
My hypothesis for the first two problems, object permanence and object morphing, was that the object simply occupies too few patches to matter to the loss. A plain token MSE averages over the whole frame, so the model can hallucinate a new object or let the real one smear into the hand while paying almost no penalty. I wanted to weight exactly those patches so that getting the object wrong costs something.

Following DexWM [1], I added a small auxiliary transformer that takes the predicted state from the main transformer and tries to reconstruct a heatmap of where the object is. Its prediction is compared against a ground-truth heatmap built from the object mask of the real next frame, and the error is added to the training loss. The gradient only reaches the main transformer through the predicted state, so the auxiliary branch acts as a check: if the heatmap cannot be recovered from the state, the state did not encode the object's position and shape, and the main model is pushed to fix that.

![alt text]({{ "/assets/images/world-models-p2/image-3.png" | relative_url }})
{: .figure}

The heatmap loss visibly lowered the training loss, but on held-out trajectories the object was still reconstructed in the wrong place. The auxiliary loss sharpened the representation of the object but did not not teach the model to understand the correlation between a given action and how the object moves in the wrist view. As a result, it fails to generalize to unseen test data with new actions.

## Is the model overfit to the training goal?
The model behaved as if every rollout were on a fixed schedule. No matter what actions I fed it, the predicted states marched toward the final grasp at the same pace, which is exactly the behaviour behind the paused-action and sideways-motion failures above. Even with the action held constant, each new step brought the object closer, as if the model were counting frames rather than reading actions.

My guess was that the training data invited this shortcut. The trajectories differ in how they approach the object, but they all end the same way, with the hand closed around it at roughly the same timestep. The cheapest way to score well on those endings is to learn "at step N, show a grasp", regardless of what happened in between. I tried two things to break that association.

First, I downweighted the loss on the final grasping frames so the shared ending counted for less. Second, I removed the temporal embedding attached to each frame, so the model could no longer condition on how far along in the episode it was and had to infer progress from the images and actions alone.

This improved the predicted states for the paused-action sequence, but the model still seemed to lack the spatial understanding needed for accurate predicted states under sideways motion.

## How do we teach spatial/movement understanding with only wrist camera input?
My first action representation was the raw joint targets, each normalised to [0, 1]. For the model to connect those numbers to what changes in the wrist image, it would have had to learn forward kinematics implicitly: how seven joint values combine into a position and orientation of the hand. That felt like too much to ask of a model that was already struggling with the image side, so I switched the action to the end-effector pose, a 3D position plus a quaternion in the robot's base frame. While it is the same number of inputs, in this space the correlations are at least linear: x increasing means the arm moves forward, y increasing means it moves left, and the camera view should shift accordingly.

To make the relationship even more explicit, I added a second, relative input: the end-effector position expressed in a frame centred on the object. This tells the model not just that the arm moved left, but how far it now sits from the object in each axis, which is exactly the quantity that determines where the object appears in the wrist view. The hope was that with both the absolute pose and the object-relative offset available, the model would stop treating the object as something that drifts on its own and start treating it as something whose position in the frame follows from where the hand is.

The object position itself came from the wrist camera: an object detector on the RGB image gave the object's pixel location, and the RealSense D455 depth at that pixel turned it into a 3D point in the camera frame.

![alt text]({{ "/assets/images/world-models-p2/wm-relative-input.png" | relative_url }})
{: .figure}

To get the object position, I used the depth output of the wrist camera realsense d455 with object detection. Now you can argue if depth is available, the model should just take the depth image as an input too. But I wanted to keep the model as simple as possible, and feeding depth would have meant a second encoder and a second stream of tokens to attend over.

### Results
I was sceptical that these ten extra numbers would change much. Their loss terms are tiny next to the hundreds of image patches, and I expected them to be drowned out. However, after some tuning of the loss weights, the combination of absolute end-effector pose and object-relative offset worked far better than I expected, both on the training set and on held-out trajectories. The object stayed where the arm's motion said it should be.

More surprising still, once these inputs were in place I could remove the heatmap loss entirely and the object was still predicted correctly. The information that the heatmap had been forcing into the state was now arriving directly through the conditioning.

In fact, I read the training details released by World Labs team for their Atlas world model and they also feed in camera pose, allowing the model to develop spatial memory. [2]

## Open question
After these changes, the predicted states now look good on held-out offline trajectories. On hardware though, with the world model driving a CEM planner, I still see confusing results and hallucination that I'm trying to debug.

One of my main suspects is the data collection itself. My initial objective was to collect expert trajectories for training but now I wonder if they're too perfect and smooth in comparison to the trajectories sampled by the CEM planner.

![alt text]({{ "/assets/images/world-models-p2/wm-data-collection.png" | relative_url }})
{: .figure}

Each training trajectory was recorded in two passes. First I puppeteered the Franka arm in guide mode, physically moving the end effector to the grasp position while dense waypoints were logged. Then I replayed that trajectory on the arm and teleoperated the Inspire hand with a finger-tracking glove to close the grasp. The result is a dataset of perfectly smooth arm motion. A CEM planner does not produce anything like that. It samples noisy action sequences and scores them, so the model is being asked to predict the consequences of action patterns it never saw during training, and it may simply be out of distribution.

![alt text]({{ "/assets/images/world-models-p2/wm-action-mismatch.png" | relative_url }})
{: .figure}

## Conclusion
Looking back, the three problems I started with were not really three problems. Object permanence, object morphing and the ignored actions were all symptoms of the same thing: the model had never been given a reason to connect what the arm did to what the camera saw. With a single wrist camera, that connection is the whole task, and nothing in a plain pixel loss or a hand-tuned frame weighting forces the model to learn it. It found cheaper explanations instead, counting frames and redrawing the object where the grasp needed it to be.

What finally worked was not a cleverer loss but a better description of the state. Once the model was told where the hand was and where the hand was relative to the object, the geometry stopped being something it had to reverse-engineer from pixels and became something it could read off directly. The heatmap loss, which I had spent the most time on, turned out to be a workaround for information that should have been in the input all along. I suspect this is a general lesson for wrist-only setups: the camera cannot see the arm, so the arm has to be described some other way.

That said, a model that predicts offline trajectories well is not yet a model that can drive a robot. The hardware results are still messy, and my best guess is the gap between the smooth demonstrations the model was trained on and the noisy action sequences a planner actually proposes. In part three, I'll get into the planning side: why a goal-image CEM did not behave the way I expected, how the learned representation collapsed under it, and what I tried to do about that.

[0] <https://arxiv.org/pdf/2510.10125>  
[1] <https://arxiv.org/pdf/2512.13644>  
[2] <https://worldlabs.ai/blog/atlas>  
[3] <https://arxiv.org/abs/2606.13672>
