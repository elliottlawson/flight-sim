# Flight Simulator Build Plan — Babylon.js

## Overview

This is a phased build plan for a browser-based flight simulator using Babylon.js. Each phase is designed to be built and tested independently before moving to the next. **Do not skip phases or combine them.** Each phase should produce a working, testable artifact.

---

## Phase 1: Flight Model (No Rendering)

Build a standalone flight dynamics model as a pure TypeScript/JavaScript class. No 3D rendering — output state to an HTML overlay or console.

### Requirements

- `FlightModel` class that maintains aircraft state: position (x, y, z), velocity vector, orientation (pitch, roll, yaw as a quaternion), airspeed, altitude, vertical speed, throttle level.
- Accepts inputs each frame: `pitchInput` (-1 to 1), `rollInput` (-1 to 1), `yawInput` (-1 to 1), `throttleInput` (0 to 1).
- Uses a **fixed physics timestep** (e.g., 1/60s) decoupled from render framerate. Accumulate delta time and step physics in fixed increments.
- Forces to model:
  - **Thrust**: forward force along the aircraft's local forward axis, proportional to throttle.
  - **Lift**: perpendicular to the aircraft's local forward axis and wings, proportional to airspeed² and angle of attack. Use a simplified lift curve — lift increases linearly with AoA up to ~15°, then drops (stall).
  - **Drag**: opposes velocity direction, proportional to airspeed². Includes both parasitic drag and induced drag (increases with lift).
  - **Gravity**: constant downward force (9.81 m/s²) applied in world space.
  - **Control surfaces**: pitch/roll/yaw inputs apply torques in the aircraft's **local coordinate space**, scaled by airspeed (controls should be less effective at low speed, more effective at high speed).
- **Critical: all rotational inputs must be in the aircraft's local space, never world space.** When the pilot pushes "pitch up," the nose goes up relative to the aircraft's current orientation, regardless of which direction it's facing.
- Add damping to angular velocity so the plane doesn't spin forever — it should return toward stable flight when inputs are released.
- Aircraft mass: pick a reasonable value (e.g., 1000 kg for a light aircraft). Tune lift coefficient so the plane achieves stable flight around 60-80 m/s airspeed.
- Expose all state as readable properties.

### Test Criteria

Display a simple HTML panel showing: altitude, airspeed, vertical speed, pitch angle, roll angle, heading, throttle %. Use keyboard input (arrow keys + throttle keys). Verify:

- With throttle up on a flat surface, the plane accelerates.
- At sufficient speed, pulling up causes a smooth climb (no oscillation).
- Releasing pitch input returns to roughly level flight over a few seconds.
- Banking left and pulling up causes a smooth left turn.
- At zero throttle, the plane gradually descends.
- No shaking, oscillation, or sudden flips at any point during normal flight.

---

## Phase 2: 3D Scene + Chase Camera (Flat Ground)

Create a Babylon.js scene with a flat ground plane, a sky, and a simple aircraft model. Connect it to the Phase 1 flight model.

### Requirements

