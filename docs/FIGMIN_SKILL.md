---
name: figmin-xr-scripting
description: >-
  Comprehensive reference for the Figmin XR Scripting API. Use this skill
  whenever writing JavaScript for Figmin XR spatial apps, creating or
  manipulating 3D objects, working with the MCP agent bridge, or building
  XR experiences. Covers object creation, transforms, physics, sketches,
  input, raycasting, multiplayer, the Agent camera object (MCP), and all
  component APIs. Platforms: Meta Quest, Steam VR, Windows, Vision OS, iOS.
  API docs: https://www.figmin.com/api
---

<!-- @topic: basics -->
# Figmin XR Scripting API Reference

Comprehensive reference for building spatial apps in Figmin XR. This document provides the core details needed to write Figmin XR spatial apps, including key concepts, APIs, and patterns. For an in-depth look including additional examples, full app demos, and downloadable code—visit the official documentation at https://www.figmin.com/api

## What is a Spatial App?

A **Spatial App** is an HTML page deployed into the 3D scene (not just viewed in Figmin's built-in browser). Spatial apps can:
- Create and manipulate 3D objects in the scene
- Control their own position, rotation, and scale in the scene (via `getAppObject()`)
- Access player body parts (hands, head) for interaction
- Use physics, lighting, and other scene features
- Persist beyond the browser session

When you load an HTML page in Figmin's browser, it runs in "browser mode". To access most API features, you must **deploy** it as a spatial app (the system prompts for this, or you can call `figmin.requireSpatialApp()`).

**Note**: Some API functions work in browser mode (e.g., `getWindow()`, `getCamera()`, `print()`), but most require spatial app mode (e.g., `createObject()`, `getAppObject()`, input connection).

## UI Design Guidelines

Figmin XR apps may be operated via **eye tracking** and **hand tracking**, not just controllers. Design your UI accordingly:

- **Use large buttons** (minimum 48x48px, prefer 64x64px or larger)
- **Add spacing between interactive elements** to prevent mis-taps
- **Avoid small text** - use at least 16px font size
- **Provide visual feedback** for hover/focus states
- **Keep UI simple** - fewer elements work better in XR

## Quick Start Template

```html
<!DOCTYPE html>
<html>
<head>
    <meta name="figmin">
    <title>My Spatial App</title>
</head>
<body>
    <button id="spawn">Spawn Object</button>
    <script>
        document.addEventListener("FigminReady", init);

        function init() {
            // Require spatial app context
            if (!figmin.isSpatialApp()) {
                figmin.requireSpatialApp();
                return;
            }

            // Only master client creates objects in multiplayer
            if (!figmin.isMasterClient()) return;

            document.getElementById("spawn").addEventListener("click", () => {
                const obj = figmin.createObject(figmin.ObjectType.TEXT3D, {
                    position: figmin.getWindowSpawnPosition(0.3, "right"),
                    rotation: figmin.getWindow().transform.rotation,  // TEXT3D faces user directly
                    text: "Hello XR!",
                    targetSize: 0.15
                });

                obj.onStart(() => {
                    console.log("Object ready:", obj.name);
                    // Per-object update loop: obj.onUpdate((dt) => { ... });
                });
            });

            // Optional: game loop
            figmin.setUpdateFunction((deltaTime) => {
                // Called every frame
            });
        }
    </script>
</body>
</html>
```

## Coordinate System

- **+X** = Right
- **+Y** = Up
- **+Z** = Forward (Unity-style)
- Units: **meters** for positions/sizes, **degrees** for Euler rotations

**World Origin**: The coordinate `(0, 0, 0)` represents the center of the user's room/play space at floor level. Objects positioned at `y: 0` will be on the floor.

## Events

### FigminPreload
Fires early, before API is ready. Used for engine configuration before initialization.

### FigminReady
Fires when API is initialized. Start your app here:
```javascript
document.addEventListener("FigminReady", init);

function init() {
    if (!figmin.isSpatialApp()) {
        figmin.requireSpatialApp();
        return;
    }
    // Your app logic
}
```

---

## FigminObject

Core entity (similar to Unity GameObject). All scene objects are FigminObjects.

### Properties

| Property | Type | Access | Description |
|----------|------|--------|-------------|
| `oid` | string | read-only | Internal object id, scoped to this script/session (not visible to other apps, doesn't persist across reloads) |
| `name` | string | read-only | Globally addressable name, set via `createObject` params. Names are enforced unique — if taken, Figmin auto-adjusts it, so read back `obj.name` after `onStart` if you need it later |
| `objectType` | ObjectType | read-only | The `figmin.ObjectType` of this object |
| `active` | boolean | read-write | Visibility (true=visible) |
| `interactionState` | InteractionState | read-write | ENABLED or LOCKED |
| `scenePersistent` | boolean | read-only | true = survives script end |
| `parent` | FigminObject\|null | read-write | Parent object (set to parent, null to unparent) |
| `children` | FigminObject[] | read-only | Direct child objects |

### Components (always present)
- `status` - Lifecycle state
- `transform` - Position, rotation, scale
- `collider` - Collision shape

### Type-specific Components
- TEXT3D → `text3d`
- MODEL3D → `model3d`
- IMAGE → `image`
- SKETCH → `sketch`
- PORTAL → `portal`
- BODYPART → `bodypart`
- AGENT → `agent` (MCP sessions only)
- APP → `app`

### Lifecycle Methods

```javascript
// Called when object becomes READY
obj.onStart((object) => { });

// Called when object is destroyed
obj.onDestroy((object) => { });

// Called on error (e.g., model load failed)
obj.onError((object) => { });

// Called each frame (per-object)
// NOTE: Unlike other callbacks, onUpdate receives ONLY deltaTime, not (object, deltaTime).
// Access the object via its outer variable reference.
obj.onUpdate((deltaTime) => { });
obj.onUpdate(null); // stop per-object update loop
```

### Object Methods

```javascript
// Add runtime component
obj.addComponent(figmin.ComponentType.PHYSICS, { mass: 1.0 });
obj.addComponent(figmin.ComponentType.LIGHT, { type: figmin.lights.LightType.POINT });
obj.addComponent(figmin.ComponentType.MOTION);
obj.addComponent(figmin.ComponentType.PHYSICS_MATERIAL, { bounciness: 0.8 });

// Remove component
obj.removeComponent(figmin.ComponentType.PHYSICS);

// Destroy object
// NOTE: You can only destroy objects owned by this script, calls to destroy foreign objects will simply make them untracked.
obj.destroy();
```

### Object Parenting

Objects can be parented to form hierarchies. The transform provides both **world-space** and **local-space** accessors. When parented, `localPosition` / `localRotation` are relative to the parent (Unity-standard behavior — parent scale affects local-to-world conversion). `position` / `rotation` always return world-space values regardless of hierarchy depth. To parent objects, set `.parent` after creation — create objects first, then assign the hierarchy.

```javascript
// Create objects first, then parent
child.parent = parentObj;

// Unparent (returns to scene)
child.parent = null;

// Read parent
if (obj.parent) {
    console.log("Parent is: " + obj.parent.name);
}

// World vs local
console.log(child.transform.position);       // world-space position
console.log(child.transform.localPosition);  // position relative to parent
console.log(child.transform.scale);          // world scale
console.log(child.transform.localScale);     // scale relative to parent
```

**Restrictions:**
- **Physics** and parenting are mutually exclusive: remove one before adding the other
- When parent is **destroyed**, all children are also destroyed (if script-owned)

---

## Creating Objects

### figmin.createObject(type, params?)

Returns FigminObject immediately (UNINITIALIZED). Wait for `onStart()`.

**Requirements**: Master client only

### ObjectType.TEXT3D
```javascript
const obj = figmin.createObject(figmin.ObjectType.TEXT3D, {
    position: { x: 0, y: 1, z: 0 },
    rotation: figmin.getWindow().transform.rotation,  // Faces user (no flip needed)
    text: "Hello World",
    size: 0.08,                         // capital-letter height in meters
    font: "roboto",                     // see fonts list below
    material: "metallic",               // see materials list below
    depth: 0.01,                        // depth in meters
    bend: 0,                            // degrees (-360 to 360)
    frontColor: "#FFE6C6",
    sideColor: "#310E65",
    colliderType: figmin.ColliderType.BOX
});
```

**TEXT3D Sizing**:
Use `size` for readable text; it sets capital-letter height in meters. 
Use `targetSize` only to fit the entire text object into a maximum overall size. 
Do not pass both; `targetSize` takes precedence and can make long text tiny.

**Supported Fonts**: `"default"`, `"pixels"`, `"curved"`, `"classic"`, `"open sans"`, `"roboto"`, `"caveat"`

**Supported Materials**: `"standard"`, `"metallic"`, `"unlit"`, `"matcap"`, `"hologram"`, `"fire"`, `"hypercolor"`, `"hueshift"`, `"diamond"`, `"rainbow"`

### ObjectType.MODEL3D
```javascript
const obj = figmin.createObject(figmin.ObjectType.MODEL3D, {
    position: { x: 0, y: 0, z: 1 },
    rotation: { x: 0, y: 0, z: 0 },
    url: "https://example.com/model.glb",  // GLB only (not GLTF)
    targetSize: 0.5,
    colliderType: figmin.ColliderType.APPROXIMATE
    // transparent: false — optional; alpha-blended rendering (not recommended, cutout is default)
});

obj.onStart((o) => {
    // Access animations (only available after onStart)
    console.log(o.model3d.getAnimationNames());  // string[]
    o.model3d.play("Walk", true);                // name, loop
    o.model3d.setAnimationSpeed(1.5);
    o.model3d.stop();
});
```

### ObjectType.IMAGE
```javascript
const obj = figmin.createObject(figmin.ObjectType.IMAGE, {
    position: { x: 0, y: 1.5, z: 0 },
    url: "https://example.com/image.png",
    targetSize: 0.3
});
```

<!-- @topic: basics, sketches -->
### ObjectType.SKETCH
```javascript
const obj = figmin.createObject(figmin.ObjectType.SKETCH, {
    position: { x: 0, y: 1, z: 0 },
    brushStrokes: [stroke1, stroke2]  // Required — see SKETCH gotcha above
});
```

<!-- @topic: basics -->
### ObjectType.PORTAL
```javascript
const obj = figmin.createObject(figmin.ObjectType.PORTAL, {
    position: { x: 0, y: 0, z: 2 },
    targetSize: 1.0,
    scene: "https://share.figmin.com/asset/4807d1be-0464-11ed-8b86-0236d6272b39"
});
```

### ObjectType.AGENT
**MCP sessions only** — The Agent object carries a camera that lets you visually verify your own work. Create and manipulate objects first, then use the agent camera to check the results when needed.

**You do NOT need to create an agent camera at the start of every session.** Only create one when you want to see something you've built or modified. The scene may be empty when you start — there's nothing to look at until you've created something.

**Limitations**: Agent objects are **local-only** (not networked). They do not support `addComponent`, cannot be found via `findObjectByName`, have no colliders, and cannot be created with `scenePersistent`. They are automatically destroyed when the script/session ends.

```javascript
// After creating some objects, set up a camera to check your work
window.agentObj = figmin.createObject(figmin.ObjectType.AGENT, {
    position: figmin.getWindowSpawnPosition(0.5, "front"),
    rotation: figmin.getWindow().transform.rotation,
    fieldOfView: 60,
    backgroundColor: "#0F486C"
});

// Point the camera at a specific object you created, then capture
window.agentObj.agent.lookAtObject(myCreatedObject);
const img = await window.agentObj.agent.captureView(512, 80);
console.image(img.data, "image/" + img.format);
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `fieldOfView` | number | 50 | Starting field of view of the agent camera (degrees) |
| `backgroundColor` | string | "#0F486C" | Camera background color as hex |

<!-- @topic: apps -->
### ObjectType.APP
Creates a spatial app — a browser surface living in 3D space. Through the `app` component you can load HTML or URLs, access the app's console logs, and query ownership information.

This is particularly useful for AI agents running live sessions (e.g. through MCP). Instead of modifying their own app context, agents can create independent APP objects, load self-contained HTML apps into them, and monitor their output via console logs. Each spawned app runs in its own isolated context and persists independently, enabling the agent to build and deploy spatial apps into the scene rather than managing everything within a single script.

**Security**: In multiplayer sessions, only the master client (the player who owns the app) can perform state-changing operations on it. Cross-player modification is forbidden — see `isMasterClient` in the app component.

```javascript
const myApp = figmin.createObject(figmin.ObjectType.APP, {
    position: figmin.getWindowSpawnPosition(0.3, "right"),
    rotation: figmin.getWindow().transform.rotation
});

myApp.onStart(() => {
    // Load custom HTML
    myApp.app.loadHTML(`<html>
      <body style="background:#1a1a2e;color:#eee;">
        <h1>Hello from Figmin XR</h1>
      </body>
    </html>`);

    // Or load a URL
    myApp.app.loadURL("https://www.figmin.com");

    // Monitor logs
    myApp.app.showConsole(true);
    const logs = await myApp.app.getConsoleLogs(10);
    console.log(logs);
});
```

**Cross-app messaging**: Apps can communicate with each other using `sendMessage` and `onMessage`. Always send from your own app and target the other app by name.

```javascript
// Agent creates a child app with an echo bot
const child = figmin.createObject(figmin.ObjectType.APP, {
    position: figmin.getWindowSpawnPosition(0.3, "right"),
    rotation: figmin.getWindow().transform.rotation
});

child.onStart(() => {
    child.app.loadHTML(`<html>
      <head><meta name="figmin"></head>
      <body>
        <h1>Echo Bot</h1>
        <script>
          document.addEventListener('FigminReady', function() {
              var app = figmin.getAppObject();
              app.app.onMessage(function(senderName, data) {
                  // Reply back to whoever sent the message
                  app.app.sendMessage(
                      { action: 'echo', original: data },
                      figmin.MessageRecipient.ALL,
                      senderName
                  );
              });
          });
        </script>
      </body>
    </html>`);
});

// Agent listens for replies on its own app
const myApp = figmin.getAppObject();
myApp.app.onMessage((senderName, data) => {
    console.log("Reply from " + senderName + ": " + JSON.stringify(data));
});

// Agent sends a message from its own app, targeting the child
myApp.app.sendMessage({ action: "ping" }, figmin.MessageRecipient.ALL, child.name);
```

<!-- @topic: basics -->
### Common Parameters
| Parameter | Type | Description |
|-----------|------|-------------|
| `position` | {x,y,z} | Initial world-space position in meters |
| `rotation` | {x,y,z} | Initial Euler degrees |
| `targetSize` | number | Uniform scale (largest dimension = this) |
| `colliderType` | ColliderType | BOX, SPHERE, APPROXIMATE, EXACT, NONE |
| `scenePersistent` | boolean | Survives script end (default: false) |

---

## Object Lifecycle

Objects created via `createObject()` go through lifecycle states, but **use event callbacks** instead of checking state:
- `onAwake()` - Called when object is created (before loading)
- `onStart()` - Called when object is **READY** - use this before accessing properties/components
- `onError()` - Called if object fails to load
- `onDestroy()` - Called when object is destroyed

**No promise support**: These lifecycle events are **callback-only**. They do not return promises. Unlike raycast functions, lifecycle events are not guaranteed to fire (e.g. an object may silently fail to load, get stuck in LOADING, or be destroyed before reaching READY), so awaiting them could hang indefinitely.

```javascript
const obj = figmin.createObject(figmin.ObjectType.MODEL3D, params);
obj.onStart(() => {
    // Safe to access properties and add components here
    obj.addComponent(figmin.ComponentType.PHYSICS);
});
obj.onError(() => {
    console.error("Failed to load object");
});
```

---

## Components Reference

### Component: status
```javascript
obj.status.state           // ObjectState enum
obj.status.code            // Error code (if ERROR state)
obj.status.description     // Error description

obj.status.onStatusChanged((object) => { });
```

**ObjectState**: `UNINITIALIZED`, `LOADING`, `READY`, `DESTROYED`, `ERROR`

### Component: transform
```javascript
// World-space accessors (walk the parent hierarchy; for root objects, same as local)
obj.transform.position   // {x, y, z} world-space meters
obj.transform.rotation   // {x, y, z} world-space Euler degrees
obj.transform.rotationq  // {x, y, z, w} world-space quaternion
obj.transform.scale      // number (world scale — the value synced with C#)

// Local-space accessors (relative to parent; for root objects, same as world)
obj.transform.localPosition   // {x, y, z} local-space meters (Unity-standard: parent scale affects conversion)
obj.transform.localRotation   // {x, y, z} local-space Euler degrees
obj.transform.localRotationq  // {x, y, z, w} local-space quaternion
obj.transform.localScale      // number (scale relative to parent, derived from world scales)

// Read-only
obj.transform.size       // {x, y, z} world-space bounding-box (bounds × world scale)
obj.transform.bounds     // {x, y, z} raw geometry bounding-box, unaffected by scale
obj.transform.forward    // {x, y, z} world-space unit vector
obj.transform.up         // {x, y, z} world-space unit vector
obj.transform.right      // {x, y, z} world-space unit vector

// Events
obj.transform.onBoundsChanged((objectRef, bounds) => {
    // bounds: {x, y, z} - raw geometry bounding-box size in meters
    // objectRef.transform.size - world-space bounding-box (accounts for scale)
});

// Methods
obj.transform.lookAt(position, up?);
// Rotates this object so transform.forward points at a world-space target position.
// Works correctly for parented objects (automatically converts to local rotation).
// position: {x,y,z} - world-space target point (not a direction)
// up: {x,y,z} - optional world up direction, defaults to {x:0, y:1, z:0}

// Example: make an object face the player camera
const target = figmin.getCamera().transform.position;
obj.transform.lookAt(target);
```

### Component: collider
```javascript
obj.collider.type = figmin.ColliderType.SPHERE;

// Collision callback — onCollisionEnter is the ONLY collision event.
// There are no exit or trigger callbacks in the API.
obj.collider.onCollisionEnter((payload) => {
    console.log("Hit:", payload.objectName, "at", payload.point);
}, ["objectName", "point", "normal", "velocity"]);  // requestedData array

obj.collider.onCollisionEnter(null);  // unregister — collision events are only computed while a callback is registered

// requestedData options: "velocity", "impulse", "layer", "point", "normal", "object", "objectName"
// Request only the fields you need — fewer fields = better performance.
```

**Note**: `payload.object` is `null` when the hit object wasn't created by this script. Use `payload.objectName` with `findObjectByName()` to start tracking it; after that, `object` will be valid in future callbacks.

**ColliderType**: `NONE`, `BOX`, `SPHERE`, `APPROXIMATE`, `EXACT`
- Physics requires: BOX, SPHERE, or APPROXIMATE (not EXACT/NONE)

### Component: text3d
```javascript
obj.text3d.text = "New Text";
obj.text3d.font = "roboto";           // see fonts list in ObjectType.TEXT3D
obj.text3d.material = "metallic";     // see materials list in ObjectType.TEXT3D
obj.text3d.size = 0.1;                // capital letter height in meters
obj.text3d.depth = 0.03;              // extrusion thickness in meters 
obj.text3d.bend = 45;                 // arc angle in degrees
obj.text3d.frontColor = "#00FF00";    // front face color
obj.text3d.sideColor = "#005500";     // side/extrusion color
```

**Note**: Changing `text` rebuilds the geometry — the object cycles **READY → LOADING → READY** and `onBoundsChanged` fires again. Use that to react to the new size (see the dynamic text background pattern).

### Component: model3d
**Supported format**: GLB only. Animation data / Node access available only after `onStart`.

```javascript
// Properties (read-only)
obj.model3d.url                    // string - model URL
obj.model3d.transparent            // boolean - transparency mode

// Animation playback
obj.model3d.play(name?, loop?, speed?, normalizedStart?);  // Play animation
obj.model3d.stop();                                         // Stop animation
obj.model3d.setAnimationSpeed(speed);                       // Set playback speed
obj.model3d.getAnimationSpeed();                            // Returns number

// Animation info (call after onStart)
obj.model3d.getAnimationNames();              // Returns string[]
obj.model3d.hasAnimation(name);               // Returns boolean
obj.model3d.getAnimationDuration(name);       // Returns number (seconds)
obj.model3d.isLooping();                      // Returns boolean
obj.model3d.getCurrentAnimationName();        // Returns string | null
obj.model3d.getCurrentAnimationTime();        // Returns number (seconds)
obj.model3d.getCurrentAnimationNormalizedTime();  // Returns number (0..1)

// Node access (for animating parts of the model) 
const names = await obj.model3d.getNodeNames();  // Returns string[] of all node names
const node = obj.model3d.getNode("wheel-front-left");  // Returns ModelNode handle

// ModelNode — local-space transform (write-only, relative to parent in GLB hierarchy). Always make full assignments
node.localPosition = { x: 0, y: 0.2, z: 0 };       // meters
node.localRotation = { x: 0, y: 45, z: 0 };         // Euler degrees
node.localRotationq = { x: 0, y: 0.38, z: 0, w: 0.92 }; // quaternion
node.localScale = { x: 1.2, y: 1, z: 1 };           // per-axis (unlike FigminObject's uniform scale)
node.active = false;                                  // hide node and children
```

**Note**: Animation names are case-insensitive.

**Note**: Node names are **case-sensitive**. Use `getNodeNames()` to discover available names at runtime. `getNode()` is currently write-only — default bone values are not sent from C#.

### Component: image
```javascript
obj.image.url        // string (read-only, set at creation)
obj.image.material   // "unlit" | "lit" | "unlit_transp" | "lit_transp" (read-only, set at creation)
```

**Supported formats**: PNG, JPG, WEBP, GIF. Default material is `"unlit"` (unlit cutout — recommended); `lit` variants receive scene lighting, `_transp` variants use smooth alpha blending. Both properties are fixed at creation — to change them, destroy and recreate the object.

### Component: portal
Present only on PORTAL objects. Links to another published scene.
```javascript
obj.portal.scene       // string (read-only, set at creation) — target scene id
obj.portal.activate(); // Request the scene switch. WARNING: ends the current script/app context.
```

<!-- @topic: sketches -->

Figmin XR includes a procedural sketch system derived from Tilt Brush. The entire brush catalog (flat paints, 3D tubes, convex hulls, particles, audio-reactive variants) is available via the API. You can generate 3D geometry procedurally using shape generators (boxes, spheres, cylinders, etc.) and stroke manipulation functions.

**Sketch components are not limited to SKETCH objects** — you can add a sketch component to any object using `addComponent(figmin.ComponentType.SKETCH)`. This is useful for attaching procedural geometry to TEXT3D, MODEL3D, or other objects (e.g. adding a background panel behind text).

### Component: sketch
```javascript
// Set brush strokes (positions and rotations are in the sketch object's local space)
obj.sketch.setBrushStrokes([stroke1, stroke2]);

// Get brush strokes (async, returns strokes in the sketch object's local space)
obj.sketch.getBrushStrokes((strokes) => {
    // strokes: array of {id, c, s, pts}
});
```

<!-- @topic: physics -->
### Component: physics
**Requires**: Compatible collider (BOX, SPHERE, APPROXIMATE)

```javascript
obj.addComponent(figmin.ComponentType.PHYSICS, {
    mass: 1.0,
    drag: 0.1,
    angularDrag: 0.05,
    gravityStrength: 1.0,
    kinematic: false,
    sleeping: false,
    freezePosition: false,                                // freezes translation on ALL axes (single boolean)
    freezeRotationAxes: { x: false, y: false, z: false }, // freeze rotation per axis
    collisionDetectionMode: figmin.physics.CollisionDetectionMode.CONTINUOUS  // default
    // velocity, angularVelocity, maxAngularVelocity (default 14), motorTorque also accepted here
});

// Properties
obj.physics.mass = 2.0;
obj.physics.velocity = { x: 0, y: 5, z: 0 };
obj.physics.angularVelocity = { x: 0, y: 1, z: 0 };
obj.physics.maxAngularVelocity = 30;                // rad/s (default: 14)
obj.physics.gravityStrength = 0.5;
obj.physics.kinematic = true;
obj.physics.sleeping = false;
obj.physics.motorTorque = { x: 0, y: 2, z: 0 };     // constant angular acceleration, local space (rad/s²)
obj.physics.freezePosition = true;                  // boolean — all axes
obj.physics.freezeRotationAxes = { x: false, y: true, z: false };

// Force field (create gravity wells, etc.)
obj.physics.forceFieldType = figmin.physics.ForceFieldType.SPHERE;
obj.physics.forceFieldAcceleration = 10;
obj.physics.forceFieldReach = 2.0;
obj.physics.forceFieldRepulsive = false;
obj.physics.forceFieldForceType = figmin.physics.ForceFieldForceType.SQUARED;  // SQUARED (falls off) or CONSTANT
obj.physics.forceFieldDirection = figmin.physics.ForceFieldDirection.UP;       // for FLAT fields
obj.physics.forceFieldDrag = 0;          // extra drag applied to bodies inside the field
obj.physics.forceFieldAngularDrag = 0;   // extra angular drag inside the field

// Apply forces (local space)
obj.physics.applyForce({ x: 0, y: 100, z: 0 });      // Newtons
obj.physics.applyImpulse({ x: 0, y: 10, z: 0 });     // N·s
obj.physics.applyTorque({ x: 0, y: 5, z: 0 });       // N·m
obj.physics.applyAcceleration({ x: 0, y: 9.8, z: 0 }); // m/s²
obj.physics.applyAngularAcceleration({ x: 0, y: 3, z: 0 }); // rad/s²
```

**Notes**: All physics vectors (force, impulse, acceleration, torque, motorTorque) are in the object's **local** space. Mass scales linearly with `transform.scale` (deliberate simplification). Vector-like properties (`velocity`, `angularVelocity`, `motorTorque`, `freezeRotationAxes`) follow the assign-a-new-object rule — never mutate in place.

### Component: physicsMaterial
```javascript
obj.addComponent(figmin.ComponentType.PHYSICS_MATERIAL, {
    bounciness: 0.8,
    staticFriction: 0.3,
    dynamicFriction: 0.3,
    bounceCombine: figmin.physics.Combine.AVERAGE,
    frictionCombine: figmin.physics.Combine.AVERAGE
});

// Properties
obj.physicsMaterial.bounciness = 0.4;
obj.physicsMaterial.staticFriction = 0.6;
obj.physicsMaterial.dynamicFriction = 0.6;
obj.physicsMaterial.bounceCombine = figmin.physics.Combine.MAXIMUM;
obj.physicsMaterial.frictionCombine = figmin.physics.Combine.MINIMUM;
```

**Important**: Physics requires a built collider, so always add it inside `onStart` — not immediately after `createObject`:

```javascript
const ball = figmin.createObject(figmin.ObjectType.MODEL3D, {
    url: "https://example.com/ball.glb",
    position: spawnPos,
    targetSize: 0.1,
    colliderType: figmin.ColliderType.SPHERE
});

ball.onStart((b) => {
    b.addComponent(figmin.ComponentType.PHYSICS, {
        mass: 0.5,
        gravityStrength: 1.0
    });
    b.physics.velocity = { x: 0, y: 5, z: 3 };

    b.collider.onCollisionEnter((hit) => {
        console.log("Hit:", hit.objectName);
    });
});
```

<!-- @topic: lighting -->
### Component: light
```javascript
obj.addComponent(figmin.ComponentType.LIGHT, {
    type: figmin.lights.LightType.SPOT,
    color: "#FFFFFF",
    range: 5.0,
    spotlightAngle: 30,
    rotation: { x: -45, y: 0, z: 0 },
    shadowType: figmin.lights.ShadowType.SOFT,
    shadowNearPlane: 0.1,
    effect: figmin.lights.LightEffect.NONE,
    projectionPattern: figmin.lights.LightProjection.NONE
});

// Runtime changes
obj.light.color = "#FF0000";
obj.light.range = 10;
obj.light.type = figmin.lights.LightType.POINT;
```

**LightType**: `POINT` (default), `SPOT`, `DIRECTIONAL`
**ShadowType**: `NONE`, `HARD`, `SOFT`
**LightEffect**: `NONE`, `FLICKER`, `AUDIO_REACTIVE`, `PULSE`, `RAINBOW`
**LightProjection**: `NONE`, `FLASHLIGHT1`, `SPOTLIGHT1`, `SPOTLIGHT2`

**Defaults**: color `#d47e13` (warm orange — set explicitly if you want white), range `4.0` m (POINT/SPOT only), shadowType `NONE`, shadowNearPlane `0.15`. For `rotation`, assign a new `{x,y,z}` object (never mutate in place).

<!-- @topic: keyframes -->
### Component: motion
```javascript
obj.addComponent(figmin.ComponentType.MOTION, {
    speed: 1.0,
    endDelay: 0,
    randomizeDelay: false,
    endBehavior: figmin.MotionEndBehavior.LOOP
});

// Set keyframes
obj.motion.setKeyframes([
    { p: {x:0, y:0, z:0}, r: {x:0, y:0, z:0}, t: 0 },
    { p: {x:0, y:1, z:0}, r: {x:0, y:90, z:0}, t: 1 },
    { p: {x:0, y:0, z:0}, r: {x:0, y:180, z:0}, t: 2 }
], false);  // fixedPlacement: false = keyframes relative to the object; true = world space

// Get keyframes (rotations always come back as quaternions)
obj.motion.getKeyframes((data) => {
    // data: { frames: [{p:{x,y,z}, r:{x,y,z,w}, t:number}, ...], fixedPlacement: boolean }
});
```

**MotionEndBehavior**: `LOOP`, `PING_PONG`

<!-- @topic: audio -->
### Component: audio
Adds spatial audio with support for multiple simultaneous clips. Each named clip has independent playback and spatial settings backed by its own AudioSource.
```javascript
obj.addComponent(figmin.ComponentType.AUDIO);

// Load clips (supported formats: .ogg, .wav, .mp3). Callback is optional
var engine = obj.audio.loadClip("engine", "https://example.com/engine.ogg", function(success, duration) {
    if (success) {
        console.log("Ready! Duration: " + duration + "s");
    }
});

// Per-clip properties (read-write)
engine.spatialize = true;   // 3D positional audio (default: false = flat 2D)
engine.dopplerLevel = 1.0;  // 0–5, pitch shift on moving objects (default: 0 = off; only applies when spatialize is true)
engine.volume = 0.7;        // 0–1 (default: 0.7)
engine.pitch = 1.0;         // 0.1–3.0 (default: 1.0)
engine.loop = true;         // default: true
engine.playing = false;     // read-write playback state (default: true — clips start playing when loaded; set false before load to prevent autoplay)

// Playback is a persistent and networked state (default: true)
engine.play();
engine.stop();

// Read-only
engine.name;                // "engine"
engine.url;                 // the loaded URL
engine.loaded;              // true after load completes
engine.duration;            // clip length in seconds (available after load)

// Component-level
obj.audio.getClip("horn");  // AudioClip or null
obj.audio.clipNames;        // ["engine", "horn"]
obj.audio.stopAll();
```

**loadClip callback**: `loadClip(name, url, callback?)` — optional callback fires when the clip finishes loading or fails: `(success: boolean, duration: number)`. Properties like volume, pitch, loop can be set before the clip finishes downloading — they are applied when ready.

**Spatialize**: When `true`, audio is 3D — volume and panning change as the listener moves relative to the object. When `false`, audio is 2D (same volume everywhere). Use `false` for background music or UI sounds.

<!-- @topic: special_effects -->
### Component: trail
Adds dynamic brush trails to an object, supporting multiple simultaneous trail emitters. Each named trail uses a brush from the `figmin.brushes` catalog to generate stroke geometry that follows the object as it moves. Use for smoke, fire, light trails, and motion effects.
```javascript
obj.addComponent(figmin.ComponentType.TRAIL);  // takes no creation params — configure emitters after

// Load trails using brushes from figmin.brushes
var flame = obj.trail.loadTrail("flame", figmin.brushes.Light2);
var smoke = obj.trail.loadTrail("smoke", figmin.brushes.Smoke);

// Per-trail properties (read-write)
flame.color = "#AAAAAA";                        // hex #RRGGBB (default: "#AAAAAA")
flame.size = 0.01;                              // stroke width in parent-local units (default: 0.01)
flame.lifetime = 0.5;                           // seconds before points fade (default: 0.5)
flame.maxLength = 0.05;                         // max trail length in parent-local units, negative = unlimited (default: -1)
flame.baseVelocity = {x: 0, y: 0.25, z: 0};    // drift in parent-local units, e.g. fire rising (default: {x:0, y:0.015, z:0})
flame.localPosition = {x: 0, y: 0.1, z: 0};    // offset relative to parent (default: {x:0, y:0, z:0})
flame.localRotation = {x: 0, y: 0, z: 0};      // Euler degrees relative to parent (default: {x:0, y:0, z:0})
flame.localRotationq = {x: 0, y: 0, z: 0, w: 1}; // quaternion alternative (default: identity)
flame.reverseDirection = false;                 // render trail backwards (default: false)
flame.active = true;                            // toggle emission (default: true). Existing points fade smoothly when disabled.

// Read-only
flame.name;                     // "flame"
flame.brush;                    // brush id string

// Component-level
obj.trail.getTrail("smoke");    // BrushTrail or null
obj.trail.removeTrail("smoke"); // remove by name
obj.trail.trailNames;           // ["flame", "smoke"]
```

**Brush selection**: Pass a brush object from `figmin.brushes` (e.g., `figmin.brushes.Smoke`, `figmin.brushes.Light2`, `figmin.brushes.Fire`). The brush is validated against the catalog — invalid brushes log an error. Good trail brushes include `Smoke`, `SmokeTiny`, `Light2`, `Fire`, `Embers`, `NeonPulse`, and `SoftHighlighter`.

**Size and scale**: `size`, `baseVelocity`, and `maxLength` are in the parent's local space, not world meters. They are automatically multiplied by the parent's current scale at render time. To achieve a desired world size: `trail.size = desiredWorldSize / obj.transform.scale`. Subsequent scale changes are handled automatically.

**maxLength vs lifetime**: Use `lifetime` to control how long trail points persist (fade over time). Use `maxLength` to spatially cap the trail (e.g., flames that should not stretch when moving fast). Set `maxLength` to a negative value to disable the spatial cap.

<!-- @topic: interaction -->
### Component: bodypart
Retrieved via `figmin.getPlayerBodypart()`, not addable.

```javascript
const player = figmin.getLocalPlayer();
const hand = figmin.getPlayerBodypart(player, figmin.BodypartType.RIGHT_HAND);

hand.onStart((handObj) => {
    // Read-only properties
    handObj.bodypart.tracked         // boolean
    handObj.bodypart.playerId        // number
    handObj.bodypart.bodypartType    // BodypartType
    handObj.bodypart.inputType       // InputType

    // Ray origin (for pointing) — all read-only
    handObj.bodypart.rayPosition     // {x,y,z} world space
    handObj.bodypart.rayRotation     // {x,y,z} Euler
    handObj.bodypart.rayRotationq    // {x,y,z,w} quaternion
    handObj.bodypart.rayForward      // {x,y,z} direction
    handObj.bodypart.rayUp           // {x,y,z} up direction
    handObj.bodypart.rayLocalPosition   // ray origin offset relative to the bodypart's transform
    handObj.bodypart.rayLocalRotation   // (also rayLocalRotationq)
    handObj.bodypart.pointerDistance // recommended ray-to-cursor distance — place pointers/tools
                                     // at rayPosition + rayForward * pointerDistance

    // Callbacks
    handObj.bodypart.onTrackingChanged((obj, isTracked) => { });
    handObj.bodypart.onGrabChanged((obj, isGrab, targetName, targetObj) => { });
    // targetObj is null if the grabbed object isn't tracked by this script —
    // use findObjectByName(targetName) to start tracking it.
});
```

**BodypartType**: `HEAD`, `LEFT_HAND`, `RIGHT_HAND`
**InputType**: `HAND_TRACKING`, `MOTION_CONTROLLER`, `TOUCH_SCREEN`

<!-- @topic: basics -->
### Component: agent
**MCP sessions only** — Present only on objects of type `figmin.ObjectType.AGENT`. Provides camera capture capabilities so you can visually verify your work during an MCP session. Use it to check the results of objects you've created or modified — not as a first step before doing anything.

Access via the `agent` property on a FigminObject.

```javascript
// Properties (read-write)
obj.agent.fieldOfView        // number - camera FOV in degrees
obj.agent.backgroundColor    // string - camera background color hex (e.g. "#0F486C")

// Methods

// captureView(resolution?, quality?, callback?)
// Captures a screenshot from the agent's camera.
// resolution: number - size in pixels of the square capture (default: 512)
// quality: number - 0 = PNG (lossless), 1-100 = JPG at that quality (default: 0)
// Returns Promise<{format, data}> if no callback; data is base64-encoded.
const img = await obj.agent.captureView(1024, 80);
console.image(img.data, "image/" + img.format);  // Send image back through MCP

// focus(rect)
// Reframe the camera to focus on a normalized screen-space rectangle.
// Rotates the agent so the rect center becomes the view center,
// then narrows fieldOfView so the rect roughly fills the square capture.
// Camera position does NOT change.
// rect: {x, y, w?, h?} - normalized coords, (0,0)=top-left, (1,1)=bottom-right
obj.agent.focus({ x: 0.25, y: 0.25, w: 0.5, h: 0.5 });

// lookAtObject(figminObject)
// Rotate and zoom the camera so a target object fits maximally in view.
// Adjusts fieldOfView so the object's bounding box fills the square capture.
// Camera position does NOT change.
obj.agent.lookAtObject(someObject);
```

<!-- @topic: apps -->
### Component: app
Present on objects of type `figmin.ObjectType.APP`, and the object returned by `figmin.getAppObject()`.

Provides control over a spatial app instance — a browser surface living in 3D space. This is particularly useful for AI agents running live sessions (e.g. through MCP). Instead of modifying their own app context, agents can create independent APP objects, load self-contained HTML apps into them, and monitor their output via console logs. Each spawned app runs in its own isolated context and persists independently, enabling the agent to build and deploy spatial apps into the scene rather than managing everything within a single script. This component also enables inter-app communication as well as communication between different multiplayer instances of the same app.

**Security**: Most operations require `isMasterClient` to be `true` except for messaging. In multiplayer sessions the player who created the app is the master.

Access via the `app` property on a FigminObject.

```javascript
// Properties (read-only)
obj.app.isMasterClient       // boolean - whether this app belongs to the master client

// showConsole(doShow)
// Shows or hides this app's log console.
// The console MUST be visible before calling getConsoleLogs() or clearConsoleLogs().
// Requires isMasterClient to be true.
obj.app.showConsole(true);

// getConsoleLogs(logCount?, callback?)
// Retrieves the latest log entries from this app's console.
// Returns Promise<string|null> if no callback. Resolves to null if not master client.
// Requires isMasterClient to be true and console to be visible.
obj.app.showConsole(true);
const logs = await obj.app.getConsoleLogs(10);
console.log(logs);

// clearConsoleLogs()
// Clears all log entries from this app's console.
// Requires isMasterClient to be true and console to be visible.
obj.app.clearConsoleLogs();

// reload()
// Reloads this app, refreshing its current page content.
// Requires isMasterClient to be true.
obj.app.reload();

// loadHTML(html)
// Loads an HTML string into this app, replacing its current content.
// Base64-encoded internally — double quotes and special chars are supported.
// Can include inline <script> tags and CSS.
// Requires isMasterClient to be true.
// NOTE: Literal unicode characters (e.g. emoji) are not preserved by the
// base64 encoding. Use HTML entities instead (e.g. &#x1F3AE; for 🎮).
obj.app.loadHTML(`<html>
  <body style="background:#1a1a2e;color:#eee;">
    <h1>Hello World</h1>
    <script>console.log("App loaded!");</script>
  </body>
</html>`);

// loadURL(url)
// Navigates this app to a URL, replacing its current content.
// Requires isMasterClient to be true.
obj.app.loadURL("https://www.figmin.com");

// sendMessage(data, recipient, targetAppName?)
// Sends a message to this app's own instances or to a different app by name.
// Enables inter-app communication and multiplayer master/non-master coordination.
// In multiplayer, non-master clients can send messages to the master via
// figmin.MessageRecipient.MASTER, allowing other players to communicate
// their intent (button presses, answers, votes, etc.) so the master can
// act on their behalf.
// data must be a JSON-serializable object (not a plain string or number) — it is
// serialized via JSON.stringify() and reconstructed on the receiving end.
// ANY client can send messages — isMasterClient is NOT required.
// Do not call this on other apps — use your own sendMessage with their name instead.

// A non-master client sends a message to the master to communicate user intent
appObj.app.sendMessage({ action: "answer", choice: "B" }, figmin.MessageRecipient.MASTER);

// A message is sent to all multiplayer instances of this app
appObj.app.sendMessage({ action: "reset" }, figmin.MessageRecipient.ALL);

// Send a message to a different app (e.g. a scoreboard) in a multiplayer session all instances would receive this message
appObj.app.sendMessage({ action: "setScore", points: 100 }, figmin.MessageRecipient.ALL, scoreboardObj.name);

// Send a message to a specific player (e.g. a late joiner)
appObj.app.sendMessage({ action: "welcome", state: gameState }, figmin.MessageRecipient.player(playerId));

// figmin.MessageRecipient values:
//   MASTER            — only the master client receives it
//   OTHERS            — everyone except the sender
//   ALL               — everyone including the sender
//   player(playerId)  — only the specified player receives it

// WARNING: sendMessage is intended for small, infrequent messages
// (events, button presses, state changes) — not real-time streaming
// or large payloads. Sending too much data or too often will result
// in disconnection. Keep payloads compact and avoid sending every frame.

// onMessage(callback)
// Registers a callback to receive messages sent to this app.
// Messages can originate from other instances of this app in multiplayer,
// or from any other app in the scene. The senderName parameter identifies
// which app sent the message, allowing you to differentiate between your own
// app's instances and messages from other apps.
// Only one handler can be active at a time — calling onMessage again replaces
// the previous handler.
// ANY client can listen for messages — isMasterClient is NOT required.
// Do not call this on other apps.
obj.app.onMessage((senderName, data) => {
    if (data.action === "answer") {
        console.log(senderName + " answered: " + data.choice);
    }
});

// onPeerReady(callback)
// Registers a callback that fires when a non-master peer's app instance
// has finished loading and is ready to communicate.
// This is useful for detecting late joiners, sending welcome data,
// assigning teams, or updating player lists. The callback fires once
// per peer at the moment their app instance becomes ready — not when
// they join the room.
// Only one handler can be active at a time — calling onPeerReady again
// replaces the previous handler.
// Only the master client receives this event — isMasterClient is required.
obj.app.onPeerReady((playerId) => {
    var peerName = figmin.getPlayerById(playerId).displayName;
    console.log(peerName + " is ready");
    // Send current game state to the new peer
    obj.app.sendMessage({ action: "welcome", state: gameState }, figmin.MessageRecipient.player(playerId));
});
```

---

<!-- @topic: basics -->
## Core Functions

### Update Loop
```javascript
figmin.setUpdateFunction((deltaTime) => {
    // Called every frame (synchronized with XR render loop)
    // deltaTime is seconds since last frame
    obj.transform.rotation = {
        x: obj.transform.rotation.x,
        y: obj.transform.rotation.y + 90 * deltaTime,
        z: obj.transform.rotation.z
    };
});
```

### Tracked vs Untracked Objects

**Tracked objects** are created by your script via `createObject()` - you already have a reference.

**Untracked objects** exist in the scene but weren't created by your script (placed by hand, created by other apps, or pre-existing). Use `findObjectByName()` to get a reference:

```javascript
// Find an untracked object in the scene
const existingCube = figmin.findObjectByName("MyCube");
existingCube.onStart(() => {
    // Now you can manipulate it
    existingCube.transform.position = { x: 0, y: 2, z: 0 };
});
existingCube.onError(() => {
    console.log("Object not found in scene");
});
```

This is essential for interacting with objects the user placed manually or that exist from other sources.

The parent hierarchy is automatically discovered.
To discover an object's entire hierarchy, pass `findHierarchy: true`. The tree is resolved in both directions — upward through parents and downward through children — so the complete hierarchy is reconstructed. This operation is recursive and make take a few frames to complete:

```javascript
const root = figmin.findObjectByName("MyGroup", true);
root.onStart(() => {
    // root.parent and all descendants are now tracked and linked
});
```

**Important**: You can't destroy foreign objects, calling destroy on a found object will only make it untracked for your script.

### Scene Discovery & App Encapsulation

There is no `listAllObjects()` or `getSceneGraph()`. This is by design — Figmin XR is a shared spatial environment where multiple apps coexist, each owning its own objects. Exposing a global object list would break app encapsulation. If you need to find other objects in the scene, you can use `findObjectByName()` when you know a specific object's name, or use raycasts and spherecasts (see the `physics` help topic) to probe the space around you.

### Window & Positioning
```javascript
const window = figmin.getWindow();       // Read-only transform
const app = figmin.getAppObject();          // Mutable (spatial app only)
const camera = figmin.getCamera();       // Player camera

// Spawn position relative to window
const pos = figmin.getWindowSpawnPosition(0.3, "right");
// directions: "right", "left", "up", "down", "front", "back"
```

### Scene
```javascript
const scene = figmin.getScene();
scene.mainLightColor = "#FFFFFF";
scene.mainLightRotation = { x: 50, y: -30, z: 0 };
scene.secondaryLightColor = "#8888FF";
scene.ambientLightColor = "#333333";
scene.floorVisibility = true;
scene.boundarySize = { x: 10, y: 5, z: 10 }; // or null to disable
scene.shadowMode = figmin.lights.SceneShadowMode.FORCE_ON;

// getScene() returns immediately with defaults; real values arrive from the runtime.
// Use scene.status / scene.onStatusChanged(callback) to know when it's READY.
scene.onStatusChanged((sceneObj) => { });
```

### Session Info
```javascript
figmin.isSpatialApp();     // true if running as spatial app
figmin.requireSpatialApp(); // Prompt deploy if not spatial
figmin.isMasterClient();   // true if this client has authority
figmin.isMultiplayer();    // true if in multiplayer session
figmin.getVersion();       // Runtime version string
figmin.API_VERSION;        // API version
```

### State Persistence
HTML elements with `figmin-persist-state` attribute are saved:
```html
<input type="text" figmin-persist-state="myKey" value="default">
```
```javascript
figmin.saveState("optionalCustomData");
figmin.loadState((customData) => {
    console.log("Loaded:", customData);
});
```

<!-- @topic: multiplayer -->
### Multiplayer UI Sync
HTML elements with the `figmin-sync` attribute are automatically synchronized across all peers in a multiplayer session. This lets you write single-player UI logic and get multiplayer behavior for free — no `sendMessage` or `isMasterClient` checks needed.

**Important**: The early-out pattern (`if (!figmin.isMasterClient()) return;`) is still recommended for app logic that creates or modifies objects. `figmin-sync` handles UI synchronization, but non-master clients should avoid running game logic that would result in silent no-ops and wasted CPU cycles.

**How it works**: When a non-master interacts with a synced element, the interaction is intercepted and relayed to the master. The master processes it (handlers fire, state updates), and resulting changes are broadcast back to all non-masters automatically. In single-player sessions, `figmin-sync` has zero overhead — interceptors are not installed.

**Supported elements**:
- `<button>` — click events relayed to master
- `<input type="checkbox">` — checked state synced on change
- `<input type="radio">` — selection synced on change
- `<select>` — selected value synced on change
- `<input type="range">` — value synced on change (when released)
- Text elements (`<span>`, `<div>`, `<p>`, etc.) — `textContent` observed on master and broadcast to non-masters

```html
<!-- Tag elements with a unique sync key -->
<button figmin-sync="likeBtn">Like</button>
<input type="checkbox" figmin-sync="musicToggle">
<select figmin-sync="teamPick">
    <option value="red">Red</option>
    <option value="blue">Blue</option>
</select>
<span figmin-sync="score">0</span>
```

**Important notes**:
- Each `figmin-sync` value must be unique within the app.
- Only user interactions are synced. Programmatic changes (e.g. `element.value = 5`) do not trigger sync — this is by design so the developer retains full control.
- Text elements are synced via MutationObserver on the master. Only elements present at `FigminReady` are observed — dynamically added text elements are not picked up.
- Text label broadcasts are throttled (one message per second max) to stay within bandwidth limits.
- `figmin-sync` and `figmin-persist-state` are independent systems. Use both on the same element if you want multiplayer sync AND cross-session persistence.

<!-- @topic: basics -->
### Display Modes
```javascript
// App mode
figmin.setAppMode(figmin.AppMode.DEFAULT);     // Normal window
figmin.setAppMode(figmin.AppMode.MINIMIZED);   // Icon only
figmin.setAppMode(figmin.AppMode.OBJECT);      // 3D billboard
figmin.setAppMode(figmin.AppMode.HUD);         // Head-locked
figmin.setAppMode(figmin.AppMode.FULLSCREEN);  // Full FOV

// Transparent background (preferred for HUD/FULLSCREEN)
figmin.setTransparentBackground(true, true);  // transparent, raysPassthrough
// Requires: <meta name="transparent" content="true"> in HTML head

// 3D stereo rendering
figmin.setRenderMode(figmin.RenderMode.STANDARD);
figmin.setRenderMode(figmin.RenderMode.SBS_3D);   // Side-by-side
figmin.setRenderMode(figmin.RenderMode.OE_3D);    // Over-under

// Object mode bounds
figmin.setObjectModeBounds({ x: 0.5, y: 0.5, z: 0.5 });
```

### Advanced Rendering (custom 3D with three.js)
Spatial apps can render their **own 3D graphics with web technologies** — custom shaders and materials, particle systems, post-processing, complex procedural geometry — anything the native object API can't express. The app renders in head-tracked stereo (one image per eye), so the content has genuine depth and shifts perspective as the viewer moves, like a real object.

Figmin provides a **premade three.js library** that does the stereo and head-tracking math for you, with three modes:
- **Object mode** — a single grabbable, inspectable 3D prop rendered by your code
- **Display mode** — a head-tracked stereo display for 3D apps and games (you author the camera; head movement adds depth within your shot)
- **Portal mode** — the app window becomes a one-to-one opening into your three.js scene, anchored in the room at true scale (a *rendering* mode — not the PORTAL object type)

Prefer native Figmin objects when they suffice — custom rendering is more expensive (the scene draws twice per frame in stereo). When you need it, get the library (`figmin-three-1.0.3.js`) and downloadable working examples for each mode from the **Advanced Rendering** section of the official docs: https://www.figmin.com/api

### Utility
```javascript
figmin.print("Message");                        // Toast message
figmin.showNavigationArrow(startPos, endPos);   // Animated arrow
figmin.debugDraw([p1, p2, p3], "#FF0000");      // Debug line (one frame)
figmin.showLogConsole();                        // Show dev console

// Offer downloadable content (3D models, images, spatial apps): opens an import
// dialog so the USER deploys the asset into their scene. The standard way for
// AI agents to deliver files.
figmin.downloadURL(url, title);

// Build a {x,y,z,w} quaternion looking along `forward` (up defaults to {x:0,y:1,z:0}).
// Pairs with transform.rotationq.
figmin.lookRotationq(forward, up?);

figmin.getObjectModeBounds();                   // {x,y,z} or null — getter for setObjectModeBounds
```

---

<!-- @topic: interaction -->
## Input System

### Enable Input Connection
```javascript
figmin.enableInputConnect(
    "Game Controller",
    [
        figmin.InputCode.A,
        figmin.InputCode.B,
        figmin.InputCode.SELECT_LEFT,
        figmin.InputCode.SELECT_RIGHT,
        figmin.InputCode.AXIS_X_LEFT,
        figmin.InputCode.AXIS_Y_LEFT
    ],
    (connected, playerId) => {
        console.log(connected ? "Connected" : "Disconnected", playerId);
    },
    {
        targetObject: optionalObj,       // Show connect icon on this object instead of window
        grabbableObjects: [obj1, obj2]   // Only these objects are grabbable while connected
    }
);

figmin.disableInputConnect();   // Remove connect icon
figmin.disconnectFromInput();   // Disconnect active session
```

### Read Input (in update loop)
```javascript
figmin.setUpdateFunction((dt) => {
    // Buttons (0 or 1)
    if (figmin.getInputDown(figmin.InputCode.A)) { /* just pressed */ }
    if (figmin.getInputUp(figmin.InputCode.B)) { /* just released */ }

    // Analog (0 to 1)
    const trigger = figmin.getInput(figmin.InputCode.SELECT_RIGHT);

    // Axes (-1 to 1 for thumbsticks)
    const axisX = figmin.getInput(figmin.InputCode.AXIS_X_LEFT);
    const axisY = figmin.getInput(figmin.InputCode.AXIS_Y_LEFT);
});
```

### InputCode Enum
| Name | Description |
|------|-------------|
| `A`, `B`, `X`, `Y` | Face buttons |
| `GRIP_LEFT`, `GRIP_RIGHT` | Grip buttons |
| `SELECT_LEFT`, `SELECT_RIGHT` | Triggers |
| `AXIS_X_LEFT`, `AXIS_Y_LEFT` | Left stick |
| `AXIS_X_RIGHT`, `AXIS_Y_RIGHT` | Right stick |
| `THUMB_LEFT`, `THUMB_RIGHT` | Stick press |

---

<!-- @topic: physics -->
## Raycasting

All four raycast functions support **callbacks** and **promises**. If a callback is provided it behaves as before; if omitted, a Promise is returned so you can `await` the result.

### Basic Raycast
```javascript
// Callback style
figmin.raycast({
    origin: { x: 0, y: 1, z: 0 },
    direction: { x: 0, y: 0, z: 1 },
    maxDistance: 10,
    layers: [figmin.Layers.SURFACE, figmin.Layers.PHYSICS_OBJ],
    lookupObject: true  // Include object reference in result
}, (hit) => {
    if (!hit) return;
    // hit: { point, normal, distance, layer, objectName?, object? }
});

// Promise style (recommended for MCP)
const hit = await figmin.raycast({
    origin: { x: 0, y: 1, z: 0 },
    direction: { x: 0, y: 0, z: 1 },
    maxDistance: 10,
    lookupObject: true
});
```

**Note**: Setting `lookupObject: true` is slower because it retrieves the full object reference. Only enable it when you need to access the actual FigminObject that was hit.

### All Hits
```javascript
const hits = await figmin.raycastAll(params);
// hits: array or null
```

### Sphere Cast
```javascript
const hit = await figmin.raycastSphere({
    origin: { x: 0, y: 1, z: 0 },
    direction: { x: 0, y: 0, z: 1 },
    radius: 0.1,
    maxDistance: 10
});

const hits = await figmin.raycastSphereAll(params);
```

**Layers**: `SURFACE`, `PHYSICS_OBJ`, `STATIC_OBJ`, `UI`

---

<!-- @topic: interaction -->
## Players API

### Get Players
```javascript
const local = figmin.getLocalPlayer();     // PlayerInfo: { id, displayName }
const all = figmin.getPlayers();           // PlayerInfo[]
const player = figmin.getPlayerById(123);  // PlayerInfo or null
const local = figmin.getMasterPlayer();    // PlayerInfo of the master of this app

figmin.onPlayerConnected((player) => { });
figmin.onPlayerDisconnected((player) => { });
```

### Get Body Parts
```javascript
const hand = figmin.getPlayerBodypart(player, figmin.BodypartType.RIGHT_HAND);
hand.onStart((handObj) => {
    // Use handObj.bodypart for tracking data
});
```

---

<!-- @topic: sketches -->
## Sketch Functions

All sketch positions and rotations are in the **sketch object's local space**.

### Create Strokes

**Note**: Use `point()` + `stroke()` for freeform paths only. For regular geometry use the shape generators below to avoid stroke simplification artifacts.

```javascript
// Create control point (local space of the sketch object)
const pt = figmin.sketch.point(
    { x: 0, y: 0, z: 0 },      // position (local space)
    { x: 0, y: 0, z: 0 },      // rotation (optional, local space)
    1.0                         // pressure 0-1 (optional)
);

// Create stroke
const stroke = figmin.sketch.stroke(
    figmin.brushes.MetallicTube,  // brush object or ID
    "#FF5500",                     // color
    0.02,                          // max size (meters)
    [pt1, pt2, pt3]               // control points (local space)
);

// Apply to sketch
obj.sketch.setBrushStrokes([stroke]);
```

### Stroke Manipulation
```javascript
// Clone (deep copy)
const copy = figmin.sketch.cloneStroke(stroke);

// Transform (mutates in place, operates in local space)
figmin.sketch.translateStroke(stroke, { x: 0, y: 0.5, z: 0 });
figmin.sketch.rotateStroke(stroke, { x: 0, y: 90, z: 0 }, pivot);
figmin.sketch.scaleStroke(stroke, 1.5, pivot);  // uniform
figmin.sketch.scaleStroke(stroke, { x: 1, y: 2, z: 1 }, pivot);  // non-uniform

// Get center (local space)
const center = figmin.sketch.calculateStrokeCenter(stroke);
```

### Shape Generators
```javascript
// All return stroke objects centered at the local-space origin
figmin.sketch.boxStroke(brush, color, strokeSize, width, height, depth);
figmin.sketch.rectangleStroke(brush, color, strokeSize, width, height);
figmin.sketch.roundedRectangleStroke(brush, color, strokeSize, width, height, roundness);
figmin.sketch.circleStroke(brush, color, strokeSize, radius);
figmin.sketch.sphereStroke(brush, color, strokeSize, radius);
figmin.sketch.cylinderStroke(brush, color, strokeSize, radiusTop, radiusBottom, height, loops);
figmin.sketch.coneStroke(brush, color, strokeSize, radius, height, loops);
figmin.sketch.capsuleStroke(brush, color, strokeSize, radius, length, loops);
figmin.sketch.roundedBoxStroke(brush, color, size, width, height, depth, cornerXY, cornerZ);
```

**Note on `loops` parameter**: The `loops` param in `cylinderStroke`, `coneStroke`, and `capsuleStroke` controls the number of full spiral turns from top to bottom. Defaults to `1` (one full loop). Use values > 1 to create springs, coils, or spiral shapes. Can be fractional (e.g., `2.5` for two and a half turns).

### Animating Sketches

**Important**: Do NOT modify `brushStrokes` repeatedly for animation purposes - this is expensive and not meant for per-frame updates.

Instead, use these approaches:

**Moving/rotating sketch geometry**: Manipulate the sketch object's `transform` (position, rotation, scale) rather than regenerating strokes.

```javascript
// Good: animate via transform
sketch.transform.position = { x: newX, y: newY, z: newZ };
sketch.transform.rotation = { x: 0, y: angle, z: 0 };
```

**Frame-by-frame animation**: Create multiple sketch objects (one per frame) and cycle their `active` state.

```javascript
// Create frames once at init
const frames = [];
const frameCount = 4;  // example: 4-frame animation
let framesReady = 0;

for (let i = 0; i < frameCount; i++) {
    const frame = figmin.createObject(figmin.ObjectType.SKETCH, {
        position: { x: 0, y: 1, z: 0 },
        brushStrokes: [frameStrokes[i]]  // Each frame has different strokes
    });
    frame.onStart(() => {
        frame.active = (i === 0);  // Only first frame visible initially
        framesReady++;
    });
    frames.push(frame);
}

// Cycle visibility in update loop
let currentFrame = 0;
let elapsed = 0;
const frameDuration = 0.1;  // seconds per frame

figmin.setUpdateFunction((dt) => {
    if (framesReady < frameCount) return;  // Wait until all frames ready

    elapsed += dt;
    if (elapsed >= frameDuration) {
        elapsed = 0;
        frames[currentFrame].active = false;
        currentFrame = (currentFrame + 1) % frameCount;
        frames[currentFrame].active = true;
    }
});
```

---

<!-- @topic: sketches -->
## Brush Catalog

Access via `figmin.brushes.<Name>`. Each brush has: `id`, `name`, `category`, `minPressure`, `audioReactive`, `description`.

### Categories
- **FLAT**: 2D paint strokes (auto-segments at sharp angles)
- **3D**: Volumetric tubes/shapes
- **CONVEX_HULL**: Solid convex shapes (use small size)
- **PARTICLES**: Particle effects

### Common Brushes

**FLAT Brushes**:
`OilPaint`, `Ink`, `ThickPaint`, `WetPaint`, `Gouache`, `DryBrush`, `Flat`, `Marker`, `TaperedFlat`, `DoubleTaperedFlat`, `TaperedMarker`, `DoubleTaperedMarker`, `TaperedHueShift`, `DoubleTaperedHue`, `SoftHighlighter`, `Highlighter`, `VelvetInk`, `Paper`, `CoarseBristles`, `Charcoal`, `WigglyGraphite`, `CelVinyl`, `DuctTape`, `Light`, `Light2`, `Fire`, `Fire2`, `Electricity`, `Plasma`, `Waveform`, `WaveformFFT`, `ChromaticWave`, `Streamers`, `Comet`, `Wind`, `Splatter`, `Fairy`, `Feather`, `ArrowsBrush`, `Hypercolor`, `HypercolorTransparent`, `NeonPulse`, `Rain`, `Rainbow`, `HyperGrid`

**3D Brushes**:
`MetallicTube`, `TubeAdditive`, `Icing`, `Toon`, `MylarTube`, `MetallicWire`, `DiamondWire`, `ToonWire`, `Wire`, `Spikes`, `MetallicSpikes`, `SpikesToon`, `Lofted`, `LoftedMetallic`, `LoftedToon`, `LoftedHueShift`, `BubbleWand`, `SquarePaper`, `Petal`, `Muscle`, `Guts`, `Disco`, `LightWire`

**CONVEX_HULL Brushes**:
`ShinyHull`, `MatteHull`, `UnlitHull`, `DiamondHull`, `MetallicHull`, `StainedGlassHull`, `ToonHull`, `SmoothHull`, `SmoothMatteHull`, `MetallicSmoothHull`, `SmoothDiamondHull`, `InvisibleHull`

**PARTICLES Brushes**:
`Embers`, `Smoke`, `SmokeTiny`, `Stars`, `Snow`, `Bubbles`, `RisingBubbles`, `Hearts`, `Dots`

**Audio-Reactive** (append `_AR`):
`SoftHighlighter_AR`, `VelvetInk_AR`, `Light_AR`, `Fire_AR`, `Fire2_AR`, `Embers_AR`, `Stars_AR`, `Waveform_AR`, `WaveformFFT_AR`, `ChromaticWave_AR`, `Plasma_AR`, `Electricity_AR`, `Streamers_AR`, `Hypercolor_AR`, `HypercolorTransparent_AR`, `HyperGrid_AR`, `Snow_AR`, `Dots_AR`, `Toon_AR`, `Disco_AR`, `NeonPulse_AR`, `LightWire_AR`, `Rainbow_AR`, `WigglyGraphite_AR`, `Comet_AR`, `Feather_AR`, etc.

---

<!-- @topic: basics -->
## Enums Reference

### ObjectType
`MODEL3D`, `TEXT3D`, `IMAGE`, `SKETCH`, `PORTAL`, `BODYPART`, `AGENT`, `APP`

### ComponentType
`PHYSICS`, `PHYSICS_MATERIAL`, `LIGHT`, `MOTION`, `SKETCH`, `AUDIO`, `TRAIL`

### ObjectState
`UNINITIALIZED`, `LOADING`, `READY`, `DESTROYED`, `ERROR`

### InteractionState
`ENABLED`, `LOCKED`

### ColliderType
`NONE`, `BOX`, `SPHERE`, `APPROXIMATE`, `EXACT`

### BodypartType
`HEAD`, `LEFT_HAND`, `RIGHT_HAND`

### InputType
`HAND_TRACKING`, `MOTION_CONTROLLER`, `TOUCH_SCREEN`

### Layers
`SURFACE`, `PHYSICS_OBJ`, `STATIC_OBJ`, `UI`

### AppMode
`DEFAULT`, `MINIMIZED`, `OBJECT`, `HUD`, `FULLSCREEN`

### MessageRecipient
`MASTER`, `OTHERS`, `ALL`, `player(playerId)`

### RenderMode
`STANDARD`, `SBS_3D`, `OE_3D`

### MotionEndBehavior
`LOOP`, `PING_PONG`

### lights.LightType
`POINT`, `SPOT`, `DIRECTIONAL`

### lights.ShadowType
`NONE`, `HARD`, `SOFT`

### lights.LightEffect
`NONE`, `FLICKER`, `AUDIO_REACTIVE`, `PULSE`, `RAINBOW`

### lights.LightProjection
`NONE`, `FLASHLIGHT1`, `SPOTLIGHT1`, `SPOTLIGHT2`

### lights.SceneShadowMode
`DEFAULT`, `FORCE_ON`, `FORCE_OFF`

### physics.CollisionDetectionMode
`DISCRETE`, `CONTINUOUS`, `CONTINUOUS_DYNAMIC`, `CONTINUOUS_SPECULATIVE`

### physics.ForceFieldType
`NONE`, `SPHERE`, `FLAT`

### physics.ForceFieldForceType
`SQUARED`, `CONSTANT`

### physics.ForceFieldDirection
`UP`, `DOWN`, `LEFT`, `RIGHT`, `FORWARD`, `BACK`

### physics.Combine
`AVERAGE`, `MULTIPLY`, `MINIMUM`, `MAXIMUM`

---

<!-- @topic: _none -->
## Common Patterns

<!-- @topic: _none -->
### Spawn with Physics
```javascript
const ball = figmin.createObject(figmin.ObjectType.MODEL3D, {
    url: "https://example.com/ball.glb",
    position: spawnPos,
    targetSize: 0.1,
    colliderType: figmin.ColliderType.SPHERE
});

ball.onStart((b) => {
    b.addComponent(figmin.ComponentType.PHYSICS, {
        mass: 0.5,
        gravityStrength: 1.0
    });
    b.physics.velocity = { x: 0, y: 5, z: 3 };

    b.collider.onCollisionEnter((hit) => {
        console.log("Hit:", hit.objectName);
    });
});
```

<!-- @topic: interaction -->
### Player Hand Interaction
```javascript
let rightHand = null;

function setupHands() {
    const player = figmin.getLocalPlayer();
    const hand = figmin.getPlayerBodypart(player, figmin.BodypartType.RIGHT_HAND);

    hand.onStart((h) => {
        rightHand = h;
        h.bodypart.onTrackingChanged((obj, tracked) => {
            if (!tracked) rightHand = null;
        });
    });
}

function update(dt) {
    if (!rightHand?.bodypart.tracked) return;

    // Cast ray from hand
    figmin.raycast({
        origin: rightHand.bodypart.rayPosition,
        direction: rightHand.bodypart.rayForward,
        maxDistance: 5,
        lookupObject: true
    }, (hit) => {
        if (hit?.object) {
            // Do something with hit object
        }
    });
}
```

<!-- @topic: keyframes -->
### Animated Text
```javascript
const text = figmin.createObject(figmin.ObjectType.TEXT3D, {
    position: figmin.getWindowSpawnPosition(0.3, "right"),
    rotation: figmin.getWindow().transform.rotation,  // TEXT3D faces user directly
    text: "Spinning!",
    targetSize: 0.15
});

text.onStart((t) => {
    t.addComponent(figmin.ComponentType.MOTION, {
        speed: 1.0,
        endBehavior: figmin.MotionEndBehavior.LOOP
    });

    const frames = [];
    for (let i = 0; i <= 8; i++) {
        frames.push({
            p: { x: 0, y: 0, z: 0 },
            r: { x: 0, y: i * 45, z: 0 },
            t: i / 8
        });
    }
    t.motion.setKeyframes(frames, false);
});
```

<!-- @topic: sketches -->
### Procedural Sketch Geometry
```javascript
// Build strokes BEFORE creating the object
const strokes = [];

// Create a box wireframe
const box = figmin.sketch.boxStroke(
    figmin.brushes.Wire, "#00FF00", 0.005,
    0.3, 0.3, 0.3
);
strokes.push(box);

// Create a sphere above the box
const sphere = figmin.sketch.sphereStroke(
    figmin.brushes.ToonHull, "#FF5500", 0.002, 0.1
);
figmin.sketch.translateStroke(sphere, { x: 0, y: 0.3, z: 0 });
strokes.push(sphere);

// Pass strokes directly — SKETCH objects require brushStrokes in createObject
const sketch = figmin.createObject(figmin.ObjectType.SKETCH, {
    position: { x: 0, y: 1, z: 0 },
    brushStrokes: strokes
});
```

### Dynamic Text Background with onBoundsChanged
Add a background that auto-resizes when text changes:

```javascript
const text = figmin.createObject(figmin.ObjectType.TEXT3D, {
    text: "Hello",
    targetSize: 0.15
});

text.onStart((t) => {
    t.addComponent(figmin.ComponentType.SKETCH);

    t.transform.onBoundsChanged((obj, bounds) => {
        // Use 'size' (scaled) for world-space dimensions, not 'bounds' (unscaled)
        const size = obj.transform.size;
        const padding = 0.02;

        // Create rounded rectangle background
        const bg = figmin.sketch.roundedRectangleStroke(
            figmin.brushes.ToonHull,
            "#333333",
            0.002,
            size.x + padding * 2,
            size.y + padding * 2,
            0.3  // corner roundness
        );
        figmin.sketch.translateStroke(bg, { x: 0, y: 0, z: -size.z / 2 - 0.01 });
        obj.sketch.setBrushStrokes([bg]);
    });
});
```

---

<!-- @topic: multiplayer -->
## Multiplayer Considerations

1. **Authority**: Only master client creates/modifies objects
2. **Early out on init**: Check `isMasterClient()` once at startup, not on every action
3. **State Sync**: Object changes auto-replicate to all peers
4. **UI Sync**: Use `figmin-sync` attribute for automatic multiplayer UI synchronization — no messaging code required
5. **Messaging**: Use `sendMessage`/`onMessage` for custom multiplayer logic that `figmin-sync` can't handle (e.g. player identity, ephemeral events, complex state)
6. **Late Joiners**: Use `onPeerReady` to detect when a non-master's app instance is loaded and send them current state
7. **Bandwidth**: Peer-to-peer; avoid large/frequent updates
8. **Master Election**: If master disconnects, new master elected and script reloads

```javascript
function init() {
    // Early out pattern - check once at init
    if (!figmin.isMasterClient()) return;

    // Master-only logic
    createGameObjects();
    figmin.setUpdateFunction(gameLoop);

    // Handle late joiners
    var appObj = figmin.getAppObject();
    appObj.app.onPeerReady((playerId) => {
        var name = figmin.getPlayerById(playerId).displayName;
        console.log(name + " joined, sending state...");
        appObj.app.sendMessage({ action: "gameState", state: gameState }, figmin.MessageRecipient.player(playerId));
    });
}
```

---

<!-- @topic: basics -->
## Critical Rules & Gotchas

### Vector Assignment
**CRITICAL**: Never mutate vector properties directly. Always assign a NEW object:
```javascript
// WRONG - won't update
obj.transform.position.x = 5;

// CORRECT - assign new object
obj.transform.position = { x: 5, y: obj.transform.position.y, z: obj.transform.position.z };

// CORRECT - full assignment
obj.transform.position = { x: 5, y: 1, z: 0 };
```

### TEXT3D Orientation
TEXT3D's readable face points opposite to its transform +Z. This means when you copy the window's rotation directly, the text faces the user correctly - **no flip needed**:

```javascript
const obj = figmin.createObject(figmin.ObjectType.TEXT3D, {
    rotation: figmin.getWindow().transform.rotation,  // Faces user directly
    text: "Facing User"
});
```

### SKETCH Objects Require Initial Strokes
SKETCH objects must have `brushStrokes` passed in `createObject()` params or they will remain stuck in `LOADING` state and `onStart` will never fire.
```javascript
// WRONG — stays in LOADING forever
const obj = figmin.createObject(figmin.ObjectType.SKETCH, {
    position: { x: 0, y: 1, z: 0 }
});

// CORRECT — create strokes first, pass them in
const stroke = figmin.sketch.circleStroke(figmin.brushes.Electricity, "#00ffff", 0.008, 0.15);
const obj = figmin.createObject(figmin.ObjectType.SKETCH, {
    position: { x: 0, y: 1, z: 0 },
    brushStrokes: [stroke]
});
```

**Note**: This is different from MODEL3D and other object types, where the model's front typically aligns with +Z. For those, you may need to flip the rotation to face the user.

- **Grabbing and Interactions**
Objects are grabbable with just a collider (set via `colliderType` in `createObject`). The physics component is **optional** for adding dynamics. Use `bodypart.onGrabChanged` for grab/release detection on static or dynamic objects.

### Multiplayer Authority
- Only **master client** can create/modify objects
- **Early out on init** if not master client (don't check on every mutation)
- Non-master calls are silently ignored (no-op)

```javascript
function init() {
    if (!figmin.isMasterClient()) return;  // Early out pattern
    // Master-only setup code
}
```

---

<!-- @topic: basics -->

Some features are not yet available but can be worked around:

### No Native 3D Buttons
There are no built-in 3D clickable buttons. **Workaround**: Use raycasting from hand/controller + a highlight object + enableInputConnect for trigger detection.

```javascript
// Create highlight object (hidden by default)
const highlight = figmin.createObject(figmin.ObjectType.SKETCH, { ... });
highlight.onStart(() => { highlight.active = false; });

// In update loop, raycast from hand to detect "focus"
function checkFocus(hand) {
    figmin.raycast({
        origin: hand.bodypart.rayPosition,
        direction: hand.bodypart.rayForward,
        maxDistance: 5,
        lookupObject: true
    }, (hit) => {
        if (hit?.object === myButton) {
            highlight.active = true;
            highlight.transform.position = myButton.transform.position;
        } else {
            highlight.active = false;
        }
    });
}

// Use enableInputConnect + getInputDown for "click"
if (figmin.getInputDown(figmin.InputCode.SELECT_RIGHT)) {
    // Trigger pressed while focused = click
}
```

### Click Events Require Input Connection
To receive reliable trigger/button input, you must use `enableInputConnect()`. Without it, trigger presses aren't captured.

---

## Best Practices

1. **Always use FigminReady** - Don't call API before this event
2. **Wait for onStart** - Objects aren't ready immediately after creation
3. **Assign new vector objects** - Never mutate `.x`, `.y`, `.z` directly
4. **Early out if not master** - Check `isMasterClient()` once at init, not every mutation
5. **Use setUpdateFunction** - Not requestAnimationFrame (wrong framerate)
6. **Handle errors** - Use onError for load failures
7. **Clean up** - Objects are auto-destroyed when script ends (unless scenePersistent)
8. **Lock interaction** - Use `interactionState = LOCKED` for game objects
9. **Use targetSize** - Let the system scale objects consistently
10. **Test in 2D mode** - Use Steam version for faster iteration

---

<!-- @topic: _none -->
## Development Workflow

### MCP Agent Bridge (Recommended)

If you have access to the Figmin XR MCP bridge, use it. The AI assistant connects directly to the running Figmin XR session, executes scripts in real-time, and can see the scene through an agent camera — no copy-pasting errors, no manual reloading. This is the fastest development loop available.

### Manual Development (Without MCP)

If MCP is not available, use the following workflow:

#### Use the Steam 2D Version for Development

**Highly recommended**: Use the 2D version of Figmin XR (Steam) while developing:
- Easier to copy errors from the log console to your AI assistant
- No headset on/off constantly
- Faster iteration

Download from: https://store.steampowered.com/app/1890220/Figmin_XR/

Run in windowed mode (with SteamVR off).

#### Keep the Log Console Open

Always have the log console open while developing:
1. Open settings on your spatial app
2. Press the Console toggle
3. Press reload to catch any previous errors
4. Use the copy button to share errors with your AI assistant

Or call `figmin.showLogConsole()` programmatically.

#### Local Development (Web Server)

For quick iteration, serve your HTML from a local web server:

```bash
# In your app folder, start Python's built-in server
python -m http.server 8000
```

Then in Figmin XR browser, navigate to `<your-ip>:8000` (e.g., `192.168.0.23:8000`).

Find your IP:
- **Windows**: Run `ipconfig` in Command Prompt, look for "IPv4 Address"
- **Mac**: System Preferences → Network, or run `ifconfig` in Terminal

Edit files on your computer, then reload in Figmin XR to see changes instantly.

#### Release Workflow (Embed HTML)

For publishing, offer your HTML file for download in Figmin XR's browser:
1. Upload your `.html` file to Discord, a web host, or get a direct link from your AI
2. Open that link in Figmin XR's browser
3. Figmin XR will prompt to deploy the spatial app
4. The HTML is embedded directly - no server needed at runtime

**Warning**: Do NOT publish apps pointing to local IP addresses (192.168.x.x). Only use real internet URLs or embedded files.

**Note**: Embedded HTML cannot be updated after publishing. For updatable apps, host on a real web server.

---

*For the latest API documentation, visit: https://www.figmin.com/api*