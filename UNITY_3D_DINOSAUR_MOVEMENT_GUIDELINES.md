# Unity 6.6 Dinosaur Movement Guidelines

Use these guidelines for smooth third-person movement in a 3D dinosaur survival game. The controls are:

- `WASD`: Move relative to the camera
- Mouse: Orbit the camera and aim the movement direction
- `Left Shift`: Sprint
- `Escape`: Release the mouse cursor
- Left mouse click: Lock the cursor again

## AI Agent Implementation Contract

This document is an implementation specification, not a list of optional ideas.
An AI agent implementing the movement system must:

1. Inspect the existing Unity project and reuse its assembly definitions, namespaces,
   and folder conventions. Replace an existing player movement implementation rather
   than running two implementations at once.
2. Implement all behavior marked as required in this document.
3. Keep input, locomotion, and camera responsibilities separate.
4. Provide working Inspector defaults and validate required serialized references.
5. Avoid placeholder methods, pseudocode in production scripts, unresolved TODOs, and
   missing Input Action references.
6. Preserve unrelated project behavior and adapt existing player or camera code
   rather than creating a second competing movement system.
7. Compile the project and run the smallest relevant Edit Mode or Play Mode tests.
8. Test the controls in Play Mode and report anything that cannot be verified
   automatically.

Required deliverables:

- `DinosaurControls.inputactions`, containing the `Player` action map defined below.
- `DinosaurInputReader.cs`, the sole owner of Input System callbacks and input state.
- `DinosaurMovementController.cs`, the sole owner of player translation, gravity,
  sprint selection, and player-facing rotation.
- `ThirdPersonOrbitCamera.cs`, the sole owner of camera yaw, pitch, position,
  smoothing, and collision.
- Scene configuration for `DinosaurPlayer`, `CameraTarget`, and `Main Camera`.
- Edit Mode tests for camera-relative direction and diagonal-speed clamping.

Use the exact file and class names above. Existing equivalent components must be
adapted into these components or replaced; they must not remain enabled alongside
the new implementation.

The finished feature must be usable after assigning clearly identified scene
references. It must not require an undocumented code change to begin working.

## Non-negotiable Control Behavior

Mouse look and WASD movement are independent inputs and must work simultaneously.
Neither input is allowed to consume, disable, replace, or reset the other.

Required behavior:

| Input combination | Result |
|---|---|
| Mouse only | Orbit the camera; do not move the dinosaur |
| `W` | Move toward the camera's flat forward direction |
| `S` | Move opposite the camera's flat forward direction |
| `A` | Move toward the camera's flat left direction |
| `D` | Move toward the camera's flat right direction |
| `W+A`, `W+D`, `S+A`, or `S+D` | Move diagonally at the same maximum speed as a single direction |
| Mouse + `A` | Orbit the camera while continuously moving camera-relative left |
| Mouse + `D` | Orbit the camera while continuously moving camera-relative right |
| Mouse + any WASD combination | Orbit and move at the same time using the latest camera orientation |

`A` and `D` do **not** rotate the camera. They produce lateral, camera-relative
movement. Mouse X rotates the camera orbit. These controls do not conflict.

As the mouse changes camera yaw while `A` or `D` is held, the world-space movement
direction must update continuously so it remains left or right relative to the
camera. Do not cache a world-space movement direction when the key is first pressed.

This specification uses free camera-relative locomotion: the dinosaur smoothly
faces its final movement direction. It is not a tank-control scheme, and `A`/`D`
must not merely rotate the dinosaur in place.

## Unity 6.6 Compatibility

Use APIs supported by Unity 6.6:

- Use the **Input System** package, not the legacy `Input` class.
- Use a `CharacterController` for responsive player movement.
- Read input and call `CharacterController.Move` from `Update`.
- Use `Time.deltaTime` for frame-rate-independent movement.
- Do not multiply motion by `Time.deltaTime` more than once.
- Do not directly modify the player transform's position while a
  `CharacterController` is responsible for movement.
