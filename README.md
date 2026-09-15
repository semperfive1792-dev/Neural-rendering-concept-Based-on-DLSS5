# Neural-rendering-concept-Based-on-DLSS5
# Neural Rendering on Separate Dies: A Concept for a Streaming SRAM Buffer Between GPU and NPU

## Abstract

NVIDIA’s DLSS 5 Neural Rendering delivers a significant leap in visual realism by processing the final upscaled frame with a neural network. However, this comes at a steep performance cost—often halving frame rates—because the workload runs on shared tensor cores within the GPU.

This paper proposes a conceptual architecture that decouples neural rendering from the main graphics pipeline: a dedicated NPU die on a common substrate with the GPU, connected via a streaming SRAM buffer acting as a low‑latency data bus. The concept leverages existing technologies (3D V‑Cache, Infinity Cache, multi‑die packaging) and primarily requires architectural integration rather than fundamentally new engineering.

---

## 1. The Problem

DLSS 5 Neural Rendering is the final stage of the graphics pipeline, operating on the full output resolution after upscaling. The neural network processes every pixel of the final image to refine lighting, materials, skin, hair, and shadows, delivering a noticeable improvement in visual fidelity.

The trade‑off is severe performance loss. Because the network works on the full upscaled frame, changing Super Resolution modes (Quality → Balanced → Performance) has almost no effect on the load. Tests on the RTX 5090 show nearly a 2× drop in FPS when DLSS 5 is enabled; on an RTX 5070 Ti at 1440p, the drop is approximately 2.6×.

The root cause is hardware contention: tensor cores must share resources with rasterization, ray tracing, DLSS Super Resolution, and Ray Reconstruction. Neural rendering competes directly with the game itself for compute and power.

---

## 2. Existing Approaches and Their Limitations

### 2.1. More Tensor Cores on the GPU Die

This is NVIDIA’s current path. It’s limited by two factors:

- **Die area.** Tensor cores compete for space with CUDA cores, RT cores, and memory controllers. Increasing their share means cutting other blocks.
- **Power delivery.** Tests show the RTX 5090 hits its power limit at 575 W with a single 12V‑2×6 connector when DLSS 5 is active. The MSI Lightning Z variant with dual connectors and a 1000 W limit loses less FPS. Here, the bottleneck is not compute—it’s power delivery.

### 2.2. Neural Upstream (Neural Network Before Upscaling)

The Neural Upstream mod (by matiasLombo) runs the neural network on the internal render resolution before upscaling, not after. This yields up to a 61% FPS gain on an RTX 4080.

NVIDIA deliberately avoids this approach to preserve image quality. With less input data, the network produces weaker material detail, stronger uncanny valley effects, and more artifacts.

### 2.3. A Second GPU as a Neural Coprocessor

In the Neural Coprocessor project (Marcelo Guibout), one RTX 5060 Ti renders the game and a second processes the frame with DLSS 5. This recovers most of the FPS lost to neural rendering (e.g., from 54 to 91 FPS in DLSS Quality).

Limitations: dual GPUs, dual monitors, ReShade add‑on, PCIe frame transfer. This is not viable for mainstream users.

---

## 3. Proposed Architecture

### 3.1. Core Idea

Separate neural rendering and main graphics at the hardware level: GPU and NPU as distinct dies on a common substrate, linked by a streaming SRAM buffer that acts as a low‑latency interconnect.

### 3.2. Structure

```
        ┌─────────────────┐
        │     GPU die     │  Rasterization, Ray Tracing,
        │   (main)        │  DLSS Super Resolution, Ray Reconstruction,
        │                 │  Display Controller, HDMI/DP outputs
        └────────┬────────┘
                 │ TSV (~10 µm, ~1 ns latency)
        ┌────────┴────────┐
        │  SRAM buffer    │  16–32 MB, streaming FIFO
        │  (active layer) │  Hardware bypass multiplexer
        └────────┬────────┘
                 │ TSV (~10 µm, ~1 ns latency)
        ┌────────┴────────┐
        │     NPU die     │  Neural rendering only
        │  (AI‑specialized)│  Tensor‑optimized, single‑model focus
        │                 │
        └─────────────────┘
```

### 3.3. Data Flow

1. GPU renders the frame (raster + RT + upscaling) and writes it in strips to the SRAM buffer.
2. NPU reads the previous strip, processes it, and writes the result back to the buffer.
3. A hardware bypass multiplexer routes the result to the GPU’s display controller → screen.
4. If NPU is disabled, the multiplexer passes the frame directly from GPU to display controller; the buffer is unused.

### 3.4. Key Architectural Choices

**Streaming buffering instead of full‑frame storage.** The SRAM buffer acts as a FIFO queue: 4–8 MB strips enter and exit sequentially. This reduces SRAM requirements from ~100 MB (full frames) to 16–32 MB (streaming strips). For context, NVIDIA AD102 has 48 MB L2 cache; AMD RDNA 3 Infinity Cache reaches up to 128 MB.

**Hardware bypass.** The multiplexer is purely physical, not software‑controlled. When NPU is off (no DLSS 5 support), the frame goes straight from GPU to display controller—zero latency, zero NPU power. Full backward compatibility with existing games is automatic.

**Display controller stays on the GPU.** The NPU has no video outputs. It processes the frame and returns it to the GPU buffer; the GPU’s own display controller handles HDMI/DP, HDCP, VRR, and color spaces. This avoids duplicating complex display logic on the NPU die.

---

## 4. Why SRAM Buffer Instead of a Direct Interconnect

### 4.1. Latency

