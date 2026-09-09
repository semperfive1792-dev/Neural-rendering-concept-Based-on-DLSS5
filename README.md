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
