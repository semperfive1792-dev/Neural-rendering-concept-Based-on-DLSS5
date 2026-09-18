# Photonic Rendering Pipeline

**Author:** Semperfive  
**Date:** September 2026  
**License:** Creative Commons Attribution 4.0 International (CC BY 4.0)  
**Publication status:** Architectural concept / research proposal / defensive publication

---

## Abstract

Modern graphics systems increasingly compensate for the limitations of their underlying architecture by adding additional processing stages.

Higher image quality introduces additional reconstruction algorithms.  
Higher resolution introduces wider interfaces and compression.  
Variable refresh introduces synchronization protocols.  
HDR introduces additional processing and display-side control.  
Neural rendering introduces increasingly complex data movement between compute units, memory and display pipelines.

This document proposes a different architectural approach.

Instead of continuously optimizing the conventional digital display pipeline, the proposed architecture removes unnecessary representations between rendering and light emission.

The central idea is to divide the graphics system into two domains:

1. **Electronic domain** — general-purpose computation, control, memory, simulation, operating-system interaction, scheduling and other tasks for which conventional electronics remain appropriate.
2. **Photonic domain** — neural rendering, selected image transformations, high-bandwidth signal distribution and optical image formation.

The system therefore does not attempt to build a fully photonic computer. Instead, a conventional electronic processor is combined with one or more photonic chiplets. Neural rendering and image-domain operations that naturally map to optical computation are performed in the photonic domain. The resulting optical representation is then transmitted directly to an optical display subsystem without first being converted back into a conventional digital framebuffer and serialized through a traditional display interface.

```
Application / Engine
        │
        ▼
Electronic CPU / GPU / NPU
        │
        │ scene data, control, geometry,
        │ simulation, memory, scheduling
        ▼
Photonic Rendering Chiplet
        │
        │ neural rendering
        │ reconstruction
        │ optical image transformation
        ▼
Optical Image Representation
        │
        │ waveguide / fiber / photonic interconnect
        ▼
Optical Output Module
        │
        │ spatial optical formation
        ▼
Laser / Phosphor / Projection System
        │
        ▼
Human visual system
```

---

## 1. Core Thesis

The fundamental observation behind this architecture is that a display ultimately produces light.

Nevertheless, conventional graphics systems often perform a long sequence of conversions before reaching that final physical representation:

```
Scene
  ↓
Electronic computation
  ↓
Digital framebuffer
  ↓
Digital image processing
  ↓
Digital serialization
  ↓
Digital transmission
  ↓
Digital reconstruction
  ↓
Panel timing
  ↓
Pixel driving
  ↓
Optical emission
```

The proposal asks a simple question: **Which of these intermediate representations are actually necessary?**

If a portion of rendering is already being performed by an analog or photonic computational system, and the final destination is an optical field, then repeatedly converting the information between electronic and optical representations may be unnecessary.

The proposed architecture therefore attempts to preserve the optical representation for as much of the final image pipeline as practical.

> "Do not optimize an unnecessary representation. Remove it."

---

## 2. Design Philosophy

The proposal is based on five principles.

### 2.1 Heterogeneity instead of technological purity

The system should not attempt to replace electronics wholesale. Electronics remain the preferred medium for:

- operating-system execution
- application logic
- scene management
- physics simulation
- control flow
- branching
- memory management
- scheduling
- general-purpose computation
- device management
- error handling
- configuration
- system control

Photonic hardware is introduced only where its properties provide a useful architectural advantage.

### 2.2 Compute where the mathematics is naturally suited to optics

Many neural-network operations consist primarily of large linear transformations, weighted combinations and massively parallel signal propagation. These are natural candidates for photonic computation.

The purpose is not to "make the GPU photonic." The purpose is to move selected computational workloads into a photonic accelerator when doing so reduces energy, latency or data movement.

### 2.3 Preserve optical data as optical data

The conventional architecture frequently follows:

```
optical → electronic → digital → electronic → optical
```

The proposed architecture attempts to replace this with:

```
electronic → photonic → optical
```

where appropriate. This is particularly relevant when the output of a photonic neural renderer can remain in an optical representation until it reaches the display subsystem.

### 2.4 Separate architecture from implementation technology

The concept does not depend on one particular implementation. The photonic processing element could use different technologies, including:

- Mach–Zehnder interferometer networks
- microring-based architectures
- wavelength-division multiplexing
- free-space optical computing
- diffractive optical elements
- photonic crystal or waveguide structures
- integrated laser sources
- hybrid silicon photonics
- heterogeneous photonic/electronic integration

Likewise, the optical output stage could use:

