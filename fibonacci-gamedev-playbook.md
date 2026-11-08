# Fibonacci & Golden-Ratio Game-Dev Playbook

Six practical, performance-minded uses of the golden ratio (φ) in game code.

---

## Background: why everything lands on 1.618

Start with **any two numbers** and keep adding the last two:

```
a, b, a+b, a+2b, 2a+3b, 3a+5b, ...
```

The ratio `term[n+1] / term[n]` converges to **φ = 1.6180339887…**, and the reverse ratio converges to **1/φ = 0.6180339887…** (which is exactly φ − 1).

Example with seeds −500,000,000 and 14:

| Step | Term | Ratio to previous |
|---|---|---|
| 1 | −500,000,000 | |
| 2 | 14 | ≈ 0 |
| 3 | −499,999,986 | ≈ −35,714,285 |
| 4 | −499,999,972 | 1.000 |
| 5 | −999,999,958 | 2.000 |
| 6 | −1,499,999,930 | 1.500 |
| 7 | −2,499,999,888 | 1.667 |
| 8 | −3,999,999,818 | 1.600 |
| 9 | −6,499,999,706 | 1.625 |
| 10 | −10,499,999,524 | 1.615 |

**Why:** every such sequence is `A·φⁿ + B·(−1/φ)ⁿ`. The second part decays, so the first part takes over.

- **Convergence speed:** the error shrinks by about 0.38× per step, roughly two correct digits every five terms.
- **Exceptions:** seeds of `0, 0`, or seeds already in the exact ratio −0.618… (impossible with whole numbers).
- **Overflow warning:** with the seeds above, term 7 already overflows a 32-bit int. Standard Fibonacci overflows `int32` at F47 and `int64` at F93.

---

## 1. Fibonacci hashing

**What:** multiply the key by 2⁶⁴/φ and keep the **top** bits.

**Why it's fast:** one multiply and one shift, with no modulo. Multiplying by an irrational-derived constant spreads consecutive keys across the table instead of clustering them. This is useful when keys are sequential IDs or grid coordinates.

```cpp
// Table size must be 2^bits.
inline uint32_t fib_hash(uint64_t key, int bits) {
    return uint32_t((key * 0x9E3779B97F4A7C15ull) >> (64 - bits));
}

// Spatial hash for a 2D grid cell (broad-phase collision, etc.)
inline uint32_t cell_hash(int32_t cx, int32_t cy, int bits) {
    uint64_t k = (uint64_t(uint32_t(cx)) << 32) | uint32_t(cy);
    return fib_hash(k, bits);
}
```

**Where to use it:** entity lookup tables, spatial-hash buckets for collision broad-phase, sound and asset caches, and combining hashes (`seed ^= h + 0x9E3779B9 + (seed << 6) + (seed >> 2)`, Boost-style).

**Pitfalls:**
- Take the **high** bits. The low bits of the product are poorly mixed.
- This is not a cryptographic hash and won't stop adversarial keys. That rarely matters in a game.

---

## 2. Even spreading without random numbers (golden-ratio sequence)

**What:** `x[n] = fract(x[n−1] + 0.6180339887)`. Each new value lands in the largest remaining gap of [0, 1). This is a *low-discrepancy* sequence.

**Why it's an optimisation:** random samples clump and leave holes, so you need more of them to hide noise. Evenly spread samples reach the same quality with fewer samples, which means less GPU work per frame.

```cpp
// 1D: golden-ratio sequence
inline float golden1D(uint32_t n) {
    double v = 0.5 + n * 0.6180339887498949;   // double avoids precision loss at large n
    return float(v - floor(v));
}

// 2D: R2 sequence (uses the "plastic number" g ≈ 1.32471795724)
inline void r2(uint32_t n, float& x, float& y) {
    const double g  = 1.32471795724474602596;
    const double a1 = 1.0 / g, a2 = 1.0 / (g * g);
    double vx = 0.5 + a1 * n, vy = 0.5 + a2 * n;
    x = float(vx - floor(vx));
    y = float(vy - floor(vy));
}

// Distinct colours: step the hue by 0.618 each time
float hue = 0.0f;
Color nextTeamColour() {
    hue = fmodf(hue + 0.6180339887f, 1.0f);
    return hsvToRgb(hue, 0.65f, 0.95f);
}
```

