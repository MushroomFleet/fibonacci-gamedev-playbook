# 🌀 Fibonacci Game-Dev Playbook

**Practical, performance-minded uses of the golden ratio (φ ≈ 1.618) in game development, with copy-paste code.**

It started with a simple observation: take **any two numbers**, positive or negative, and keep adding the last two together, Fibonacci-style. The ratio between neighbouring terms always settles at **1.618** (or **0.618** read the other way). Seeds as wild as `-500,000,000` and `14` still get there within a few dozen steps.

That number turns up throughout performance-sensitive game code. This playbook collects six places where it does real work, explains why each one helps, and gives you code you can drop straight into an engine.

---

## 🎯 What it's for

This repo is a **reference for game programmers and technical artists** who want cheap, deterministic tools for:

- faster hash tables and spatial hashing
- noise-free sampling for rendering (TAA, AO, shadows, dithering)
- even procedural placement without random clumping
- tuning parameters with fewer expensive evaluations
- less memory fragmentation in growable buffers
- integer-only progression curves (XP, costs, scaling)

Each technique is short, needs no dependencies, and fits in a few lines of C++ or shader code.

---

## 📖 What's inside

The full write-up is in **[`fibonacci-gamedev-playbook.md`](./fibonacci-gamedev-playbook.md)**.

| # | Technique | What it optimises |
|---|---|---|
| 0 | **Background: why everything lands on 1.618** | The maths, how fast it converges, overflow warnings |
| 1 | **Fibonacci hashing** (`0x9E3779B97F4A7C15`) | One multiply and one shift spreads keys evenly; good for spatial hashing and broad-phase collision |
| 2 | **Golden-ratio and R2 sequences** | Even sampling for TAA jitter, temporal dithering, distinct colours and spawn points |
| 3 | **Golden-angle point sets** (Fibonacci sphere, Vogel disk) | SSAO kernels, PCF shadow taps, light probes, procedural scattering |
| 4 | **Golden-section search** | Finds an optimum with about 55–60% fewer evaluations than ternary search |
| 5 | **Growth factor below φ** | Lets dynamic arrays reuse freed memory and fragment less |
| 6 | **Additive growth curves** | Exponential XP and cost tables using only integer additions, with no `pow()` |

The playbook ends with a **quick-reference table of constants** and a recommended order to try things in.

---

## ⚡ Quick taste

```cpp
// Fibonacci hashing: table size = 2^bits
inline uint32_t fib_hash(uint64_t key, int bits) {
    return uint32_t((key * 0x9E3779B97F4A7C15ull) >> (64 - bits));
}

// Golden-ratio sequence: each value fills the biggest gap in [0, 1)
inline float golden1D(uint32_t n) {
    double v = 0.5 + n * 0.6180339887498949;
    return float(v - floor(v));
}

// Near-uniform points on a sphere, no rejection loop
Vec3 fibSphere(int i, int N) {
    const float GOLDEN_ANGLE = 2.39996322972865332f;
    float y = 1.0f - (i + 0.5f) * 2.0f / N;
    float r = sqrtf(1.0f - y * y);
    float t = i * GOLDEN_ANGLE;
    return { r * cosf(t), y, r * sinf(t) };
}
```

---

## 🚀 Where to start

1. **Fibonacci hashing.** It's a one-line change to any power-of-two hash table. Measure collision chains before and after.
2. **Golden-ratio and R2 sequences.** Replace random jitter or sampling, then compare noise at the same sample count.
3. **Vogel disk / Fibonacci hemisphere.** Swap in better sample kernels for SSAO and soft shadows.

---

## 🧮 The core idea in one line

Every "add the last two" sequence equals `A·φⁿ + B·(−1/φ)ⁿ`. The second part fades away, so **φ always wins**. The only exceptions are seeds of `0, 0` or seeds already in the exact ratio −0.618…, which whole numbers can never be.

---

## 🤝 Contributing

Have another golden-ratio trick that earns its place in a game engine? Found a bug in a snippet? Issues and pull requests are welcome. Please include a short explanation of *why* the technique helps performance or quality, plus a minimal code example.

---

## 📚 Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{fibonacci_gamedev_playbook,
  title = {Fibonacci Game-Dev Playbook: Practical golden-ratio techniques for game development optimisation},
  author = {Drift Johnson},
  year = {2026},
  url = {https://github.com/MushroomFleet/fibonacci-gamedev-playbook},
  version = {1.0.0}
}
```

### Donate:

[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