- laser-phosphor systems
- direct laser projection
- scanning optical systems
- spatial optical emitters
- free-space optical coupling
- other emerging optical display technologies

### 2.5 Remove unnecessary representations

The architecture is therefore intended to remain technology-agnostic at the component level. The guiding principle is architectural: identify which intermediate representations between computation and light emission are unnecessary, and remove them.

---

## 3. Proposed System Architecture

The complete system can be divided into six functional domains.

```
┌──────────────────────────────────────────────────────────┐
│                 ELECTRONIC COMPUTE DOMAIN                │
│                                                          │
│ CPU / GPU / NPU / Memory / Control / Simulation          │
└─────────────────────────┬────────────────────────────────┘
                          │
                          │ neural rendering inputs
                          │ and control
                          ▼
┌──────────────────────────────────────────────────────────┐
│                  PHOTONIC CHIPLET                        │
│                                                          │
│ Neural rendering                                        │
│ Neural reconstruction                                   │
│ Image-domain transforms                                 │
│ Optical broadcast / routing                             │
└─────────────────────────┬────────────────────────────────┘
                          │
                          │ optical image representation
                          ▼
┌──────────────────────────────────────────────────────────┐
│                OPTICAL INTERCONNECT                      │
│                                                          │
│ Waveguide / fiber / photonic interposer                 │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│                OPTICAL OUTPUT MODULE                     │
│                                                          │
│ Spatial conversion / coupling / beam formation          │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│                  OPTICAL DISPLAY                         │
│                                                          │
│ Laser source → phosphor / projection medium → light     │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ▼
                    HUMAN VISUAL SYSTEM
```

---

## 4. Electronic Domain

The electronic processor remains the primary general-purpose computing system. It may contain:

- CPU cores
- conventional GPU compute units
- tensor/NPU accelerators
- memory controllers
- cache hierarchy
- system memory
- storage
- I/O
- operating-system interfaces

The important architectural change is that the electronic processor does not necessarily have to construct and transmit a conventional display frame as its final product. Instead, it provides the photonic subsystem with the information necessary to construct the visual output.

```
Game / application
       ↓
Scene state
       ↓
Geometry / physics / lighting information
       ↓
Neural rendering inputs
       ↓
Photonic rendering chiplet
```

The exact partition is implementation-dependent. The architecture does not require all rendering to become photonic.

---

## 5. Photonic Rendering Chiplet

The photonic chiplet is the central accelerator of the architecture. It is not intended to replace the entire GPU. Its purpose is to execute selected image-generation operations for which photonic computation is advantageous.

Candidate workloads include:

- neural rendering
- neural reconstruction
- super-resolution
- ray reconstruction
- learned denoising
- neural frame synthesis
- feature transformations
- linear image transforms
- optical filtering
- image-domain broadcast
- selected matrix operations

```
Electronic scene representation
             │
             ▼
      photonic input
             │
             ▼
    optical neural network
             │
             ▼
      optical features
             │
             ▼
     image reconstruction
             │
             ▼
      optical image field
```

The photonic chiplet may be integrated into the same package as electronic compute chiplets.

---

## 6. Chiplet Architecture

A heterogeneous chiplet package is a natural implementation model.

```
┌─────────────────────────────────────────────┐
│                 PACKAGE                     │
│                                             │
│  ┌───────┐  ┌───────┐  ┌───────────────┐  │
│  │ CPU   │  │ GPU   │  │ Memory / I/O  │  │
│  └───────┘  └───────┘  └───────────────┘  │
│                                             │
│              ┌────────────────┐             │
│              │ Photonic       │             │
│              │ Rendering      │             │
│              │ Chiplet        │             │
│              └────────────────┘             │
│                                             │
│       Electronic / optical interconnect     │
└─────────────────────────────────────────────┘
```

Different manufacturing processes can be used for different functions:

- advanced CMOS for general compute
- specialized SRAM/cache technology
- silicon photonics for optical processing
- separate laser technology
- photonic interposer for high-bandwidth communication

The architecture therefore avoids the requirement that a single semiconductor process must simultaneously be optimal for every function.

SEECHIP (ICPP 2023) has demonstrated a chiplet-based GPU architecture using photonic inter-chiplet links, achieving reductions in both execution time and energy consumption compared to metallic interconnects, with good scalability [web_15_0_0_11].

---

## 7. Why Photonic Compute Is Used Selectively

A common misunderstanding of this proposal is that it requires a fully photonic processor. It does not.

