# DinoBorn — Evrima-Style Fast Locomotion

### High speed, intentional momentum sliding, and no foot sliding

---

## Purpose

This document defines a locomotion system for DinoBorn that allows dinosaurs — especially
short-legged fast animals such as Troodon and Raptor — to:

- move at **high gameplay speeds**,
- **slide and drift on purpose**, the way The Isle: Evrima does,
- and still **never show unintentional foot sliding**.

The target feel is:

- The dinosaur can move much faster than the apparent size of its steps might suggest.
- Legs remain relatively short and animations do not need exaggerated giant strides.
- Animation playback adapts to actual movement speed.
- Feet remain visually synchronized with the ground **while running**.
- The body carries **momentum**: it cannot change direction instantly at speed.
- Sharp turns and hard braking at speed produce a **visible, controllable skid**.
- Acceleration, deceleration, turning, braking and sliding all feel connected.
- The system works identically for player-controlled and AI-controlled dinosaurs.
- Gameplay movement remains authoritative; animation visually follows movement.
- Root motion is not the primary source of gameplay velocity.
- Uneven terrain is handled with foot IK / procedural placement where appropriate.

> **The core claim of this document:** speed, sliding and foot-lock are not three separate
> features. They are one system built on a single value — the relationship between the
> dinosaur's **velocity vector** and its **facing vector**.

---

## Table of Contents

**Part I — Foundations**
1. Core Principle
2. Two Kinds of Sliding
3. Why Fast Movement Normally Causes Foot Sliding
4. The Stride-Matching Concept
5. DinoBorn Architecture
6. Do Not Start With Root Motion

**Part II — Increasing Speed Safely**
7. What Actually Breaks When You Raise Max Speed
8. Speed Tiers, Sprint and Stamina
9. Selling Speed Without Increasing It

**Part III — Animation and Synchronization**
10. Prepare the Animations
11. Determine the Natural Speed of Each Animation
12. Blended Natural Speed — The Correct Playback Formula
13. Locomotion Blend Tree
14. Use Actual Velocity, Not Input Magnitude
15. Animation Curves and Foot Contact
16. Stride Warping / Speed Warping

**Part IV — Momentum, Grip and Intentional Sliding**
17. What Evrima-Style Sliding Actually Is
18. The Grip Model
19. Slip Angle — The Single Value That Drives Everything
20. The Slide State Machine
21. Slide Triggers
22. Steering and Recovering From a Slide
23. Animating a Slide
24. Foot IK and Foot Locking During a Slide
25. Slide Feedback — FX, Audio, Camera
26. Sliding as a Gameplay Mechanic
27. Networking Momentum and Sliding

**Part V — Acceleration, Deceleration and Turning**
28. Acceleration
29. Deceleration and Braking
30. Turning
31. Turn Animation

**Part VI — Foot Placement**
32. Foot IK
33. Foot IK Concept
34. Do Not Overuse IK
35. Foot Locking

**Part VII — Implementation**
36. Data-Driven Locomotion Config
37. The Movement Component
38. The Animation Driver
39. The Foot IK Component
40. Suggested Animator Parameters
41. Recommended Animator Structure
42. AI Must Use the Same Locomotion System

**Part VIII — Tuning, Testing and Delivery**
43. Example — Raptor
44. Example — Troodon
45. What NOT To Do
46. Debug Mode
47. How to Test for Foot Sliding
48. How to Test Intentional Sliding
49. Testing Sequence for DinoBorn
50. Recommended Implementation Stages
51. The Relationship Between Speed and Animation
52. The Evrima-Like Target
53. Recommended Final DinoBorn Architecture
54. Final Recommendation
55. Implementation Acceptance Criteria
56. Changelog

---
---

# PART I — FOUNDATIONS

---

# 1. Core Principle

The most important rule is:

> **Animation speed and gameplay movement speed are separate concepts.**

Do not build the system around:

```text
Animation -> Root Motion -> Gameplay Speed
```

Prefer:

```text
Gameplay Movement
       |
       +---- Actual Velocity
       |
       v
    Animator
       |
       +---- Locomotion Blend
       |
       +---- Playback/Stride Adjustment
       |
       v
 Foot Placement / IK
       |
       v
 Final Visual Result
```

The dinosaur's movement system decides how fast the dinosaur actually travels.

The animation system receives that velocity and chooses/adjusts the animation that
visually represents it.

A second rule, added for momentum and sliding:

> **Facing direction and velocity direction are also separate concepts.**

Most simple character controllers assume the character always moves exactly where it
looks. Evrima-style locomotion breaks that assumption on purpose. Everything in Part IV
follows from allowing those two vectors to disagree.

---

# 2. Two Kinds of Sliding

This is the single most important distinction in this document, and the reason it was
revised.

## 2.1 Foot sliding — a bug

The body moves at one speed, the legs animate for a different speed, so the planted foot
skates across the ground.

```text
Facing    ───────────────>
Velocity  ───────────────>      (same direction!)

Foot should be planted:
        🦶 ──────> slides forward with no cause
```

The body and the velocity **agree**. Nothing in the fiction explains why the foot is
moving. The player reads it as broken animation.

**Cause:** animation cadence ≠ ground travel.
**Fix:** stride matching (Part III).

## 2.2 Momentum sliding — a feature

The dinosaur turns its body, but its momentum still carries it along the old direction.
The feet skid because the animal is genuinely skidding.

```text
Facing    ──────>
Velocity      ╲
               ╲
                ╲──────>       (velocity lags behind the turn)

Foot on the ground:
        🦶 ══════> skids sideways, and it looks correct
```

The body and the velocity **disagree**, and the player can see exactly why. This reads as
weight, mass and speed — not as a bug.

**Cause:** limited lateral grip vs. momentum.
**Fix:** none needed. This is the thing you are building.

## 2.3 The rule that keeps them apart

```text
Velocity aligned with facing   ->  feet MUST NOT slide
Velocity diverging from facing ->  feet SHOULD slide
```

A single value expresses this: the **slip angle**, the signed angle between facing and
velocity. Section 19 defines it. Once you have it:

- Slip angle ≈ 0 → run stride-matched, foot-locked, IK at full weight.
- Slip angle large → switch to skid pose, **disable foot locking**, reduce IK weight.

Foot locking exists to answer *"is this foot moving when it shouldn't be?"* During a
deliberate slide the answer is: **it should be.** Leaving foot locking on during a slide
is one of the most common ways a good slide mechanic ends up looking broken.

---

# 3. Why Fast Movement Normally Causes Foot Sliding

Consider a walk animation.

Suppose one complete stride visually covers approximately 2 meters.

If the animation plays at 1 cycle/second:

```text
Visual stride speed ≈ 2 m/s
```

If gameplay moves the dinosaur at 5 m/s while the animation remains unchanged:

```text
Animation:     2 m/s
Gameplay:      5 m/s
```

The feet cannot stay synchronized with the ground.

The result looks like:

```text
foot planted
    |
    |-----> foot visibly slides
```

This is foot sliding.

The solution is to synchronize:

```text
Actual ground travel
        ≈
Visual stride travel
```

Note that this problem gets **worse the faster you go**, which is why Part II (raising max
speed) and Part III (stride matching) must be done together. Raising the speed number
alone will only make the existing sliding more obvious.

---

# 4. The Stride-Matching Concept

A useful conceptual formula is:

```text
Animation Playback Rate
    =
Actual Movement Speed
/
Animation's Natural Speed
```

Example:

```text
Animation natural speed = 2 m/s
Actual dinosaur speed    = 5 m/s

Playback rate = 5 / 2
              = 2.5x
```

The animation cycles approximately 2.5 times faster.

The important point is that this does NOT mean the dinosaur needs enormous leg movements.

Instead:

```text
Short legs
+
Higher stride frequency
+
Correct movement synchronization
=
Fast-looking dinosaur
```

This is particularly useful for:

- Troodon
- Velociraptor-like dinosaurs
- Small theropods
- Other short-legged but fast creatures

A 2.5x playback rate on a single clip is, however, too much in practice — the legs start
to look like they are vibrating. Section 12 shows the correct version, where a blend tree
covers most of the speed range and playback correction only handles the remainder.

---

# 5. DinoBorn Architecture

Use the following separation:

```text
                MOVEMENT SYSTEM
                       |
                       v
                Desired Velocity
                       |
                       v
              MOMENTUM + GRIP SOLVER
                       |
                       v
              Actual World Velocity
                       |
          +------------+------------+
          |                         |
          v                         v
     Character              Velocity vs Facing
     Controller                     |
                                    v
                             Slip Angle / Slide
                                    |
                                    v
                            Animator Parameters
                                    |
                                    v
                             Locomotion Blend
                                    |
                                    v
                            Playback Adjustment
                                    |
                                    v
                        Foot IK (slide-aware)
```

## Responsibilities

### Movement System

Responsible for:

- Maximum speed
- Acceleration
- Deceleration
- Braking
- Turning
- Slope handling
- Ground detection
- AI navigation
- Player input
- Movement constraints
- Sprinting
- Stopping

### Momentum + Grip Solver

Responsible for:

- Keeping velocity as a real vector rather than "speed along forward"
- Limiting how fast velocity can be rotated toward the new facing
- Producing the slip angle
- Deciding when the dinosaur is sliding
- Surface grip differences (mud, sand, wet rock, snow)

### Animator

Responsible for:

- Idle
- Walk
- Run
- Sprint
- Start
- Stop
- Turn
- Skid / slide
- Airborne
- Landing
- Special locomotion states

### Foot IK / Procedural Placement

Responsible for:

- Ground contact
- Uneven terrain correction
- Foot height
- Optional foot rotation
- Reducing visible sliding **when the dinosaur is not deliberately sliding**
- Standing down gracefully when it is

---

# 6. Do Not Start With Root Motion

For DinoBorn, movement should preferably be gameplay-authoritative.

Recommended:

```text
Character Controller / Rigidbody
        |
        v
Move dinosaur
        |
        v
Calculate actual velocity
        |
        v
Animator.SetFloat("Speed", speed)
```

Avoid making the entire gameplay movement depend on:

```text
Animation root motion
        |
        v
Character position
```

## Why?

Gameplay-authoritative movement is easier to control for:

- Player movement
- AI movement
- Multiplayer
- Prediction/interpolation
- Sprint speed
- Stamina
- Knockback
- Attacks
- Chase behavior
- Pathfinding
- Network synchronization

Root motion can still be used selectively for specific animations, but it should not be
the fundamental locomotion authority.

**And for sliding specifically:** root motion is fundamentally incompatible with a
momentum slide. A slide is defined by velocity that the animation did *not* produce. If
the animation is producing the motion, there is no momentum to preserve and no slide to
see. Keep root motion for attacks, feeding, and scripted one-shots only.

---
---

# PART II — INCREASING SPEED SAFELY

---

# 7. What Actually Breaks When You Raise Max Speed

Raising `sprintSpeed` from 7 to 11 is one line of config. Everything below is what
silently breaks when you do it. Work through this list every time you raise a speed.

## 7.1 Animation cadence

The playback rate needed goes up proportionally. Past roughly **1.3x** on a single clip,
legs read as sped-up video rather than a faster animal.

**Fix:** add a faster clip to the blend tree rather than stretching the existing one
(Section 13), and keep the playback clamp tight (Section 12).

## 7.2 Ground detection

A ground raycast of fixed length starts missing, because the dinosaur travels further per
frame than the ray reaches ahead or below.

```text
At 4 m/s, 60fps:   6.6 cm per frame
At 12 m/s, 60fps: 20.0 cm per frame
At 12 m/s, 30fps: 40.0 cm per frame
```

**Fix:** scale ground and foot ray lengths with `speed * deltaTime`, and add a margin.

## 7.3 Collision tunnelling

`CharacterController` and swept rigidbodies can skip thin colliders at speed.

**Fix:** raise the physics tick rate, enable continuous collision detection, and avoid
paper-thin world colliders on traversable ground.

## 7.4 Turn rate becomes absurd

