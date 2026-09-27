# Unity 6.6 Mobile Dinosaur Movement Guidelines

This document specifies smooth third-person dinosaur movement for Android and iOS
in landscape orientation. It is an implementation contract for an AI coding agent.

The controls are:

- Fixed virtual joystick at bottom-left: camera-relative movement
- Drag in the camera-look area: orbit camera
- Hold the sprint button: sprint
- Tap the attack button: request one primary attack
- Hold the Eat/Drink button: consume the currently selected valid resource
- Tap the interact button: request one interaction
- Tap the roar button: request one roar
- Menu button: request the game's menu

This file is standalone and authoritative for mobile implementation. The desktop
guideline is not required to interpret it. Where both documents are used in one
cross-platform project, this file overrides desktop input and cursor rules on
Android and iOS; the numeric locomotion and camera defaults are intentionally
identical.

## Required Behavior

Movement and camera touch input must work simultaneously.

| Touch input | Required result |
|---|---|
| Joystick up | Move toward the camera's horizontal forward direction |
| Joystick down | Turn and move opposite the camera's horizontal forward direction |
| Joystick left | Turn and move toward the camera's horizontal left |
| Joystick right | Turn and move toward the camera's horizontal right |
| Joystick diagonal | Move diagonally without exceeding the configured speed |
| Camera-area drag | Orbit the camera without moving the dinosaur |
| Joystick plus camera drag | Move and orbit simultaneously |
| Joystick plus sprint hold | Sprint in the joystick direction |
| Attack tap | Submit one attack request |
| Eat/Drink hold | Keep consumption requested until release |
| Interact tap | Submit one interaction request |
| Roar tap | Submit one roar request |

The dinosaur uses free camera-relative locomotion:

- It turns toward the final movement direction.
- It does not use tank controls.
- Joystick down does not make it backpedal; it turns and moves forward in the
  camera-relative backward direction.
- Joystick left and right do not make it strafe while facing forward; it turns and
  moves toward those directions.
- Camera drag never directly rotates or translates the dinosaur.

As the camera yaw changes while the joystick is held, recalculate the world-space
movement direction every frame. Never cache the world direction from the start of a
touch.

## Unity 6.6 Requirements

- Use Unity 6.6.
- Use the Input System package supported by the installed Unity 6.6 editor.
- Set Active Input Handling to `Input System Package (New)`.
- Set Input System Update Mode to `Process Events In Dynamic Update`.
- Use an `EventSystem` with `InputSystemUIInputModule`.
- Set `InputSystemUIInputModule.Pointer Behavior` to `All Pointers As Is`.
- Assign the Input System UI module's default UI actions, including Point and
  Left Click. These actions route touches through the EventSystem only; they are not
  gameplay movement actions.
- Do not use `StandaloneInputModule`.
- Do not call legacy `Input` APIs.
- Use `CharacterController`, not Rigidbody, for the player root.
- Process touch state and player movement in `Update`.
- Apply the final camera transform in `LateUpdate`.
- Do not allocate managed memory each frame.

Lock the application orientation to landscape:

- Default Orientation: `Landscape Left`
- Allow Auto Rotation: enabled only for `Landscape Left` and `Landscape Right`
- Portrait and Portrait Upside Down: disabled

Support both landscape directions and recalculate safe-area layout after an
orientation change. When width and height swap, clear every owned pointer and input
value before applying the new safe area.

## Required Deliverables

Use these exact file and class names. Adapt existing equivalents into these
components or replace them; do not leave competing implementations enabled:

- `DinosaurInputReader.cs`: owns normalized movement, look delta in degrees, sprint
  and consume held states, one-frame action requests, and control-block state.
- `DinosaurMovementController.cs`: owns movement, gravity, and dinosaur rotation.
- `ThirdPersonOrbitCamera.cs`: owns camera yaw, pitch, smoothing, follow, and
  collision.
- `MobileMovementJoystick.cs`: owns the movement-joystick pointer.
- `MobileLookArea.cs`: owns the camera-drag pointer.
- `MobileActionButton.cs`: owns one action-button pointer and dispatches its
  configured action.
- `MobileSafeArea.cs`: fits the control root to `Screen.safeArea`.
- `SurvivalStatusPresenter.cs`: maps food and hydration state to the layered
  hunger and thirst indicators.
- `DinosaurMobileControls.prefab`: contains the complete mobile controls.

Do not enable desktop and mobile input adapters at the same time. Select the active
adapter using the project's platform configuration. Mobile builds enable the mobile
adapter; desktop builds enable the keyboard/mouse adapter.

The mobile UI components write input only through these
`DinosaurInputReader` methods:

```csharp
public void SetMobileMove(Vector2 value);
public void AddMobileLookDegrees(Vector2 deltaDegrees);
public void SetMobileSprint(bool held);
public void RequestMobileAttack();
public void SetMobileConsume(bool held);
public void RequestMobileInteract();
public void RequestMobileRoar();
```