```
             ELECTRONIC
         ┌─────────────────┐
         │ CPU             │
         │ GPU             │
         │ memory          │
         │ control         │
         │ simulation      │
         └────────┬────────┘
                  │
                  ▼
             PHOTONIC
         ┌─────────────────┐
         │ Neural render   │
         │ reconstruction  │
         │ transformations │
         │ optical routing │
         └────────┬────────┘
                  │
                  ▼
                LIGHT
```

The boundary should be determined experimentally. The guiding criterion is not "electronic versus photonic." The criterion is:

> "Which representation and processing medium minimizes total system cost for a given operation?"

---

## 8. Optical Data Distribution

One potentially important advantage of photonics is optical broadcast.

A conventional electronic architecture may require multiple consumers to read the same data repeatedly:

```
Memory
  ├──→ Compute block A
  ├──→ Compute block B
  ├──→ Compute block C
  └──→ Compute block D
```

An optical architecture can distribute one signal through optical splitting or wavelength/spatial multiplexing:

```
                   ┌──→ Block A
                   │
Optical signal ────┼──→ Block B
                   │
                   ├──→ Block C
                   │
                   └──→ Block D
```

This does not mean that optical splitting is free. Optical power, insertion loss, noise, detector sensitivity and signal integrity remain engineering constraints. The architectural opportunity is that broadcast can occur in the physical domain without requiring equivalent duplication of digital memory traffic.

---

## 9. Optical Image Representation

The most important conceptual transition occurs at the output of the photonic renderer.

The system does not necessarily need to produce `3840 × 2160 × RGB × N bits` as a conventional digital framebuffer. Instead, it can produce an optical field:

> **I(x, y, λ, t)**

where:
- x and y describe spatial position
- λ describes wavelength/spectral content
- t describes time

This does not imply infinite resolution. The resulting image remains limited by:

- diffraction
- optical transfer function
- numerical aperture
- aberrations
- optical noise
- laser bandwidth
- modulation depth
- spatial addressing
- detector/emitter characteristics
- phosphor response
- human visual resolution

The important difference is that resolution becomes a property of an optical system rather than exclusively a property of a fixed rectangular pixel matrix.

---

## 10. Continuous Optical Reconstruction

A conventional display samples an image onto discrete pixels. The optical architecture instead aims to reconstruct the image using a continuous or quasi-continuous optical point-spread function.

```
Digital pixel representation:       Optical reconstruction:

████████                                .+.
████████                              .+### +.
████████                              +#######+
                                        .+### +.
                                          .+.
```

A Gaussian-like point-spread function can act as a physical reconstruction filter.

---

## 11. Anti-Aliasing as a System Property

Anti-aliasing should be considered as a combination of:

1. **Scene sampling** — how the 3D scene is sampled during rendering
2. **Reconstruction** — how the sampled data is reconstructed into a continuous image
3. **Display sampling** — how the continuous image is sampled by the display aperture
4. **Human visual integration** — how the eye integrates the final light field

The proposed architecture moves part of this chain into optics.

```
Scene
  ↓
Neural / electronic sampling
  ↓
Photonic reconstruction
  ↓
Optical PSF
  ↓
Human visual system
```

rather than:

```
Scene
  ↓
Digital rendering
  ↓
Digital AA
  ↓
Digital framebuffer
  ↓
Pixel sampling
  ↓
Panel optics
  ↓
Eye
```

The optical display can perform part of the reconstruction operation physically, potentially reducing the amount of computation required for display-sampling correction. Neural rendering can then address the remaining scene-sampling problem.

The architecture does not claim "optics makes anti-aliasing free." The more precise statement is that some work traditionally performed numerically can become a property of the physical image-forming system.

This should be validated experimentally using controlled spatial-frequency test patterns.

---

## 12. No Traditional Raster Scan

Traditional flat-panel displays typically depend on a spatially addressed pixel matrix and timing controller. The proposed optical display does not need to preserve this architecture.

If the optical system forms the image field directly, there is no inherent requirement to scan rows of a TFT matrix. This can eliminate a class of raster synchronization artifacts.

However, this does not mean that every temporal artifact becomes impossible. The system can still have:

- frame-transition discontinuities
- temporal modulation artifacts
- laser response limitations
- phosphor persistence
- synchronization errors between rendering and optical emission

The claim is specifically: raster-scan tearing is not an inherent requirement of the proposed display architecture.

---

## 13. Optical Interconnect

The optical image representation can leave the photonic chiplet through:

- integrated waveguides
- photonic interposers
- optical fibers
- free-space coupling

The key architectural goal is to avoid converting the image back into a high-speed digital electrical stream merely for transport.

```
Photonic chiplet
      │
      ▼
optical waveguide
      │
      ▼
optical connector / fiber
      │
      ▼
optical output module
```