A direct multi‑die interconnect (e.g., NV‑HBI in server GPUs) offers high bandwidth but adds routing latency. An SRAM buffer connected via TSVs in a 3D stack achieves 1–2 ns access latency—comparable to L2/L3 cache.

### 4.2. Asynchrony

A direct link requires tight synchronization: the GPU must wait for the NPU to be ready. The SRAM buffer decouples them: GPU writes the next strip while NPU processes the previous one. Neither die idles waiting for the other.

### 4.3. Scalability

The NPU die can be upgraded independently of the GPU. New neural models, new process nodes, or larger compute capacity can be introduced by swapping just the NPU die, leaving the GPU unchanged. This mirrors how AMD updates the cache die in Ryzen X3D while keeping the CCD the same.

---

## 5. Comparison of Approaches

| Parameter | Tensor Cores on GPU (current) | Neural Upstream (mod) | 2nd GPU as Coprocessor (mod) | **GPU + NPU + SRAM Buffer (proposed)** |
|---|---|---|---|---|
| FPS penalty from neural rendering | ~2× | ~0.4× | ~0× (on render GPU) | **~0× (on GPU)** |
| Image quality | Maximum | Reduced (uncanny valley, artifacts) | Maximum | **Maximum** |
| Power consumption | High (resource contention) | Low | High (2 cards) | **Low (NPU optimized)** |
| Backward compatibility | Full | Full | Limited | **Full (hardware bypass)** |
| Mass‑market viability | Yes | Yes (as mod) | No | **Yes** |
| Manufacturing cost | Base | Base | 2× (2 cards) | **+1 die + buffer** |

---

## 6. Technological Readiness

All components of the proposed architecture are already in mass production:

- **3D stacking with TSVs:** AMD 3D V‑Cache—millions of CPUs shipped with an SRAM die stacked over the CCD.
- **Multi‑die on a substrate:** NVIDIA GB200—two compute dies on a single substrate with NV‑HBI.
- **Cache as an intermediate buffer:** AMD Infinity Cache—128 MB SRAM on the GPU die acting as a large buffer between compute and memory controller.
- **NPU in consumer chips:** AMD plans to integrate an NPU into desktop Ryzen 10 000 (Zen 6, 2027), trading integrated graphics for a neural coprocessor.
- **Dedicated AI accelerators in GPUs:** Apple A19 Pro includes Neural Accelerators inside GPU cores.

No fundamentally new technology is required—only architectural integration, software model design, and NPU optimization for a specific neural model.

---

## 7. Expected Challenges

**Software model.** Even with a hardware bypass, an API is needed to route frames to/from the NPU. A likely path is an extension to NVIDIA’s RTX SDK, transparent to game developers.

**Area and cost.** Three dies (GPU + SRAM buffer + NPU) are more expensive than a single die. The first generation would likely be flagship‑tier. However, streaming buffering (16–32 MB vs 100+ MB) and independent NPU scaling reduce costs compared to a brute‑force GPU enlargement.

**Pipeline latency.** The NPU must process each strip faster than the GPU writes the next one. At 90 FPS and 4 MB strips for 1440p, that’s ~0.5 ms per strip. If the NPU lags, a 1–2 frame delay occurs. This is acceptable for single‑player games but not for competitive esports.

---

## 8. Conclusion

DLSS 5 Neural Rendering represents a qualitative leap in visual quality, but its current implementation on shared tensor cores is inherently inefficient: it causes a ~2× FPS drop and pushes flagship GPUs to their power limits.

The proposed architecture—a dedicated NPU die on the same substrate, linked to the GPU via a streaming SRAM buffer—addresses both issues without sacrificing image quality. All required technologies already exist in mass production. What’s needed is architectural commitment, not scientific breakthrough.

This concept is not a full engineering design; detailed calculations of die area, power budget, and precise latency require RTL‑level simulation. The goal is to clearly articulate a promising direction that the industry is likely already considering, and to give it a public formulation.

---

## References

1. NVIDIA. *DLSS 5: 3D‑Guided Neural Rendering*. nvidia.com  
2. TweakTown. *DLSS 5 in NBA 2K27: Performance Analysis*. tweaktown.com  
3. GameGPU. *DLSS 5 Neural Rendering: 2× FPS Difference in The Witcher 3*. gamegpu.com  
4. ixbt.com. *DLSS 5 Activation Reduces Performance by 2.5×*. ixbt.com  
5. TechPowerUp. *Early DLSS 5 Testing: RTX 5090 Power Connector Bottleneck*. techpowerup.com  
6. TechPowerUp. *Modders Rework DLSS 5 Pipeline for Performance Boost*. techpowerup.com  
7. TechSpot. *Modders Found Two Ways to Make DLSS 5 Faster*. techspot.com  
8. TechPowerUp. *Mod Revives Multi‑GPU with Dedicated DLSS 5 GPU*. techpowerup.com  
9. Wikipedia. *RDNA (microarchitecture)*. en.wikipedia.org  
10. Wikipedia. *CPU cache*. en.wikipedia.org  
11. Wikipedia. *AMD 3D V‑Cache*. en.wikipedia.org  
12. Wikipedia. *Nvidia Blackwell*. en.wikipedia.org  
13. DNS Club. *AMD to Drop Integrated Graphics for NPU in Ryzen 10 000*. club.dns-shop.ru  
14. Articsledge. *Neural Processing Unit (NPU)*. articsledge.com  

---

*Author is a conceptual thinker, not an ASIC architect. This paper presents an architectural idea, not a complete engineering design. Die area, power budget, and exact latency require RTL‑level simulation. The purpose of this publication is to publicly anchor the idea and give it a chance to reach engineering teams capable of implementing it.*


---

## Photonic Implementation: From Chiplet to Optical Compute