A dinosaur that can rotate 180°/s at 4 m/s and still rotate 180°/s at 12 m/s looks like a
hovering camera, not an animal.

**Fix:** turn rate must fall as speed rises (Section 30). This is also what *creates* the
slide, so it is not optional here.

## 7.5 Camera

At high speed a rigid camera makes the world feel like it is teleporting past.

**Fix:** camera spring lag, slight FOV widening, and a small amount of look-ahead along
the velocity vector (not the facing vector — see Section 25).

## 7.6 AI pathing

NavMesh corner-cutting that looked fine at walking pace becomes visible overshoot at
sprint.

**Fix:** AI should feed *desired direction* into the same movement component (Section 42)
and be allowed to overshoot and correct, exactly like a player.

## 7.7 Networking

Higher speed means larger position error per unit of latency.

**Fix:** see Section 27.

## 7.8 Stamina and pacing

If top speed rises but stamina does not change, chases end faster and the game gets
shallower.

**Fix:** Section 8.

## Checklist

```text
[ ] Re-measured natural animation speeds
[ ] Added / verified a clip at the new top speed
[ ] Playback clamp still within 0.85 - 1.30
[ ] Blend tree thresholds updated
[ ] Ground + foot ray lengths scale with speed
[ ] Physics tick rate adequate
[ ] Turn rate curve re-tuned for the new top speed
[ ] Grip re-tuned (higher speed = more slide at the same grip)
[ ] Stamina drain re-balanced
[ ] Camera FOV / lag re-tuned
[ ] AI tested at the new speed
```

---

# 8. Speed Tiers, Sprint and Stamina

Rather than one `maxSpeed`, define tiers. Evrima's feel comes largely from the fact that
top speed is **rationed**, not constant.

```text
TIER        SPEED        STAMINA        NOTES

Walk        1.8 m/s      regenerates    default, quiet
Trot/Run    4.0 m/s      neutral        sustainable travel
Sprint      7.0 m/s      drains         chase / escape
Burst       9.0 m/s      drains fast    short, 2-4 seconds, cooldown
```

Conceptual behaviour:

```text
Stamina
  ^
  |‾‾‾‾‾‾╲
  |       ╲          sprint
  |        ╲
  |         ╲___
  |             ‾‾‾╲     burst
  |                 ╲
  |                  ╲________
  |                           ╱‾‾‾  regen after a delay
  +------------------------------------> time
```

Key points:

- **Exhausted ≠ instantly slow.** When stamina hits zero, lower the *target* speed and let
  normal deceleration carry the dinosaur down. A hard speed cut is very visible.
- **Burst should not be spammable.** A cooldown plus a stamina floor is enough.
- **Grip should drop during burst.** A dinosaur at maximum effort is less able to turn.
  This is a very cheap way to make top speed feel dangerous rather than just numerically
  larger, and it feeds directly into the slide system.

```csharp
// inside the movement config resolution
float staminaGripPenalty = isBursting ? 0.6f : 1f;
grip *= staminaGripPenalty;
```

---

# 9. Selling Speed Without Increasing It

Before raising numbers again, be aware that perceived speed is mostly not the number.
These are cheaper and they do not break animation:

- **FOV**: widen by 6–12° between run and sprint, eased in over ~0.3s.
- **Camera lag**: let the camera trail slightly and settle, rather than being welded on.
- **Ground FX**: dust, grass displacement, debris scaled by speed.
- **Near-field motion**: foliage brushing, screen-edge particles.
- **Footstep audio cadence**: driven by the *foot contact curves* (Section 15), so audio
  is automatically in sync with the stride and speeds up correctly.
- **Wind / breathing audio** layered by speed tier.
- **Slight camera shake** at sprint, more at burst, and a distinct texture during slides.
- **Motion blur**, used sparingly.

A raptor at 7 m/s with all of the above reads faster than one at 11 m/s without any of it,
and the 7 m/s version will not fight your animation system.

---
---

# PART III — ANIMATION AND SYNCHRONIZATION

---

# 10. Prepare the Animations

For each dinosaur, ideally have:

```text
Idle
Walk
Run
Sprint
Start
Stop
Turn Left
Turn Right
```

Plus, for the sliding system (Part IV):

```text
Skid Left        (body leaning into a leftward slide, legs braced)
Skid Right
Brake Slide      (straight-line skid, both feet planted forward)
```

You do NOT necessarily need many speed-specific animations.

For example, a Raptor can use:

```text
Walk
Run
Sprint
```

instead of:

```text
Walk_1
Walk_2
Walk_3
Run_1
Run_2
Run_3
Run_4
Sprint_1
Sprint_2
Sprint_3
...
```

The locomotion system should provide the speed variation.

## Note on the skid clips

The skid clips can be **additive** or a thin override layer rather than full-body loops.
An additive lean pose over the normal run cycle already reads as a drift, and it costs
far less to author than a full skid cycle per dinosaur. Author them as:

```text
Skid pose = torso lean into the turn
          + tail counter-swing
          + head stays on target
          + legs braced wider
```

Keep the legs' *cycle* in the base layer and let the slide only affect posture and
playback rate. This is the cheapest path to a convincing drift.

---

# 11. Determine the Natural Speed of Each Animation

This is one of the most important setup steps.

For every locomotion animation, determine approximately how much distance one complete
stride travels.

Example:

```text
Raptor Run Animation

Stride distance: 2.2 m
Animation duration: 0.55 sec

Natural speed:

2.2 / 0.55
≈ 4.0 m/s
```

Store this as the animation's natural speed.

Example configuration:

```text
Walk   = 1.8 m/s
Run    = 4.0 m/s
Sprint = 6.5 m/s
```

These values do not have to be the dinosaur's final gameplay speeds.

They are synchronization reference values.

## How to measure it reliably

1. Put the clip on the rig with root motion **enabled temporarily**, in an empty scene.
2. Play exactly one full cycle and record the root's travel distance.
3. Divide by the clip's duration.
4. Disable root motion again.

If the clip is authored in place (no root travel), measure instead from a single foot:
the distance a planted foot travels backwards relative to the root during its contact
phase, divided by the time of that contact phase.

---

# 12. Blended Natural Speed — The Correct Playback Formula

A naive implementation:

```csharp
float actualSpeed = movementVelocity.magnitude;
float naturalAnimationSpeed = 4.0f;          // <- one fixed value
float playbackRate = actualSpeed / naturalAnimationSpeed;
playbackRate = Mathf.Clamp(playbackRate, 0.75f, 1.75f);
```

This is wrong across the full speed range, because the blend tree is not playing one clip
— it is cross-fading two. At speed 5.0 the pose is roughly half Run and half Sprint, so
its effective natural speed is somewhere between the two.

**The correct version interpolates the natural speed the same way the tree interpolates
the clips:**

```csharp
float NaturalSpeedFor(float speed)
{
    if (speed <= cfg.walkSpeed)
        return cfg.walkNaturalSpeed;

    if (speed <= cfg.runSpeed)
        return Mathf.Lerp(
            cfg.walkNaturalSpeed,
            cfg.runNaturalSpeed,
            Mathf.InverseLerp(cfg.walkSpeed, cfg.runSpeed, speed));

    return Mathf.Lerp(
        cfg.runNaturalSpeed,
        cfg.sprintNaturalSpeed,
        Mathf.InverseLerp(cfg.runSpeed, cfg.sprintSpeed, speed));
}
```

Then:

```csharp
float natural  = NaturalSpeedFor(matchSpeed);
float playback = natural > 0.01f ? matchSpeed / natural : 1f;
playback = Mathf.Clamp(playback, cfg.minPlaybackRate, cfg.maxPlaybackRate);
```

With blend thresholds placed at the clips' natural speeds, the playback rate now sits near
**1.0 across the whole range**, and only departs from it to cover acceleration, slopes,
turn slowdown and tuning drift.

Recommended clamp:

```text
minPlaybackRate = 0.85
maxPlaybackRate = 1.30
```

If the required rate is regularly pinned at the clamp, that is the system telling you a
clip is missing — add one, do not widen the clamp.

## What speed to match against

Critically, `matchSpeed` is **not** `velocity.magnitude`. It is the component of velocity
along the facing direction:

```csharp
float matchSpeed = Mathf.Max(0f, move.ForwardSpeed);   // Vector3.Dot(velocity, forward)
```

During a slide, total speed can be high while the legs are producing very little forward
drive. Matching against total speed would make the legs sprint during a sideways skid.
Matching against the forward component makes the legs do exactly as much work as they are
actually doing, and the remaining motion is read by the player as momentum — which is
precisely the effect you want.

---

# 13. Locomotion Blend Tree

Create a 1D Blend Tree driven by:

```text
Speed
```

Example:

```text
Speed:

0.0       Idle
1.8       Walk
4.0       Run
6.5       Sprint
```

Conceptually:

```text
Idle ───── Walk ───────── Run ───────── Sprint
0         1.8            4.0            6.5 m/s
```

**Place each threshold at that clip's measured natural speed**, not at a round number.
That is what keeps the playback rate near 1.0 (Section 12).

The exact values must be tuned per dinosaur.

Do not assume a Raptor and Troodon should use identical speed thresholds.

If your top gameplay speed is meaningfully above your fastest clip's natural speed, add a
clip rather than relying on playback stretch.

---

# 14. Use Actual Velocity, Not Input Magnitude

Do not simply do:

```csharp
animator.SetFloat("Speed", input.magnitude);
```

This can produce incorrect animation when:

- The dinosaur is blocked.
- The dinosaur is turning.
- The dinosaur is sliding on a slope.
- The dinosaur is drifting through a turn.
- The AI cannot reach its target.
- The character is pushed.
- Knockback occurs.
- Network interpolation is active.

Prefer actual movement velocity:

```csharp
Vector3 horizontalVelocity = new Vector3(velocity.x, 0f, velocity.z);
float actualSpeed = horizontalVelocity.magnitude;
```

And, per Section 12, feed the animator the forward component of that velocity:

```csharp
float forwardSpeed = Vector3.Dot(horizontalVelocity, transform.forward);
animator.SetFloat("Speed", Mathf.Max(0f, forwardSpeed));
```

Best of all, derive the velocity from the position actually achieved after collision
resolution:

```csharp
Vector3 before = transform.position;
characterController.Move(velocity * dt);
Vector3 measured = (transform.position - before) / dt;
```

This is the only value that is guaranteed to be what the player sees. The animation should
represent what the dinosaur is actually doing.

---

# 15. Animation Curves and Foot Contact

A useful advanced technique is adding animation curves.

For example:

```text
LeftFootContact
RightFootContact
```

The animation can contain a curve:

```text
1 = planted
0 = swinging
```

Example:

```text
Time:

0----0.25----0.50----0.75----1.0

Left:
    ____████████____

Right:
    ████____██████
```

The locomotion system can use these curves to know when a foot should be locked.

This is more reliable than guessing foot contact from time alone.

These same curves should drive:

- Foot locking (Section 35)
- Footstep audio
- Dust / impact FX
- Camera step-bob, if used

Driving all of them from one curve means they stay in sync automatically as the playback
rate changes with speed.

---

# 16. Stride Warping / Speed Warping

For an advanced implementation, consider stride warping.

Instead of only changing animation playback speed:

```text
Playback Speed
```

you can adjust the apparent stride length.

Conceptually:

```text
Original:

👣------👣


Warped:

👣----------👣
```

This can help maintain natural cadence over a wider range of movement speeds.

Use this only after the basic system works.

Do not begin with stride warping.

**Related technique — orientation warping.** Where stride warping adjusts step *length*,
orientation warping rotates the lower body to point along the velocity vector while the
upper body keeps facing the target. That is a partial, animation-side approximation of the
slide in Part IV. It is a good polish pass **on top of** a real momentum model, but it is
not a substitute for one: warping the pose does not give the dinosaur mass, and gameplay
will still feel like it turns on a dime.

---
---

# PART IV — MOMENTUM, GRIP AND INTENTIONAL SLIDING

---