This is especially attractive for systems where the compute package and display are physically separated.

---

## 14. Optical Output Module

A standardized optical output module can serve as the interface between the photonic compute system and different physical display implementations.

```
Photonic processor
       │
       ▼
Standard optical interface
       │
       ├──→ laser-phosphor display
       ├──→ direct laser projection
       ├──→ AR optical engine
       ├──→ VR optical engine
       └──→ other optical systems
```

The interface should define:

1. physical optical connection
2. wavelength conventions
3. optical power range
4. spatial representation
5. temporal modulation
6. synchronization
7. calibration
8. error handling
9. safety limits

The objective is not to create another closed proprietary video protocol. The objective is to define a physical optical representation that can be implemented by multiple vendors.

---

## 15. Display Architecture

A representative implementation:

```
Optical input
     ↓
Optical coupling
     ↓
Laser source / emitter array
     ↓
Spatial optical formation
     ↓
Phosphor / projection medium
     ↓
Visible image
```

The display no longer requires a conventional TFT pixel matrix for image formation. Potentially removable components include:

- conventional scaler
- display-side framebuffer
- T-CON
- row/column pixel drivers
- conventional pixel DAC infrastructure
- conventional backlight
- polarizers
- RGB color-filter matrix

The actual component reduction depends on the chosen optical implementation.

---

## 16. Laser-Phosphor as One Possible Implementation

Laser-phosphor technology is a particularly interesting candidate because it already provides:

- high optical brightness
- long operating lifetime
- no conventional LCD backlight leakage
- potentially wide spectral output
- scalable optical excitation

**Prysm LPD (Laser Phosphor Display)** is a commercial implementation of this technology. Key measured characteristics [web_15_0_0_27][web_15_0_0_28]:

- 75% lower power consumption than competing large-format display technologies
- contrast ratio constant vs. viewing angle (laser OFF during black video = true black)
- 240 Hz refresh rate with <2–3 μs phosphor response time
- 178° viewing angle
- aggregate "excitation light to viewer light" throughput >20%
- no motion blur

However, laser-phosphor is not a mandatory component of the architecture. The architecture can also be implemented using direct laser emission or other spatial optical systems.

The general requirement is: the display must accept an optical representation and transform it into the desired visual field without reconstructing a conventional digital framebuffer.

---

## 17. Contrast and Dynamic Range

A laser-based optical system can achieve a very low black level because optical emission can be reduced toward zero rather than being generated by a continuously illuminated backlight.

Prysm LPD demonstrates this: the laser is turned off during black video, producing true black with contrast ratio constant across viewing angles [web_15_0_0_27].

Practical contrast depends on:

- stray light
- phosphor persistence
- optical scattering
- laser extinction ratio
- ambient light
- optical leakage
- measurement methodology

The architecture should treat very high contrast as a potential capability, not as a guaranteed numerical specification.

---

## 18. Color

A photonic display can use wavelength-selective optical sources. Potential implementations include:

- RGB lasers
- multiple wavelength channels
- wavelength-division multiplexing
- broadband excitation plus spectral filtering
- multi-primary systems

**TriLite Trixel 3** demonstrates 214% color gamut over sRGB in a commercial laser beam scanner display [web_15_0_0_30][web_15_0_0_34].

Potential advantages include high spectral purity and gamuts beyond conventional sRGB. However, a larger color gamut is not automatically equivalent to better image quality. The system must consider:

- colorimetric calibration
- white point
- spectral power distribution
- human cone response
- HDR color volume
- laser speckle
- metamerism

---

## 19. Brightness and Power

The potential power advantage comes from architectural simplification, not from the assumption that lasers are inherently efficient.

The total system power must be calculated as:

```
Wall power
   ↓
electronic compute
   ↓
photonic compute
   ↓
laser electrical-to-optical efficiency
   ↓
optical coupling losses
   ↓
waveguide/fiber losses
   ↓
optical conversion efficiency
   ↓
visible optical power
   ↓
perceived image
```

Potential power savings may arise from:

- removal of display-side processing
- elimination of large pixel-driver networks
- elimination of conventional backlight architectures
- reduced digital image movement
- photonic computation efficiency
- optical broadcast
- direct optical image formation

Prysm LPD consumes 75% less power than competing display technologies [web_15_0_0_28]. TriLite Trixel 3 operates at ~320 mW for a full AR display engine [web_15_0_0_30].

---

## 20. Latency

The architecture potentially removes several buffering and serialization stages.

```
Scene update
   ↓
electronic scheduling
   ↓
photonic neural inference
   ↓
optical propagation
   ↓
optical emission
   ↓
visual response
```

