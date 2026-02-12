# Fit Horizontal Single-Axis Box Centering for SUSTechPOINTS

> A new annotation tool that horizontally centers a 3D bounding box on the point cloud without affecting vertical position, box size, or rotation.

---

## Overview

When annotating 3D LiDAR data, the standard **Fit Position** moves the box along all axes simultaneously, which often causes unwanted vertical drift. **Fit Horizontal** solves this by shifting the box **only along the screen-horizontal axis**, snapping it to the center of the detected point cluster.

![Horizontal](https://github.com/user-attachments/assets/6ec486fb-99c3-44c7-9ce3-531cf592014b)
![Fit Horizontal](https://github.com/user-attachments/assets/241d6dce-c124-4982-aff3-fb2b31cf8ca4)

### Algorithm in Detail

1. **Search zone** — A virtual expanded box is created with X and Y scaled by **1.3x** (Z unchanged). This extends the search area by 15% beyond each face, enough to catch nearby points without pulling in too much noise.

2. **Point collection** — All LiDAR points inside the expanded box are gathered, then their coordinates are recalculated relative to the **original** box center and rotation.

3. **Ground filtering** — The lowest Z-coordinate among all collected points is found. Every point within **0.5 m** above that minimum is discarded. This removes road/ground points that would otherwise skew the horizontal centering. If all points are filtered out, the filter is bypassed (fallback).

4. **Centering** — From the remaining points, the min and max values along the target axis are found. The box is translated so its center sits at `(min + max) / 2` on that axis. No other axes are touched.
---

## Tunable Parameters

| Parameter | Value | Location | Purpose |
|-----------|-------|----------|---------|
| X/Y expansion | `1.3x` | `box_op.js` | Search area beyond box edges |
| Z expansion | `1.0x` | `box_op.js` | No vertical expansion |
| Ground filter | `0.5 m` | `box_op.js` | Height from lowest point to cut |

**Adjusting expansion:** Lower values (e.g., `1.1x`) reduce noise from nearby objects but require the box to already be close to the target. Higher values (e.g., `1.5x`) reach further but may capture irrelevant points.

**Adjusting ground filter:** For trucks or tall vehicles, `0.5 m` works well. For smaller objects (pedestrians, cyclists), consider reducing to `0.3-0.5 m`.

---

## Installation

### Modified Files

```
public/js/box_op.js        — New function: fit_position_axis()
public/js/side_view_op.js  — Wiring: parameter, handlers, constructors
index.html                 — UI: button with SVG icon
```

---

### 1. `public/js/box_op.js`

**Location:** After the `fit_size` function (around line 72), before `justifyAutoAdjResult`.

**Action:** Add the following function:

```javascript
    // Fit position along a single axis (horizontal shift only)
    // Does NOT change other axes, scale, or rotation
    // Centers box on midpoint of points along the target axis
    this.fit_position_axis = function(box, axis){
        // 1. Search points in slightly expanded area (1.3x)
        let expandedBox = {
            position: {x: box.position.x, y: box.position.y, z: box.position.z},
            rotation: {x: box.rotation.x, y: box.rotation.y, z: box.rotation.z},
            scale:    {x: box.scale.x * 1.3, y: box.scale.y * 1.3, z: box.scale.z},
            world: box.world,
        };
        let points_indices = box.world.lidar.get_points_of_box(expandedBox, 1.0).index;
        if (points_indices.length === 0) return;

        // 2. Get point positions relative to original box
        let ret = box.world.lidar._get_points_of_box(
            box.world.lidar.points, box, 1, points_indices);
        let points = ret.position;
        if (points.length === 0) return;

        // 3. Filter out ground: remove points within 1m of the lowest point
        let minZ = Infinity;
        for (let i = 0; i < points.length; i++){
            if (points[i][2] < minZ) minZ = points[i][2];
        }
        let filtered = points.filter(function(p){ return p[2] - 0.5 > minZ; });
        if (filtered.length === 0) filtered = points; // fallback if all filtered out

        // 4. Find min/max along target axis
        let axisIndex = axis === 'x' ? 0 : (axis === 'y' ? 1 : 2);
        let min = Infinity, max = -Infinity;
        for (let i = 0; i < filtered.length; i++){
            let v = filtered[i][axisIndex];
            if (v < min) min = v;
            if (v > max) max = v;
        }

        // 5. Center box on midpoint along target axis
        this.translate_box(box, axis, (max + min) / 2);
    };
```

---

### 2. `public/js/side_view_op.js`

Ten changes labeled **A** through **J**:

#### A. Constructor parameter

**Where:** `ProjectiveView` function signature (line ~15), after `on_fit_size,`

```diff
  on_fit_size,
+ on_fit_horizontal,
  on_auto_rotate,
```

#### B. Store parameter

**Where:** Inside constructor, after `this.on_fit_size=on_fit_size;` (line ~37)

```diff
  this.on_fit_size=on_fit_size;
+ this.on_fit_horizontal=on_fit_horizontal;
```

#### C. Button reference

**Where:** `this.buttons` object, after `fit_position:` (line ~76)

```diff
  fit_position: ui.querySelector("#v-fit-position"),
+ fit_horizontal: ui.querySelector("#v-fit-horizontal"),
  fit_size: ui.querySelector("#v-fit-size"),
```

#### D. Click handler

**Where:** `install_buttons()`, after the `buttons.fit_position` block (line ~853)

```javascript
if (buttons.fit_horizontal && this.on_fit_horizontal){
    buttons.fit_horizontal.onclick = event=>{
        this.on_fit_horizontal();
    };
}
```

#### E. Z-view handler function

**Where:** After `on_z_fit_size`, before `on_z_auto_rotate` (line ~1166)

```javascript
function on_z_fit_horizontal(){
    scope.boxOp.fit_position_axis(scope.box, 'y');
    scope.on_box_changed(scope.box);
}
```

#### F. Z-view constructor call

**Where:** `new ProjectiveView` for `z_view_handle` (line ~1205)

```diff
  on_z_fit_size,
+ on_z_fit_horizontal,
  on_z_auto_rotate,
```

#### G. Y-view handler function

**Where:** After `on_y_auto_rotate`, before `new ProjectiveView` for Y (line ~1329)

```javascript
function on_y_fit_horizontal(){
    scope.boxOp.fit_position_axis(scope.box, 'x');
    scope.on_box_changed(scope.box);
}
```

#### H. Y-view constructor call

**Where:** `new ProjectiveView` for `y_view_handle` (line ~1343)

```diff
  null,
+ on_y_fit_horizontal,
  on_y_auto_rotate,
```

#### I. X-view handler function

**Where:** After `on_x_auto_rotate`, before `new ProjectiveView` for X (line ~1466)

```javascript
function on_x_fit_horizontal(){
    scope.boxOp.fit_position_axis(scope.box, 'y');
    scope.on_box_changed(scope.box);
}
```

#### J. X-view constructor call

**Where:** `new ProjectiveView` for `x_view_handle`

```diff
  null,
+ on_x_fit_horizontal,
  on_x_auto_rotate,
```

---

### 3. `index.html`

**Where:** After `div#v-fit-position`, before `div#v-fit-rotation` (line ~1124)

**Action:** Add the button:

```html
<div id="v-fit-horizontal" class="ui-button" title="Fit horizontal position">
    <svg viewBox="0 0 24 24" preserveAspectRatio="xMidYMid meet"
         focusable="false" class="svg-button"><g>
        <rect x="4" y="6" width="12" height="12" stroke-dasharray="1"/>
        <rect x="8" y="6" width="12" height="12"/>
        <line x1="14" y1="20" x2="18" y2="20" stroke-width="1.5"/>
        <line x1="15" y1="19" x2="18" y2="20" stroke-width="1"/>
        <line x1="15" y1="21" x2="18" y2="20" stroke-width="1"/>
    </g></svg>
</div>
```

## License

This feature is an extension of [SUSTechPOINTS](https://github.com/naurril/SUSTechPOINTS), licensed under the same terms as the original project.