- Use `LateUpdate` for the camera after the dinosaur has moved.

Install the Input System from:

`Window > Package Manager > Unity Registry > Input System`

Then set:

`Edit > Project Settings > Player > Active Input Handling > Input System Package (New)`

Restart the Unity Editor if prompted.

Do not hardcode a package version in these guidelines. Use the Input System version
officially supported by the installed Unity 6.6 editor release.

## Required Hierarchy

```text
DinosaurPlayer
├── Model
└── CameraTarget

Main Camera
```

### DinosaurPlayer

Add:

- `CharacterController`
- `PlayerInput`
- `DinosaurInputReader`
- `DinosaurMovementController`

The root object owns movement and horizontal rotation. Keep the model as a child so
its visual rotation or animation can be adjusted without changing collision.

### CameraTarget

Place this child at the visual center of the dinosaur's upper torso. The
third-person camera orbits this target instead of the dinosaur's feet. Its local
rotation must be identity and its local scale must be `(1, 1, 1)`.

### Main Camera

Attach `ThirdPersonOrbitCamera` to `Main Camera`. This specification uses the custom
camera defined below and does not use Cinemachine. Disable or remove Cinemachine
Brain, virtual cameras, and every other component that writes the gameplay camera's
transform while this controller is active.

Assign `CameraTarget` to the camera controller and assign the camera controller to
the movement controller. Missing required references must log an error naming the
missing field and disable the affected component in `Awake`.

## Input Actions

Create an Input Actions asset named `DinosaurControls`.

Add an action map named `Player` with these actions:

| Action | Action Type | Control Type | Binding |
|---|---|---|---|
| `Move` | Value | Vector2 | 2D Vector composite: WASD |
| `Look` | Value | Vector2 | Mouse Delta |
| `Sprint` | Button | Button | Left Shift |
| `ReleaseCursor` | Button | Button | Escape |
| `LockCursor` | Button | Button | Left Mouse Button |

For `Move`, configure the 2D Vector composite:

- Up: `W`
- Down: `S`
- Left: `A`
- Right: `D`

Configure `PlayerInput` as follows:

- Actions: `DinosaurControls`
- Default Map: `Player`
- Behavior: `Invoke Unity Events`

Wire the action events to `DinosaurInputReader`. The reader stores:

- `Move`: the latest `Vector2`; set it to `Vector2.zero` on cancellation.
- `Look`: accumulated mouse delta for the current frame; clear it only after the
  camera consumes it.
- `Sprint`: `true` while pressed and `false` when released or cancelled.
- `ReleaseCursor`: react only to `performed` and only while controls are enabled.
- `LockCursor`: react only to `performed`, while the application has focus, and
  while no menu, death, cutscene, or interaction block is active.

Do not generate a second input polling path. Do not call legacy `Input` APIs. Keep
`Move` and `Look` enabled together throughout active gameplay.

Set:

`Edit > Project Settings > Input System Package > Update Mode > Process Events In Dynamic Update`

Do not use `Process Events In Fixed Update` for this controller. Do not add a
processor that scales `Look` by frame time. These settings ensure the Input System
delivers mouse delta before `Update`, and the implementation applies each physical
mouse delta exactly once without multiplying by `Time.deltaTime`.

## Movement Rules

### Camera-relative direction

Convert the two-dimensional input into a world direction from
`ThirdPersonOrbitCamera.CurrentYaw` every frame:

```csharp
Quaternion yawRotation =
    Quaternion.Euler(0f, cameraController.CurrentYaw, 0f);
Vector3 cameraForward = yawRotation * Vector3.forward;
Vector3 cameraRight = yawRotation * Vector3.right;
Vector3 desiredDirection =
    cameraForward * moveInput.y +
    cameraRight * moveInput.x;

desiredDirection = Vector3.ClampMagnitude(desiredDirection, 1f);
```