# 17. What Evrima-Style Sliding Actually Is

It is worth being precise, because "sliding" gets used for several different things.

What Evrima does well, mechanically, is this:

1. **Velocity is a vector with mass behind it.** The dinosaur's momentum is stored in the
   world, not as "speed along my forward axis". Rotating the body does not rotate the
   momentum.
2. **Turning authority is limited by speed.** The faster you go, the less you can rotate
   per second.
3. **The feet can only apply so much sideways force.** When the turn demands more lateral
   force than the feet can supply, the animal skids outward.
4. **The skid is controllable.** You can steer through it, and you can shorten it by
   giving up speed.
5. **The skid is a real cost.** During the slide you are not accelerating, you have less
   directional control, and you are committed for a moment.

That last point is what makes it feel good rather than annoying. The slide is the price of
carrying speed into a corner, and skilled players learn to set up their turns.

```text
        WITHOUT MOMENTUM              WITH MOMENTUM (target)

        ────────┐                     ────────╮
                │                              ╲
                │                               ╲
                │                                ╲─────
                │
    instant 90° corner            speed carries you wide,
    no cost, no skill             you plan the corner
```

## What it is not

- It is not a button that plays a slide animation. Pressing a key to "do a slide" is a
  different mechanic (a dive/slide ability) and feels scripted.
- It is not low friction everywhere. A dinosaur that slides while walking feels like it
  is on ice.
- It is not an animation trick. If gameplay still turns instantly and only the mesh leans,
  players feel the mismatch immediately.

Momentum sliding must be implemented in the **movement solver**, and the animation layer
just visualizes it.

---

# 18. The Grip Model

This is the whole mechanic, and it is surprisingly small.

## The idea

Store velocity in **world space**. Each frame:

1. Rotate the body toward the desired direction (limited by turn rate).
2. Re-express the existing world velocity in the **new** facing frame. A lateral component
   appears automatically, because the frame rotated and the velocity did not.
3. Bleed that lateral component toward zero at a limited rate — this is **grip**.
4. Apply forward acceleration / braking to the forward component.
5. Recompose and move.

```text
Step 1: body rotates            Step 2: velocity now has a lateral part

   facing ──>                      facing ──>
   vel    ──>                      vel      ╲  = forward part + lateral part

Step 3: grip pulls the velocity back toward facing, at a limited rate

   high grip:  velocity snaps to facing    -> no slide
   low grip:   velocity lags behind        -> slide
```

`lateralGrip` is expressed in **m/s²** — how much sideways acceleration the feet can
generate. That is all the tuning surface you need for the basic mechanic.

## The code

```csharp
// after the body has already been rotated this frame
Vector3 fwd   = transform.forward;
Vector3 right = transform.right;

float vF = Vector3.Dot(horizontalVelocity, fwd);     // forward component
float vL = Vector3.Dot(horizontalVelocity, right);   // lateral component

float grip = cfg.lateralGrip
           * surfaceGrip                              // mud, wet rock, snow
           * (isBraking  ? cfg.brakeGripMultiplier : 1f)
           * (isBursting ? cfg.burstGripMultiplier : 1f);

// bleed the sideways motion away, but only as fast as the feet allow
vL = Mathf.MoveTowards(vL, 0f, grip * Time.deltaTime);

// forward drive / braking happens on vF (Sections 28-29)
vF = Mathf.MoveTowards(vF, targetForwardSpeed, driveRate * Time.deltaTime);

horizontalVelocity = fwd * vF + right * vL;
```

That is the entire momentum-slide system. Everything else in Part IV is detection,
presentation and tuning on top of these lines.

## Why this model, and not friction / physics materials

- It is **deterministic and cheap** — no solver, no physics material tuning, no jitter.
- It is **one readable parameter per dinosaur** rather than an emergent property of
  colliders.
- It works identically for `CharacterController`, `Rigidbody` and custom movement.
- It behaves correctly at any frame rate.
- AI gets the same behaviour for free, because AI feeds the same desired direction.

## When does a slide begin?

You do not need a trigger condition. It emerges: if the turn rotates the facing faster
than `lateralGrip` can rotate the velocity, a slip angle opens up on its own. The physical
relationship is:

```text
Lateral acceleration demanded by a turn  =  speed × yawRate(radians/s)

If  demanded > lateralGrip   ->  the dinosaur slides
```

Worked example for a raptor with `lateralGrip = 12 m/s²`:

```text
At 4 m/s, turning 120°/s (2.09 rad/s):
    demanded = 4 × 2.09 = 8.4 m/s²   ->  8.4 < 12  ->  grips, no slide

At 7 m/s, turning 120°/s:
    demanded = 7 × 2.09 = 14.6 m/s²  ->  14.6 > 12 ->  slides

At 7 m/s, turning 60°/s (1.05 rad/s):
    demanded = 7 × 1.05 = 7.3 m/s²   ->  7.3 < 12  ->  grips
```

So the same animal slides or grips depending on how hard it is asked to turn at speed.
That is exactly the Evrima behaviour, and you did not have to script a single case.

## Tuning grip

```text
lateralGrip     Feel

25+             Rails. Effectively no sliding. Arcade.
14 - 20         Slight drift at top speed only. Forgiving.
 9 - 13         Evrima-like. Slides on committed turns.      <- start here
 5 -  8         Slippery, heavy, hard to control. Large animals.
 < 5            Ice. Usually a bug.
```

Larger dinosaurs should have **lower** grip relative to their speed, which is what makes
them feel heavy. A T. rex that corners like a raptor loses all sense of mass.

## Surface grip

```csharp
// set by the ground probe, from the surface under the foot
float surfaceGrip = 1.0f;   // dry dirt / rock
// mud        0.55
// wet rock   0.70
// sand       0.75
// snow       0.50
// shallow water 0.65
```

Multiplying into the grip term is enough. It instantly makes terrain matter, costs
nothing, and reuses the entire slide presentation you already built.

---

# 19. Slip Angle — The Single Value That Drives Everything

Define:

```text
slipAngle = signed angle between facing direction and velocity direction, in degrees
```

```csharp
Vector3 velDir = horizontalVelocity.normalized;
float slipAngle = Vector3.SignedAngle(transform.forward, velDir, Vector3.up);
```

```text
             velocity
                ╲
                 ╲  slipAngle
                  ╲
   facing ─────────>

slipAngle  ≈   0°   running true
slipAngle  =  20°   drifting slightly right of facing
slipAngle  = -45°   hard slide to the left
slipAngle  ≈ 180°   moving backwards (sliding after a reversal)
```

Guard it: below a minimum speed the direction of a tiny velocity is noise, so freeze the
slip angle to 0 when `speed < slideMinSpeed * 0.5`.

From this one value, derive everything:

```csharp
float absSlip = Mathf.Abs(slipAngle);

// 0 at the entry threshold, 1 at "fully committed slide"
float slideAmount = Mathf.InverseLerp(cfg.slideSlipEnter, cfg.slideSlipFull, absSlip);
slideAmount *= Mathf.InverseLerp(cfg.slideMinSpeed * 0.5f, cfg.slideMinSpeed, speed);
```

| Consumer | Uses |
|---|---|
| Slide state machine | `absSlip`, `speed` |
| Skid layer weight | `slideAmount` |
| Skid left/right blend | `sign(slipAngle)` |
| Playback rate | `ForwardSpeed`, `slideAmount` |
| Foot locking | `slideAmount` (disables it) |
| Foot IK weight | `slideAmount` (reduces it) |
| Dust / skid FX | `slideAmount`, `LateralSpeed` |
| Skid audio | `slideAmount`, `speed` |
| Camera shake / roll | `slideAmount`, `sign(slipAngle)` |
| Stamina cost | `slideAmount` |

Driving everything from one smoothed value is what makes the feature feel like one
mechanic rather than a pile of effects.

---

# 20. The Slide State Machine

Use hysteresis so the state does not flicker at the boundary.

```text
                 absSlip > enterAngle
                 AND speed > minSpeed
    GRIPPED ─────────────────────────────> SLIDING
        ^                                     │
        │                                     │
        │      absSlip < exitAngle            │
        │      OR speed < minSpeed*0.6        │
        └──── RECOVERING <────────────────────┘
                  │  ^
                  │  │ absSlip rises again
                  v  │
               GRIPPED
```

Suggested values for a raptor:

```text
slideSlipEnter   = 12°     start showing the slide
slideSlipFull    = 45°     fully committed slide
slideSlipExit    =  6°     stop showing it
slideMinSpeed    = 3.5 m/s below this, no sliding at all
recoveryTime     = 0.25 s  blend-out time for the skid layer
```

Implementation:

```csharp
void UpdateSlideState(float absSlip, float speed, float dt)
{
    if (!IsSliding)
    {
        if (absSlip > cfg.slideSlipEnter && speed > cfg.slideMinSpeed)
            IsSliding = true;
    }
    else
    {
        if (absSlip < cfg.slideSlipExit || speed < cfg.slideMinSpeed * 0.6f)
            IsSliding = false;
    }

    float target = IsSliding
        ? Mathf.InverseLerp(cfg.slideSlipEnter, cfg.slideSlipFull, absSlip)
        : 0f;

    // asymmetric smoothing: enter quickly, leave gently
    float rate = target > SlideAmount ? cfg.slideAttack : cfg.slideRelease;
    SlideAmount = Mathf.MoveTowards(SlideAmount, target, rate * dt);
}
```

`slideAttack ≈ 6` and `slideRelease ≈ 3` per second works well: the skid snaps on when the
animal breaks loose and eases off as it regains grip, which is how it looks in reality.

**Do not gate the slide on a button.** It is a consequence of speed, turn input and grip.
If the player has to ask for it, it stops being momentum.

---

# 21. Slide Triggers

All of these fall out of the grip model. They are listed so you can verify each one works,
not because each needs separate code.

## 21.1 Turn drift — the main case

Sprinting, then a sharp turn. Yaw exceeds what grip can follow, slip angle opens, the
dinosaur arcs wide.

```text
        ╭──── intended path
       ╱
      ╱  ····· actual path (wider)
     ╱  ·
    🦖 ·
```

## 21.2 Brake slide

Braking hard at speed reduces available grip (`brakeGripMultiplier`) while also
decelerating. If the player turns while braking, the slide is dramatic. If not, it is a
straight-line skid with the feet forward.

```csharp
float grip = cfg.lateralGrip * (isBraking ? cfg.brakeGripMultiplier : 1f);
float decel = isBraking ? cfg.brakeDeceleration : cfg.deceleration;
```

Detect the straight-line variant for animation purposes:

```csharp
bool isBrakeSlide = isBraking
                 && speed > cfg.slideMinSpeed
                 && Mathf.Abs(slipAngle) < cfg.slideSlipEnter;
```

## 21.3 Slope slide

On a steep slope, project gravity onto the slope plane and add it to velocity, and scale
grip down with slope angle.

```csharp
if (slopeAngle > cfg.slopeSlideAngle)
{
    Vector3 down = Vector3.ProjectOnPlane(Vector3.down, groundNormal).normalized;
    horizontalVelocity += down * cfg.slopeSlideAccel * dt;
    grip *= Mathf.InverseLerp(cfg.slopeMaxAngle, cfg.slopeSlideAngle, slopeAngle);
}
```

This reuses the whole system: descending a wet hill too fast now produces exactly the
same skid presentation.

## 21.4 Landing at speed

On the landing frame, reduce grip for a short window. A dinosaur that lands from a jump
at full sprint should scrabble for a moment before it bites into the ground.

```csharp
if (Time.time - lastLandTime < cfg.landingGripRecovery)
    grip *= cfg.landingGripMultiplier;   // ~0.5
```

## 21.5 Panic turn / AI evasion

AI feeding a hard direction change into the same movement component gets the same drift
for free. Prey animals juking a predator will naturally slide, and predators will
naturally overshoot. This is a large part of why chases read well.

---

# 22. Steering and Recovering From a Slide

A slide the player cannot influence feels like a punishment. A slide they can work with
feels like a skill.