`SetMobileMove` replaces the stored movement value.
`AddMobileLookDegrees` accumulates into the current frame's look value.
`SetMobileSprint` and `SetMobileConsume` replace their stored held values.
The three `Request` methods store `Time.frameCount`. These methods ignore incoming
gameplay values while any control block is active. UI components must not reference
`DinosaurMovementController`, `ThirdPersonOrbitCamera`, or gameplay action systems
directly.

Expose these read-only states:

```csharp
public bool SprintHeld { get; }
public bool ConsumeHeld { get; }
public bool AttackPressedThisFrame =>
    attackRequestFrame == Time.frameCount;
public bool InteractPressedThisFrame =>
    interactRequestFrame == Time.frameCount;
public bool RoarPressedThisFrame =>
    roarRequestFrame == Time.frameCount;
```

Initialize request-frame fields to `-1`. A tap request is valid only during the
frame in which its pointer-down event occurs; it does not queue into a later frame.
Exactly one gameplay system reads each action property during `Update`. Gameplay
systems, not UI components, enforce cooldowns, stamina costs, valid targets, damage,
consumption progress, and action animations.

## Required Scene Hierarchy

```text
DinosaurPlayer
├── Model
└── CameraTarget

Main Camera

EventSystem

MobileControlsCanvas
└── SafeArea
    ├── MovementJoystick
    │   ├── Background
    │   └── Handle
    ├── CameraLookArea
    ├── SprintButton
    ├── AttackButton
    ├── EatDrinkButton
    ├── InteractButton
    ├── RoarButton
    ├── SurvivalStatus
    │   ├── HungerStatus
    │   │   ├── Fill
    │   │   └── Frame
    │   └── ThirstStatus
    │       ├── Fill
    │       └── Frame
    └── MenuButton
```

Required component placement:

- `DinosaurPlayer`: `CharacterController`, `DinosaurInputReader`,
  `DinosaurMovementController`
- `Main Camera`: `ThirdPersonOrbitCamera`
- `EventSystem`: `EventSystem`, `InputSystemUIInputModule`
- `MobileControlsCanvas`: `Canvas`, `CanvasScaler`, `GraphicRaycaster`
- `SafeArea`: `MobileSafeArea`
- `MovementJoystick`: `MobileMovementJoystick`
- `CameraLookArea`: `MobileLookArea`
- `SprintButton`, `AttackButton`, `EatDrinkButton`, `InteractButton`,
  `RoarButton`: one `MobileActionButton` each
- `MenuButton`: `Button`
- `SurvivalStatus`: `SurvivalStatusPresenter`

There must be exactly one enabled `EventSystem`, one enabled gameplay camera
controller, and one enabled mobile controls canvas.

Place `CameraTarget` at the visual center of the dinosaur's upper torso. Its local
rotation must be identity and its local scale must be `(1, 1, 1)`.

`MobileMovementJoystick`, `MobileLookArea`, and `MobileActionButton` must implement
`IPointerDownHandler`, `IDragHandler`, `IPointerUpHandler`, and
`ICancelHandler`. `MobileActionButton.OnDrag` does not change button state; it exists
only so the original pointer retains ownership after leaving the button rectangle.

## Serialized Reference Contract

Assign these references in the prefab or scene:

| Component | Required references |
|---|---|
| `DinosaurInputReader` | `MobileMovementJoystick`, `MobileLookArea`, array containing exactly five `MobileActionButton` instances |
| `DinosaurMovementController` | `CharacterController`, `DinosaurInputReader`, `ThirdPersonOrbitCamera` |
| `ThirdPersonOrbitCamera` | `CameraTarget`, environment collision mask |
| `MobileMovementJoystick` | Background RectTransform, Handle RectTransform, `DinosaurInputReader` |
| `MobileLookArea` | Look-area RectTransform, `DinosaurInputReader` |
| `MobileActionButton` | Button RectTransform, `DinosaurInputReader`, action type, trigger mode |
| `MobileSafeArea` | SafeArea RectTransform, `DinosaurInputReader`, joystick, look area, and all five action buttons |
| `SurvivalStatusPresenter` | Hunger Fill Image, Hunger Frame Image, Thirst Fill Image, Thirst Frame Image |

Use `[RequireComponent(typeof(CharacterController))]` on
`DinosaurMovementController`. In each component's `Awake`, validate every required
reference. A missing reference must produce one `Debug.LogError` containing the
component type, GameObject name, and missing field name, then disable that component.
Do not search the scene to silently replace missing references.

## Canvas and Safe Area

Configure `MobileControlsCanvas`:

- Render Mode: `Screen Space - Overlay`
- Sort Order: `100`
- Canvas Scaler UI Scale Mode: `Scale With Screen Size`
- Reference Resolution: `1920 x 1080`
- Screen Match Mode: `Match Width Or Height`
- Match: `0.5`

`MobileSafeArea` must convert `Screen.safeArea` into normalized anchor minimum and
maximum values and apply them to the `SafeArea` RectTransform. Reapply the safe area
when screen width, screen height, or `Screen.safeArea` changes.

Use this conversion:

```csharp
Rect safe = Screen.safeArea;
Vector2 screenSize = new(Screen.width, Screen.height);

safeAreaRect.anchorMin = safe.position / screenSize;
safeAreaRect.anchorMax =
    (safe.position + safe.size) / screenSize;
safeAreaRect.offsetMin = Vector2.zero;
safeAreaRect.offsetMax = Vector2.zero;
```

Before applying a changed layout, `MobileSafeArea` calls `CancelActiveTouches()` on
the joystick, look area, and all five action buttons, then clears stored movement,
look, sprint, consume, and one-frame action requests. Cache the last width, height,
and safe-area Rect; do not rewrite the layout when none changed.

All controls must remain completely inside `SafeArea`.

## Touch Ownership

Each control tracks one pointer ID:

- `MobileMovementJoystick` owns the pointer that begins inside its interaction area.
- `MobileLookArea` owns the pointer that begins inside the camera-look area.
- Each `MobileActionButton` owns the pointer that presses that button.
- A control ignores additional pointers until its owned pointer ends or cancels.
- A pointer cannot control two surfaces.
- Moving a pointer outside its starting control does not transfer ownership.
- Releasing or cancelling a pointer clears only that pointer's state.

Clear all movement, look, sprint, consume, and one-frame action-request state on:

- `OnDisable`
- Application focus loss
- Application pause
- Touch cancellation
- Landscape Left/Right orientation change
- A control-block transition from none to blocked

Do not identify touches by their order in a touch collection. Store and compare the
pointer ID supplied by `PointerEventData`.

## Movement Joystick

### Visual assets

Use:

- `MobileJoystickAssets/dinosaur_joystick_outer.svg` for
  `MovementJoystick/Background`
- `MobileJoystickAssets/dinosaur_joystick_inner.svg` for
  `MovementJoystick/Handle`

Copy both files into the Unity project's `Assets/UI/Mobile/Joystick` folder. Install
the Vector Graphics package version supported by the installed Unity 6.6 editor so
Unity imports SVG files as vector sprites.

For both UI `Image` components:

- Source Image: the corresponding imported SVG sprite
- Image Type: `Simple`
- Preserve Aspect: enabled
- Color: white with full RGB values; control opacity through the Image alpha
- Raycast Target: enabled for Background and disabled for Handle

The outer SVG has a `512 x 512` viewbox and renders at the required `240 x 240` UI
size. The inner SVG has a `256 x 256` viewbox and renders at the required
`100 x 100` UI size. Do not rasterize, crop, stretch, or add an opaque background.

### Layout

Use a fixed joystick:

- Anchor: bottom-left of `SafeArea`
- Background size: `240 x 240` reference-resolution pixels
- Center offset from safe-area bottom-left: `(170, 170)` pixels
- Handle size: `100 x 100` pixels
- Maximum handle travel radius: `90` pixels
- Visual opacity while idle: `0.55`
- Visual opacity while held: `0.9`

The joystick center does not move to the initial touch position.
Set Background and Handle pivots to `(0.5, 0.5)`. The joystick center in Background
local space is `Vector2.zero`. Use `eventData.pressEventCamera` in
`RectTransformUtility.ScreenPointToLocalPointInRectangle`; it is `null` for the
required Screen Space Overlay canvas.

### Pointer calculation

On pointer down:

1. Reject the event if the joystick already owns another pointer.
2. Store `eventData.pointerId`.
3. Convert the pointer position into local joystick coordinates with
   `RectTransformUtility.ScreenPointToLocalPointInRectangle`.
4. Update movement immediately.

On drag, update only when `eventData.pointerId` equals the owned pointer ID.

Calculate normalized movement:

```csharp
Vector2 rawOffset = localPointerPosition - joystickCenter;
Vector2 normalized = rawOffset / handleTravelRadius;
normalized = Vector2.ClampMagnitude(normalized, 1f);

float magnitude = normalized.magnitude;
Vector2 moveInput = magnitude <= deadZone
    ? Vector2.zero
    : normalized.normalized *
      Mathf.InverseLerp(deadZone, 1f, magnitude);
```

Serialized defaults:

| Setting | Default |
|---|---:|
| Dead zone | `0.12` |
| Handle travel radius | `90 px` |

Set the handle position to `normalized * handleTravelRadius`. The remapped
`moveInput` controls speed, so a partly displaced joystick produces slower movement.

On pointer up or cancellation:

- Set movement to `Vector2.zero`.
- Return the handle to the center immediately.
- Release the owned pointer ID.

## Camera Look Area

### Layout

`CameraLookArea` must:

- Cover the right `62%` of `SafeArea`.
- Stretch from safe-area bottom to top.
- Render no visible graphic.
- Use a transparent `Image` with alpha `0.001` and Raycast Target enabled.
- Be behind all five action buttons and `MenuButton` in canvas sibling order.

`MovementJoystick.Background`, every action button, and `MenuButton` must each have
an `Image` with Raycast Target enabled. `MovementJoystick.Handle` and action-button
icon/label children must have Raycast Target disabled so they cannot intercept the
parent control's pointer.

Touches that begin on the sprint or menu button belong to those buttons and must not
start camera look.

### Drag calculation

On pointer down:

1. Reject the event if the look area already owns a pointer.
2. Store `eventData.pointerId`.
3. Store the current pointer position.
4. Produce zero look delta for that event.

