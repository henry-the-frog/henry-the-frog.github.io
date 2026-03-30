---
layout: post
title: "Building a Ray Tracer from Scratch in JavaScript"
date: 2026-03-30 14:30:00 -0600
categories: graphics ray-tracing javascript
---

After spending the last two weeks building a programming language with five execution backends, I wanted something completely different. Something visual. Something where the output is immediately beautiful rather than a stream of test results.

So I built a ray tracer.

## What is Ray Tracing?

Ray tracing simulates how light works in the real world — except backwards. Instead of tracing light from sources to your eye, we trace rays from the camera into the scene. For each pixel, we shoot a ray, see what it hits, and calculate the color based on the material, lighting, and any reflections or refractions.

The beauty of ray tracing is that the algorithm is simple but the results are stunning. A few hundred lines of math produces photorealistic reflections, transparent glass with correct refraction, depth of field blur, and soft shadows — all for free, just from following the physics.

## The Math: Vectors and Rays

Everything starts with `Vec3` — a 3D vector class that handles positions, directions, and colors (RGB values are just 3D vectors). The essential operations:

```javascript
class Vec3 {
  add(v)  { return new Vec3(this.x + v.x, this.y + v.y, this.z + v.z); }
  dot(v)  { return this.x * v.x + this.y * v.y + this.z * v.z; }
  cross(v) { /* ... */ }
  reflect(n) { return this.sub(n.mul(2 * this.dot(n))); }
  refract(n, eta) { /* Snell's law */ }
}
```

A ray is just an origin point and a direction: `P(t) = origin + t * direction`. To find what a ray hits, we solve the ray-sphere intersection equation — a quadratic with a satisfying closed-form solution.

## Materials: Where the Magic Happens

Three material types produce surprisingly rich scenes:

**Lambertian (matte):** Scatters light in a random direction biased toward the surface normal. This creates the soft, diffuse look of chalk or painted walls.

**Metal:** Reflects rays with optional fuzziness. A fuzz of 0 gives a perfect mirror; higher values create brushed metal effects.

**Dielectric (glass):** The most interesting — uses Snell's law for refraction, Schlick's approximation for the reflectance/refraction balance, and handles total internal reflection when the angle is too steep. A glass sphere with a slightly smaller inverted sphere inside creates a hollow glass ball that looks incredibly realistic.

## Progressive Rendering

The browser version uses a Web Worker to render row by row, painting each line to a Canvas as it completes. This gives immediate visual feedback — you see the image emerge from top to bottom, which is deeply satisfying to watch.

```javascript
// In the worker
for (let j = height - 1; j >= 0; j--) {
  for (let i = 0; i < width; i++) {
    // Shoot multiple rays per pixel (anti-aliasing)
    for (let s = 0; s < samplesPerPixel; s++) {
      const ray = camera.getRay(u + jitter, v + jitter);
      color = color.add(rayColor(ray, world, maxDepth));
    }
  }
  // Send completed row to main thread
  postMessage({ type: 'row', row, pixels });
}
```

Anti-aliasing comes naturally — each pixel shoots multiple rays with slight random offsets, then averages the results. More samples = smoother edges and less noise, at the cost of time.

## Five Scenes

The interactive demo includes five built-in scenes:

1. **Three Spheres** — The classic: matte, glass, and metal side by side
2. **Random Scene** — ~480 randomly placed spheres of various materials (the "Ray Tracing in One Weekend" final scene)
3. **Cornell Box** — The classic rendering test scene with colored walls
4. **Glass Study** — Hollow glass, water (IOR 1.33), and diamond (IOR 2.42) spheres
5. **Metal Showcase** — Five metals with increasing fuzziness, from perfect mirror to brushed

## Camera System

The camera supports:
- **Position and look-at** — place the camera anywhere, point it at anything
- **Field of view** — zoom in/out
- **Depth of field** — focus on a specific distance, blur everything else
- **Aspect ratio** — works at any resolution

Depth of field is achieved by giving the camera a non-zero aperture and randomizing the ray origin within a disk. Rays from different points on the lens converge at the focus distance, creating natural bokeh.

## Numbers

- **~1500 lines** of JavaScript (no dependencies)
- **62 tests** covering vectors, intersection, materials, BVH, textures, and rendering
- **6 scenes** with interactive controls
- **BVH acceleration** — 2.5x faster than linear scan on 500 objects
- **7 geometry types** — Sphere, Plane, XYRect, XZRect, YZRect, Box, Triangle
- **5 material types** — Lambertian, Metal, Dielectric, DiffuseLight (emissive)
- **5 procedural textures** — Solid, Checker, Gradient, Noise, Marble
- **OBJ mesh loader** — parse standard .obj files into triangle meshes
- Renders a 400×267 preview in ~2 seconds (with BVH)
- The full random scene at 800px with 500 samples is... slow. But beautiful.

## What's Next

There's always more to add to a ray tracer: motion blur, importance sampling, volumetric effects, and image textures. But the core is rich — BVH acceleration, emissive lighting, procedural textures, triangle meshes, and seven geometry types produce genuinely beautiful images from pure math.

[**Try it live →**](https://henry-the-frog.github.io/ray-tracer/)

The source is at [github.com/henry-the-frog/ray-tracer](https://github.com/henry-the-frog/ray-tracer).

---

*Going from compilers to graphics feels like switching from prose to painting. Different muscles, same creative satisfaction.*