## What the player can do mid-slide

**Counter-steer.** Rotating the body during a slide is allowed, at a reduced rate:

```csharp
float turnRate = TurnRateForSpeed(speed) *
                 Mathf.Lerp(1f, cfg.slideTurnMultiplier, SlideAmount);   // ~0.6
```

Because the velocity chases the facing through grip, turning into the slide shortens it
and turning further away extends it. That is a real, learnable control loop and it comes
free from the model.

**Release the throttle.** Lower speed means lower demanded lateral acceleration, so grip
catches up. Slowing down ends a slide — the correct intuition.

**Brake.** Reduces speed fast but also reduces grip. Braking mid-drift is a trade-off, not
a universal fix. This is good design tension; keep it.

## Forward drive during a slide

Do **not** let the dinosaur accelerate at full rate while sliding. The feet are busy.

```csharp
float driveScale = Mathf.Lerp(1f, cfg.slideDriveMultiplier, SlideAmount);  // ~0.25
```

This is what makes carrying speed into a corner cost something, which is what makes
taking the corner cleanly feel like a win.

## Recovery

No special code. As slip angle closes, `slideAmount` falls, the skid layer blends out,
foot locking re-enables, and drive returns. Keep `slideRelease` slower than `slideAttack`
so the recovery reads as the animal settling rather than snapping.

---

# 23. Animating a Slide

The animation layer never decides anything. It reads `SlideAmount` and `SlipAngle` and
presents them.

## Layer setup

```text
Base Layer            Idle / Walk / Run / Sprint blend tree
                      (playback-rate matched, Section 12)

Skid Layer            weight = SlideAmount
(additive or          1D blend on SlipAngleNormalized:
 upper-body mask)         -1  SkidLeft
                           0  Neutral
                          +1  SkidRight

Brake Layer           weight = BrakeSlideAmount
(additive)            straight-line skid pose
```

Parameters fed in:

```csharp
animator.SetFloat("SlideAmount", move.SlideAmount, 0.05f, dt);
animator.SetFloat("SlipAngleNormalized",
    Mathf.Clamp(move.SlipAngle / cfg.slideSlipFull, -1f, 1f), 0.05f, dt);
animator.SetBool("IsSliding", move.IsSliding);
animator.SetLayerWeight(skidLayer, move.SlideAmount);
```

## What the skid pose should contain

```text
Torso        leans INTO the turn (toward the inside of the arc)
Hips         rotate slightly toward the velocity direction
Legs         braced wider, knees more bent, stance lower
Tail         swings OUT, counterbalancing — this sells it more than anything
Head/neck    stays locked on the direction of travel or the target
```

The tail is the highest-value part. A raptor tail whipping out through a drift
communicates mass and speed better than any leg detail.

## Playback rate during a slide

The legs are bracing, not cycling. Blend the stride cadence down:

```csharp
float playback = Mathf.Clamp(matchSpeed / natural,
                             cfg.minPlaybackRate, cfg.maxPlaybackRate);

playback = Mathf.Lerp(playback, cfg.slidePlaybackRate, move.SlideAmount);  // ~0.35
```

Combined with `matchSpeed` being the *forward* component (Section 12), a full sideways
slide almost stops the leg cycle, which is correct: the animal is being carried, not
running.

## The critical ordering

```text
SlideAmount rises
      ↓
Foot locking OFF          <-- must happen first
      ↓
Foot IK weight reduced
      ↓
Skid layer blends in
      ↓
Playback rate blends down
```

If foot locking is still active when the skid begins, the locked foot will fight the
sliding body and the leg will visibly stretch or snap. Section 24.

---

# 24. Foot IK and Foot Locking During a Slide

This is where most implementations of this feature go wrong, so it gets its own section.

## The conflict

```text
Foot locking says:     "this foot is planted, hold its world position"
The slide says:        "the whole animal is moving sideways across the ground"

Both at once:          the leg stretches toward a point the body has left
                       -> knee pops, foot snaps, IK jitter
```

## The resolution

Treat foot locking as **something the grip system grants permission for**:

```csharp
// foot locking strength
float lockWeight = footContactCurve          // from the animation curve, Section 15
                 * (1f - move.SlideAmount);  // revoked as the slide develops

// general terrain IK weight
float ikWeight = Mathf.Lerp(cfg.footIKWeight,
                            cfg.footIKSlideWeight,   // ~0.35
                            move.SlideAmount);
```

Terrain IK is still useful during a slide — the feet should stay on the ground surface —
but it should be gentler, and it must not try to hold a fixed world position.

## Summary table

| | Gripped | Sliding |
|---|---|---|
| Foot locking | On, full | **Off** |
| Terrain IK height | Full weight | Reduced (~0.35) |
| Foot rotation to normal | Full | Reduced |
| Pole/knee correction | Full | Reduced, heavily damped |
| Stride matching | Active | Blended down |
| Visible foot slide | **Never** | **Expected and correct** |

## A useful debugging rule

If a foot slides, ask one question first: **is `SlideAmount` above zero?**

```text
SlideAmount = 0  and feet slide   ->  BUG. Go to Part III (stride matching).
SlideAmount > 0  and feet slide   ->  WORKING AS DESIGNED.
```

This single check converts the vague complaint "the feet slide" into a specific,
answerable question, and it is the main practical reason to expose `SlideAmount` in the
debug overlay.

---

# 25. Slide Feedback — FX, Audio, Camera

The mechanic is in the solver; the *feel* is here. All of it reads from `SlideAmount` and
`LateralSpeed`.

## Particles

```csharp
var emission = skidDust.emission;
emission.rateOverTime = move.SlideAmount * Mathf.Abs(move.LateralSpeed) * cfg.dustScale;
```

Emit from the **foot positions**, not the root, and orient the burst against the lateral
velocity direction. Tint from the surface material so mud, sand and snow differ.

## Ground decals

Skid marks stamped along the velocity path while `SlideAmount > 0.3`, fading over time.
Cheap, and they let players *read the terrain* after a chase.

## Audio

```text
Gripped:   footstep impacts on contact curve
Sliding:   continuous skid/scrape loop, volume = SlideAmount,
           pitch = speed, sample set = surface type
```

Crossfade rather than cut. The transition into a skid sound is most of the impact.

## Camera

```text
Roll:        small, toward the inside of the slide, proportional to SlipAngle
Shake:       texture increases with SlideAmount
Look-ahead:  bias toward the VELOCITY direction, not the facing direction
FOV:         hold the sprint FOV through the slide, don't drop it
```

The look-ahead point is worth emphasising: during a drift the player wants to see where
they are going, which is no longer where the dinosaur is pointing. Following the facing
vector during a slide makes the camera feel like it is fighting the player.

---

# 26. Sliding as a Gameplay Mechanic

Sliding should have costs and uses, or it is just decoration.

## Costs

- **No acceleration during a slide** (`slideDriveMultiplier`).
- **Reduced turn authority** (`slideTurnMultiplier`).
- **Stamina drain** proportional to `SlideAmount` — skidding is hard work.
- **Commitment** — a long drift is a window during which you cannot change plan.

## Uses

- **Cornering at speed** is faster overall than slowing to turn cleanly, if executed well.
- **Juking** — a prey animal can break a predator's lock by forcing an overshoot.
- **Terrain reading** — knowing which surfaces are low grip becomes real knowledge.
- **Size differentiation** — heavy animals drift long and wide; small ones are nimble.
  This is a *free* characterization tool, and it costs one number per dinosaur.

## Balance warning

Do not let sliding be faster than not sliding in a straight line. If a drift preserves
more speed than running, players will drift permanently and the mechanic collapses.
The forward-drive penalty is what prevents this — verify it.

---

# 27. Networking Momentum and Sliding

Sliding is unusually friendly to networking, because it is derived rather than stated.

## Replicate

```text
position
rotation (yaw)
velocity (world, full vector)     <- essential
grounded
surfaceGrip (or surface id)
stamina / tier flags
```

## Do NOT replicate

```text
SlideAmount
SlipAngle
IsSliding
playback rate
```

Every one of those is a **pure function of velocity and rotation**. Remote proxies
recompute them locally from the interpolated velocity and yaw, and arrive at the same
answer. This is the main reason to keep the slide derived from a slip angle rather than
stored as a state flag: it costs zero bandwidth and it cannot desync from the motion the
client is actually rendering.

```csharp
// identical code path on owner and proxy
float slip = Vector3.SignedAngle(transform.forward, velocity.normalized, Vector3.up);
UpdateSlideState(Mathf.Abs(slip), velocity.magnitude, dt);
```

## Prediction

The grip model is deterministic and frame-rate independent, so it re-simulates cleanly
under client-side prediction and rollback. Keep it free of `Random` and of any per-frame
state that is not in the replicated set.

## Interpolation caution

If a proxy's velocity is reconstructed by differencing interpolated positions, smooth it
before computing the slip angle, or the skid layer will flicker on network jitter. A
simple exponential smooth over ~0.1s is enough.

---
---

# PART V — ACCELERATION, DECELERATION AND TURNING

---

# 28. Acceleration

A fast dinosaur should not instantly jump from:

```text
0 -> 7 m/s
```

Use acceleration.

Example:

```text
0
|
|       acceleration
|      /
|     /
|    /
|___/________________
        time
```

Conceptual values:

```text
Troodon:

Acceleration: 8 m/s²
Max speed:    7 m/s
```

The exact values should be tuned through gameplay testing.

Acceleration is applied to the **forward component** of velocity only (Section 18). A
dinosaur cannot accelerate sideways, and applying acceleration to the whole velocity
vector would quietly cancel out the slide you are trying to create.

```csharp
vF = Mathf.MoveTowards(vF, targetForwardSpeed, accel * driveScale * dt);
```

where `driveScale` falls with `SlideAmount` (Section 22).

---

# 29. Deceleration and Braking

Stopping is just as important as starting.

Avoid:

```text
7 m/s
  |
  |
  v
0 m/s instantly
```

Instead:

```text
7
 \
  \
   \
    \
     0
```

The animation should transition toward a stop animation as the actual velocity decreases.

Recommended states:

```text
Run
 |
 | release movement
 v
Decelerate
 |
 v
Stop
 |
 v
Idle
```

## Three distinct rates

```text
deceleration        no input, coasting to a stop      ~ 9 m/s²
brakeDeceleration   active brake input               ~ 14 m/s²
exhaustedDecel      stamina ran out, target drops    reuse deceleration
```

Braking additionally lowers grip (`brakeGripMultiplier`, Section 21.2), which is what
turns a hard stop at speed into a visible skid rather than a smooth glide down to zero.

## Stopping must not skate

The classic failure is the dinosaur reaching 0 velocity while a run cycle is still
playing out. Two safeguards:

1. Keep driving the blend tree from **measured** velocity, so the tree walks itself back
   down through Run → Walk → Idle as the speed falls.
2. Only transition to a dedicated `Stop` clip when velocity is genuinely low, and let
   that clip's own root-relative foot motion finish the deceleration visually.

---

# 30. Turning

Turning is one of the major reasons locomotion can look unnatural.

Do not allow:

```text
Raptor running forward
        |
        |
        +----------> instantly 90 degrees
```

Instead, use:

```text
Running
   |
   v
Turn influence
   |
   +---- reduce turn RATE as speed rises
   |
   +---- reduce target SPEED for large heading changes
   |
   v
Turn (possibly with slide)
   |
   v
Accelerate again
```

## 30.1 Turn rate must fall with speed

This is the input side of the slide mechanic.

```csharp
float TurnRateForSpeed(float speed)
{
    float t = Mathf.Clamp01(speed / cfg.sprintSpeed);
    return Mathf.Lerp(cfg.turnRateAtRest, cfg.turnRateAtSprint, t);
}
```

Example:

```text
Speed        Turn rate

0.0 m/s      200 °/s     turn in place, agile
1.8 m/s      170 °/s
4.0 m/s      130 °/s
7.0 m/s       85 °/s     committed, wide
```