On drag:

1. Verify the owned pointer ID.
2. Calculate pixel delta from the previous stored position.
3. Store the new position.
4. Convert the pixel delta to degrees using the current safe-area height.

```csharp
Vector2 normalizedDelta =
    pixelDelta / Screen.safeArea.height;

Vector2 lookDeltaDegrees = new(
    normalizedDelta.x * yawDegreesPerScreen,
    normalizedDelta.y * pitchDegreesPerScreen);
```

Serialized defaults:

| Setting | Default |
|---|---:|
| Yaw degrees per safe-area height | `180 degrees` |
| Pitch degrees per safe-area height | `120 degrees` |
| Invert vertical look | `false` |

Apply vertical inversion in `MobileLookArea`, then forward degrees to the input
reader:

```csharp
Vector2 cameraDeltaDegrees = new(
    lookDeltaDegrees.x,
    invertVerticalLook
        ? lookDeltaDegrees.y
        : -lookDeltaDegrees.y);

inputReader.AddMobileLookDegrees(cameraDeltaDegrees);
```

The input reader accumulates all look degrees received before
`DinosaurMovementController.Update`. The movement controller consumes and clears
them exactly once per frame.

Do not multiply touch look delta by `Time.deltaTime`. Normalizing by safe-area height
makes the same fraction-of-screen drag produce the same rotation on different
resolutions.

On pointer up or cancellation:

- Produce zero additional look delta.
- Release the owned pointer ID.
- Do not reset camera yaw or pitch.

## Action Button Cluster

Define:

```csharp
public enum MobileActionType
{
    Sprint,
    Attack,
    Consume,
    Interact,
    Roar
}

public enum MobileActionTrigger
{
    Press,
    Hold
}
```

Configure one `MobileActionButton` instance per row:

| Button | Type | Trigger | Size | Center offset from safe-area bottom-right |
|---|---|---|---:|---:|
| `SprintButton` | `Sprint` | `Hold` | `170 x 170` | `(-145, 145)` |
| `AttackButton` | `Attack` | `Press` | `190 x 190` | `(-145, 345)` |
| `EatDrinkButton` | `Consume` | `Hold` | `150 x 150` | `(-340, 130)` |
| `InteractButton` | `Interact` | `Press` | `150 x 150` | `(-340, 310)` |
| `RoarButton` | `Roar` | `Press` | `140 x 140` | `(-510, 220)` |

Every button is anchored to the bottom-right of `SafeArea`. Offsets and sizes use
the `1920 x 1080` reference-resolution coordinate system.

Common pointer rules:

1. On pointer down, reject the event if that button already owns a pointer.
2. Store `eventData.pointerId`.
3. Set the button's pressed visual immediately.
4. Dispatch according to its configured type and trigger.
5. Ignore other pointers until the owned pointer releases or cancels.
6. Pointer drag never changes action state and never transfers ownership.
7. On pointer up, cancellation, `OnDisable`, focus loss, application pause, or
   orientation change, release hold state, clear the visual, and release ownership.

Dispatch rules:

- `Sprint/Hold`: pointer down calls `SetMobileSprint(true)`; release calls
  `SetMobileSprint(false)`.
- `Consume/Hold`: pointer down calls `SetMobileConsume(true)`; release calls
  `SetMobileConsume(false)`.
- `Attack/Press`: pointer down calls `RequestMobileAttack()` exactly once; holding
  does not repeat.
- `Interact/Press`: pointer down calls `RequestMobileInteract()` exactly once;
  holding does not repeat.
- `Roar/Press`: pointer down calls `RequestMobileRoar()` exactly once; holding does
  not repeat.

Reject Inspector configurations other than the five table rows. In `Awake`, log a
specific error and disable a button whose trigger does not match its required action
type.

The five-element action-button array must contain exactly one of every
`MobileActionType`. `DinosaurInputReader.Awake` must reject null entries, duplicates,
missing types, and arrays whose length is not five. Log the invalid condition and
disable the input reader.

Each action button exposes `SetAvailable(bool available)` for its gameplay consumer.
Visual states use:

| State | Image alpha |
|---|---:|
| Available, idle | `0.75` |
| Available, pressed | `0.95` |
| Unavailable or cooling down | `0.35` |

An unavailable button must still keep its `Image.raycastTarget` enabled so its
screen region cannot leak touches into `CameraLookArea`. It accepts and owns the
pointer for that press but dispatches no input state or request. Availability
changes do not cancel joystick movement or camera look.

Sprint affects movement only while post-dead-zone `moveInput.sqrMagnitude > 0f`.
Holding Sprint while stationary does not move or rotate the dinosaur.

Eat/Drink uses one contextual hold button:

- The gameplay targeting system owns one current consumable target.
- A food target interprets `ConsumeHeld` as eating.
- A water target interprets `ConsumeHeld` as drinking.
- With no valid current consumable target, holding the button performs no gameplay
  action.
- Switching or losing the target stops the old target's consumption immediately;
  continued holding may begin consumption on the new valid target.