rather than:

```
Scene update
   ↓
render
   ↓
framebuffer
   ↓
post-processing
   ↓
scanout
   ↓
serialization
   ↓
transport
   ↓
monitor buffering
   ↓
scaler
   ↓
T-CON
   ↓
pixel scan
   ↓
light
```

ACCEL (Tsinghua, Nature 2023) achieves 72 ns/frame latency — compared to 0.26 ms/frame on NVIDIA A100 — by processing image data entirely in the analog optical domain without ADC conversion [web_15_0_0_4]. OPCA (Tsinghua, Optica 2024) achieves 6 ns response time for end-to-end image processing, transmission, and reconstruction [web_15_0_0_6].

The correct measurement boundary should be: time from a defined scene-state change to the corresponding measured optical change at the display output.

---

## 21. Bandwidth

A common shorthand is to say that "light has unlimited bandwidth." That is not technically accurate.

The correct statement is: optical systems are not intrinsically constrained by the fixed bitrate of a display protocol such as HDMI or DisplayPort. Their usable bandwidth is instead determined by the physical optical system.

Relevant limits include:

- modulation bandwidth
- wavelength count
- spatial channels
- temporal channels
- signal-to-noise ratio
- detector/emitter bandwidth
- coupling efficiency
- optical nonlinearities
- dynamic range

OPCA demonstrates a processing bandwidth of up to 100 billion pixels [web_15_0_0_6].

---

## 22. Resolution

The proposed architecture does not have a fixed pixel count in the same sense as an LCD panel. Instead, image resolution becomes a function of the optical system.

Relevant parameters include:

- wavelength
- numerical aperture
- optical transfer function
- spot size
- spatial modulation capability
- aberration
- phosphor grain/response
- optical geometry
- viewing distance

"Floating resolution" should be understood as: continuously configurable optical spatial resolution within the limits of the optical transfer function.

The MIT ski-jump waveguide achieves submicron diffraction-limited beam spots (0.66 μm × 0.50 μm) with a footprint-adjusted spot rate of 68.6 mega-spots/s·mm² — sufficient for 1 megapixel at 100 Hz from a 1.5 mm² footprint [web_15_0_0_17]. A 64-element array demonstrates a pathway to >1 gigaspot resolution at kHz rates within a sub-5-cm footprint.

---

## 23. What the Architecture Potentially Removes

The goal is not to claim that every existing component is always unnecessary. Rather, the architecture identifies components that may become unnecessary when the optical representation is preserved.

### Compute side — potentially reduced:

| Component | Status |
|-----------|--------|
| Conventional display controller | Potentially removed |
| Scanout engine | Potentially removed |
| High-speed display serializer | Potentially removed |
| DSC (Display Stream Compression) for display transport | Potentially removed |
| HDMI/DisplayPort PHY | Potentially removed |
| Protocol-specific display logic | Potentially removed |

### Interconnect — potentially reduced:

| Component | Status |
|-----------|--------|
| High-speed electrical differential pairs | Potentially removed |
| Retimers | Potentially removed |
| Redrivers | Potentially removed |
| Protocol conversion | Potentially removed |

### Display side — potentially reduced:

| Component | Status |
|-----------|--------|
| Scaler | Potentially removed |
| Display framebuffer | Potentially removed |
| T-CON | Potentially removed |
| Row/column drivers | Potentially removed |
| Conventional pixel DAC architecture | Potentially removed |
| Conventional backlight | Potentially removed |
| Polarizers | Potentially removed |
| RGB color-filter matrix | Potentially removed |

The exact reduction depends on implementation.

---

## 24. What Does NOT Disappear

The architecture does not eliminate the need for:

- **Memory** — large persistent state remains primarily electronic
- **Control logic** — scheduling, configuration, error handling
- **Calibration** — wavelength, temperature, emitter intensity, phase, spatial alignment, phosphor response
- **Thermal management** — lasers and photonic components generate heat
- **Power electronics** — laser drivers, voltage regulation, power distribution
- **Safety systems** — laser safety limits, interlocks, eye-safety compliance
- **Synchronization** — between rendering and optical emission
- **Optical alignment** — manufacturing tolerances, coupling alignment
- **Signal conditioning** — where electronic-to-optical or optical-to-electronic conversion is needed
- **Error correction** — where necessary for reliable operation

It also does not eliminate the need for general-purpose electronic computation. This is a heterogeneous architecture, not a claim that electronics are obsolete.

---

## 25. Transitional Architecture

A major advantage of the concept is that full photonic rendering is not required for the first prototype.