## 30.2 Speed penalty for large heading changes

```text
Angle difference     Speed multiplier

0°                    1.00
15°                   0.95
30°                   0.85
60°                   0.65
90°                   0.40
```

These are example values only. Tune them per dinosaur.

Note what these two mechanisms do together: the turn-rate limit decides *how fast the body
can rotate*, the speed penalty decides *how much speed the animal is willing to carry*,
and the grip limit (Section 18) decides *what the momentum does about it*. The slide lives
in the gap between them.

```text
Speed
+
Turning
+
Grip
+
Animation
```

## 30.3 Turn in place

At very low speed, the dinosaur should rotate without translating, using a turn-in-place
animation rather than a stride. Gate it on `speed < turnInPlaceMaxSpeed` (~0.5 m/s) and a
sustained heading error, so it does not fire on tiny corrections.

---

# 31. Turn Animation

For stronger turns, use separate animations:

```text
TurnLeft
TurnRight
```

The decision can depend on signed angular velocity.

Conceptually:

```csharp
float turnAmount = ...;   // applied turn rate, deg/s, signed

if (turnAmount > threshold)
{
    // Turn right
}
else if (turnAmount < -threshold)
{
    // Turn left
}
```

For very small corrections, normal locomotion can handle the turn.

For larger turns, transition into turn animations.

## Turn lean vs skid lean

Keep these separate:

```text
Turn lean   driven by TurnRate      -> the animal banking into a turn it is making
Skid lean   driven by SlipAngle     -> the animal being carried through a turn it is losing
```

They can be additive on the same layer stack, but they mean different things and should
not share a parameter. A dinosaur banking cleanly through a fast turn and one drifting
sideways look different, and players read the difference.

---
---

# PART VI — FOOT PLACEMENT

---

# 32. Foot IK

Foot IK is useful but should not be treated as the only solution to foot sliding.

The correct order is:

```text
1. Correct movement speed
2. Correct animation/stride synchronization
3. Then use IK for terrain adaptation
```

Do not try to solve fundamentally incorrect movement speed using IK.

And, per Section 24: do not try to use IK to *prevent* a slide you deliberately created.

---

# 33. Foot IK Concept

For each foot:

```text
Animation foot position
        |
        v
Raycast downward
        |
        v
Ground hit
        |
        v
Calculate correction
        |
        v
Move foot toward ground
```

Example:

```text
       leg
        |
        |
        O  foot
        |
        |
--------X--------- ground
        ^
     raycast hit
```

If the terrain is uneven:

```text
             foot
              O
             /
            /
-------____/________ terrain
```

IK can move the foot to the terrain.

Remember to scale the ray length with speed (Section 7.2), and to cast from slightly
**ahead** of the foot along the velocity vector at high speed, so the correction is not
one frame late.

---

# 34. Do Not Overuse IK

Too much IK can create:

- Broken-looking legs
- Excessive knee bending
- Foot jitter
- Unnatural posture
- Animation fighting
- Performance cost

Use IK primarily for:

- Uneven terrain
- Small height differences
- Ground contact correction

Do not use it to completely reconstruct locomotion.

---

# 35. Foot Locking

For high-quality locomotion, the important moment is when a foot is planted.

Conceptually:

```text
FOOT CYCLE

Lift
  \
   \
    Contact
       |
       | LOCK
       |
       |
    Release
      /
     /
```

During the planted portion:

```text
Foot world position ≈ stable
```

During the swing phase:

```text
Foot follows animation
```

This reduces visible skating.

## Foot locking is conditional

Lock strength is the product of two things:

```csharp
float lockWeight = footContactCurve * (1f - slideAmount);
```

- `footContactCurve` — is this foot supposed to be planted right now? (Section 15)
- `1 - slideAmount` — is the animal entitled to a planted foot right now? (Section 24)

Also release the lock if the required correction exceeds a distance threshold (~0.3 m),
which catches teleports, knockback, network corrections and any case the grip model did
not anticipate.

---
---

# PART VII — IMPLEMENTATION

---

# 36. Data-Driven Locomotion Config

All tuning lives in one asset per dinosaur. The locomotion code itself is shared.

```csharp
using UnityEngine;

[CreateAssetMenu(menuName = "DinoBorn/Locomotion Config")]
public class DinoLocomotionConfig : ScriptableObject
{
    [Header("Speeds (m/s)")]
    public float walkSpeed   = 1.8f;
    public float runSpeed    = 4.0f;
    public float sprintSpeed = 7.0f;
    public float burstSpeed  = 9.0f;

    [Header("Drive (m/s^2)")]
    public float acceleration      = 7f;
    public float deceleration      = 9f;
    public float brakeDeceleration = 14f;

    [Header("Turning")]
    public float turnRateAtRest   = 200f;   // deg/s
    public float turnRateAtSprint = 85f;    // deg/s
    public float turnInPlaceMaxSpeed = 0.5f;
    [Tooltip("x = heading error / 180, y = speed multiplier")]
    public AnimationCurve turnSpeedPenalty =
        AnimationCurve.EaseInOut(0f, 1f, 1f, 0.35f);

    [Header("Grip / Sliding")]
    public float lateralGrip          = 12f;   // m/s^2 of sideways force the feet can make
    public float brakeGripMultiplier  = 0.45f;
    public float burstGripMultiplier  = 0.60f;
    public float landingGripMultiplier= 0.50f;
    public float landingGripRecovery  = 0.35f; // seconds
    public float slideMinSpeed        = 3.5f;
    public float slideSlipEnter       = 12f;   // deg
    public float slideSlipFull        = 45f;   // deg
    public float slideSlipExit        = 6f;    // deg
    public float slideAttack          = 6f;    // per second
    public float slideRelease         = 3f;    // per second
    [Range(0f, 1f)] public float slideDriveMultiplier = 0.25f;
    [Range(0f, 1f)] public float slideTurnMultiplier  = 0.60f;

    [Header("Slopes")]
    public float slopeSlideAngle = 38f;   // deg, start sliding downhill
    public float slopeMaxAngle   = 60f;
    public float slopeSlideAccel = 6f;    // m/s^2

    [Header("Animation natural speeds (m/s)")]
    public float walkNaturalSpeed   = 1.8f;
    public float runNaturalSpeed    = 4.0f;
    public float sprintNaturalSpeed = 6.5f;
    public float minPlaybackRate    = 0.85f;
    public float maxPlaybackRate    = 1.30f;
    [Range(0f, 1f)] public float slidePlaybackRate = 0.35f;

    [Header("Foot IK")]
    public bool  footIKEnabled     = true;
    [Range(0f, 1f)] public float footIKWeight      = 1.0f;
    [Range(0f, 1f)] public float footIKSlideWeight = 0.35f;
    public float footRayExtra      = 0.35f;   // metres, plus speed * dt
    public float footLockBreakDistance = 0.3f;

    [Header("Misc")]
    public float gravity = -25f;
    public float strideScale = 1f;

    public float MaxSpeed => Mathf.Max(sprintSpeed, burstSpeed);
}
```

Then create:

```text
RaptorConfig
TroodonConfig
TyrannosaurusConfig
TriceratopsConfig
...
```

all using the same locomotion code.

---

# 37. The Movement Component

This is the authoritative movement solver. Everything downstream reads from it.

```csharp
using UnityEngine;

[RequireComponent(typeof(CharacterController))]
public class DinoMovement : MonoBehaviour
{
    public DinoLocomotionConfig cfg;

    // ---- Input, written by the player controller or the AI ----------------
    public Vector3 DesiredDirection { get; set; }  // world space, XZ, normalized
    public float   DesiredSpeed     { get; set; }  // m/s
    public bool    BrakeRequested   { get; set; }
    public bool    IsBursting       { get; set; }
    public float   SurfaceGrip      { get; set; } = 1f;

    // ---- Output, read by animation / FX / audio / debug -------------------
    public Vector3 HorizontalVelocity { get; private set; }
    public float   Speed         { get; private set; }  // |horizontal velocity|
    public float   ForwardSpeed  { get; private set; }  // along facing
    public float   LateralSpeed  { get; private set; }  // across facing
    public float   SlipAngle     { get; private set; }  // signed degrees
    public float   SlideAmount   { get; private set; }  // 0..1, smoothed
    public bool    IsSliding     { get; private set; }
    public bool    IsBrakeSlide  { get; private set; }
    public float   TurnRate      { get; private set; }  // applied, deg/s
    public float   SlopeAngle    { get; private set; }
    public bool    IsGrounded    { get; private set; }

    CharacterController cc;
    Vector3 velocity;          // world space, includes Y
    Vector3 groundNormal = Vector3.up;
    float   lastLandTime = -99f;
    bool    wasGrounded;

    void Awake() => cc = GetComponent<CharacterController>();

    void Update()
    {
        float dt = Time.deltaTime;
        if (dt <= 0f) return;

        ProbeGround();
        ApplyTurn(dt);
        ApplyGripAndDrive(dt);
        ApplySlope(dt);
        ApplyGravity(dt);
        Integrate(dt);
        UpdateDerivedState(dt);
    }

    // ----------------------------------------------------------------------
    void ProbeGround()
    {
        wasGrounded = IsGrounded;
        IsGrounded  = cc.isGrounded;

        if (!wasGrounded && IsGrounded)
            lastLandTime = Time.time;

        float rayLength = cc.height * 0.5f + cfg.footRayExtra + Speed * Time.deltaTime;

        if (Physics.Raycast(transform.position + Vector3.up * 0.1f, Vector3.down,
                            out RaycastHit hit, rayLength))
        {
            groundNormal = hit.normal;
            SlopeAngle   = Vector3.Angle(hit.normal, Vector3.up);
            // SurfaceGrip can be read here from the hit material / terrain layer
        }
        else
        {
            groundNormal = Vector3.up;
            SlopeAngle   = 0f;
        }
    }

    // ----------------------------------------------------------------------
    void ApplyTurn(float dt)
    {
        if (DesiredDirection.sqrMagnitude < 0.0001f) { TurnRate = 0f; return; }

        float baseRate = Mathf.Lerp(cfg.turnRateAtRest, cfg.turnRateAtSprint,
                                    Mathf.Clamp01(Speed / cfg.sprintSpeed));

        // steering authority is reduced, but never removed, during a slide
        float rate = baseRate * Mathf.Lerp(1f, cfg.slideTurnMultiplier, SlideAmount);

        float error   = Vector3.SignedAngle(transform.forward, DesiredDirection, Vector3.up);
        float applied = Mathf.Clamp(error, -rate * dt, rate * dt);

        transform.Rotate(0f, applied, 0f, Space.World);
        TurnRate = applied / dt;
    }

    // ----------------------------------------------------------------------
    void ApplyGripAndDrive(float dt)
    {
        // The body has already rotated this frame. Re-express the UNCHANGED world
        // velocity in the NEW facing frame: the lateral component appears by itself.
        Vector3 fwd   = transform.forward;
        Vector3 right = transform.right;
        Vector3 hv    = new Vector3(velocity.x, 0f, velocity.z);

        float vF = Vector3.Dot(hv, fwd);
        float vL = Vector3.Dot(hv, right);

        // ---- lateral grip -------------------------------------------------
        float grip = cfg.lateralGrip * SurfaceGrip;
        if (BrakeRequested) grip *= cfg.brakeGripMultiplier;
        if (IsBursting)     grip *= cfg.burstGripMultiplier;
        if (Time.time - lastLandTime < cfg.landingGripRecovery)
            grip *= cfg.landingGripMultiplier;
        if (SlopeAngle > cfg.slopeSlideAngle)
            grip *= Mathf.InverseLerp(cfg.slopeMaxAngle, cfg.slopeSlideAngle, SlopeAngle);
        if (!IsGrounded) grip = 0f;                 // no grip in the air

        vL = Mathf.MoveTowards(vL, 0f, grip * dt);

        // ---- forward drive --------------------------------------------------
        float headingError = DesiredDirection.sqrMagnitude > 0.0001f
            ? Vector3.Angle(fwd, DesiredDirection)
            : 0f;

        float penalty = cfg.turnSpeedPenalty.Evaluate(Mathf.Clamp01(headingError / 180f));
        float target  = DesiredSpeed * penalty;

        float rate;
        if (BrakeRequested)        rate = cfg.brakeDeceleration;
        else if (target > vF)      rate = cfg.acceleration
                                        * Mathf.Lerp(1f, cfg.slideDriveMultiplier, SlideAmount);
        else                       rate = cfg.deceleration;

        if (!IsGrounded) rate *= 0.15f;             // almost no control in the air

        vF = Mathf.MoveTowards(vF, BrakeRequested ? 0f : target, rate * dt);

        hv = fwd * vF + right * vL;
        velocity = new Vector3(hv.x, velocity.y, hv.z);
    }

    // ----------------------------------------------------------------------
    void ApplySlope(float dt)
    {
        if (!IsGrounded || SlopeAngle <= cfg.slopeSlideAngle) return;

        Vector3 downhill = Vector3.ProjectOnPlane(Vector3.down, groundNormal).normalized;
        float strength = Mathf.InverseLerp(cfg.slopeSlideAngle, cfg.slopeMaxAngle, SlopeAngle);
        velocity += downhill * cfg.slopeSlideAccel * strength * dt;
    }

    void ApplyGravity(float dt)
    {
        velocity.y = IsGrounded && velocity.y < 0f
            ? -2f                                   // keep it pinned to the ground
            : velocity.y + cfg.gravity * dt;
    }

    // ----------------------------------------------------------------------
    void Integrate(float dt)
    {
        Vector3 before = transform.position;
        cc.Move(velocity * dt);

        // Trust the position that collision actually produced, not the one we asked for.
        Vector3 measured = (transform.position - before) / dt;
        velocity = new Vector3(measured.x, velocity.y, measured.z);
        HorizontalVelocity = new Vector3(measured.x, 0f, measured.z);
    }

    // ----------------------------------------------------------------------
    void UpdateDerivedState(float dt)
    {
        Speed = HorizontalVelocity.magnitude;
        ForwardSpeed = Vector3.Dot(HorizontalVelocity, transform.forward);
        LateralSpeed = Vector3.Dot(HorizontalVelocity, transform.right);

        // Below a minimum speed the velocity direction is noise.
        SlipAngle = Speed > cfg.slideMinSpeed * 0.5f
            ? Vector3.SignedAngle(transform.forward, HorizontalVelocity.normalized, Vector3.up)
            : 0f;

        float absSlip = Mathf.Abs(SlipAngle);

        if (!IsSliding)
        {
            if (absSlip > cfg.slideSlipEnter && Speed > cfg.slideMinSpeed)
                IsSliding = true;
        }
        else
        {
            if (absSlip < cfg.slideSlipExit || Speed < cfg.slideMinSpeed * 0.6f)
                IsSliding = false;
        }

        float target = IsSliding
            ? Mathf.InverseLerp(cfg.slideSlipEnter, cfg.slideSlipFull, absSlip)
            : 0f;

        float smooth = target > SlideAmount ? cfg.slideAttack : cfg.slideRelease;
        SlideAmount = Mathf.MoveTowards(SlideAmount, target, smooth * dt);

        IsBrakeSlide = BrakeRequested
                    && Speed > cfg.slideMinSpeed
                    && absSlip < cfg.slideSlipEnter;
    }
}
```