- The UI does not select targets, grant resources, or decide consumption duration.

Attack, Interact, and Roar are requests, not guaranteed actions. Their sole gameplay
consumers validate current state, cooldown, stamina, and targets. A rejected request
must not move the dinosaur or alter camera input.

### Action button visual assets

Copy the SVG files from `MobileActionAssets` into the Unity project's
`Assets/UI/Mobile/Actions` folder and assign:

| UI use | SVG |
|---|---|
| `SprintButton` | `dinosaur_button_sprint.svg` |
| `AttackButton` | `dinosaur_button_attack.svg` |
| `EatDrinkButton` with a food target | `dinosaur_button_eat.svg` |
| `EatDrinkButton` with a water target | `dinosaur_button_drink.svg` |
| `RoarButton` | `dinosaur_button_roar.svg` |
| Reserved jump button artwork | `dinosaur_button_jump.svg` |

The gameplay targeting system must switch `EatDrinkButton` between the Eat and
Drink sprites when the current consumable target type changes. Keep the last sprite
when no target exists and show the button's unavailable alpha of `0.35`.

The Jump SVG is visual artwork only. This movement contract does not implement
jumping and must not display or wire a Jump button until a separate jump
specification defines airborne movement, cooldown, animation, and grounding rules.

For every action-button `Image`:

- Image Type: `Simple`
- Preserve Aspect: enabled
- Raycast Target: enabled only on the button's root Image
- Color RGB: white
- Alpha: controlled by the action-button visual-state table

## Menu Button

Layout:

- Anchor: top-right of `SafeArea`
- Size: `120 x 120` reference-resolution pixels
- Center offset from safe-area top-right: `(-90, -90)` pixels

On click, request the existing menu system to set
`ControlBlockReason.Menu`. The menu must clear only its own block when it closes.
Creating a menu screen is outside this movement task.

If the project has no menu system, keep the button disabled and document that
integration requirement. Do not make the button silently change `Time.timeScale`.

## Hunger and Thirst Status

### Assets and hierarchy

Copy the SVG files from `MobileStatusAssets` into the Unity project's
`Assets/UI/Mobile/Status` folder.

Assign:

| UI object | SVG | Image configuration |
|---|---|---|
| `HungerStatus/Fill` | `hunger_stomach_fill.svg` | Filled, Vertical, Bottom origin |
| `HungerStatus/Frame` | `hunger_hex_frame.svg` | Simple |
| `ThirstStatus/Fill` | `thirst_drop_fill.svg` | Filled, Vertical, Bottom origin |
| `ThirstStatus/Frame` | `thirst_hex_frame.svg` | Simple |

Within each status object, Fill must be the first sibling and Frame the second
sibling so the fixed hexagon and organ outline render above the changing fill.
All four Images must preserve aspect and have Raycast Target disabled.

### Layout

Anchor both indicators to the top-left of `SafeArea`:

| Indicator | Size | Center offset from safe-area top-left |
|---|---:|---:|
| Hunger | `120 x 120` | `(90, -90)` |
| Thirst | `120 x 120` | `(225, -90)` |

The hexagon frame never empties, scales, rotates, or changes shape. Only the stomach
or water-drop Fill Image changes.

### Resource values

Use resource-level values where `1` means full and `0` means empty:

```csharp
float foodLevel01 = Mathf.Clamp01(
    currentFood / maximumFood);

float hydrationLevel01 = Mathf.Clamp01(
    currentHydration / maximumHydration);

hungerFill.fillAmount = foodLevel01;
thirstFill.fillAmount = hydrationLevel01;
```

Perform division as floating point. `maximumFood` and `maximumHydration` must be
greater than zero. If either maximum is invalid, log a specific error and disable
`SurvivalStatusPresenter`; do not display a false full or empty state.

If an existing gameplay model exposes need values instead, where `0` means
satisfied and `1` means starving or dehydrated, convert them:

```csharp
foodLevel01 = 1f - Mathf.Clamp01(hungerNeed01);
hydrationLevel01 =
    1f - Mathf.Clamp01(thirstNeed01);
```

Do not feed need values directly into `fillAmount`.

### Emptying behavior

- At `1.0`, the stomach or water drop is completely filled.
- At `0.75`, its top quarter is empty.
- At `0.5`, its upper half is empty.
- At `0.25`, only its bottom quarter remains filled.
- At `0.0`, no colored fill remains, but the hexagon and stomach/drop outline remain
  visible.
- Fill decreases from top to bottom because the Image Fill Origin is Bottom.
- Increasing food or hydration refills from bottom to top.

Update the Image only when its normalized value changes. Set the initial fill in
`Start` before the first visible gameplay frame. Do not animate toward an incorrect
intermediate value when loading or respawning.

### Status validation

The status implementation passes when:

1. Setting food to maximum fills the entire stomach.
2. Reducing food from maximum to zero continuously empties the stomach from top to
   bottom without affecting the hexagon.
3. Setting hydration to maximum fills the entire drop.
4. Reducing hydration from maximum to zero continuously empties the drop from top
   to bottom without affecting the hexagon.