- Use Babylon.js (import from CDN).
- Create a large flat ground plane (e.g., 10km x 10km) with a grid or simple texture so you can perceive movement.
- Add a hemispheric light and a directional light (sun).
- Use Babylon's built-in skybox or a gradient sky shader for atmosphere.
- Aircraft model: load a free glTF model of a small plane if possible (e.g., from Sketchfab or similar). If not available, build a simple but recognizable plane shape from merged meshes — fuselage cylinder, wing boxes, tail. **Do not use a single box.**
- **Connect the flight model to the 3D mesh**: each frame, set the mesh's position and rotation quaternion from the `FlightModel` state. The flight model is the source of truth; the mesh just follows it.
- **Chase camera**: 
  - Position the camera behind and above the aircraft (e.g., 20m behind, 5m above, relative to the plane's local space).
  - Use **lerp** for position smoothing (factor ~0.05-0.1 per frame) and **Quaternion.Slerp** for rotation smoothing (factor ~0.03-0.08).
  - The camera should feel like it's on a soft tether — it follows the plane but doesn't jerk or snap.
  - Camera always looks at the aircraft (or slightly ahead of it).
- **Ground constraint**: the aircraft's altitude (y position) cannot go below 0 (ground level). If it would, clamp it to 0 and zero out downward velocity. This is temporary — Phase 5 replaces this with terrain collision.
- Maintain the HUD overlay from Phase 1 showing flight data.

### Test Criteria

- Camera follows the plane smoothly through turns, climbs, and dives with no jitter.
- Banking the plane left makes the ground appear to move right from the camera's perspective.
- Pitching up tilts the view upward smoothly.
- Flying close to the ground and pulling up does not clip through the ground.
- The plane visually rotates in a way that matches the control inputs (stick right = roll right, stick back = pitch up).

---

## Phase 3: Input Polish + HUD

Refine controls and add a proper heads-up display.

### Requirements

- **Input mapping**:
  - Arrow Up = pitch up (nose goes up). Arrow Down = pitch down.
  - Arrow Left = roll left. Arrow Right = roll right.
  - W = increase throttle. S = decrease throttle.
  - A = yaw left. D = yaw right.
  - These must feel correct regardless of the aircraft's current orientation. **Test specifically**: fly north, turn to face east, then verify arrow up still pitches the nose up (not left or right).
- **Input smoothing**: don't apply raw 0/1 values from key presses. Ramp inputs up and down smoothly (e.g., lerp toward target input at a rate of ~5/second). This prevents jerky movement.
- **HUD** (HTML overlay, not 3D):
  - Airspeed indicator (in knots or m/s — pick one and be consistent)
  - Altimeter
  - Vertical speed indicator
  - Artificial horizon (pitch + roll visualization — a simple CSS element that rotates is fine)
  - Heading indicator
  - Throttle bar
  - Stall warning (visible alert when angle of attack exceeds stall threshold)
- **Throttle behavior**: throttle should stay where set (not spring back to zero). W and S increment/decrement it.

### Test Criteria

- All controls feel responsive but smooth — no instant snapping.
- HUD updates in real time and matches the aircraft's actual state.
- Controls never feel "reversed" regardless of aircraft heading or orientation.
- Stall warning appears when flying too slowly with nose too high.

---

## Phase 4: Landing + Ground Interaction

Add takeoff and landing capability on a flat runway.

### Requirements

- **Runway**: a visually distinct flat rectangle on the ground (darker color or painted texture). Start the aircraft on the runway at zero speed.
- **Ground state**: when the aircraft is on the ground (altitude ≈ 0 and vertical speed ≈ 0):
  - Disable roll and pitch below a threshold — the plane sits level on its wheels.
  - Apply ground friction to slow the plane.
  - Allow throttle to accelerate along the ground (like taxiing).
  - At sufficient speed (~60 m/s), the player can pull up to take off.
- **Landing detection**: when the aircraft descends to ground level:
  - If vertical speed is gentle (< ~3 m/s descent) and pitch is roughly level and wings roughly level → successful landing. Transition to ground state. Apply braking friction.
  - If vertical speed is too high or extreme pitch/roll → crash. Display "CRASHED" message and offer restart.
- **Gear abstraction**: don't model landing gear separately. Just use the ground contact logic above.
- The plane should not be able to go below y=0 under any circumstances. Clamp and adjust velocity on contact.

### Test Criteria

- Start on runway, throttle up, reach speed, pull back, plane takes off smoothly.
- Fly a circuit, come back, reduce throttle, descend gradually, touch down without crashing.
- Hard landings correctly trigger crash state.
- On the ground, the plane doesn't bounce, vibrate, or clip through.
- Plane cannot go below the ground ever.

---

## Phase 5: Terrain

Replace the flat ground with heightmap-based terrain.

### Requirements

- Use Babylon.js `GroundMesh` with `CreateGroundFromHeightMap` or equivalent.
- Terrain should cover a large area (at least 10km x 10km). Use a procedurally generated heightmap or a real-world one.
- Terrain should have visible elevation changes: hills, valleys, maybe a mountain. Max elevation should be noticeable (e.g., 500-1000m).
- Apply a terrain texture (green for low, brown/gray for high, or a proper splat map).
- **Terrain collision**: replace the simple y=0 ground check with actual terrain height queries. Babylon provides `getHeightAtCoordinates()` on ground meshes. Each physics step:
  - Get terrain height at the aircraft's current x,z position.
  - If aircraft altitude ≤ terrain height → ground contact (same landing/crash logic as Phase 4, but relative to terrain height, not zero).
  - The aircraft must **never** pass through terrain. If it would, clamp altitude to terrain height.
- **Terrain-aware altitude display**: HUD should show both absolute altitude (above sea level) and AGL (above ground level) using terrain height queries.
- **Visual**: the terrain should be visible from the air and give a sense of flying over varied landscape.

### Test Criteria

- Flying over hills, the AGL reading changes even at constant absolute altitude.
- Flying into a mountainside triggers a crash.
- Landing on a flat(ish) area of terrain works the same as landing on the runway.
- No clipping through terrain from any angle or speed.
- Terrain looks reasonable from altitude (not a flat green plane).

---

## Phase 6: Polish + Atmosphere (Optional Enhancements)

These are stretch goals once everything above works solidly.

- **Clouds**: simple billboard clouds or volumetric cloud layer at a set altitude.
- **Fog/haze**: distance fog to give depth and hide terrain edges.
- **Propeller animation**: spinning disc on the nose, speed linked to throttle.
- **Sound**: engine hum that changes pitch with throttle/airspeed (use Web Audio API).
- **Multiple camera views**: chase cam, cockpit cam, free look.
- **Minimap**: small overhead view showing plane position on terrain.
- **Wind**: add a constant or gusting wind vector that affects the flight model.

---

## General Rules for All Phases

1. **Use Babylon.js loaded from CDN.** The artifact should be a single HTML file.
2. **All physics in local space.** Rotational inputs always relative to the aircraft, never the world.
3. **Fixed physics timestep.** Do not tie physics to framerate.
4. **No magic numbers without comments.** Every coefficient (lift, drag, mass, etc.) should have a comment explaining what it represents and what a reasonable range is.
5. **Test each phase before moving on.** If it doesn't fly right, fix it before adding terrain or landing.
6. **The flight model class should be separate from the rendering code.** Don't mix physics logic into the render loop. The render loop reads state from the flight model and updates visuals — that's it.
7. **Camera smoothing is mandatory.** Never rigidly attach the camera to the plane. Always lerp/slerp.
8. **Clamp everything.** Velocity, angular velocity, position — nothing should go to infinity or NaN. Add sanity checks.