## Notes on this implementation

- **Order matters.** Turn first, then grip. Reversing them removes the slide entirely,
  because the lateral component would be cancelled before the rotation creates it.
- **Velocity is measured after `Move`.** Slopes, steps and blocked collisions are then
  reflected in the animation automatically.
- **No `Random`, no hidden state.** It re-simulates identically, which matters for
  prediction (Section 27).
- **Frame-rate independent.** Everything is a rate multiplied by `dt`; run the whole
  `Update` body from `FixedUpdate` instead if you prefer, with no changes.

---

# 38. The Animation Driver

Reads the movement component. Decides nothing.

```csharp
using UnityEngine;

[RequireComponent(typeof(Animator))]
public class DinoAnimationDriver : MonoBehaviour
{
    public DinoLocomotionConfig cfg;
    public DinoMovement move;
    public int skidLayer  = 1;
    public int brakeLayer = 2;

    static readonly int PSpeed      = Animator.StringToHash("Speed");
    static readonly int PPlayback   = Animator.StringToHash("PlaybackRate");
    static readonly int PTurnRate   = Animator.StringToHash("TurnRate");
    static readonly int PSlip       = Animator.StringToHash("SlipAngleNormalized");
    static readonly int PSlide      = Animator.StringToHash("SlideAmount");
    static readonly int PIsSliding  = Animator.StringToHash("IsSliding");
    static readonly int PGrounded   = Animator.StringToHash("Grounded");
    static readonly int PVertical   = Animator.StringToHash("VerticalSpeed");
    static readonly int PSlopeAngle = Animator.StringToHash("SlopeAngle");

    Animator animator;

    void Awake() => animator = GetComponent<Animator>();

    void Update()
    {
        float dt = Time.deltaTime;
        if (dt <= 0f) return;

        // The blend tree is driven by FORWARD speed, not total speed, so a sideways
        // slide does not make the legs sprint. (Section 12)
        float matchSpeed = Mathf.Max(0f, move.ForwardSpeed);

        float natural  = NaturalSpeedFor(matchSpeed);
        float playback = natural > 0.01f ? matchSpeed / natural : 1f;
        playback = Mathf.Clamp(playback, cfg.minPlaybackRate, cfg.maxPlaybackRate);

        // During a slide the legs brace rather than cycle.
        playback = Mathf.Lerp(playback, cfg.slidePlaybackRate, move.SlideAmount);

        animator.SetFloat(PSpeed,    matchSpeed, 0.08f, dt);
        animator.SetFloat(PPlayback, playback,   0.08f, dt);
        animator.SetFloat(PTurnRate, move.TurnRate, 0.08f, dt);
        animator.SetFloat(PSlide,    move.SlideAmount, 0.05f, dt);
        animator.SetFloat(PSlip,
            Mathf.Clamp(move.SlipAngle / cfg.slideSlipFull, -1f, 1f), 0.05f, dt);
        animator.SetFloat(PSlopeAngle, move.SlopeAngle, 0.10f, dt);
        animator.SetFloat(PVertical, 0f, 0.05f, dt);     // wire to vertical velocity
        animator.SetBool (PIsSliding, move.IsSliding);
        animator.SetBool (PGrounded,  move.IsGrounded);

        animator.SetLayerWeight(skidLayer,  move.SlideAmount);
        animator.SetLayerWeight(brakeLayer, move.IsBrakeSlide ? 1f : 0f);
    }

    // The blend tree cross-fades two clips, so the effective natural speed is the
    // interpolation of their natural speeds. (Section 12)
    float NaturalSpeedFor(float speed)
    {
        if (speed <= cfg.walkSpeed) return cfg.walkNaturalSpeed;

        if (speed <= cfg.runSpeed)
            return Mathf.Lerp(cfg.walkNaturalSpeed, cfg.runNaturalSpeed,
                              Mathf.InverseLerp(cfg.walkSpeed, cfg.runSpeed, speed));

        return Mathf.Lerp(cfg.runNaturalSpeed, cfg.sprintNaturalSpeed,
                          Mathf.InverseLerp(cfg.runSpeed, cfg.sprintSpeed, speed));
    }
}
```

Wire `PlaybackRate` into the blend tree's speed multiplier (a `Multiplier` parameter on
the blend tree's motion entries), **not** into `animator.speed` — `animator.speed` scales
every layer, including attacks and one-shots.

---

# 39. The Foot IK Component

A sketch. The important part is how it reacts to `SlideAmount`.

```csharp
using UnityEngine;

[RequireComponent(typeof(Animator))]
public class DinoFootIK : MonoBehaviour
{
    public DinoLocomotionConfig cfg;
    public DinoMovement move;

    Animator animator;
    Vector3 leftLock, rightLock;
    bool leftLocked, rightLocked;

    void Awake() => animator = GetComponent<Animator>();

    void OnAnimatorIK(int layerIndex)
    {
        if (!cfg.footIKEnabled) return;

        // Terrain adaptation stays on during a slide, but gentler.
        float ikWeight = Mathf.Lerp(cfg.footIKWeight, cfg.footIKSlideWeight,
                                    move.SlideAmount);

        SolveFoot(AvatarIKGoal.LeftFoot,  "LeftFootContact",  ikWeight,
                  ref leftLock,  ref leftLocked);
        SolveFoot(AvatarIKGoal.RightFoot, "RightFootContact", ikWeight,
                  ref rightLock, ref rightLocked);
    }

    void SolveFoot(AvatarIKGoal goal, string contactCurve, float ikWeight,
                   ref Vector3 lockPos, ref bool locked)
    {
        float contact = animator.GetFloat(contactCurve);   // 1 planted, 0 swinging

        // Foot LOCKING is revoked as the slide develops. During a real slide the
        // foot is supposed to move across the ground. (Section 24)
        float lockWeight = contact * (1f - move.SlideAmount);

        Vector3 animPos = animator.GetIKPosition(goal);

        // Ray length grows with speed so fast movement does not out-run the probe.
        float rayLen = 0.5f + cfg.footRayExtra + move.Speed * Time.deltaTime;
        Vector3 origin = animPos + Vector3.up * 0.5f;

        if (!Physics.Raycast(origin, Vector3.down, out RaycastHit hit, rayLen))
        {
            locked = false;
            animator.SetIKPositionWeight(goal, 0f);
            animator.SetIKRotationWeight(goal, 0f);
            return;
        }

        Vector3 grounded = new Vector3(animPos.x, hit.point.y, animPos.z);

        if (lockWeight > 0.5f)
        {
            if (!locked) { lockPos = grounded; locked = true; }

            // Break the lock if it is being stretched too far (teleport, knockback,
            // network correction, or a slide the grip model did not predict).
            if (Vector3.Distance(lockPos, grounded) > cfg.footLockBreakDistance)
            {
                lockPos = grounded;
            }

            animator.SetIKPosition(goal, Vector3.Lerp(grounded, lockPos, lockWeight));
        }
        else
        {
            locked = false;
            animator.SetIKPosition(goal, grounded);
        }

        animator.SetIKPositionWeight(goal, ikWeight);
        animator.SetIKRotationWeight(goal, ikWeight * 0.8f);
        animator.SetIKRotation(goal, Quaternion.FromToRotation(
            Vector3.up, hit.normal) * animator.GetIKRotation(goal));
    }
}
```

---

# 40. Suggested Animator Parameters

Use a small set of parameters.

```text
Speed                 float    forward-relative speed, m/s
PlaybackRate          float    stride synchronization multiplier
MoveDirection         float    for directional blends, if used
TurnRate              float    applied yaw rate, deg/s, signed
SlipAngleNormalized   float    -1 .. +1, slide direction and magnitude
SlideAmount           float    0 .. 1, how committed the slide is
IsSliding             bool
IsBrakeSlide          bool
Grounded              bool
VerticalSpeed         float
IsSprinting           bool
IsTurning             bool
```

Optional:

```text
Acceleration          float
SlopeAngle            float
Stamina01             float
```

Avoid creating dozens of parameters unless there is a real need. Note that
`SlideAmount` and `SlipAngleNormalized` between them replace what would otherwise be a
dozen slide-specific booleans.

---

# 41. Recommended Animator Structure

A practical structure:

```text
Base Layer
|
+-- Locomotion (blend tree, Speed)
|     |
|     +-- Idle
|     +-- Walk
|     +-- Run
|     +-- Sprint
|
+-- Start
+-- Stop
+-- TurnInPlace
+-- Airborne
+-- Landing

Skid Layer        (additive / lower-body+torso mask, weight = SlideAmount)
|
+-- Skid blend (SlipAngleNormalized)
      +-- SkidLeft
      +-- Neutral
      +-- SkidRight

Brake Layer       (additive, weight = IsBrakeSlide)
|
+-- BrakeSlide

Action Layer      (upper-body mask)
|
+-- Eat
+-- Drink
+-- Attack
+-- Bite
+-- Sleep
```