5. Refilling either resource reverses the same path from bottom to top.
6. Both indicators remain inside the safe area in Landscape Left and Landscape
   Right.
7. Neither indicator intercepts joystick, camera, or action-button touches.

## Locomotion

Use this camera-relative calculation:

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

Preserve analog magnitude. Do not normalize `desiredDirection` after clamping.

Use these defaults:

| Setting | Default |
|---|---:|
| Walk speed | `3.5 m/s` |
| Sprint speed | `6.5 m/s` |
| Acceleration | `14 m/s²` |
| Deceleration | `18 m/s²` |
| Dinosaur rotation speed | `540 degrees/s` |
| Gravity | `-25 m/s²` |
| Grounded vertical velocity | `-2 m/s` |

Target velocity is:

```csharp
float maximumSpeed = sprintHeld ? sprintSpeed : walkSpeed;
Vector3 targetVelocity =
    desiredDirection * maximumSpeed;
```

Use acceleration while target speed magnitude is greater than current horizontal
speed magnitude. Use deceleration while it is less, including joystick release and
sprint release.

Use this exact velocity update:

```csharp
float rate =
    desiredDirection.sqrMagnitude <= 0f ||
    targetVelocity.sqrMagnitude <= horizontalVelocity.sqrMagnitude
        ? deceleration
        : acceleration;

horizontalVelocity = Vector3.MoveTowards(
    horizontalVelocity,
    targetVelocity,
    rate * Time.deltaTime);
```

The joystick component has already applied and remapped the raw `0.12` dead zone.
Do not apply that dead zone a second time. Rotate the dinosaur toward
`desiredDirection` whenever `desiredDirection.sqrMagnitude > 0f`:

```csharp
if (desiredDirection.sqrMagnitude > 0f)
{
    Quaternion targetRotation =
        Quaternion.LookRotation(desiredDirection, Vector3.up);

    transform.rotation = Quaternion.RotateTowards(
        transform.rotation,
        targetRotation,
        rotationSpeed * Time.deltaTime);
}
```

Do not rotate from camera drag alone.

Track vertical velocity separately:

```csharp
if (controller.isGrounded && verticalVelocity < 0f)
{
    verticalVelocity = groundedVerticalVelocity;
}
else
{
    verticalVelocity += gravity * Time.deltaTime;
}

Vector3 motion = horizontalVelocity;
motion.y = verticalVelocity;
controller.Move(motion * Time.deltaTime);
```

Call `CharacterController.Move` exactly once per `Update`. Apply
`Time.deltaTime` exactly once to the combined horizontal and vertical velocity.
Use `CharacterController.isGrounded` as the only grounded source. Jumping, falling
damage, ledge detection, stamina consumption, and combat-facing behavior are not
part of this movement implementation.

## CharacterController Configuration

`DinosaurPlayer` must have local scale `(1, 1, 1)`. Configure its controller from
the imported model:

- Center: horizontal center of the torso and half the capsule height above the
  lowest foot position
- Height: vertical distance from the lowest foot position to the top of the torso,
  excluding head crests and decorative spikes
- Radius: half the maximum torso width, excluding head and tail
- Slope Limit: `50 degrees`
- Step Offset: the smaller of `0.3 m` and `20%` of controller height
- Skin Width: `10%` of controller radius, clamped to `0.01-0.1 m`
- Min Move Distance: `0`

The head and tail do not define the movement capsule. Hit-detection colliders,
when present, must be triggers and must not move the player root.

## Animation Integration

Animation integration is conditional on `Model` already having an Animator with
compatible parameters. Do not create animation clips, controllers, or blend trees.

When those parameters exist, update:

- `Speed`: horizontal velocity magnitude
- `Forward`: local forward velocity
- `Strafe`: local sideways velocity
- `IsGrounded`: `CharacterController.isGrounded`
- `IsSprinting`: sprint is held and `desiredDirection.sqrMagnitude > 0f`

Use `0.1 s` damping for Animator float parameters. Disable Animator root motion.
The Animator must not write `DinosaurPlayer` position or rotation.

## Camera

Use one custom third-person orbit camera. Do not enable Cinemachine Brain, virtual
cameras, or another script that writes `Main Camera` while
`ThirdPersonOrbitCamera` is enabled.

| Setting | Default |
|---|---:|
| Minimum pitch | `-30 degrees` |
| Maximum pitch | `65 degrees` |
| Camera distance | `6 m` |
| Initial pitch | `15 degrees` |
| Rotation smooth time | `0.07 s` |
| Distance return smooth time | `0.12 s` |
| Collision sphere radius | `0.25 m` |
| Collision safety offset | `0.05 m` |

In `Awake`:

1. Validate `CameraTarget`, camera Transform, and collision mask references.
2. Initialize target and current yaw from `CameraTarget.eulerAngles.y`.
3. Initialize target and current pitch to `15 degrees`.
4. Initialize current camera distance to `6 m`.
5. Use `Physics.CheckSphere` with radius `0.25 m`, the camera collision mask, and
   `QueryTriggerInteraction.Ignore` to verify `CameraTarget` does not overlap an
   environment collider.
6. Log a specific error and disable the component if validation fails.