This is the only direction calculation. Do not also derive direction from the
camera transform. Clamping prevents diagonal input from moving faster than straight
input.

The input reader must preserve both `Move` and accumulated `Look` values until
`DinosaurMovementController.Update` consumes them. The camera yaw must be updated
before locomotion calculates its camera-relative direction.
`DinosaurMovementController.Update` must execute this exact sequence:

1. Read the stored input state.
2. Call `ThirdPersonOrbitCamera.AdvanceOrbit(lookDelta, Time.deltaTime)`.
3. Clear the consumed look delta.
4. Read `ThirdPersonOrbitCamera.CurrentYaw`.
5. Calculate desired movement direction.
6. Update horizontal and vertical velocity.
7. Rotate the dinosaur.
8. Call `CharacterController.Move` exactly once.

`ThirdPersonOrbitCamera` must not implement `Update`; its `LateUpdate` performs only
camera position, camera rotation, and collision. This explicit call sequence removes
MonoBehaviour execution-order ambiguity.

### Smooth acceleration

Do not instantly switch between zero and maximum speed. Maintain a current
horizontal velocity and move it toward the target velocity:

```csharp
Vector3 targetVelocity = desiredDirection * targetSpeed;
float rate =
    desiredDirection.sqrMagnitude <= movementInputDeadZone * movementInputDeadZone ||
    targetVelocity.sqrMagnitude <= horizontalVelocity.sqrMagnitude
        ? deceleration
        : acceleration;

horizontalVelocity = Vector3.MoveTowards(
    horizontalVelocity,
    targetVelocity,
    rate * Time.deltaTime);
```

Use separate acceleration and deceleration values with these serialized defaults:

| Setting | Default value |
|---|---:|
| Walk speed | `3.5 m/s` |
| Run speed | `6.5 m/s` |
| Acceleration | `14 m/s²` |
| Deceleration | `18 m/s²` |
| Rotation speed | `540 degrees/s` |
| Gravity | `-25 m/s²` |
| Grounded vertical velocity | `-2 m/s` |
| Movement input dead zone | `0.01` |

Designers may tune serialized values after implementation. The agent must implement
and validate the defaults above.

The `540 degrees/s` rotation default is intentionally responsive: a 180-degree turn
takes approximately `0.33 s`. Do not silently lower it based on the dinosaur's
visual size. During Play Mode validation, report visible foot sliding or animation
turning that cannot keep up with this rate; animation tuning requires designer
approval and is outside this movement task.

### Smooth facing rotation

When movement input is present, turn the dinosaur toward the desired movement
direction:

```csharp
if (desiredDirection.sqrMagnitude >
    movementInputDeadZone * movementInputDeadZone)
{
    Quaternion targetRotation =
        Quaternion.LookRotation(desiredDirection, Vector3.up);

    transform.rotation = Quaternion.RotateTowards(
        transform.rotation,
        targetRotation,
        rotationSpeed * Time.deltaTime);
}
```

Do not rotate the dinosaur from raw mouse delta.
The mouse rotates the camera; movement rotates the dinosaur toward the
camera-relative movement direction.

Aim-mode and combat-specific facing behavior are outside this movement task and
must not be added.

### Gravity and grounding

`CharacterController.Move` does not apply gravity automatically. Track vertical
velocity separately:

```csharp
if (controller.isGrounded && verticalVelocity < 0f)
{
    verticalVelocity = -2f;
}
else
{
    verticalVelocity += gravity * Time.deltaTime;
}

Vector3 motion = horizontalVelocity;
motion.y = verticalVelocity;
controller.Move(motion * Time.deltaTime);
```

The small negative grounded velocity keeps the controller in contact with slopes.

Use `CharacterController.isGrounded` as the only grounded source for this
implementation. Jumping, falling damage, and ledge detection are outside scope.

### Sprinting

Sprint only while:

- Sprint is held.
- Movement input is present.
- Controls are enabled.