Locomotion should be separated from actions.

For example:

```text
Locomotion
+
Attack Overlay
```

rather than creating:

```text
RunAttack
WalkAttack
SprintAttack
RunBite
WalkBite
...
```

for every combination.

The same argument applies to sliding: do **not** author `SprintSkidLeftBite`. The skid is
a layer, and it composes.

---

# 42. AI Must Use the Same Locomotion System

Do not create a separate animation system for AI.

Player:

```text
Player Input
    ↓
DesiredDirection + DesiredSpeed
    ↓
DinoMovement
    ↓
Actual Velocity
    ↓
Animator
```

AI:

```text
NavMesh / AI Steering
    ↓
DesiredDirection + DesiredSpeed
    ↓
DinoMovement
    ↓
Actual Velocity
    ↓
Animator
```

Both write the **same two fields** on the same component. Everything downstream — grip,
slip angle, sliding, stride matching, foot IK — is then identical by construction.

```csharp
// AI, per tick
Vector3 toCorner = (nextPathCorner - transform.position);
toCorner.y = 0f;
move.DesiredDirection = toCorner.normalized;
move.DesiredSpeed     = chasing ? cfg.sprintSpeed : cfg.runSpeed;
move.BrakeRequested   = distanceToTarget < brakingDistance;
```

This is important for DinoBorn because prey, predators, and player dinosaurs should all
visually obey the same locomotion rules — **and because AI gets momentum sliding for
free.** A pursuing raptor that overshoots a juking prey animal is emergent behaviour from
this one design decision, not something anyone has to script.

One caution: the NavMesh agent must be used for **pathfinding only**. Set
`agent.updatePosition = false` and `agent.updateRotation = false`, and feed the corner
direction into `DinoMovement`. If the agent also moves the transform, it will fight the
grip model and cancel the slide.

---
---

# PART VIII — TUNING, TESTING AND DELIVERY

---

# 43. Example — Raptor

Example tuning:

```text
Raptor

Walk:               1.8 m/s
Run:                4.0 m/s
Sprint:             7.0 m/s
Burst:              9.0 m/s (3 s, heavy stamina cost)

Acceleration:       7.0 m/s²
Deceleration:       9.0 m/s²
Brake deceleration: 14.0 m/s²

Turn rate at rest:   200 °/s
Turn rate at sprint:  85 °/s

Lateral grip:       12.0 m/s²
Brake grip mult:    0.45
Burst grip mult:    0.60
Slide enter:        12°
Slide full:         45°
Slide exit:          6°
Slide min speed:    3.5 m/s

Natural speeds:     Walk 1.8 / Run 4.0 / Sprint 6.5
Playback clamp:     0.85 – 1.30
Slide playback:     0.35
```

These are starting values, not final gameplay values.

Example, running clean:

```text
Actual speed = 6.0 m/s
Slip angle   = 2°

Animator:
    Speed = 6.0 (forward component)
Blend:
    mostly Sprint
Playback:
    ~0.95, stride synchronized
Foot lock:
    ON
Foot IK:
    full weight, corrects terrain contact
```

Example, cornering hard at speed:

```text
Actual speed = 6.4 m/s
Turn input   = hard left at 85°/s
Demanded lateral = 6.4 × 1.48 = 9.5 m/s²  -> under grip, holds
Turn input   = hard left while bursting (grip 12 × 0.6 = 7.2)
Demanded lateral = 9.5 m/s²               -> EXCEEDS grip

Slip angle   = 31°
SlideAmount  = 0.58

Animator:
    Speed = 5.5 (forward component only)
Blend:
    Run/Sprint
Playback:
    blended down toward 0.35 — legs bracing
Skid layer:
    weight 0.58, leaning left, tail out
Foot lock:
    OFF
Foot IK:
    reduced to ~0.62 weight
FX:
    dust from both feet, skid decals, scrape audio
```

Result:

```text
Short-legged raptor
+
high step frequency
+
fast world movement
+
stable foot contacts when running true
+
a visible, controllable skid when it over-commits
=
fast, believable, and readable movement
```

---

# 44. Example — Troodon

Troodon can use a similar system but with different proportions and tuning.

```text
Troodon

Walk:               1.5 m/s
Run:                3.5 m/s
Sprint:             6.0 m/s

Turn rate at rest:   240 °/s     (more agile)
Turn rate at sprint: 120 °/s

Lateral grip:       16.0 m/s²    (lighter, grips better)
Slide enter:        15°
Slide full:         50°
```

Because its legs are shorter, avoid simply making its stride enormous.

Prefer:

```text
Short stride
+
higher cadence
+
appropriate playback
+
foot synchronization
```

This is the visual trick that makes a small dinosaur feel fast.

## Grip as characterization

Compare across the roster — this one column does a lot of work:

```text
Dinosaur          Sprint     Grip      Feel

Troodon           6.0 m/s    16 m/s²   darting, agile, hard to catch
Raptor            7.0 m/s    12 m/s²   fast, commits to turns, drifts
Carnotaurus       9.0 m/s     8 m/s²   very fast, terrible cornering
Tyrannosaurus     7.0 m/s     6 m/s²   heavy, enormous turning circle
Triceratops       6.5 m/s     9 m/s²   ploughs, hard to redirect
```

A T. rex is not scary because its number is bigger. It is scary because it cannot stop,
and both the player and the T. rex know it.

---

# 45. What NOT To Do

## Do not do this

```csharp
transform.position += animationRootMotion;
```

as the primary gameplay system. It is also fundamentally incompatible with momentum
sliding (Section 6).

## Do not do this

```csharp
animator.speed = 3.0f;
```

for all locomotion. Animation speed should depend on actual movement and the current
locomotion animation — and `animator.speed` scales every layer including attacks.

## Do not do this

```text
Speed = input magnitude
```

when actual movement velocity is available.

## Do not solve sliding only with IK

If:

```text
Gameplay = 7 m/s
Animation = 2 m/s
```

IK cannot magically make the whole locomotion correct.

## Do not store velocity as a scalar

```csharp
float speed;                       // <- kills momentum sliding dead
transform.position += transform.forward * speed * dt;
```

If velocity only exists along the forward axis, rotating the body rotates the momentum
instantly, and no amount of animation work will make the dinosaur feel like it has mass.
Velocity must be a **world-space vector**.

## Do not apply grip before turning

```csharp
ApplyGrip(dt);     // WRONG ORDER
ApplyTurn(dt);
```

The lateral component is created *by* the rotation. Cancel it first and there is nothing
left to slide.

## Do not leave foot locking on during a slide

The leg will stretch toward a world position the body has already left. See Section 24.

## Do not make sliding a button

It is a consequence of speed, turn input and grip, not an ability.

## Do not make sliding free

No forward-drive penalty means players drift permanently and the mechanic collapses into
a faster way to travel. See Section 26.

## Do not widen the playback clamp to hide a missing clip

If `PlaybackRate` sits pinned at 1.30, add a faster animation. Stretching one clip to 2x
looks like the video is fast-forwarding.

---

# 46. Debug Mode

Create a locomotion debug mode.

Display:

```text
Actual Speed:       5.83 m/s
Forward Speed:      5.41 m/s
Lateral Speed:      2.18 m/s
Desired Speed:      6.00 m/s

Animation:          Sprint (blend 0.62)
Natural Speed:      5.50 m/s
Playback Rate:      1.06

Slip Angle:         21.9°
Slide Amount:       0.42
Sliding:            TRUE
Brake Slide:        FALSE

Grip (effective):   12.0 m/s²
Demanded Lateral:   14.8 m/s²      <- exceeds grip, hence the slide
Surface Grip:       1.00 (dirt)

Grounded:           TRUE
Turn Rate:          78°/s
Slope:              7°
Stamina:            0.63
```

The `Demanded Lateral` vs `Grip (effective)` pair is the single most useful readout in the
whole overlay — it tells you immediately *why* the dinosaur is or is not sliding.

Also draw in the scene:

```text
Velocity vector           (yellow)
Forward vector            (blue)
Lateral component         (red)
The slip angle arc between velocity and forward
Foot raycasts
Foot contact points
Locked foot positions     (green when locked, grey when released)
```

Drawing the velocity and forward vectors as two separate lines makes the entire mechanic
visible at a glance, and makes tuning grip dramatically faster.

---

# 47. How to Test for Foot Sliding

Use a flat test surface first. Do not start on complicated terrain.

**Test with sliding disabled.** Temporarily set `lateralGrip` to a very high value (200)
so no momentum slide can occur. Any foot sliding you see now is a genuine bug.

Test:

```text
1. Walk forward
2. Run forward
3. Sprint forward
4. Accelerate
5. Decelerate
6. Turn gently
7. Turn sharply
8. Stop
9. Start again
```

Watch the planted foot.

If the foot moves across the ground while it should be planted:

```text
Check SlideAmount (should be 0 in this test)
        ↓
Check actual speed vs the speed sent to the animator
        ↓
Check animation natural speed measurements
        ↓
Check blend thresholds match natural speeds
        ↓
Check playback rate (is it pinned at the clamp?)
        ↓
Check root motion is disabled
        ↓
Check foot locking / IK
```

Then restore `lateralGrip` to its real value and move to Section 48.

---

# 48. How to Test Intentional Sliding

```text
 1. Sprint in a straight line          -> slip angle ~0, NO slide
 2. Sprint, gentle turn                -> slip angle small, NO slide
 3. Sprint, hard turn                  -> slide begins, dust, skid pose
 4. Hold the turn                      -> slide sustains, arcs wide
 5. Counter-steer into the slide       -> slide shortens
 6. Steer further out of it            -> slide extends
 7. Release throttle mid-slide         -> recovers smoothly
 8. Brake mid-slide                    -> speed drops, slide deepens briefly
 9. Hard brake, straight               -> straight-line brake skid
10. Walk, hard turn                    -> NO slide (below min speed)
11. Sprint onto mud                    -> slide with less provocation
12. Land from a jump at sprint         -> brief scrabble
13. Sprint down a steep slope          -> slope slide
14. Slide into a wall                  -> velocity resolves, no jitter, no stretch
15. AI chase around an obstacle        -> AI overshoots and recovers
16. Watch a remote client mid-slide    -> matches the owner's skid
```

For each, check the four things that must stay consistent:

```text
[ ] The skid direction matches the sign of the slip angle
[ ] Foot locking released as the slide developed
[ ] Leg cadence slowed rather than sprinting sideways
[ ] Recovery blended out rather than snapping
```

## Common symptoms and causes

| Symptom | Likely cause |
|---|---|
| Never slides | Grip too high; turn rate not falling with speed |
| Always slides | Grip too low; slide enter angle too small |
| Slide flickers on/off | Missing hysteresis; noisy velocity; no smoothing |
| Legs sprint sideways | Matching against total speed, not forward speed |
| Leg stretches during slide | Foot locking still active (Section 24) |
| Slide feels uncontrollable | `slideTurnMultiplier` too low |
| Drifting is faster than driving | `slideDriveMultiplier` too high or missing |
| Slides while walking | `slideMinSpeed` too low |
| Skid pose pops in | `slideAttack` too high; no damping on layer weight |

---

# 49. Testing Sequence for DinoBorn

Use one dinosaur first.

Recommended:

```text
Raptor
```

Do not implement the entire dinosaur roster immediately.

Build:

```text
Raptor
    ↓
Locomotion
    ↓
Foot synchronization
    ↓
Momentum + grip
    ↓
Slide presentation
    ↓
Terrain IK
    ↓
AI locomotion
```

Once it looks good:

```text
Reuse system
    ↓
Troodon
    ↓
Other theropods
    ↓
Other dinosaur types
```

The movement system should be data-driven so each dinosaur supplies its own parameters.

---

# 50. Recommended Implementation Stages

## Stage 1 — Basic movement