`AdvanceOrbitDegrees(Vector2 lookDeltaDegrees, float deltaTime)` must:

1. Add horizontal look degrees to target yaw.
2. Add the already-inverted vertical look degrees to target pitch.
3. Clamp target pitch to `-30` through `65` degrees.
4. Smooth current yaw and pitch with `Mathf.SmoothDampAngle`, smooth time `0.07 s`,
   maximum speed `Mathf.Infinity`, and the supplied `deltaTime`.

```csharp
targetYaw += lookDeltaDegrees.x;
targetPitch = Mathf.Clamp(
    targetPitch + lookDeltaDegrees.y,
    minPitch,
    maxPitch);

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
```

Expose current yaw as a read-only property:

```csharp
public float CurrentYaw => currentYaw;
```

`DinosaurMovementController.Update` performs:

1. Read stored mobile input.
2. Pass accumulated look degrees to
   `ThirdPersonOrbitCamera.AdvanceOrbitDegrees`.
3. Clear consumed look.
4. Read the updated `CurrentYaw`.
5. Calculate camera-relative movement.
6. Update velocity and dinosaur facing.
7. Call `CharacterController.Move`.

`ThirdPersonOrbitCamera.LateUpdate` follows the moved player, resolves camera
collision, and writes the camera transform.

Touching the camera area while holding the joystick must update camera yaw and
movement direction in the same frame.

In `LateUpdate`, build the camera orbit:

```csharp
Quaternion orbitRotation =
    Quaternion.Euler(currentPitch, currentYaw, 0f);

Vector3 backwardDirection =
    -(orbitRotation * Vector3.forward);
```

Sphere cast from `CameraTarget.position` in `backwardDirection` for the configured
`cameraDistance`. Use collision radius `0.25 m`, the environment collision mask,
and `QueryTriggerInteraction.Ignore`.

Set the target distance:

```csharp
float targetDistance = cameraDistance;
if (Physics.SphereCast(
        cameraTarget.position,
        collisionRadius,
        backwardDirection,
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
```

Move inward immediately. Smooth outward only toward the obstruction distance
calculated during the current frame:

```csharp
currentDistance = targetDistance < currentDistance
    ? targetDistance
    : Mathf.SmoothDamp(
        currentDistance,
        targetDistance,
        ref distanceVelocity,
        distanceReturnSmoothTime);
```

Calculate and apply the final transform:

```csharp
Vector3 resolvedPosition =
    cameraTarget.position +
    backwardDirection * currentDistance;

transform.SetPositionAndRotation(
    resolvedPosition,
    orbitRotation);
```

The collision mask must include terrain and environment collision layers and
exclude the player, triggers, UI, and hit-detection-only layers. Camera collision
must never move the dinosaur.

## Control Blocks and Mobile Lifecycle

Define the complete blocker enum in `DinosaurInputReader.cs`:

```csharp
[System.Flags]
public enum ControlBlockReason
{
    None = 0,
    Menu = 1 << 0,
    Dead = 1 << 1,
    Cutscene = 1 << 2,
    Interaction = 1 << 3,
    FocusLost = 1 << 4,
    ApplicationPaused = 1 << 5
}
```

`DinosaurInputReader` owns a private `ControlBlockReason activeBlocks` field.
Controls are enabled only when `activeBlocks == ControlBlockReason.None`.

Expose:

```csharp
public bool ControlsEnabled =>
    activeBlocks == ControlBlockReason.None;

public void SetControlBlock(
    ControlBlockReason reason,
    bool blocked);
```

`SetControlBlock` accepts exactly one non-`None` flag. It adds that flag when
`blocked` is `true` and removes only that flag when `blocked` is `false`. Throw
`ArgumentException` for `None` or a combined value. One system must never clear
another system's reason.

Whenever a reason changes from clear to blocked, call `CancelActiveTouches()` on
the joystick, look area, and all five action buttons. Clear stored movement, look,
sprint, consume, and action-request frame stamps before the next movement update.
Removing a reason never restores old touch state.

Rules:

- Menu open adds `Menu`; menu close removes only `Menu`.
- Player death/revival adds or removes only `Dead`.
- Cutscene start/end adds or removes only `Cutscene`.
- Interaction lock/unlock adds or removes only `Interaction`.
- `OnApplicationPause(true)` adds `ApplicationPaused` and clears touch state.
- `OnApplicationPause(false)` removes only `ApplicationPaused`.
- `OnApplicationFocus(false)` adds `FocusLost` and clears touch state, even when
  `ApplicationPaused` is already active.
- `OnApplicationFocus(true)` removes only `FocusLost`.
- Controls are enabled only when no block is active.
- Gravity continues while controls are blocked.
- Horizontal movement decelerates to zero while controls are blocked.
- `DinosaurMovementController.Update` must continue running while blocked so
  deceleration and gravity are applied; it substitutes zero movement, zero look,
  and sprint released instead of returning early.

`DinosaurInputReader` owns `OnApplicationPause` and `OnApplicationFocus`. It uses
its serialized touch-control references to cancel their owned pointers before
clearing stored values and changing the corresponding block. No other component
may independently add or clear `ApplicationPaused` or `FocusLost`.

