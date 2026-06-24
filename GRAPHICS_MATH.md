# 3D Graphics Math — A Computational Reference

This document explains and **shows how to compute** every piece of math used in
the two projects in this folder:

- `webgl-cube-mvp.html` — the Model · View · Projection (MVP) pipeline.
- `cube_lighting.html` — surface normals, the normal matrix, and the
  Ambient + Lambert + Blinn‑Phong lighting model.

It is written so you can **work each formula out by hand** with a calculator.
Every section has: the idea, the formula, and a fully worked numeric example.

> **Conventions used everywhere in this folder**
> - **Column vectors**: a point is a column `v`, transformed as `v' = M · v`.
>   The matrix on the **left is applied last**.
> - **Column‑major storage**: WebGL `Float32Array` lists column 0, then column
>   1, etc. Element at (row `r`, col `c`) lives at index `r + 4·c`.
> - **Right‑handed coordinates**: +X right, +Y up, +Z toward the viewer; the
>   camera looks down **−Z**.
> - Angles are in **radians** (`deg · π/180`).

---

## Table of contents

1. [Vectors: the atoms](#1-vectors-the-atoms)
2. [Dot product (the workhorse of lighting)](#2-dot-product)
3. [Cross product (building camera axes & normals)](#3-cross-product)
4. [Normalization (unit vectors)](#4-normalization)
5. [Matrices & homogeneous coordinates](#5-matrices--homogeneous-coordinates)
6. [Translation](#6-translation)
7. [Rotation](#7-rotation)
8. [Matrix multiplication & transform order](#8-matrix-multiplication--order)
9. [The View matrix (camera)](#9-the-view-matrix)
10. [The Perspective Projection matrix](#10-the-perspective-projection-matrix)
11. [The perspective divide & clip → NDC → screen](#11-the-perspective-divide)
12. [The full MVP pipeline, traced end‑to‑end](#12-full-mvp-trace)
13. [Surface normals](#13-surface-normals)
14. [The Normal Matrix (inverse‑transpose)](#14-the-normal-matrix)
15. [Lighting I — Ambient](#15-lighting-i--ambient)
16. [Lighting II — Lambert diffuse](#16-lighting-ii--lambert-diffuse)
17. [Lighting III — Blinn‑Phong specular](#17-lighting-iii--blinn-phong-specular)
18. [A complete per‑pixel lighting calculation](#18-complete-lighting-calc)
19. [Quick formula cheat‑sheet](#19-cheat-sheet)

---

<a name="1-vectors-the-atoms"></a>
## 1. Vectors: the atoms

A 3D vector `a = (aₓ, a_y, a_z)` is used for two different things:

- a **point/position** (a location in space), or
- a **direction** (an arrow with length and orientation).

The distinction matters when we add the 4th homogeneous coordinate `w`
(Section 5): **points use `w = 1`, directions use `w = 0`.**

**Basic operations** (component‑wise):

```
a + b = (aₓ+bₓ, a_y+b_y, a_z+b_z)
a − b = (aₓ−bₓ, a_y−b_y, a_z−b_z)
s · a = (s·aₓ,  s·a_y,  s·a_z)     (scalar scaling)
```

**Length (magnitude):**

```
|a| = √(aₓ² + a_y² + a_z²)
```

> In code (`cube_lighting.html`): `sub3`, `dot3`, `cross3`, `normalize3`, and
> `Math.hypot(x,y,z)` implement exactly these.

**Worked example.** Light at `Lp = (4, 2.2, 0)`, surface point at
`P = (1, 1, 1)`. The vector *from the surface to the light* is:

```
Lp − P = (4−1, 2.2−1, 0−1) = (3, 1.2, −1)
|Lp − P| = √(3² + 1.2² + (−1)²) = √(9 + 1.44 + 1) = √11.44 ≈ 3.382
```

---

<a name="2-dot-product"></a>
## 2. Dot product — the workhorse of lighting

```
a · b = aₓ·bₓ + a_y·b_y + a_z·b_z          (a single number, a scalar)
```

The geometric meaning is the **whole reason lighting works**:

```
a · b = |a| · |b| · cos(θ)
```

where `θ` is the angle between the vectors. So if **both vectors are unit
length** (`|a| = |b| = 1`), then:

```
a · b = cos(θ)
```

That is why we normalize before taking dot products in lighting — the dot
*becomes* the cosine of the angle, and brightness follows the cosine
(Lambert's law, Section 16).

**Sign tells you orientation:**

| `a · b` | angle θ | meaning in lighting |
|--------:|:-------:|:--------------------|
| `> 0`   | < 90°   | surfaces face each other → lit |
| `= 0`   | = 90°   | edge‑on / grazing → no light |
| `< 0`   | > 90°   | facing away → clamp to 0 |

**Worked example.** Normal `N = (0, 1, 0)` (a top face), light direction
(already normalized) `L = (0.6, 0.8, 0)`:

```
N · L = 0·0.6 + 1·0.8 + 0·0 = 0.8
```

Both are unit length, so `cos(θ) = 0.8`, i.e. `θ ≈ 37°`. The face receives
about 80% of the maximum diffuse light.

---

<a name="3-cross-product"></a>
## 3. Cross product — perpendiculars

The cross product returns a **new vector perpendicular to both inputs**:

```
a × b = ( a_y·b_z − a_z·b_y,
          a_z·bₓ − aₓ·b_z,
          aₓ·b_y − a_y·bₓ )
```

Direction follows the **right‑hand rule**; magnitude is
`|a||b|·sin(θ)`. We use it to:

- build the camera's **right** and **up** axes in `lookAt` (Section 9), and
- (conceptually) compute a face normal from two edge vectors:
  `N = normalize((v1 − v0) × (v2 − v0))`.

**Worked example.** `up = (0,1,0)`, forward `z = (0,0,1)`:

```
right = up × z = (1·1 − 0·0,  0·0 − 0·1,  0·0 − 1·0) = (1, 0, 0)
```

That's +X — exactly the right axis you'd expect for a camera looking down −Z.

---

<a name="4-normalization"></a>
## 4. Normalization — unit vectors

To make a vector length 1 while keeping its direction, divide by its length:

```
â = a / |a|
```

**Why we need it constantly:** the dot‑product‑equals‑cosine identity
(`a·b = cos θ`) only holds when both vectors are unit length. After we
*interpolate* normals across a triangle (Section 13) the result is generally
**shorter than 1**, so the fragment shader re‑normalizes before lighting.

**Worked example.** Normalize `a = (3, 1.2, −1)` from Section 1
(`|a| ≈ 3.382`):

```
â = (3/3.382, 1.2/3.382, −1/3.382) ≈ (0.887, 0.355, −0.296)
check: 0.887² + 0.355² + 0.296² ≈ 0.787 + 0.126 + 0.088 ≈ 1.000 ✓
```

---

<a name="5-matrices--homogeneous-coordinates"></a>
## 5. Matrices & homogeneous coordinates

A 3D transform is a **4×4 matrix**. Why 4×4 and not 3×3? Because a 3×3 matrix
can only do *linear* maps (rotation, scale), and those **always keep the origin
fixed** (`M·0 = 0`) — they can never *move* (translate) anything.

The fix is **homogeneous coordinates**: lift a 3D point into 4D by appending
`w`:

```
point     →  (x, y, z, 1)     (w = 1, so translation applies)
direction →  (x, y, z, 0)     (w = 0, so translation is ignored)
```

The 4×4 layout, with the **translation living in the last column**:

```
| m0  m4  m8   m12 |     m12,m13,m14 = translation (tx,ty,tz)
| m1  m5  m9   m13 |     upper‑left 3×3 = rotation + scale
| m2  m6  m10  m14 |
| m3  m7  m11  m15 |
```

In the column‑major `Float32Array`, the indices read **down columns**:

```
[ m0,m1,m2,m3,  m4,m5,m6,m7,  m8,m9,m10,m11,  m12,m13,m14,m15 ]
   column 0       column 1       column 2          column 3
```

To read element (row `r`, col `c`): index = `r + 4·c`.

---

<a name="6-translation"></a>
## 6. Translation

Slides a point by `(tx, ty, tz)` without rotating or scaling:

```
| 1 0 0 tx |   | x |   | x + tx |
| 0 1 0 ty | · | y | = | y + ty |
| 0 0 1 tz |   | z |   | z + tz |
| 0 0 0 1  |   | 1 |   |   1    |
```

The `w = 1` is what "activates" the translation column. A direction with
`w = 0` would come out unchanged — which is exactly why normals (directions)
ignore translation (Section 14).

**Worked example.** Move `(1, 1, 1)` by `(0, 0, −6)` (the view matrix in the
MVP demo) → `(1, 1, −5)`. The cube point is now 5 units in front of the camera.

---

<a name="7-rotation"></a>
## 7. Rotation

The core 2D rotation by angle `θ` of a pair `(a, b)`:

```
a' = a·cos θ − b·sin θ
b' = a·sin θ + b·cos θ
```

A 3D axis rotation applies this 2D rotation to the **other two** axes and
leaves the rotation axis untouched.

**Rotation about X** (spins the y–z plane; x fixed):

```
| 1   0       0     0 |     y' = y·cos − z·sin
| 0  cos θ  −sin θ  0 |     z' = y·sin + z·cos
| 0  sin θ   cos θ  0 |
| 0   0       0     1 |
```

**Rotation about Y** (spins the x–z plane; y fixed):

```
|  cos θ  0  sin θ  0 |     x' =  x·cos + z·sin
|   0     1   0     0 |     z' = −x·sin + z·cos
| −sin θ  0  cos θ  0 |
|   0     0   0     1 |
```

> **Column‑major gotcha.** In code, `rotationXMatrix` stores `sin` at array
> index `[6]` and `−sin` at `[9]`. That looks "transposed" versus the math
> grid above only because the array is filled column‑by‑column. Use
> `index = row + 4·col` to verify it matches the matrix.

**Worked example.** Rotate `(1, 0, 0)` about Y by `θ = 90°`
(`cos = 0`, `sin = 1`):

```
x' =  1·0 + 0·1 = 0
z' = −1·1 + 0·0 = −1
→ (0, 0, −1)
```

The point that pointed along +X now points along −Z. ✓

---

<a name="8-matrix-multiplication--order"></a>
## 8. Matrix multiplication & transform order

To **combine** transforms you multiply their matrices. The product `C = A·B`:

```
C[r][c] = Σ over k of  A[r][k] · B[k][c]      (dot of A's row r with B's col c)
```

In column‑major indices that is exactly what `multiplyMatrices` computes:
`out[r + 4c] = Σ_k a[r + 4k] · b[k + 4c]`.

**Order is not commutative: `A·B ≠ B·A`.** With column vectors
(`v' = M·v`) the **rightmost matrix acts first**. So:

```
Model = Translate · RotateY · RotateX
```

means **rotate about X, then Y, then translate** — read right to left.

- *Rotate then translate* → the object orbits the world origin.
- *Translate then rotate* → the object spins about a far pivot.

Completely different motion from the same three matrices.

**Worked 2×2 illustration** (same principle, smaller numbers). Let
`R = [[0,−1],[1,0]]` (90° rotation) and `T` conceptually "shift right". Then
`R·T` and `T·R` send a point to different places — multiply them out and you
get different matrices, proving order matters.

---

<a name="9-the-view-matrix"></a>
## 9. The View matrix (camera)

There is **no camera object** in WebGL. "Moving the camera" is implemented by
moving the **entire world by the inverse** of the camera's transform.

The simple demo uses `translationMatrix(0, 0, −6)`: pushing the world 6 units
down −Z is identical to pulling a camera back to +Z by 6.

The lighting demo uses a full **`lookAt(eye, center, up)`**, built from an
orthonormal camera basis:

```
z (forward) = normalize(eye − center)      // points from target back to eye
x (right)   = normalize(up × z)
y (trueUp)  = z × x                          // re-derived so it's exactly ⟂

View =
| xₓ  x_y  x_z  −(x·eye) |
| yₓ  y_y  y_z  −(y·eye) |
| zₓ  z_y  z_z  −(z·eye) |
|  0   0    0      1     |
```

The upper‑left 3×3 **rotates** world space into the camera's orientation; the
last column **translates** so the eye ends up at the origin. The `−(axis·eye)`
terms are the projections of the eye onto each camera axis (this is the inverse
translation).

**Worked example.** `eye = (0, 0, 8)`, `center = (0,0,0)`, `up = (0,1,0)`:

```
z = normalize((0,0,8) − (0,0,0)) = (0, 0, 1)
x = normalize((0,1,0) × (0,0,1)) = (1, 0, 0)
y = (0,0,1) × (1,0,0)            = (0, 1, 0)
translation column = (−x·eye, −y·eye, −z·eye) = (0, 0, −8)
```

So a point at the world origin maps to `(0,0,−8)` in camera space — 8 units in
front of the camera. ✓

---

<a name="10-the-perspective-projection-matrix"></a>
## 10. The Perspective Projection matrix

This is the most important matrix. It maps the **view frustum** (the truncated
pyramid of everything visible between the near and far planes) into a cube of
**clip space**, and sets up the perspective divide that makes distant things
small.

Parameters:

- `fov` — vertical field of view, in radians.
- `aspect` — viewport width / height (so the image isn't stretched).
- `near`, `far` — clipping distances (`near > 0`).

Let `f = 1 / tan(fov / 2)` (the focal length factor). The matrix:

```
| f/aspect   0         0                       0 |
|   0        f         0                       0 |
|   0        0   (far+near)/(near−far)        −1 |
|   0        0   (2·far·near)/(near−far)       0 |
```

Two things to notice:

1. `[0] = f/aspect` and `[5] = f` scale x and y; dividing x by `aspect`
   corrects for a non‑square window.
2. The `−1` in the 3rd column (array index `[11]`) **copies `−z` into the
   output `w`**. That stored `w` is what makes the perspective divide shrink
   distant geometry (next section).

**Worked example.** `fov = 45° = 0.7854 rad`, `aspect = 16/9 ≈ 1.778`,
`near = 0.1`, `far = 100`:

```
f = 1 / tan(0.3927) = 1 / 0.4142 ≈ 2.4142
f/aspect = 2.4142 / 1.778 ≈ 1.358

[10] = (100 + 0.1)/(0.1 − 100) = 100.1 / (−99.9) ≈ −1.002
[14] = (2·100·0.1)/(0.1 − 100) = 20 / (−99.9)    ≈ −0.200
```

So the projection matrix (column‑major reading as rows) is approximately:

```
| 1.358   0       0        0   |
|   0    2.414    0        0   |
|   0     0     −1.002   −0.200|
|   0     0      −1        0   |
```

---

<a name="11-the-perspective-divide"></a>
## 11. The perspective divide & clip → NDC → screen

The projection matrix outputs a 4D **clip‑space** vector
`(x_c, y_c, z_c, w_c)`. The GPU then automatically divides by `w_c`:

```
NDC = (x_c / w_c,  y_c / w_c,  z_c / w_c)     ("Normalized Device Coordinates")
```

Because the matrix set `w_c = −z_camera` (the distance in front of the eye),
**farther points have larger `w_c`, so dividing shrinks them** — that single
divide *is* perspective.

- Anything with NDC components in `[−1, 1]` is on screen; outside is clipped.
- Finally the **viewport transform** maps NDC `x,y` from `[−1,1]` to pixel
  coordinates: `screenX = (ndcX·0.5 + 0.5)·width`, similarly for y.

**Worked example.** Take a camera‑space point `(1, 1, −5)` (5 units ahead) and
the projection matrix above. The relevant outputs:

```
x_c = 1.358 · 1 = 1.358
y_c = 2.414 · 1 = 2.414
w_c = (−1) · (−5) = 5            ← the magic: w = −z = 5

NDC_x = 1.358 / 5 ≈ 0.272
NDC_y = 2.414 / 5 ≈ 0.483
```

Now move the same point twice as far, to `(1, 1, −10)`: `w_c = 10`, so
`NDC_x ≈ 0.136`, `NDC_y ≈ 0.241` — **half the screen offset.** Twice as far →
half as big. That is the perspective effect, falling straight out of the
divide.

---

<a name="12-full-mvp-trace"></a>
## 12. The full MVP pipeline, traced end‑to‑end

A vertex's journey in `webgl-cube-mvp.html`:

```
gl_Position = P · V · M · (x, y, z, 1)
```

Read **right to left**:

| Step | Matrix | Space before → after | What it does |
|-----:|:-------|:---------------------|:-------------|
| 1 | `M` (Model) | local → **world** | place/rotate the cube in the scene |
| 2 | `V` (View) | world → **camera** | re‑express relative to the eye |
| 3 | `P` (Projection) | camera → **clip** | add perspective, load `w = −z` |
| 4 | ÷w (GPU) | clip → **NDC** | perspective divide |
| 5 | viewport (GPU) | NDC → **screen** | scale to pixels |

**Worked mini‑trace.** Cube corner `(1, 1, 1)`, no rotation (`M = I`),
`V = translate(0,0,−6)`, `P` = the matrix from Section 10.

```
1. M·v = (1, 1, 1, 1)                      (identity, unchanged)
2. V·… = (1, 1, 1−6, 1) = (1, 1, −5, 1)    (translated −6 on z)
3. P·… : x_c = 1.358,  y_c = 2.414,
         w_c = (−1)·(−5) = 5
4. ÷w  : NDC = (0.272, 0.483, …)
5. on a 1600×900 canvas:
         screenX = (0.272·0.5 + 0.5)·1600 ≈ 1018 px
         screenY = (0.483·0.5 + 0.5)·900  ≈ 667 px (before y‑flip)
```

That corner lands in the upper‑right region of the screen — exactly where a
front‑top‑right cube corner should appear.

---

<a name="13-surface-normals"></a>
## 13. Surface normals

A **normal** `N` is a unit vector pointing straight out of a surface,
perpendicular to it. Lighting needs it because brightness depends on the
**angle between the surface and the light**, and the normal is how we encode
"which way this surface faces."

**Computing a cube's normals.** Each of the 6 faces points along one axis, so
the normal is simply that axis:

```
+X face → ( 1, 0, 0)      −X face → (−1, 0, 0)
+Y face → ( 0, 1, 0)      −Y face → ( 0,−1, 0)
+Z face → ( 0, 0, 1)      −Z face → ( 0, 0,−1)
```

A cube therefore uses **24 vertices, not 8** — corners are duplicated so each
face can carry its **own** normal. (Sharing 8 corners and averaging would round
the cube's shading.)

**General formula from geometry** (for arbitrary meshes): take two edges of a
triangle and cross them:

```
N = normalize( (v1 − v0) × (v2 − v0) )
```

**Interpolation.** The rasterizer linearly blends the 3 corner normals across
each triangle, giving every pixel its own normal. For a cube all 4 vertices of
a face share one normal, so the face stays flat‑shaded; for a sphere the
corner normals differ and you get smooth shading. **Interpolation shortens
vectors**, which is why the fragment shader calls `normalize()` again
(Section 4).

---

<a name="14-the-normal-matrix"></a>
## 14. The Normal Matrix (inverse‑transpose)

**You cannot transform a normal with the model matrix** when the matrix
contains non‑uniform scale. A normal is a *direction perpendicular to a
surface*; if you stretch space unevenly, the naive transform tilts the normal
so it is **no longer perpendicular**, and lighting goes wrong.

The matrix that keeps a normal perpendicular after applying model matrix `M`
is the **inverse‑transpose** of `M`'s upper‑left 3×3:

```
N_world = transpose( inverse( M₃ₓ₃ ) ) · N_local
```

**Why inverse‑transpose?** A surface tangent `t` transforms by `M` and must
stay perpendicular to the normal: `N' · t' = 0`. Working that requirement
through the algebra forces `N' = (M⁻¹)ᵀ · N`. (Derivation sketch:
`N'·t' = (A·N)·(M·t) = Nᵀ·Aᵀ·M·t`; this equals `Nᵀ·t = 0` for all `t` only if
`Aᵀ·M = I`, i.e. `A = (M⁻¹)ᵀ`.)

**When is the normal matrix just the rotation matrix?** For any rotation or
rigid / uniform‑scale transform, `M` is **orthogonal**, so `M⁻¹ = Mᵀ` and
therefore `(M⁻¹)ᵀ = M`. The inverse‑transpose collapses back to the plain
rotation — no special handling needed. The cube in `cube_lighting.html` only
rotates and translates, so its normal matrix *equals* the rotation part; the
code still computes the general inverse‑transpose so it stays correct if you
add scaling.

**Why do non‑uniform scales break normals?** Uniform scale only changes a
normal's *length* (fixed by `normalize()`), but **non‑uniform** scale shears
its *direction* off perpendicular. The inverse‑transpose precisely undoes that
shear.

**Worked example.** Scale `S = diag(2, 1, 1)` (stretch x by 2). A surface that
faces +X has normal `N = (1, 0, 0)`.

```
inverse(S)            = diag(1/2, 1, 1)
transpose(inverse(S)) = diag(1/2, 1, 1)   (diagonal ⇒ transpose unchanged)
N_world = diag(1/2,1,1) · (1,0,0) = (1/2, 0, 0)  → normalize → (1, 0, 0) ✓
```

Now check a **slanted** normal `N = normalize(1,1,0) = (0.707, 0.707, 0)`:

```
naive (wrong) S·N         = (1.414, 0.707, 0) → normalize → (0.894, 0.447, 0)
correct (S⁻¹)ᵀ·N          = (0.354, 0.707, 0) → normalize → (0.447, 0.894, 0)
```

The two disagree — and only the inverse‑transpose result stays perpendicular to
the stretched surface. That difference is exactly what would make lighting look
wrong on a non‑uniformly scaled object.

---

<a name="15-lighting-i--ambient"></a>
## 15. Lighting I — Ambient

Ambient is a constant fill that fakes light bounced off everything else in the
scene, so surfaces facing *away* from the light aren't pure black.

```
ambient = ambientStrength · objectColor
```

**Why we need it:** with only diffuse + specular, any face with `N·L ≤ 0`
would be solid black — unrealistic. Real shadowed sides are dim, not invisible.

**Worked example.** `ambientStrength = 0.12`,
`objectColor = (0.25, 0.55, 0.95)`:

```
ambient = 0.12 · (0.25, 0.55, 0.95) = (0.030, 0.066, 0.114)
```

A faint blue floor under everything.

---

<a name="16-lighting-ii--lambert-diffuse"></a>
## 16. Lighting II — Lambert diffuse

**Lambert's cosine law:** a matte surface's brightness is proportional to the
cosine of the angle between the normal `N` and the light direction `L`. Why
cosine? A fixed beam of light spreads over **more** surface area (so is dimmer
per area) as it arrives more edge‑on; that geometric spreading is exactly
`cos θ = N·L` for unit vectors.

```
diff    = max( N · L, 0 )
diffuse = diff · lightColor · objectColor
```

**Why clamp to 0?** When the light is behind the surface, `θ > 90°` and `N·L`
goes negative. Negative light is meaningless (and causes artifacts), so we
floor it at 0 with `max(…, 0)`.

**Reading the value:**

| `diff` | angle | meaning |
|------:|:-----:|:--------|
| `1.0` | 0°    | light hits dead‑on → brightest |
| `0.5` | 60°   | half brightness |
| `0.0` | ≥ 90° | grazing or behind → no diffuse |

**Worked example.** Top face normal `N = (0,1,0)`; light at `(4, 2.2, 0)`,
surface point `(0, 1, 0)`:

```
L_raw = (4−0, 2.2−1, 0−0) = (4, 1.2, 0)
|L_raw| = √(16 + 1.44) = √17.44 ≈ 4.176
L = (0.958, 0.287, 0)

diff = max(N·L, 0) = max(0.287, 0) = 0.287

diffuse = 0.287 · (1.0,0.97,0.9) · (0.25,0.55,0.95)
        = 0.287 · (0.250, 0.534, 0.855)
        ≈ (0.0717, 0.1532, 0.2453)
```

---

<a name="17-lighting-iii--blinn-phong-specular"></a>
## 17. Lighting III — Blinn‑Phong specular

Specular is the shiny mirror‑like hotspot. **Blinn‑Phong** uses the
**half‑vector** `H`, the unit vector halfway between the light direction `L`
and the view direction `V`:

```
H = normalize(L + V)
spec     = pow( max(N · H, 0), shininess )
specular = specularStrength · spec · lightColor
```

**Intuition:** if the surface were a tiny mirror, it would reflect the light
straight into your eye exactly when `N` lines up with `H`. So `N·H` measures
"how close to a perfect mirror bounce we are."

**Why Blinn‑Phong over original Phong?** Original Phong reflects `L` about `N`
to get `R`, then uses `R·V`. Blinn‑Phong's `N·H` is cheaper, avoids Phong's
artifact where the highlight is wrongly cut off past 90°, and better matches
real materials at grazing angles.

**The shininess exponent** controls highlight tightness. Since `N·H ≤ 1`,
raising it to a high power shrinks it fast:

| shininess | highlight | looks like |
|----------:|:----------|:-----------|
| 4   | broad, soft  | chalky plastic |
| 32  | medium gloss | painted plastic |
| 256 | tiny, sharp  | polished metal / glass |

**Why is specular the light's color, not the surface color?** On dielectrics
(plastic, paint), the highlight is light bouncing off the surface *before*
entering the pigment, so it keeps the **light's** color (usually white). That's
why a red ball has a white highlight.

**Worked example.** From Section 16, `L = (0.958, 0.287, 0)`. Suppose the
viewer direction is `V = (0, 0.6, 0.8)`. Then:

```
L + V = (0.958, 0.887, 0.8)
|L+V| = √(0.918 + 0.787 + 0.64) = √2.345 ≈ 1.531
H = (0.626, 0.579, 0.522)

N·H (with N = (0,1,0)) = 0.579

shininess = 32:  spec = 0.579^32 ≈ 2.6e−8   → essentially no highlight here
shininess = 4 :  spec = 0.579^4  ≈ 0.113     → a soft glow
```

This shows how a high exponent makes the highlight vanish unless `N·H` is very
close to 1 (i.e. you're looking almost exactly at the mirror bounce).

---

<a name="18-complete-lighting-calc"></a>
## 18. A complete per‑pixel lighting calculation

Putting Sections 15–17 together, the final fragment color is:

```
color = ambient + diffuse + specular
```

with specular gated to 0 whenever `diff ≤ 0` (a face turned away from the light
can't glint).

**Full worked pixel.** Use the **top face** (`N = (0,1,0)`) at point
`P = (0,1,0)`:

```
Inputs:
  lightPos    = (4, 2.2, 0)      objectColor      = (0.25, 0.55, 0.95)
  viewDir V   = (0, 0.6, 0.8)    lightColor       = (1.0, 0.97, 0.9)
  ambientStr  = 0.12             specularStr      = 0.9
  shininess   = 4

1) L = normalize(lightPos − P) = (0.958, 0.287, 0)        [Section 16]
2) ambient  = 0.12·(0.25,0.55,0.95)          = (0.030, 0.066, 0.114)
3) diff     = max((0,1,0)·L, 0) = 0.287
   diffuse  = 0.287·(1,0.97,0.9)·(0.25,0.55,0.95) ≈ (0.072, 0.153, 0.245)
4) H = normalize(L+V) = (0.626, 0.579, 0.522)
   spec     = max((0,1,0)·H,0)^4 = 0.579^4 ≈ 0.113
   specular = 0.9·0.113·(1,0.97,0.9) ≈ (0.101, 0.098, 0.091)
5) color = ambient + diffuse + specular
         ≈ (0.030+0.072+0.101, 0.066+0.153+0.098, 0.114+0.246+0.091)
         ≈ (0.203, 0.318, 0.451)
```

Final pixel ≈ RGB `(0.20, 0.32, 0.45)` — a medium‑bright blue with a slightly
whitened sheen from the specular term. Components above 1.0 would simply clamp
to white on display.

---

<a name="19-cheat-sheet"></a>
## 19. Quick formula cheat‑sheet

```
VECTORS
  |a|           = √(aₓ²+a_y²+a_z²)
  â             = a / |a|
  a·b           = aₓbₓ+a_yb_y+a_zb_z = |a||b|cosθ   (= cosθ if both unit)
  a×b           = (a_yb_z−a_zb_y, a_zbₓ−aₓb_z, aₓb_y−a_ybₓ)

TRANSFORMS (column vectors, v' = M·v, rightmost applied first)
  translate(t)  : last column = (tx,ty,tz); needs point w=1
  rotateX(θ)    : y'=y c − z s,  z'=y s + z c
  rotateY(θ)    : x'=x c + z s,  z'=−x s + z c
  perspective   : f=1/tan(fov/2);  w_out = −z_camera
  MVP           : gl_Position = P · V · M · (x,y,z,1)
  perspective ÷ : NDC = clip.xyz / clip.w

NORMALS
  faceNormal    = normalize((v1−v0)×(v2−v0))
  normalMatrix  = transpose(inverse(M₃ₓ₃))  (= rotation if no non‑uniform scale)

LIGHTING (all direction vectors unit length)
  L = normalize(lightPos − P)      V = normalize(eye − P)
  H = normalize(L + V)
  ambient  = ambientStrength · objectColor
  diffuse  = max(N·L, 0) · lightColor · objectColor
  specular = specularStrength · max(N·H, 0)^shininess · lightColor   (0 if N·L≤0)
  color    = ambient + diffuse + specular
```

---

*Cross‑references: matrix code lives in `webgl-cube-mvp.html` SECTION 3;
normal‑matrix and lighting code live in `cube_lighting.html` PART B and the
fragment shader.*