A transitional system can use an existing electronic GPU:

```
Existing GPU
     ↓
Electronic-to-optical conversion
     ↓
Optical transport
     ↓
Optical output module
     ↓
Laser/phosphor or projection display
```

This isolates the display architecture from the photonic compute problem.

The first experiment only needs to test: can an optical image representation be transported and converted into a high-quality display image without reconstructing a conventional digital display pipeline?

If successful, the photonic rendering chiplet can be introduced later.

---

## 26. Existing Technologies Supporting the Architecture

Each stage of the proposed pipeline has at least one demonstrated implementation:

| Technology | Year | Status | Relevance |
|-----------|------|--------|-----------|
| **ACCEL** (Tsinghua, Nature) | 2023 | Published | All-analog photonic-electronic chip: 3000× faster, 4M× lower energy than A100 for vision tasks [web_15_0_0_4] |
| **OPCA** (Tsinghua, Optica) | 2024 | Published | Photonic chip: 100 billion pixels, 6 ns response, all-optical processing [web_15_0_0_6] |
| **SEECHIP** (ICPP) | 2023 | Published | Chiplet-based GPU with photonic inter-chiplet links [web_15_0_0_11] |
| **MIT ski-jump waveguide** (Nature) | 2025 | Published | 68.6 M spots/s·mm², submicron beams, CMOS-foundry fabricated [web_15_0_0_17] |
| **Brilliance Laserchip** (Brilliance RGB) | 2026 | Commercial (pre-production) | Integrated RGB laserchip on SiN PIC, 10× power reduction for AR [web_15_0_0_20] |
| **Prysm LPD** (Prysm) | Commercial | Commercial | Laser-phosphor display: 75% lower power, true black, 240 Hz [web_15_0_0_27] |
| **TriLite Trixel 3** (TriLite) | 2024 | Commercial (engineering samples) | <1 cm³ laser beam scanner, 15 lm, 320 mW, 214% sRGB [web_15_0_0_30][web_15_0_0_34] |

This does not prove the complete architecture. It does mean the proposal can be evaluated as a systems-integration problem rather than requiring every underlying technology to be invented from scratch.

---

## 27. Experimental Roadmap

The architecture should be validated in stages.

### Stage 1 — Optical transport

Demonstrate:

```
Electronic image source
        ↓
E/O conversion
        ↓
Optical fiber
        ↓
Optical display
```

Measure: latency, bandwidth, SNR, optical dynamic range, color fidelity, spatial resolution.

### Stage 2 — Optical reconstruction

Compare pixel reconstruction vs. optical PSF reconstruction using:

- checkerboards
- diagonal lines
- text
- subpixel patterns
- moving edges
- high-frequency textures
- temporal patterns

Measure the resulting modulation transfer function.

### Stage 3 — Spatial optical addressing

Demonstrate I(x,y,λ,t) generation from an optical input. This is one of the most important technical milestones.

### Stage 4 — Neural rendering chiplet

Introduce a photonic neural network for a controlled workload. Measure:

- energy per operation
- latency
- optical loss
- precision
- error rate
- thermal stability
- scaling behaviour

ACCEL and OPCA demonstrate that photonic neural networks can process image data directly [web_15_0_0_4][web_15_0_0_6].

### Stage 5 — Direct photonic rendering-to-display path

```
Electronic control
       ↓
Photonic neural rendering
       ↓
Optical image field
       ↓
Optical interconnect
       ↓
Optical display
```

No intermediate digital framebuffer is required between the photonic renderer and optical output.

---

## 28. Critical Engineering Questions

The concept has several important open problems.

### 28.1 Spatial addressing

How many independently controllable optical degrees of freedom can be produced? This is arguably the central technical question. The MIT ski-jump demonstrates 68.6 M spots/s·mm² from a single device [web_15_0_0_17], but scaling to full-display resolution requires further work.

### 28.2 Optical power budget

How much optical power is required after splitting, coupling, propagation, modulation, and conversion?

### 28.3 Precision

Photonic computation is analog. The system must characterize: noise, quantization, phase error, amplitude error, thermal drift, manufacturing variation.

### 28.4 Nonlinear operations

Linear optical transformations are relatively natural. Neural networks also require nonlinearities. The architecture needs an efficient photonic/electronic strategy for nonlinear activation functions and control.

### 28.5 Memory

Photonics does not automatically solve memory. Large persistent state remains primarily an electronic problem. The proposed architecture deliberately leaves memory in the electronic domain wherever appropriate.

### 28.6 Calibration