The neural-chiplet concept naturally extends into silicon photonics. Rather than waiting for a fully photonic GPU, the chiplet architecture allows photonics to enter incrementally — as a dedicated compute tile alongside electronic ones.

### Why Chiplets Are the Ideal Bridge

- **Mixed process nodes.** Photonic components work best at 45–90 nm (larger waveguides), while electronic logic benefits from 3 nm. A monolithic die can't mix these. A chiplet package can.
- **Incremental adoption.** You don't replace the entire GPU — you add a photonic tile where it matters most: matrix multiplication for neural rendering.
- **Independent scaling.** Memory, compute, and I/O tiles can be upgraded separately.

### Proposed Architecture

| Tile | Technology | Function |
|------|-----------|----------|
| Render tile | Electronic (3 nm) | Rasterization, ray tracing, geometry |
| Neural tile | Photonic (matrix compute on light) | DLSS, neural shaders, AI upscaling |
| Memory tile | Electronic (HBM / DRAM) | Frame buffer, weights, textures |
| Interconnect | Optical (silicon waveguides on interposer) | Ultra-low-latency data transfer between all tiles |

### What Already Exists (as of 2026)

Each component of this architecture is already in development or production — just not yet combined this way:

- **Celestial AI** (backed by NVIDIA and AMD): split compute and memory tiles connected by a photonic fabric at 14.4 Tbit/s. Memory scales independently from compute. — [Kisaco Research presentation](https://www.kisacoresearch.com/sites/default/files/presentations/preet_virk_-_celestial_ai_-_photonic_fabrictm_based_scale-up_network.pdf), [The Next Platform](https://www.nextplatform.com/compute/2024/04/04/celestial-ai-wants-to-break-the-memory-wall-fuse-hbm-with-ddr5/1646782)

- **Ayar Labs** (MIT spinoff, \$3.8B valuation as of March 2026): photonic chiplet interposers mounted directly on existing processors. Their TeraPHY chip delivers 8 Tbit/s at 10 ns latency — without replacing the electronic GPU. — [andrew.ooo](https://andrew.ooo/posts/ayar-labs-500m-series-e-photonic-ai-chips/)

- **NVIDIA IEDM 2024 concept**: GPU compute tiles in a 3D stack, DRAM on top, 12 silicon-photonic channels between them. Compute and memory are electronic; the link is optical. — [TechPowerUp](https://www.techpowerup.com/329651/nvidia-shows-future-ai-accelerator-design-silicon-photonics-and-dram-on-top-of-compute)

- **Neurophos** (backed by Bill Gates' fund): optical transistors 10,000× smaller than previous designs, built using existing semiconductor processes. Their Tulkas T100 chip runs at 56 GHz and is claimed to outperform NVIDIA Vera Rubin NVL72 in FP4/INT4 workloads. Mass production target: ~2028. — [Tom's Hardware](https://www.tomshardware.com/tech-industry/semiconductors/bill-gates-backed-silicon-photonics-startup-develops-optical-transistors-10-000x-smaller-than-current-tech-optical-chip-can-process-1-000-x-1-000-multiplication-matrices)

- **Purdue University**: single-photon photonic transistors compatible with CMOS fabrication, operating at room temperature. — [SecurityLab](https://www.securitylab.ru/news/566420.php)

- **Imec**: first successful epitaxial growth of GaAs lasers directly on 300 mm silicon wafers in a standard CMOS line. No more die-to-wafer bonding for on-chip lasers. — [TechXplore](https://techxplore.com/news/2025-01-silicon-photonics-advance-paves-effective.html)

- **PICNIC** (research paper, November 2025): 3D-stacked chiplets with RRAM memory and photonic silicon interconnect — 57× energy efficiency improvement over conventional GPU. — [arXiv](https://arxiv.org/pdf/2511.04036)

### What's Still Missing

- **Optical memory.** All current photonic chips use electronic memory. Berkeley Lab demonstrated optical bistability in nanoparticles (nanoscale optical memory), but practical application is years away. — [ScienceDaily](https://www.sciencedaily.com/releases/2025/02/250226125021.htm)

- **Precision.** Photonic compute is analog, not digital. For neural rendering (8-bit tolerance) this is acceptable. For general-purpose compute — not yet.

- **Fully photonic consumer GPU.** Not before 2030–2035. But the chiplet approach doesn't require waiting for that.

### The Key Insight

The neural-chiplet concept doesn't need a fully photonic GPU to work. It needs:

1. A photonic tile that handles matrix math (DLSS / neural shaders) — **already exists** (Neurophos, PICNIC)
2. An optical interconnect between tiles — **already exists** (Ayar Labs, Celestial AI, NVIDIA IEDM concept)
3. Electronic tiles for rasterization and memory — **already standard**

The industry has all the pieces. What's missing is someone putting them together in a consumer GPU package — which is exactly what this concept proposes.
## Optical Broadcast: Why Photonics Solves the Data‑Distribution Bottleneck

One of the unique advantages of photonic computing—unavailable to pure electronics—is the ability to **broadcast identical data to multiple compute units simultaneously** without copying, buffering, or consuming extra memory bandwidth.

In an electronic system, feeding the same frame to multiple AI blocks (DLSS, Frame Generation, Ray Reconstruction, neural shaders) requires:
- Multiple reads from memory, or  
- Explicit data duplication across buses and buffers,  
which costs energy, latency, and die area. The more consumers you have, the more copies you need, and the heavier the memory traffic becomes.

Photonics bypasses this by using **optical broadcast**: a single optical signal is split into multiple identical copies via beam splitters. With $j$ splitters, you can generate $2^j$ copies of the same data stream. Each copy travels simultaneously to a different compute block—**with no additional memory access, no serialization, and no buffering**.

---

### How This Enables True Parallelism for Neural Rendering

In your proposed architecture, the render tile produces a frame that must be consumed by several neural processing units:

- **DLSS** – for upscaling  
- **Frame Generation** – for interpolating intermediate frames  
- **Ray Reconstruction** – for denoising and ray-tracing cleanup  
- **Neural shaders** – for material and lighting enhancements  

With electronic interconnects, each unit must receive its own copy of the frame, leading to repeated memory reads and bus contention. In a photonic setup, the render tile outputs the frame as an optical signal; a compact on‑chip splitter distributes it **simultaneously** to all neural tiles. All blocks start processing the same frame at virtually the same instant.

This eliminates the “fan‑out” penalty: the cost of distributing data does not grow with the number of consumers.

---

### Real‑World Implementations Already Using This Principle

- **Broadcast‑and‑Weight Protocol** (proposed 2014, demonstrated 2017): Input data is broadcast across multiple wavelengths via electro‑optic modulation, and each copy is independently weighted in its own channel. This enables one input to drive many parallel computations without data duplication.  
  Source: [HAL Science](https://hal.science/hal-04580418/document)

- **Lightening‑Transformer** (arXiv 2024): A photonic accelerator for transformers that explicitly leverages optical broadcast to share operands across cores. The paper states: “We unleash the natural optical broadcast capability to enable intra‑core and inter‑core operand sharing.” One tensor is distributed to multiple compute cores as light, not as copied data.  
  Source: [arXiv](https://arxiv.org/html/2305.19533)

- **Photonic Tensor Core** (Nature 2021): Uses wavelength‑division multiplexing (WDM) to apply one matrix to **four input vectors simultaneously**, with each vector encoded on a different wavelength. This achieves 16 multiply‑accumulate operations in a single cycle.  
  Source: [ResearchGate](https://www.researchgate.net/publication/339015092_Parallel_convolution_processing_using_an_integrated_photonic_tensor_core)

These examples show that the industry is already treating optical broadcast not as a curiosity, but as a core mechanism for efficient parallel computation.

---

### Architectural Diagram (ASCII)

```
[Render Tile] ──→ [Optical Splitter]
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
   [DLSS Tile]   [FrameGen Tile]  [RayRecon Tile]
          │            │            │
          └────────────┼────────────┘
                        ↓
              [Result Assembly] → Display
```

The render tile emits a single optical representation of the frame. The splitter creates simultaneous copies, each feeding a dedicated neural tile. All tiles operate in parallel on the same input, with no serialization or memory bottleneck.

Additionally, photonic waveguides can cross each other with minimal crosstalk—unlike metal wires, which require complex routing and buffering to avoid interference. This makes the physical layout of such broadcast networks significantly simpler and more scalable.

---

# Нейронный рендеринг на раздельных кристаллах: концепция потокового SRAM-буфера между GPU и NPU

## Аннотация

Технология DLSS 5 Neural Rendering от NVIDIA открывает качественно новый уровень визуального реализма в играх, но сталкивается с фундаментальной проблемой: обработка полноразмерного кадра нейросетью на тензорных ядрах GPU приводит к двукратному падению FPS. В статье предлагается концептуальная архитектура, решающая эту проблему через разделение вычислительных кристаллов — выделенный NPU для нейронного рендеринга на общей подложке с GPU, связанный с ним через потоковый SRAM-буфер, работающий как низколатентная шина данных. Концепция опирается на существующие технологии (3D V-Cache, Infinity Cache, multi-die сборка) и требует в первую очередь архитектурной, а не принципиально новой инженерной работы.

---

## 1. Проблема

DLSS 5 Neural Rendering — финальная стадия графического конвейера, запускаемая после апскейлера на готовом кадре в выходном разрешении. Нейросеть обрабатывает каждый пиксель итогового изображения, улучшая освещение, материалы, кожу, волосы и тени. Результат — заметный прирост визуального реализма [\[1\]](https://www.nvidia.com/en-us/geforce/news/dlss-5-3d-guided-neural-rendering/).

Платой становится производительность. Поскольку нейросеть работает с полноразмерным кадром после апскейла, изменение режима Super Resolution (Quality → Balanced → Performance) практически не влияет на нагрузку [\[2\]](https://www.tweaktown.com/articles/11596/nvidia-dlss-5-3d-guided-neural-rendering-in-nba-2k27-performance-analysis-and-more/index.html). Тесты на RTX 5090 показывают падение FPS почти вдвое при включении DLSS 5 [\[3\]](https://gamegpu.com/news/igry/nejrosetevoj-rendering-dlss-5-obespechivaet-dvukratnuyu-raznitsu-fps-v-the-witcher-3). На RTX 5070 Ti в 1440p падение составляет примерно 2,6x [\[4\]](https://www.ixbt.com/news/2026/09/04/432516-teper-oficialno-aktivaciia-dlss-5-snizaet-proizvoditelnost-v-2-5-raz-i-apskeiler-ne-pomozhet-poiavilis-testyx-vsex-kart-rtx-50.html).

Причина — аппаратная. Тензорные ядра, обрабатывающие нейросеть, разделены между растеризацией, трассировкой лучей, DLSS Super Resolution и Ray Reconstruction. Нейронный рендеринг конкурирует за вычислительные ресурсы с самой игрой.

---

## 2. Существующие подходы и их ограничения

### 2.1. Наращивание тензорных ядер на кристалле GPU

Текущий путь NVIDIA. Ограничен двумя факторами:

- **Площадь кристалла.** Тензорные ядра конкурируют за место с CUDA-ядрами, RT-ядрами и контроллерами памяти. Увеличение доли тензорных ядер означает урезание остальных блоков.
- **Энергопотребление.** Тесты RTX 5090 показывают, что при включении DLSS 5 карта выходит на энергопредел уже при 575 Вт через один коннектор 12V-2x6. Версия MSI Lightning Z с двумя разъёмами и лимитом 1000 Вт теряет меньше FPS [\[5\]](https://www.techpowerup.com/352262/early-dlss-5-testing-suggests-the-rtx-5090s-single-power-connector-might-be-the-bottleneck). То есть ограничитель — не вычисления, а питание.

### 2.2. Neural Upstream — нейросеть до апскейла

Моддерский подход (проект Neural Upstream от matiasLombo): нейросеть обрабатывает кадр во внутреннем разрешении до апскейлера, а не после. Результат — выигрыш до 61% FPS на RTX 4080 [\[6\]](https://www.techpowerup.com/352476/modders-rework-dlss-5s-rendering-pipeline-for-a-big-performance-boost).

NVIDIA сознательно отказалась от этого подхода ради качества. На уменьшенном кадре нейросеть получает меньше данных — хуже детализация материалов, сильнее эффект uncanny valley, больше артефактов [\[7\]](https://www.techspot.com/news/113775-modders-have-already-found-two-ways-make-dlss.html).

### 2.3. Вторая видеокарта как нейронный сопроцессор

Проект Neural Coprocessor от Marcelo Guibout: одна RTX 5060 Ti рендерит игру, вторая обрабатывает кадр через DLSS 5. Результат — восстановление почти всего FPS, съеденного нейросетью (с 54 до 91 FPS в DLSS Quality) [\[8\]](https://www.techpowerup.com/352459/mod-revives-multi-gpu-gaming-with-dedicated-dlss-5-gpu-for-impressive-fps-gain?cp=2).

Ограничения: две видеокарты, два монитора, ReShade-аддон MGPU Bridge, PCIe-передача кадра. Неприменимо для массового пользователя.

---

## 3. Предлагаемая архитектура

### 3.1. Принцип

Разделить нейронный рендеринг и основную графику на аппаратном уровне: GPU и NPU — отдельные кристаллы на общей подложке, соединённые через потоковый SRAM-буфер, выполняющий роль низколатентной шины данных.

### 3.2. Структура

```
        ┌─────────────────┐
        │     GPU die     │  Растеризация, трассировка лучей,
        │   (основной)    │  DLSS Super Resolution, Ray Reconstruction,
        │                 │  Display controller, видеовыходы (HDMI/DP)
        └────────┬────────┘
                 │ TSV (~10 мкм, задержка ~1 нс)
        ┌────────┴────────┐
        │  SRAM-буфер     │  16–32 МБ, потоковый коммутатор
        │  (active layer) │  Bypass-мультиплексор (железный)
        └────────┬────────┘
                 │ TSV (~10 мкм, задержка ~1 нс)
        ┌────────┴────────┐
        │     NPU die     │  Нейронный рендеринг
        │  (спец. AI)     │  Только тензорные операции
        │                 │  Оптимизирован под одну модель
        └─────────────────┘
```

### 3.3. Поток данных

1. GPU рендерит кадр (растеризация + RT + апскейл) и пишет его стрипами в SRAM-буфер.
2. NPU читает предыдущий стрип из буфера, обрабатывает, возвращает результат.
3. Bypass-мультиплексор направляет результат в display controller GPU → на экран.
4. Если NPU выключен — мультиплексор пропускает кадр напрямую из GPU в display controller, буфер не задействуется.

### 3.4. Ключевые архитектурные решения

**Потоковая буферизация вместо полного хранения кадров.** SRAM-буфер не хранит кадр целиком. Он работает как FIFO-очередь — стрипы по 4–8 МБ входят с одной стороны и выходят с другой. Это снижает требование к размеру SRAM с ~100 МБ (полные кадры) до 16–32 МБ (потоковые стрипы). Для сравнения: L2-кэш NVIDIA AD102 — 48 МБ, Infinity Cache у AMD RDNA 3 — до 128 МБ [\[9\]](https://en.wikipedia.org/wiki/RDNA_(microarchitecture)). 16–32 МБ SRAM — реалистичный объём для активного буфер-слоя.

**Bypass на уровне железа.** Мультиплексор — не программный переключатель, а аппаратный. Когда NPU не задействован (игра без поддержки DLSS 5), кадр идёт напрямую от GPU в display controller. Ноль задержки, ноль расхода энергии на NPU. Обратная совместимость со всеми существующими играми — автоматическая.

**Display controller остаётся на GPU.** NPU не имеет собственных видеовыходов. Он обрабатывает кадр и возвращает его в буфер GPU, откуда display controller выводит на экран. Это короткий возврат готовых данных — не вычислительная синхронизация, а передача кадра в буфер. Отпадает необходимость выносить HDMI/DP, HDCP, VRR и цветовые пространства на NPU-кристалл.

---

## 4. Почему именно SRAM-буфер, а не прямая шина

### 4.1. Задержка

Прямая межкристальная шина (например, NV-HBI в серверных Blackwell) обеспечивает полосу 14 ТБ/с, но добавляет задержку маршрутизации через контроллеры портов. SRAM-буфер через TSV при 3D-сборке даёт задержку доступа 1–2 нс — сопоставимо с кэшем L2/L3 [\[10\]](https://en.wikipedia.org/wiki/CPU_cache).

### 4.2. Асинхронность

Прямая шина требует синхронизации: GPU должен дождаться, пока NPU освободится. SRAM-буфер работает как асинхронный коммутатор — GPU пишет следующий стрип, пока NPU обрабатывает предыдущий. Ни один из кристаллов не простаивает в ожидании другого.

### 4.3. Масштабируемость

NPU-кристалл можно обновлять независимо от GPU. Новая модель нейросети, новый техпроцесс, больший объём вычислений — всё это меняется заменой одного die, без переработки GPU. Аналогия — AMD обновляет кэш-кристалл в Ryzen X3D, оставляя CCD без изменений [\[11\]](https://en.wikipedia.org/wiki/AMD_3D_V-Cache).

---

## 5. Сравнение подходов

| Параметр | Тензорные ядра на GPU (текущий) | Neural Upstream (мод) | 2-я видеокарта (мод) | **GPU + NPU + SRAM-буфер (концепция)** |
|---|---|---|---|---|
| Падение FPS от нейросети | ~2x | ~0,4x | ~0x (на GPU) | **~0x (на GPU)** |
| Качество изображения | Максимальное | Сниженное (uncanny valley) | Максимальное | **Максимальное** |
| Энергопотребление | Высокое (конкуренция за питание) | Низкое | Высокое (2 карты) | **Низкое (NPU оптимизирован)** |
| Обратная совместимость | Полная | Полная | Нет | **Полная (железный bypass)** |
| Массовость | Да | Да (мод) | Нет | **Да** |
| Стоимость производства | Базовая | Базовая | 2x (2 карты) | **+1 die + буфер** |

---

## 6. Технологическая готовность

Все компоненты предложенной архитектуры существуют и отработаны в массовом производстве:

- **3D-сборка с TSV** — AMD 3D V-Cache: миллионы проданных процессоров с дополнительным кристаллом SRAM поверх CCD [\[11\]](https://en.wikipedia.org/wiki/AMD_3D_V-Cache).
- **Multi-die на подложке** — NVIDIA GB200: два вычислительных die на общей подложке через NV-HBI [\[12\]](https://en.wikipedia.org/wiki/Nvidia_Blackwell).
- **Кэш как промежуточный буфер** — AMD Infinity Cache: 128 МБ SRAM на кристалле GPU, работает как массивный буфер между вычислителями и контроллером памяти [\[9\]](https://en.wikipedia.org/wiki/RDNA_(microarchitecture)).
- **NPU в потребительском сегменте** — AMD планирует встроить NPU в настольные Ryzen 10 000 (Zen 6, 2027), отказавшись от встроенной графики ради нейронного сопроцессора [\[13\]](https://club.dns-shop.ru/digest/175996-amd-lishit-ryzen-10-000-na-zen-6-vstroennoi-grafiki-radi-npu-wc/).
- **Выделенный AI-сопроцессор в GPU** — Apple A19 Pro: Neural Accelerators встроены в GPU-ядра [\[14\]](https://www.articsledge.com/post/neural-processing-unit-npu).

Ни один из компонентов не требует принципиально новых технологий. Требуется архитектурная интеграция — проектирование межкристального взаимодействия, разработка программной модели и оптимизация NPU под конкретную модель нейросети.

---

## 7. Ожидаемые сложности

**Программная модель.** Даже при железном bypass нужен API, который направляет кадр в NPU и обратно. Вероятный путь — расширение NVIDIA RTX SDK, прозрачное для разработчиков игр.

**Площадь и стоимость.** Три кристалла (GPU + SRAM-буфер + NPU) дороже одного. Первое поколение, вероятно, будет флагманским. Однако потоковая буферизация (16–32 МБ вместо 100+) и независимое масштабирование NPU снижают издержки по сравнению с «лобовым» увеличением GPU.

**Конвейерная задержка.** NPU должен обрабатывать стрип быстрее, чем GPU пишет следующий. При 90 FPS и размере стрипа 4 МБ на 1440p — ~0,5 мс на стрип. Если NPU не успевает, возникает задержка в 1–2 кадра. Для одиночных игр терпимо, для киберспорта — нет.

---

## 8. Заключение

DLSS 5 Neural Rendering — не косметическое улучшение, а качественный скачок в визуальном реализме. Но текущая реализация на общих тензорных ядрах GPU аппаратно невыгодна: двукратное падение FPS и энергопредел на флагмане.

Предложенная архитектура — выделенный NPU на подложке, связанный с GPU через потоковый SRAM-буфер — решает обе проблемы, не жертвуя качеством изображения. Все необходимые технологии уже существуют в массовом производстве. Требуется архитектурная воля, а не научный прорыв.

Концепция не претендует на инженерную полноту — расчёты площади кристалла, энергобаланса и точной задержки требуют симуляции на уровне RTL. Цель — обозначить направление, которое индустрия, по всей видимости, уже рассматривает, и дать ему публичную формулировку.

---

## Ссылки

1. NVIDIA. *DLSS 5: 3D-Guided Neural Rendering*. nvidia.com
2. TweakTown. *DLSS 5 in NBA 2K27: Performance Analysis*. tweaktown.com
3. GameGPU. *Нейросетевой рендеринг DLSS 5: двукратная разница FPS в The Witcher 3*. gamegpu.com
4. ixbt.com. *Активация DLSS 5 снижает производительность в 2,5 раза*. ixbt.com
5. TechPowerUp. *Early DLSS 5 Testing: RTX 5090 Power Connector Bottleneck*. techpowerup.com
6. TechPowerUp. *Modders Rework DLSS 5 Pipeline for Performance Boost*. techpowerup.com
7. TechSpot. *Modders Found Two Ways to Make DLSS 5 Faster*. techspot.com
8. TechPowerUp. *Mod Revives Multi-GPU with Dedicated DLSS 5 GPU*. techpowerup.com
9. Wikipedia. *RDNA (microarchitecture)*. en.wikipedia.org
10. Wikipedia. *CPU cache*. en.wikipedia.org
11. Wikipedia. *AMD 3D V-Cache*. en.wikipedia.org
12. Wikipedia. *Nvidia Blackwell*. en.wikipedia.org
13. DNS Club. *AMD лишит Ryzen 10 000 встроенной графики ради NPU*. club.dns-shop.ru
14. Articsledge. *Neural Processing Unit (NPU)*. articsledge.com

---

*Автор — не инженер-архитектор, а концептуальный мыслитель. Статья отражает архитектурную идею, а не инженерный расчёт. Расчёты площади, энергобаланса и задержки требуют симуляции на уровне RTL. Цель публикации — дать идее публичное закрепление и шанс дойти до инженерных команд, способных её реализовать.*
## Фотонная реализация: от чиплета к оптическим вычислениям

Концепция нейро-чиплета естественным образом ложится на кремниевую фотонику. Вместо того чтобы ждать появления полностью фотонного GPU, чиплетная архитектура позволяет внедрять фотонику поэтапно — в виде отдельного вычислительного тайла рядом с электронными.

### Почему чиплеты — идеальный переходный путь

- **Разные технологические нормы.** Фотонные компоненты лучше работают на 45–90 нм (из‑за больших волноводов), а электронная логика выигрывает от 3 нм. В монолитном кристалле совместить это нельзя, а в чиплетной упаковке — вполне.
- **Поэтапное внедрение.** Не нужно заменять весь GPU целиком: достаточно добавить фотонный тайл там, где он даёт максимальный эффект, — для матричных вычислений под нейрорендер.
- **Независимое масштабирование.** Тайлы памяти, вычислений и ввода‑вывода можно обновлять отдельно — как комплектующие в ПК, только на уровне кристалла.

---

### Предлагаемая архитектура

| Тайл | Технология | Функция |
| --- | --- | --- |
| Рендер‑тайл | Электроника (3 нм) | Растеризация, трассировка лучей, геометрия |
| Нейро‑тайл | Фотоника (матричные вычисления на свете) | DLSS, нейрошейдеры, AI‑апскейл |
| Memory‑тайл | Электроника (HBM / DRAM) | Буфер кадра, веса нейросетей, текстуры |
| Интерконнект | Оптика (кремниевые волноводы на интерпозере) | Передача данных между всеми тайлами с ультранизкой задержкой |

---

### Что уже существует (по состоянию на 2026 год)

Каждый компонент этой архитектуры уже находится в разработке или производстве — просто пока их не собрали вместе именно в таком виде.

- **Celestial AI** (при поддержке NVIDIA и AMD): раздельные тайлы вычислений и памяти, соединённые фотонной шиной на скорости 14,4 Тбит/с. Память масштабируется независимо от вычислений.  
  Источники: [презентация Kisaco Research](https://www.kisacoresearch.com/sites/default/files/presentations/preet_virk_-_celestial_ai_-_photonic_fabrictm_based_scale-up_network.pdf), [The Next Platform](https://www.nextplatform.com/compute/2024/04/04/celestial-ai-wants-to-break-the-memory-wall-fuse-hbm-with-ddr5/1646782).
- **Ayar Labs** (спин‑офф MIT, оценка \$3,8 млрд на март 2026): фотонные чиплеты‑прослойки, которые устанавливаются прямо на корпус существующего процессора. Их чип TeraPHY обеспечивает 8 Тбит/с при задержке 10 нс — без замены самого электронного GPU.  
  Источник: [andrew.ooo](https://andrew.ooo/posts/ayar-labs-500m-series-e-photonic-ai-chips/).
- **Концепт NVIDIA на IEDM 2024**: вычислительные тайлы GPU в 3D‑стеке, сверху — DRAM, между ними — 12 кремний‑фотонных каналов. Вычисления и память остаются электронными, связь — оптической.  
  Источник: [TechPowerUp](https://www.techpowerup.com/329651/nvidia-shows-future-ai-accelerator-design-silicon-photonics-and-dram-on-top-of-compute).
- **Neurophos** (при финансировании фонда Билла Гейтса): оптические транзисторы в 10 000 раз меньше прежних решений, созданные по существующим полупроводниковым техпроцессам. Чип Tulkas T100 работает на частоте 56 ГГц и, по заявлениям, превосходит NVIDIA Vera Rubin NVL72 в задачах FP4/INT4. Целевой срок массового производства — примерно 2028 год.  
  Источник: [Tom’s Hardware](https://www.tomshardware.com/tech-industry/semiconductors/bill-gates-backed-silicon-photonics-startup-develops-optical-transistors-10-000x-smaller-than-current-tech-optical-chip-can-process-1-000-x-1-000-multiplication-matrices).
- **Университет Пэрдью**: фотонные транзисторы на одиночных фотонах, совместимые с КМОП‑процессами, работают при комнатной температуре.  
  Источник: [SecurityLab](https://www.securitylab.ru/news/566420.php).
- **Imec**: впервые удалось вырастить GaAs‑лазеры прямо на 300‑мм кремниевых пластинах в стандартной КМОП‑линии — без отдельного приклеивания кристаллов.  
  Источник: [TechXplore](https://techxplore.com/news/2025-01-silicon-photonics-advance-paves-effective.html).
- **Проект PICNIC** (научная публикация, ноябрь 2025): 3D‑стек чиплетов с RRAM‑памятью и кремний‑фотонным интерконнектом — улучшение энергоэффективности в 57 раз по сравнению с обычными GPU.  
  Источник: [arXiv](https://arxiv.org/pdf/2511.04036).

---

### Чего пока не хватает

- **Оптическая память.** Все текущие фотонные чипы используют электронную память. В Berkeley Lab показали оптическую бистабильность в наночастицах (наноразмерная оптическая память), но до практического применения ещё далеко.  
  Источник: [ScienceDaily](https://www.sciencedaily.com/releases/2025/02/250226125021.htm).
- **Точность.** Фотонные вычисления — аналоговые, а не цифровые. Для нейрорендера (где допустима точность 8 бит) это приемлемо, для универсальных вычислений — пока нет.
- **Полностью фотонный потребительский GPU.** Не раньше 2030–2035 годов. Но чиплетный подход не требует ждать этого момента.

---

### Ключевая идея

Концепции нейро‑чиплета не нужен полностью фотонный GPU, чтобы быть рабочей. Ей нужно:

1. **Фотонный тайл для матричной математики** (DLSS, нейрошейдеры) — уже существует (Neurophos, PICNIC).
2. **Оптический интерконнект между тайлами** — уже существует (Ayar Labs, Celestial AI, концепт NVIDIA на IEDM).
3. **Электронные тайлы для растеризации и памяти** — это уже стандарт индустрии.

## Оптический broadcast: как фотоника решает проблему распределения данных

Одно из уникальных преимуществ фотонных вычислений — то, чего принципиально нет у чистой электроники: возможность **одновременно передавать одни и те же данные сразу в несколько вычислительных блоков** без копирования, буферизации и расхода пропускной способности памяти.

В электронной системе, чтобы подать один и тот же кадр на несколько ИИ‑блоков (DLSS, генерацию кадров, реконструкцию лучей, нейрошейдеры), приходится:
* либо многократно считывать данные из памяти,  
* либо явно дублировать их по шинам и буферам.  

Каждое такое копирование — это лишняя энергия, задержка и занятая площадь кристалла. Чем больше потребителей, тем больше копий нужно создать — и тем сильнее растёт нагрузка на память.

Фотоника обходит эту проблему за счёт **оптического broadcast**: один световой сигнал делится на несколько идентичных копий с помощью делителей (beam splitters). При наличии $j$ делителей можно получить $2^j$ копий одного потока данных. Каждая копия одновременно уходит в свой вычислительный блок — **без дополнительных обращений к памяти, без сериализации и без буферов**.

---

### Как это даёт настоящий параллелизм для нейрорендера

В твоей предложенной архитектуре рендер‑тайл формирует кадр, который сразу нужен нескольким нейроблокам:

* **DLSS** — для апскейла;  
* **Frame Generation** — для генерации промежуточных кадров;  
* **Ray Reconstruction** — для шумоподавления и «очистки» трассировки лучей;  
* **Нейрошейдеры** — для улучшения материалов и освещения.

При электронных соединениях каждый блок должен получить свою копию кадра — а значит, будут повторяющиеся чтения из памяти и конкуренция за шину. В фотонной схеме рендер‑тайл выдаёт кадр как оптический сигнал, а компактный делитель на кристалле **одновременно** распределяет его по всем нейротайлам. Все блоки начинают обработку одного и того же кадра практически в один и тот же момент.

Так исчезает «штраф за разветвление»: стоимость раздачи данных не растёт с увеличением числа потребителей.

---

### Реальные реализации, где уже используют этот принцип

* **Протокол Broadcast‑and‑Weight** (предложен в 2014, продемонстрирован в 2017): входные данные транслируются сразу на несколько длин волн с помощью электрооптической модуляции, и каждая копия независимо взвешивается в своём канале. Это позволяет одному входу управлять множеством параллельных вычислений без дублирования данных.  
  Источник: [HAL Science](https://hal.science/hal-04580418/document)

* **Lightening‑Transformer** (arXiv, 2024) — архитектура фотонного ускорителя для трансформеров, которая прямо опирается на оптический broadcast для совместного использования операндов между ядрами. В статье так и сказано: «Мы используем естественную возможность оптического broadcast, чтобы обеспечить совместное использование операндов внутри и между ядрами». Один тензор распределяется по нескольким вычислительным ядрам как свет, а не как скопированные данные.  
  Источник: [arXiv](https://arxiv.org/html/2305.19533)

* **Фотонный тензорный ядро** (Nature, 2021) использует частотное мультиплексирование (WDM): одна и та же матрица применяется сразу к **четырём входным векторам**, причём каждый вектор закодирован на своей длине волны, и все они проходят через одну и ту же физическую матрицу. Так за один такт выполняется 16 операций умножения‑накопления.  
  Источник: [ResearchGate](https://www.researchgate.net/publication/339015092_Parallel_convolution_processing_using_an_integrated_photonic_tensor_core)

Эти примеры показывают, что индустрия уже рассматривает оптический broadcast не как экзотику, а как базовый механизм для эффективных параллельных вычислений.

---

### Архитектурная схема (ASCII)

```
[Рендер‑тайл] ──→ [Оптический делитель]
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
   [DLSS‑тайл]   [FrameGen‑тайл]  [RayRecon‑тайл]
          │            │            │
          └────────────┼────────────┘
                        ↓
              [Сборка результата] → Дисплей
```

Рендер‑тайл испускает одно оптическое представление кадра. Делитель создаёт одновременные копии, каждая из которых идёт в свой нейротайл. Все тайлы работают параллельно над одним и тем же входом — без сериализации и узкого места в памяти.

Кроме того, фотонные волноводы могут пересекаться друг с другом с минимальными наводками — в отличие от металлических проводников, которым нужны сложные маршруты и буферы, чтобы избежать помех. Это делает физическую разводку таких широковещательных сетей заметно проще и масштабируемее.