Smoothly accelerate to the run speed. Do not snap directly from walk speed to run
speed. Stamina consumption is outside scope; a future stamina system can gate the
stored sprint state without changing locomotion.

## Mouse Camera Rules

The required camera is a third-person orbit camera centered on `CameraTarget`.
Yaw orbits horizontally around the target, pitch orbits vertically within limits,
and camera collision shortens the viewing distance when an obstacle intervenes.
Mouse movement must never directly translate the dinosaur.

### Look input

Mouse delta already represents movement accumulated by the Input System. Apply a
sensitivity multiplier, but do not multiply mouse delta by `Time.deltaTime`.

```csharp
targetYaw += lookInput.x * mouseSensitivity;
targetPitch -= lookInput.y * mouseSensitivity;
targetPitch = Mathf.Clamp(targetPitch, minPitch, maxPitch);
```

Use these serialized default values:

| Setting | Default value |
|---|---:|
| Mouse sensitivity | `0.08` |
| Minimum pitch | `-30 degrees` |
| Maximum pitch | `65 degrees` |
| Camera distance | `6 m` |
| Rotation smooth time | `0.07 s` |
| Distance return smooth time | `0.12 s` |
| Collision sphere radius | `0.25 m` |
| Collision safety offset | `0.05 m` |

Keep target and displayed angles separate:

- `targetYaw` and `targetPitch` receive mouse input.
- `currentYaw` and `currentPitch` smoothly approach the targets.
- Movement uses the yaw-only orientation from `currentYaw` after it has been updated
  for the current frame.
- Camera rendering uses the smoothed current angles.

Update the smoothed angles before calculating movement, then apply the camera
transform in `LateUpdate`. This keeps mouse plus `A` or `D` responsive while
ensuring movement uses the same `currentYaw` that the camera renders during that
frame. Movement must never use the invisible unsmoothed target yaw.

### Camera smoothing

Apply mouse input to target yaw and pitch, then use `Mathf.SmoothDampAngle` with the
`0.07 s` default to move current yaw and pitch toward those targets. Do not apply
another layer of rotational smoothing.

### Required custom camera algorithm

The custom camera must perform these steps:

1. Read the accumulated look input and update target yaw and pitch.
2. Clamp target pitch.
3. Smooth current yaw and pitch toward their targets.
4. Build an orbit rotation from current pitch and yaw.
5. Calculate the desired position behind `CameraTarget`.
6. Resolve camera collision between `CameraTarget` and the desired position.
7. Apply the resolved position and orbit rotation in `LateUpdate`.

Reference calculation:

```csharp
currentYaw = Mathf.SmoothDampAngle(
    currentYaw,
    targetYaw,
    ref yawVelocity,
    rotationSmoothTime,
    Mathf.Infinity,
    deltaTime);

currentPitch = Mathf.SmoothDampAngle(
    currentPitch,
    targetPitch,
    ref pitchVelocity,
    rotationSmoothTime,
    Mathf.Infinity,
    deltaTime);

Quaternion orbitRotation =
    Quaternion.Euler(currentPitch, currentYaw, 0f);

Vector3 desiredPosition =
    cameraTarget.position -
    orbitRotation * Vector3.forward * cameraDistance;
```

After collision correction, apply both values:

```csharp
cameraTransform.SetPositionAndRotation(
    resolvedPosition,
    orbitRotation);
```

In `Awake`, initialize target and current yaw from
`CameraTarget.eulerAngles.y`. Initialize target and current pitch to `15 degrees`.
The first `LateUpdate` must place the camera using those values; it must not preserve
an arbitrary scene-camera offset.

### Camera collision

Prevent the camera from passing through terrain, trees, rocks, or structures.
Sphere cast from `CameraTarget` toward the desired camera position and move the
camera in front of the nearest obstruction.

Requirements:

- Ignore the player's own collision layer.
- Use the `0.25 m` collision sphere radius.
- Move inward immediately when obstructed.
- Return to the normal distance with `Mathf.SmoothDamp` and the `0.12 s` default.
- Never allow camera collision correction to move the player.
- Stop the camera `0.05 m` in front of the hit surface.
- Clamp resolved distance to the range `0 m` through the configured `6 m`. A close
  obstruction takes priority over maintaining camera distance.
- Use `QueryTriggerInteraction.Ignore`.
- The collision mask must exclude the `Player` layer and include terrain and
  environment layers.
- `CameraTarget` must not overlap a collider included in the collision mask.

Check the target using the same `collisionRadius`, `collisionMask`, and trigger
policy as camera collision:

```csharp
bool targetOverlapsEnvironment = Physics.CheckSphere(
    cameraTarget.position,
    collisionRadius,
    collisionMask,
    QueryTriggerInteraction.Ignore);
```

Run this check in `Awake`. If it returns `true`, log an error naming `CameraTarget`
and disable `ThirdPersonOrbitCamera`.

Maintain `currentDistance`. Each `LateUpdate`, sphere cast from `CameraTarget`
toward the desired camera position for `cameraDistance`:

```csharp
float targetDistance = cameraDistance;
if (Physics.SphereCast(
        cameraTarget.position,
        collisionRadius,
        -(orbitRotation * Vector3.forward),
        out RaycastHit hit,
        cameraDistance,
        collisionMask,
        QueryTriggerInteraction.Ignore))
{
    targetDistance = Mathf.Clamp(
        hit.distance - collisionSafetyOffset,
        0f,
        cameraDistance);
}

currentDistance = targetDistance < currentDistance
    ? targetDistance
    : Mathf.SmoothDamp(
        currentDistance,
        targetDistance,
        ref distanceVelocity,
        distanceReturnSmoothTime);
```

Always smooth outward toward the current frame's `targetDistance`, not directly
toward `cameraDistance`. This prevents overshooting a farther obstruction while
recovering from a closer obstruction. When no obstruction is present,
`targetDistance` equals `cameraDistance`.

Use `currentDistance`, not `cameraDistance`, when calculating the final position:

```csharp
Vector3 resolvedPosition =
    cameraTarget.position -
    orbitRotation * Vector3.forward * currentDistance;
```

Initialize `currentDistance` to `cameraDistance` in `Awake`.

### Cursor state

During gameplay:

```csharp
Cursor.lockState = CursorLockMode.Locked;
Cursor.visible = false;
```

When controls are released:

```csharp
Cursor.lockState = CursorLockMode.None;
Cursor.visible = true;
```

Cursor lock and gameplay availability are separate state concerns. `Escape` adds
only the `CursorReleased` control block, unlocks the cursor, and clears
movement/look/sprint input. It does not change `Time.timeScale`.

A left click inside the Game view may lock the cursor only when the application has
focus and none of `Menu`, `Dead`, `Cutscene`, or `Interaction` is active. A valid
lock click clears `CursorReleased` and `FocusLost`, locks the cursor, and ignores
the first look delta after locking. An invalid lock click changes no state.

Losing application focus adds `FocusLost`, unlocks the cursor, and clears input.
Regaining focus does not clear `FocusLost` and does not relock the cursor. A valid
left click clears it as described above.

While any control block is active, `DinosaurInputReader` must ignore `Move`, `Look`,
and `Sprint` callbacks. It must continue accepting `LockCursor`, subject to the
eligibility rules above.

## CharacterController Setup

Configure:

- Center: horizontal center of the torso and half the capsule height above the
  lowest foot position.
- Height: vertical distance from the lowest foot position to the top of the torso;
  exclude head crests and decorative spikes.
- Radius: half the maximum torso width; exclude head and tail.
- Slope Limit: `50 degrees`.
- Step Offset: the smaller of `0.3 m` and `20%` of controller height.
- Skin Width: `10%` of controller radius, clamped to `0.01-0.1 m`.
- Min Move Distance: `0`