An optical system may require calibration across: wavelength, temperature, emitter intensity, phase, spatial alignment, phosphor response. Calibration infrastructure must be considered part of the architecture.

---

## 29. Falsifiable Predictions

A useful architecture should make testable predictions. The proposal predicts that, for selected workloads:

1. Optical broadcast can reduce data movement relative to equivalent electronic replication.
2. Photonic neural computation can reduce latency or energy for suitable linear-algebra-heavy workloads.
3. Optical image reconstruction can replace part of numerical display reconstruction.
4. Direct optical transport can remove several conventional display-interface stages.
5. A display without a conventional rasterized pixel matrix can produce useful high-resolution images.
6. A heterogeneous electronic/photonic chiplet can provide a more practical path than a fully photonic processor.
7. End-to-end system efficiency must be evaluated at wall-plug level rather than by quoting photonic MAC efficiency alone.

These predictions can be independently tested.

---

## 30. Performance Envelope

The following values are architectural targets/hypotheses, not demonstrated specifications.

| Parameter | Current | Photonic (target) |
|-----------|---------|-------------------|
| End-to-end latency | ~50–100 ms | <1 ms |
| Brightness | 500–1500 cd/m² (LCD loses 85–90% of backlight) | up to 6×10⁶ cd/m² theoretical (laser direct) |
| Contrast | 5000:1 (LCD) to 1M:1 (OLED) | 1M:1+ (Prysm LPD measured) |
| Display-chain power | 5–15 W (cable + scaler + T-CON) | eliminated |
| Color gamut | ~100% sRGB (typical) | 214% sRGB (TriLite Trixel 3 measured) |
| Photonic compute latency | 0.26 ms/frame (A100) | 72 ns/frame (ACCEL measured) |
| Image processing speed | millisecond-level | 6 ns (OPCA measured) |
| Spatial spot rate | N/A (fixed pixel matrix) | 68.6 M spots/s·mm² (ski-jump measured) |

---

## 31. HDMI Licensing and the Industry Trend

HDMI licensing costs $10,000/year + $0.05/port. HDCP, DSC, TMDS, and DisplayPort link layers — all eliminated in this architecture.

Datacenter GPUs (NVIDIA A100, H100) already ship without display engines. The trend is clear: display output is being decoupled from compute. This architecture takes that trend to its logical conclusion.

---

## 32. Comparison of Architectural Philosophies

**Conventional evolution:**

```
Existing architecture
       ↓
optimize component
       ↓
add feature
       ↓
add controller
       ↓
add protocol
       ↓
add buffer
       ↓
add algorithm
       ↓
repeat
```

**Proposed approach:**

```
Existing architecture
       ↓
identify unnecessary representation
       ↓
remove conversion
       ↓
move suitable operation into physical domain
       ↓
preserve representation
       ↓
simplify downstream system
```

The difference is architectural rather than merely technological.

---

## 33. Where the Real Architectural Novelty Lies

The novelty should not be interpreted as:

- "Photonic neural networks are new" — they are not
- "Optical displays are new" — they are not
- "Chiplets are new" — they are not

The proposed architectural contribution is the composition:

```
Electronic general-purpose compute
         +
Photonic neural rendering
         +
Optical image transformation
         +
Optical interconnect
         +
Direct optical display formation
```

with the deliberate objective of avoiding unnecessary electronic/digital representations between photonic rendering and visual output.

The architecture changes the location of the electronic/optical boundary. That boundary is the primary subject of this proposal.

---

## 34. Key Research Question

The central research question is not "Can a computer be made entirely photonic?"

It is: **How much of the graphics pipeline can remain in the optical domain once neural rendering has produced the image representation?**

A second question follows: **What is the optimal boundary between electronic and photonic computation for a practical graphics processor?**

These questions are experimentally testable.

---

## 35. What Would Constitute Success?

The concept does not require every theoretical advantage to materialize. A successful implementation would demonstrate that:

```
Electronic compute
       ↓
Photonic rendering
       ↓
Optical image
       ↓
Optical transport
       ↓
Optical display
```

can outperform or simplify an equivalent conventional path for at least one meaningful workload.

Success could be demonstrated by any combination of:

- lower latency
- lower energy
- reduced data movement
- reduced hardware complexity
- improved optical reconstruction
- higher bandwidth
- improved display quality
- simpler manufacturing

The architecture should be judged by measurable system-level results rather than by individual component specifications.

---

## 36. Broader Implication

When a system becomes increasingly complicated, the natural response is often to optimize each component. But sometimes the greater opportunity is to question whether the intermediate representation itself is necessary.

In graphics, the final output is light. Therefore a potentially unnecessary chain is:

```
light → digital → processed digital → serialized digital → reconstructed digital → light
```

The proposed architecture asks whether a significant portion of that loop can be removed.

---

## 37. Final Architecture

```
                    ELECTRONIC DOMAIN
 ┌───────────────────────────────────────────────────┐
 │                                                   │
 │  CPU / GPU / NPU / Memory / Simulation / Control  │
 │                                                   │
 └───────────────────────┬───────────────────────────┘
                         │
                         │ neural rendering inputs
                         │ and control
                         ▼
                  PHOTONIC CHIPLET
 ┌───────────────────────────────────────────────────┐
 │                                                   │
 │  Neural rendering                                │
 │  Reconstruction                                  │
 │  Image transformation                            │
 │  Optical broadcast                               │
 │                                                   │
 └───────────────────────┬───────────────────────────┘
                         │
                         │ optical image representation
                         ▼
                 OPTICAL INTERCONNECT
 ┌───────────────────────────────────────────────────┐
 │  Photonic interposer / waveguide / optical fiber │
 └───────────────────────┬───────────────────────────┘
                         │
                         ▼
                  OPTICAL OUTPUT
 ┌───────────────────────────────────────────────────┐
 │  Spatial optical conversion / emitter system     │
 └───────────────────────┬───────────────────────────┘
                         │
                         ▼
                  OPTICAL DISPLAY
 ┌───────────────────────────────────────────────────┐
 │  Laser / phosphor / projection / optical field   │
 └───────────────────────┬───────────────────────────┘
                         │
                         ▼
                        EYE
```

---

## 38. Conclusion

This proposal does not attempt to replace electronics with photonics. It attempts to use each technology where its physical properties are most appropriate.

Electronics remain responsible for general computation, control, memory and system logic. Photonics is introduced where massive parallelism, optical propagation, signal broadcast and neural linear algebra can provide an advantage.

Most importantly, the resulting image does not necessarily need to be converted back into a conventional digital framebuffer before reaching the display.

The essential sequence is:

```
COMPUTE ELECTRONICALLY
        ↓
RENDER PHOTONICALLY
        ↓
TRANSFORM OPTICALLY
        ↓
TRANSMIT OPTICALLY
        ↓
DISPLAY OPTICALLY
```

The central proposition is simple:

> "If the final information is light, there is value in asking how early in the pipeline it can become light — and how late it can remain light."

The technologies required for individual stages already exist in various forms. The research challenge is to determine whether their deliberate integration can produce a superior system architecture.

---

## Status and Scope

This document is an architectural proposal. Performance numbers are targets, estimates or potential envelopes unless explicitly identified as experimentally demonstrated.

The purpose is to establish a coherent architecture and identify experimentally testable engineering paths toward implementation.

---

## References

1. **ACCEL** — All-analog Chip Combining Electronic and Light Computing. Tsinghua University. Published in *Nature*, November 2023.  
   https://www.tomshardware.com/tech-industry/semiconductors/chinas-accel-analog-chip-promises-to-outpace-industry-best-in-ai-acceleration-for-vision-tasks

2. **OPCA** — Optical Parallel Computational Array. Tsinghua University. Published in *Optica*, June 2024.  
   https://www.optica.org/about/newsroom/news_releases/2024/june/photonic_chip_integrates_sensing_and_computing_for_ultrafast_machine_vision/

3. **SEECHIP** — Scalable and Energy-Efficient Chiplet-based GPU Architecture Using Photonic Links. ICPP 2023.  
   https://dl.acm.org/doi/10.1145/3605573.3605626

4. **MIT ski-jump waveguide** — Nanophotonic waveguide chip-to-world beam scanning. Published in *Nature*, 2025.  
   https://pmc.ncbi.nlm.nih.gov/articles/PMC12979201/

5. **Brilliance Laserchip** — Integrated RGB laserchip on silicon nitride PIC. Brilliance RGB, March 2026.  
   https://www.optica.org/about/newsroom/corporate_member_news/2026/brilliance_raises_millions_to_advance_its_laserchips_for_ar/

6. **Prysm LPD** — Laser Phosphor Display. Prysm Inc. Commercial product.  
   https://www.prysm.com/displays/lpd-6k-series/laser-phosphor-display/index.html

7. **TriLite Trixel 3** — Laser beam scanner display for AR. TriLite Technologies.  
   https://www.trilite-tech.com/product/

---

## License

Creative Commons Attribution 4.0 International (CC BY 4.0).

The architecture is intentionally published openly so that it can be studied, implemented, improved and commercialized by others. The objective is to make the architectural concept available rather than restrict its development through proprietary ownership.