Never restore a held joystick, sprint, consume action, or pending press request after
interruption. The player must touch the controls again.

## UI and Gameplay Conflict Rules

- Movement begins only from the joystick interaction area.
- Camera look begins only from `CameraLookArea`.
- Sprint begins only from `SprintButton`.
- Attack begins only from `AttackButton`.
- Consumption begins only from `EatDrinkButton`.
- Interaction begins only from `InteractButton`.
- Roar begins only from `RoarButton`.
- Menu begins only from `MenuButton`.
- Touches over action buttons, menu UI, or other raycastable UI must not move or
  rotate the camera.
- Opening any full-screen menu disables the gameplay control canvas.
- Disabling the canvas must clear every owned pointer and stored input value.
- Re-enabling the canvas starts with zero movement, zero look delta, sprint and
  consume released, and no press request.

## Performance

- Cache RectTransforms and component references.
- Do not call `Find`, `FindObjectOfType`, or `Camera.main` during per-frame methods.
- Do not use LINQ in touch processing.
- Do not allocate new collections, arrays, or event objects each frame.
- Do not poll every active touch when pointer events already identify the owner.
- Do not run camera collision more than once per rendered frame.

## Required Tests

Test on a physical Android device and a physical iOS device. Editor Device Simulator
testing does not replace physical-device testing.

Test at `30`, `60`, and `120` FPS when the device supports those refresh rates.

The implementation passes when:

1. Holding joystick right continuously moves the dinosaur toward the camera's
   current right.
2. Holding joystick down turns the dinosaur and moves it opposite camera forward;
   it does not backpedal.
3. A full diagonal joystick produces no more speed than a full cardinal direction.
4. Partial joystick displacement produces proportionally slower movement.
5. One finger can hold the joystick while a second finger orbits the camera.
6. A third finger can hold sprint without interrupting movement or camera drag.
7. Moving any owned finger across another control does not transfer ownership.
8. Releasing the look finger does not stop movement or sprint.
9. Releasing the movement finger does not reset camera yaw or pitch.
10. A drag covering half the safe-area height changes yaw by `90 degrees`.
11. Touch look sensitivity differs by no more than `2%` between tested resolutions.
12. Movement distance over 10 seconds differs by no more than `1%` between tested
    frame rates.
13. Camera collision never crosses an included environment collider.
14. The controls remain inside notches, rounded corners, and system gesture insets.
15. Rotating between Landscape Left and Landscape Right preserves correct safe-area
    placement and clears active touch state.
16. Backgrounding and restoring the app leaves movement, sprint, and consumption
    released, with no pending action request.
17. Touches over menu UI never move the dinosaur or camera.
18. No per-frame managed allocations occur during steady movement and camera drag.
19. Activate `Menu` and `Cutscene`, then clear only `Menu`; controls must remain
    disabled until `Cutscene` is also cleared.
20. Recover from a close camera obstruction while a farther obstruction remains on
    the same camera ray. The camera must smooth only to the farther obstruction and
    must never cross it for one frame.
21. Start moving, looking, and sprinting, then rotate between Landscape Left and
    Landscape Right. All three inputs must clear before the controls move to the new
    safe area.
22. Hold joystick and camera drag with two fingers, hold Sprint with a third, and
    tap Attack with a fourth. Movement, look, and sprint must continue, and exactly
    one attack request must occur.
23. Hold Attack for one second. It must produce one request, not repeated requests.
24. Hold Eat/Drink on a food target, then release. `ConsumeHeld` must remain true
    only while owned and the food system must stop consuming on release.
25. Repeat Eat/Drink on a water target. The water system, not the food system, must
    consume the request.
26. Hold Eat/Drink with no valid target. No food or water value may change.
27. Tap Interact and Roar separately. Each tap must create exactly one request and
    must not change movement or camera input.
28. Begin an action touch and drag across `CameraLookArea`. The action retains its
    pointer and the drag produces no camera rotation.
29. Activate `Menu` or `Dead`, then tap every action button. No action request or
    held state may be accepted.
30. Background the app while Sprint and Eat/Drink are held. Both states and every
    owned pointer must be cleared after restoration.
31. Mark each action button unavailable and tap it. No request may occur, and the
    touch must not rotate the camera through the button.

## Delivery Checklist

- Unity 6.6 opens the project without compilation errors.
- Input System and `InputSystemUIInputModule` are active.
- Landscape orientation settings match this document.
- The mobile controls prefab is present and configured.
- Every required reference is assigned.
- Missing references log a specific error and disable the affected component.
- Desktop input is disabled in mobile builds.
- Mobile input is disabled in desktop builds.
- Multi-touch movement, look, sprint, and press actions pass together.
- Sprint, Attack, Eat/Drink, Interact, and Roar buttons use the exact action and
  trigger configuration table.
- Action buttons never leak touches into `CameraLookArea`.
- Safe-area behavior passes in both landscape directions.
- Android and iOS development builds are tested on physical devices.
- No placeholder methods, unresolved TODOs, or undocumented setup steps remain.