```text
Player Input
    ↓
Movement Controller
    ↓
Actual Velocity
```

Verify that the dinosaur can walk, run, sprint, stop and turn — **without animation**.

## Stage 2 — Animator speed

```text
Actual Velocity
    ↓
Animator Speed parameter
```

Create the Idle / Walk / Run / Sprint blend tree.

## Stage 3 — Animation synchronization

Measure the natural speed of each animation. Add blended natural speed and playback
correction (Section 12). Tune the playback limits.

**Checkpoint:** no foot sliding at any constant speed, on flat ground.

## Stage 4 — Acceleration and braking

Acceleration, deceleration, brake, start, stop, sprint transitions.

## Stage 5 — Turning

Turn rate falling with speed, turn speed penalty, turn animations, turn in place.

## Stage 6 — Momentum and grip

Convert velocity to a world vector if it is not one already. Add the grip bleed, slip
angle, and the slide state machine. **No animation changes yet** — verify the slide with
the debug vectors only.

**Checkpoint:** the dinosaur physically drifts through fast corners, and you can see it
in the debug overlay.

## Stage 7 — Slide presentation

Skid layer, playback blending, foot-lock release, IK weight reduction, dust, audio,
camera.

**Checkpoint:** the drift is legible without the debug overlay.

## Stage 8 — Foot IK

Ground raycasts, foot position correction, foot rotation correction, foot locking, with
the slide-aware weights from Section 24.

## Stage 9 — Surfaces and slopes

Surface grip lookup, slope sliding, landing grip recovery.

## Stage 10 — AI

Feed AI through the same `DesiredDirection` / `DesiredSpeed` interface.

## Stage 11 — Networking

Replicate velocity; derive the slide locally on proxies.

## Stage 12 — Advanced polish

Only after all of the above works:

- Stride warping
- Orientation warping
- Procedural foot placement
- Slope-aware posture
- Additional locomotion states
- Per-surface FX and audio variety

---

# 51. The Relationship Between Speed and Animation

Think of locomotion as a synchronization problem.

```text
WORLD
------------------------------------>

Actual dinosaur:
        🦖------------------------->

ANIMATION
------------------------------------>

Foot:
        L    R    L    R    L
        |    |    |    |    |

These two timelines need to stay synchronized.
```

When synchronization is correct:

```text
Foot contact
      ≈
Ground movement
```

When synchronization is wrong:

```text
Foot contact
      ≠
Ground movement
```

and the player sees sliding.

## The revised version

That is true only while the dinosaur is running true. The complete statement is:

```text
Foot contact should match the FORWARD component of ground movement.

The remainder — the lateral component — is momentum,
and it SHOULD appear as a skid.
```

```text
Total ground movement
        =
Forward component     -> matched by stride  -> looks like running
        +
Lateral component     -> not matched        -> looks like sliding
```

Foot sliding is the forward component going unmatched.
Momentum sliding is the lateral component being honest.

---

# 52. The Evrima-Like Target

The desired visual result is:

```text
                SPEED
                  ↑
                  |
                  |      Sprint
                  |     /
                  |    /
                  |   Run
                  |  /
                  | Walk
                  | /
                  |/
                  +---------------->

Animation cadence increases
as movement speed increases.
```

But the dinosaur's proportions remain believable.

The goal is NOT:

```text
Make legs huge
```

The goal is:

```text
Increase cadence
+
synchronize stride
+
control actual movement
+
keep planted feet stable WHEN RUNNING TRUE
+
let momentum win WHEN IT SHOULD
```

And, for the second half of the target:

```text
            TURN AUTHORITY
                  ↑
                  |‾‾‾╲
                  |    ╲
                  |     ╲___
                  |         ‾‾‾╲___
                  +----------------> SPEED

The faster the dinosaur moves,
the less it can redirect itself,
and the more it slides when it tries.
```

Those two graphs together are the whole feel.

---

# 53. Recommended Final DinoBorn Architecture

```text
                    PLAYER
                      |
                      v
                Player Input
                      |
                      v
              Movement Controller
                      |
AI -------------------+
                      |
                      v
        DesiredDirection + DesiredSpeed
                      |
                      v
              Turn (rate limited by speed)
                      |
                      v
        MOMENTUM + GRIP SOLVER
        (velocity as a world vector,
         lateral bleed limited by grip)
                      |
                      v
                Actual Velocity
                      |
          +-----------+-----------+
          |                       |
          v                       v
      Character            Velocity vs Facing
      Position                    |
          |                       v
          |               Slip Angle / SlideAmount
          |                       |
          |          +------------+------------+
          |          |            |            |
          |          v            v            v
          |     Animator      Foot IK       FX / Audio
          |          |         weights       / Camera
          |          v            |
          |   Locomotion Blend    |
          |   (forward speed)     |
          |          |            |
          |          v            |
          |   Playback Sync       |
          |          |            |
          |          v            |
          |    Skid Layer         |
          |          |            |
          +----------+------------+
                     |
                     v
               Visible Dinosaur
```

This architecture should be the foundation for DinoBorn locomotion.

Note that `Slip Angle / SlideAmount` is a **derived value with three consumers** and no
authority of its own. Nothing writes to it; the animator, the IK and the FX all read it.
That is what keeps the feature coherent instead of becoming three loosely related systems
that drift out of agreement.

---

# 54. Final Recommendation

For DinoBorn, implement the system in this order:

```text
 1. Gameplay-authoritative movement
 2. Actual velocity extraction (world-space VECTOR, not a scalar)
 3. Speed-based Blend Tree
 4. Animation natural-speed measurement
 5. Blended natural speed + playback synchronization
 6. Acceleration / deceleration / braking
 7. Turning, with turn rate falling as speed rises
 8. Start / stop animations
 9. Momentum + grip solver, slip angle, slide state
10. Slide presentation (skid layer, playback blend, FX)
11. Foot IK
12. Foot locking, released while sliding
13. Surfaces, slopes, landing recovery
14. AI through the same interface
15. Networking (replicate velocity, derive the slide)
16. Stride warping only if still needed
```

The two critical insights are:

> **Fast dinosaurs do not need giant or exaggerated leg animations. They need the
> relationship between world velocity, stride cadence, and foot contact to remain
> synchronized.**

> **Sliding is not the failure of that synchronization — it is what happens to the part
> of the velocity the legs were never producing. Keep velocity as a world vector, limit
> how fast grip can rotate it toward the facing direction, and both the speed and the
> slide fall out of the same three lines of code.**

That is the system DinoBorn should build around.

---

# 55. Implementation Acceptance Criteria

## Speed and synchronization

- [ ] A Raptor can sprint at high speed without obvious foot sliding.
- [ ] A Troodon can move quickly despite its short legs.
- [ ] Gameplay speed is independent from root-motion animation.
- [ ] Animator speed is driven by actual, measured velocity.
- [ ] The animator is driven by the **forward component** of velocity, not total speed.
- [ ] Blend thresholds match the clips' measured natural speeds.
- [ ] Playback rate stays within 0.85–1.30 during normal play.
- [ ] Walk / run / sprint blend smoothly.
- [ ] Acceleration looks natural.
- [ ] Deceleration looks natural.
- [ ] Stopping does not produce skating.
- [ ] Starting from idle does not instantly teleport into full-speed locomotion.
- [ ] Raising max speed does not require new code, only config and possibly one clip.

## Momentum and sliding

- [ ] Velocity is stored as a world-space vector, not a scalar along forward.
- [ ] Turn rate falls measurably as speed rises.
- [ ] Sharp turns at sprint produce a visible slide; gentle turns do not.
- [ ] Turns at walking pace never produce a slide.
- [ ] The slide direction always matches the sign of the slip angle.
- [ ] The slide can be steered through and shortened by counter-steering.
- [ ] Sliding costs forward acceleration, so drifting is not faster than driving.
- [ ] The slide state does not flicker at its threshold.
- [ ] Low-grip surfaces (mud, snow) visibly change handling.
- [ ] Heavy dinosaurs slide longer and wider than light ones, from config alone.
- [ ] Braking at speed produces a straight-line skid.
- [ ] Steep slopes produce a downhill slide.
- [ ] Landing at sprint produces a brief loss of grip.

## Presentation

- [ ] The skid pose leans into the turn and the tail counterbalances.
- [ ] Leg cadence slows during a slide instead of sprinting sideways.
- [ ] Foot locking is released while sliding, and the legs never stretch or pop.
- [ ] Foot IK weight reduces during a slide but feet stay on the ground surface.
- [ ] Dust, decals and scrape audio scale with slide amount and surface type.
- [ ] The camera biases toward the velocity direction during a slide.
- [ ] Recovery from a slide blends out rather than snapping.

## Systems

- [ ] AI and player movement use the same locomotion component and interface.
- [ ] AI overshoots when it over-commits to a turn, with no special-case code.
- [ ] The NavMesh agent does not move the transform.
- [ ] Uneven terrain does not cause obvious floating feet.
- [ ] Foot IK does not visibly distort the legs.
- [ ] Locomotion parameters are configured per dinosaur in a single asset.
- [ ] Remote clients reproduce the slide from replicated velocity alone.
- [ ] A debug mode exposes actual speed, forward/lateral speed, slip angle, slide amount,
      effective grip, demanded lateral acceleration, playback rate and foot contact.
- [ ] The debug overlay draws the velocity and forward vectors separately.
- [ ] The system works without requiring a unique animation for every possible movement
      speed or slide angle.

---

# 56. Changelog

## Revision 2 — added momentum sliding and speed guidance

**Nothing from the original document was removed.** Every original section is still
present; several were extended, and the numbering was reorganised into eight parts.

### New sections

| Section | Content |
|---|---|
| 2 | Two Kinds of Sliding — the foot-sliding vs momentum-sliding distinction |
| 7 | What Actually Breaks When You Raise Max Speed |
| 8 | Speed Tiers, Sprint and Stamina |
| 9 | Selling Speed Without Increasing It |
| 12 | Blended Natural Speed — the corrected playback formula |
| 17–27 | Part IV in full: momentum, grip model, slip angle, slide state machine, triggers, steering, slide animation, slide-aware foot IK, feedback, balance, networking |
| 36–39 | Complete implementation: config asset, movement solver, animation driver, foot IK |
| 48 | How to Test Intentional Sliding, with a symptom/cause table |

### Extended sections

| Section | Change |
|---|---|
| Purpose / Title | Now covers intentional sliding, not only its absence |
| 1 | Added the facing-vs-velocity separation principle |
| 5 | Architecture now includes the momentum/grip solver |
| 6 | Added why root motion is incompatible with momentum sliding |
| 10 | Added skid clip authoring guidance |
| 11 | Added a reliable measurement procedure |
| 13 | Thresholds should sit at measured natural speeds |
| 14 | Feed forward speed; measure velocity after collision resolution |
| 16 | Added orientation warping and its limits |
| 28–31 | Drive applies to the forward component; turn rate falls with speed; turn lean vs skid lean separated |
| 35 | Foot locking is now conditional on slide amount |
| 40–41 | Added slide parameters and the skid/brake layers |
| 42 | AI inherits sliding for free; NavMesh agent caution |
| 43–44 | Added grip values and a cross-roster grip comparison |
| 45 | Added six new failure modes, all slide-related |
| 46 | Debug overlay now exposes grip vs demanded lateral acceleration |
| 47 | Test foot sliding with grip pinned high to isolate real bugs |
| 50 | Stages expanded from 7 to 12, with checkpoints |
| 51–54 | Revised to state the forward/lateral split explicitly |
| 55 | Acceptance criteria grouped and expanded from 17 to 48 items |

### The one-paragraph summary of the change

The original document treated all foot movement across the ground as a defect. That is
correct only while the velocity vector is aligned with the facing vector. Once velocity is
stored in world space and grip is allowed to be finite, the two vectors can disagree, and
the resulting skid is the Evrima-style slide. The slip angle between them is the single
value that decides whether a given frame should be stride-matched and foot-locked, or
braced and sliding.

---