**Where to use it:**
- **TAA / upscaler jitter:** sub-pixel offsets per frame (R2 or Halton).
- **Temporal dithering:** offset a blue-noise texture by `golden1D(frameIndex)` each frame, so the noise cycles evenly over time.
- **Distinct colours:** for players, debug draw, or map regions. Every new colour is maximally far from the ones before.
- **Spawn and loot placement:** looks random, never piles up.

**Pitfalls:** in single-precision floats, `n * 0.618` loses precision once `n` reaches the millions. Use doubles, or accumulate and wrap each step.

---

## 3. Points on a sphere / disk (golden angle)

**What:** the golden angle is `π·(3 − √5) ≈ 2.39996 rad ≈ 137.508°`. Rotating by it for each new point, while stepping height or radius evenly, gives near-uniform coverage. It needs no rejection sampling and no lookup table.

```cpp
const float GOLDEN_ANGLE = 2.39996322972865332f;

// N near-uniform points on a unit sphere
Vec3 fibSphere(int i, int N) {
    float y = 1.0f - (i + 0.5f) * 2.0f / N;   // -1..1
    float r = sqrtf(1.0f - y * y);
    float t = i * GOLDEN_ANGLE;
    return { r * cosf(t), y, r * sinf(t) };
}

// Hemisphere (e.g. SSAO kernel, z-up)
Vec3 fibHemisphere(int i, int N) {
    float z = 1.0f - (i + 0.5f) / N;          // 1..0
    float r = sqrtf(1.0f - z * z);
    float t = i * GOLDEN_ANGLE;
    return { r * cosf(t), r * sinf(t), z };
}

// Disk ("Vogel spiral"): PCF shadow taps, depth-of-field bokeh
Vec2 vogelDisk(int i, int N, float rotation) {
    float r = sqrtf((i + 0.5f) / N);
    float t = i * GOLDEN_ANGLE + rotation;
    return { r * cosf(t), r * sinf(t) };
}
```

**Where to use it:**
- **SSAO / GTAO sample kernels:** good coverage with fewer taps.
- **Soft-shadow PCF:** Vogel disk taps. Rotate per pixel with blue noise to turn banding into fine grain.
- **Light probes and reflection captures** placed around a sphere.
- **Procedural scattering:** asteroids, stars, foliage (sunflower heads, pine cones).

**Pitfalls:** this is deterministic. Without a per-pixel or per-frame rotation, the same pattern repeats and can show up as visible structure.

---

## 4. Golden-section search

**What:** finds the minimum (or maximum) of a function that has a single best point in an interval `[a, b]`. Each step shrinks the interval to 0.618 of its size, and **one** of the two interior points is reused, so each step needs only **one** new evaluation.

**Why it's an optimisation:** naive ternary search shrinks by 2/3 per step but needs two new evaluations each time. Golden-section search reaches the same precision with **roughly 55–60% fewer function calls**. That matters when each call is expensive: a raycast, a physics sub-step, or a simulated AI rollout.

```cpp
template <class F>
double goldenSectionMin(F f, double a, double b, double tol = 1e-4) {
    const double invphi = 0.6180339887498949;
    double c = b - invphi * (b - a);
    double d = a + invphi * (b - a);
    double fc = f(c), fd = f(d);
    while (b - a > tol) {
        if (fc < fd) {            // minimum is in [a, d]
            b = d; d = c; fd = fc;
            c = b - invphi * (b - a); fc = f(c);
        } else {                  // minimum is in [c, b]
            a = c; c = d; fc = fd;
            d = a + invphi * (b - a); fd = f(d);
        }
    }
    return 0.5 * (a + b);
}
```

**Where to use it:**
- **Camera systems:** the best boom distance that avoids occlusion while staying close to the preferred framing.
- **Line search** inside IK solvers, cloth or physics solvers, and gradient-descent tuners.
- **Projectile aiming:** the launch angle that minimises miss distance.
- **Offline tuning:** a balance parameter that maximises a simulated win-rate score.

