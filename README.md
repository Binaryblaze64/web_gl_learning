# WebGL Learning

A hands-on, from-scratch exploration of **raw WebGL** — no Three.js, no helper
libraries, no build step. Every matrix, shader, and lighting term is written out
by hand and explained in comments so the whole graphics pipeline is visible and
hackable. This is a personal learning project, but each file doubles as a
self-contained tutorial you can read top-to-bottom.

> Everything here is plain WebGL 1 + GLSL. Just open the HTML files in a browser.

## What's inside

| File | What it demonstrates |
|------|----------------------|
| [`webgl-cube-mvp.html`](webgl-cube-mvp.html) | A rotating, vertex-colored cube built around the **Model · View · Projection** matrix pipeline. Matrix math (identity, translate, rotate, perspective, multiply) is implemented from scratch, with a live debug panel printing the matrices as they update. |
| [`cube_lighting.html`](cube_lighting.html) | A lit cube using surface normals, the normal matrix, and the **Ambient + Lambert diffuse + Blinn-Phong specular** lighting model. Includes an orbiting light, mouse-drag camera, and toggles to isolate each lighting term. |
| [`GRAPHICS_MATH.md`](GRAPHICS_MATH.md) | A computational reference for *all* the math used above — vectors, dot/cross products, homogeneous coordinates, the view & perspective matrices, the perspective divide, normals, and the lighting equations. Each topic gives the idea, the formula, and a fully worked numeric example you can verify by hand. |

## Topics covered

- Compiling and linking GLSL vertex/fragment shaders
- Building 4×4 transform matrices from scratch (translation, rotation, perspective)
- Column-vector / column-major conventions and why they matter to WebGL
- Interleaved vertex buffers (`[x, y, z, r, g, b]`) and attribute pointers
- Passing uniforms (matrices, light parameters) to the GPU
- The `requestAnimationFrame` render loop and depth testing
- Surface normals and the inverse-transpose normal matrix
- The ambient + diffuse + specular lighting model, term by term

## Running it

No server, no dependencies, no install. Just open a file in any modern browser:

- Double-click `webgl-cube-mvp.html` or `cube_lighting.html`, **or**
- Serve the folder locally if you prefer:

  ```bash
  python -m http.server 8000
  # then visit http://localhost:8000/cube_lighting.html
  ```

### Controls (`cube_lighting.html`)

| Input | Action |
|-------|--------|
| `1` | Full lighting (ambient + diffuse + specular) |
| `2` | Ambient only |
| `3` | Diffuse only (Lambert) |
| `4` | Specular only (Blinn-Phong) |
| `5` | Visualize normals as RGB |
| `L` | Pause / resume the orbiting light |
| Mouse drag | Orbit the camera |
| Scroll wheel | Zoom |
| Sliders | Adjust ambient & specular strength |

## Why raw WebGL?

Libraries like Three.js hide the interesting parts. The goal here is to
understand *exactly* what happens between a list of vertices and a lit pixel on
screen — so everything is built up by hand and explained inline.