The tail and head must not define the movement capsule. Hit-detection colliders are
outside this movement task.

After changing the dinosaur's scale, retune the controller dimensions and movement
speeds. `DinosaurPlayer` must have local scale `(1, 1, 1)` during gameplay.

## Animation Integration

Animation is conditional: perform this section only when `Model` already has an
Animator and compatible locomotion parameters. Do not create animation clips,
controllers, or blend trees as part of this movement task.

Feed animation from actual local velocity, not only from input:

- `Speed`: horizontal world velocity magnitude
- `Forward`: local forward velocity
- `Strafe`: local sideways velocity
- `IsGrounded`: current grounded state
- `IsSprinting`: current sprint state

Set Animator float damping to `0.1 s`. Keep existing blend-tree thresholds; changing
animation assets is outside scope.

Use code-driven movement and disable Animator root motion. The Animator must never
write player-root position or rotation.

## State and Gameplay Integration

Do not represent control availability with one writable boolean. Use a
`[Flags]` enum so independent systems cannot re-enable controls owned by another
system:

```csharp
[System.Flags]
public enum ControlBlockReason
{
    None = 0,
    CursorReleased = 1 << 0,
    Menu = 1 << 1,
    Dead = 1 << 2,
    Cutscene = 1 << 3,
    Interaction = 1 << 4,
    FocusLost = 1 << 5
}
```

`DinosaurInputReader` owns a private `ControlBlockReason activeBlocks` field and
exposes:

```csharp
public bool ControlsEnabled =>
    activeBlocks == ControlBlockReason.None;

public void SetControlBlock(
    ControlBlockReason reason,
    bool blocked);
```

`SetControlBlock` adds the supplied flag when `blocked` is `true` and removes only
that supplied flag when `blocked` is `false`. Reject `None` and combined flags with
`ArgumentException`; callers must change one reason per call. Whenever the active
mask changes from `None` to any blocked state, clear move, look, and sprint input
immediately. Horizontal velocity decelerates normally, gravity continues, and
camera look stops.

Existing systems use their own reasons:

- Menu opening sets `Menu`, then sets `CursorReleased`, unlocks the cursor, and makes
  it visible. Menu closing clears only `Menu`; `CursorReleased` remains until an
  eligible lock click.
- Player death/revival sets or clears `Dead`.
- Cutscene start/end sets or clears `Cutscene`.
- Interaction lock/unlock sets or clears `Interaction`.
- Escape sets `CursorReleased`.
- Focus loss sets `FocusLost`.

If menu, death, cutscene, or interaction systems do not exist, do not create them.
No system may clear a reason owned by another system.

`LockCursor` is eligible only when:

```csharp
const ControlBlockReason nonCursorBlocks =
    ControlBlockReason.Menu |
    ControlBlockReason.Dead |
    ControlBlockReason.Cutscene |
    ControlBlockReason.Interaction;

bool canLockCursor =
    Application.isFocused &&
    (activeBlocks & nonCursorBlocks) == ControlBlockReason.None;
```

When eligible, `LockCursor` clears only `CursorReleased` and `FocusLost`. It cannot
clear `Menu`, `Dead`, `Cutscene`, or `Interaction`. This rule prevents a click from
re-enabling movement while another gameplay system owns the character.

Keep these responsibilities separate:

- Input collection
- Player locomotion
- Camera orbit and collision
- Animation updates
- Stamina and gameplay restrictions

## Performance and Physics

- Do not use `Rigidbody` and `CharacterController` to move the same root object.
- Do not allocate memory every frame in movement or camera code.
- Cache component and camera transform references.
- Use layer masks for ground and camera collision queries.
- Use `Update` for input and `CharacterController` movement.
- Use `LateUpdate` for a custom follow camera.
- Use `FixedUpdate` only for Rigidbody-based physics, not for this controller.

## Required Validation

Test at several frame rates, including `30`, `60`, `120`, and uncapped FPS.

The implementation is ready when:

- Measured travel distance over 10 seconds differs by no more than `1%` between
  `30`, `60`, `120`, and uncapped FPS.
- Diagonal speed differs from straight speed by no more than `1%`.
- From rest, walk speed reaches `3.5 m/s` in `0.26 s`, within one rendered frame.
- From walk speed, releasing movement reaches `0 m/s` in `0.20 s`, within one
  rendered frame.
- Dinosaur yaw never changes by more than `rotationSpeed * Time.deltaTime` per frame.
- The same physical mouse delta changes target yaw by the same amount at every
  tested frame rate.
- The camera collision sphere does not penetrate colliders included in its mask.
- The controller remains grounded on a continuous `45-degree` test slope.
- Holding sprint reaches `6.5 m/s`; releasing sprint converges to `3.5 m/s` using
  the configured deceleration.
- Cursor locking and unlocking pass the cursor tests below after releasing controls
  and changing application focus.
- When an Animator with the listed parameters exists, each parameter reflects the
  corresponding measured movement state.
- No per-frame garbage allocations appear in the Unity Profiler during normal
  movement.

### Simultaneous-input acceptance tests

These tests are required because they catch conflicts between mouse look and
movement:

1. Hold `A` for at least two seconds while continuously moving the mouse left and
   right. The camera must orbit and the dinosaur must continuously move to the
   camera's current left without stopping or changing to tank rotation.
2. Repeat with `D`. The dinosaur must continuously move to the camera's current
   right.
3. Hold `W+A` while orbiting the camera through at least 180 degrees. Movement must
   remain diagonal relative to the camera and must not become faster than `W`.
4. Hold `W`, then move the mouse rapidly by 90 degrees. The movement direction must
   follow the new camera yaw without a persistent old-direction drift.
5. Move the mouse without pressing WASD. The dinosaur must remain in place while
   the camera orbits.
6. Press opposite directions (`A+D` or `W+S`). The matching axis must cancel to
   zero while the other input axis and mouse look continue working.
7. Open a menu, then left-click in the Game view. `Menu` must remain set, controls
   must remain disabled, and the dinosaur must not move. Close the menu, then click;
   the click may clear `CursorReleased` and restore controls.

### Camera acceptance tests

1. Orbit 360 degrees around the dinosaur without discontinuities or angle snapping.
2. Move to minimum and maximum pitch; the camera must stop at the configured limits.
3. Walk toward a wall with the wall between the camera and dinosaur. The camera
   must move inward instead of clipping through the wall.
4. Walk away from the wall. The camera must return smoothly to its configured
   distance without popping.
5. Walk and sprint across uneven ground while orbiting. Player movement completes
   in `Update`, and the camera follows the resulting position in the same frame's
   `LateUpdate`; there must be no one-frame positional lag.
6. Press Escape, click to relock, remove application focus, and restore focus. The
   cursor must remain unlocked after focus returns; clicking must relock it, and
   camera look must resume without an angle jump.
7. Place two walls on the same camera ray, first test recovery from the closer wall
   while the farther wall remains. The camera must smooth outward only to the
   farther wall and must not cross it for one frame.

## Compatibility Checklist

Before delivering the movement system:

- Confirm the project opens in Unity 6.6 without script compilation errors.
- Confirm the Input System is enabled.
- Confirm every Input Action is assigned in the Inspector.
- Confirm all serialized references are validated and missing references produce a
  clear Unity error.
- Confirm no legacy `Input.GetAxis`, `Input.GetKey`, or `Input.mousePosition` calls
  remain in the new movement system.
- Confirm obsolete Unity APIs are not used.
- Confirm movement and camera scripts work in a development build, not only in the
  Editor.
- Confirm only one component controls the active gameplay camera.
- Confirm `A`/`D` and mouse input pass all simultaneous-input acceptance tests.
- Confirm every camera acceptance test passes.
- Confirm the implementation has no unresolved placeholders or setup steps omitted
  from its delivery report.