**Pitfalls:**
- It only works if the function has a single best point in the interval. With several local minima, it finds *a* minimum, not necessarily the best one.
- To maximise, minimise `-f(x)`.

---

## 5. Dynamic array growth factor below φ

**What:** when a growable buffer reallocates, multiply its capacity by a factor `k`. If **k < φ**, the memory freed by earlier sizes eventually adds up to enough for a new allocation, so the allocator can reuse that space. If **k ≥ φ** (for example 2×), the freed blocks are always too small, because each new block is larger than all the previous ones combined.

**Why it's an optimisation:** less fragmentation and better reuse of memory in custom allocators, arenas and pools. This is why MSVC's `std::vector` and Folly's `fbvector` grow by 1.5×.

```cpp
template <class T>
void growIfFull(T*& data, size_t& size, size_t& cap) {
    if (size < cap) return;
    size_t newCap = cap ? cap + cap / 2 : 8;   // 1.5x growth, below phi
    T* nd = allocator.alloc<T>(newCap);
    memcpy(nd, data, size * sizeof(T));        // trivially copyable T
    allocator.free(data);
    data = nd; cap = newCap;
}
```

**The maths:** with factor `k`, the new block has size `kⁿ`. The freed blocks before the current one total `1 + k + … + kⁿ⁻²`. That total can eventually match `kⁿ` only when `k² < k + 1`, which means `k < φ`.

**Pitfalls:**
- The benefit only appears if the allocator places these blocks next to each other and merges them. A general-purpose `malloc` may not.
- 2× growth means fewer reallocations. Choose based on whether memory or CPU time is tighter. **Best fix of all:** `reserve()` up front when you know the size.

---

## 6. Cheap exponential growth curves

**What:** any two integer seeds converge to ×1.618 growth per step, so you can build an exponential progression table with **two integer additions per entry**. You never call `pow()`, and integers never drift the way floats do.

```cpp
// seedA/seedB shape the early game; later entries converge to ~x1.618 per level.
void buildXpTable(int64_t seedA, int64_t seedB, int64_t* out, int levels) {
    int64_t a = seedA, b = seedB;
    for (int i = 0; i < levels; ++i) {
        out[i] = a;
        int64_t c = a + b;
        if (c < b) { /* overflow guard: clamp or stop */ c = INT64_MAX; }
        a = b; b = c;
    }
}

// Example: buildXpTable(100, 150, table, 30)
// -> 100, 150, 250, 400, 650, 1050, 1700, ...
```

**Where to use it:** XP-per-level tables, upgrade costs, enemy HP scaling, idle and incremental games, and wave sizes.

**Tuning tips:**
- **Gentler than φ:** use a weighted recurrence such as `c = a + b/2`, or blend with a linear term. Pure ×1.618 per level is steep.
- **Steeper start:** pick seeds that are far apart. The ratio swings early, then settles.
- The table is integer-only and deterministic, so it gives the same values on every platform. That's good for multiplayer and save files.

**Pitfalls:**
- **Overflow:** use 64-bit integers, guard or clamp, or switch to `double` for idle games where numbers get huge.
- Negative or zero seeds can flip signs during the first few steps.

---

## Quick reference

| Constant | Value | Used for |
|---|---|---|
| φ | 1.6180339887498949 | Growth ratio, growth-factor limit |
| 1/φ = φ − 1 | 0.6180339887498949 | 1D sequence, golden-section search |
| 2⁶⁴/φ | `0x9E3779B97F4A7C15` | 64-bit Fibonacci hashing |
| 2³²/φ | `0x9E3779B9` | 32-bit hashing, hash combining |
| Golden angle | 2.39996322972865332 rad (137.508°) | Sphere, hemisphere, disk point sets |
| Plastic number g | 1.32471795724474602596 | R2 2D sequence |

## Where to start

1. **Fibonacci hashing (#1):** a one-line change in any power-of-two hash table. Measure collision chains before and after.
2. **Golden-ratio and R2 sequences (#2):** swap out random jitter or sampling and compare image noise at equal sample counts.
3. **Vogel disk / Fibonacci hemisphere (#3):** replace hand-written or random sample kernels in SSAO and shadows.
