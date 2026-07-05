# Multi-Channel Light Barrier Array with Frequency Multiplexing

Projektbericht – Praxissemester-Modul (PSM), STM32F767 / ARM Cortex-M7
Authors: D. Isermann, P. Gramescu

A compact LED-Photodiode array that detects the interruption of four LEDs using a single ADC channel via Signal Sumation and FFT-based frequency separation. Four LEDs are each PWM-modulated at a distinct frequency; the combined photodiode signal is analog-summed, sampled, and decomposed via FFT on an STM32F767 microcontroller. 

An extended application repurposes the array as a finger-position controller (for VCO/synth control) using weighted peak-coverage interpolation.

## Motivation

Classic light barriers require one full receiver chain per channel. This project reduces hardware cost by combining multiple optical channels onto a single ADC input, then separating them digitally via FFT.

## System Overview

- 4 LEDs, each driven by PWM at a distinct frequency: **15 kHz, 20 kHz, 25 kHz, 30 kHz**
- 4 photodiodes, one per LED, each followed by a transimpedance amplifier
- All 4 amplified channels are combined into a single summed signal via an inverting adder stage
- The summed signal is digitized by a single ADC channel and decomposed via FFT
- Amplitude at each known modulation frequency indicates whether the corresponding light barrier is blocked or clear
- Each barrier's state is shown on a dedicated status LED

## Hardware

### Analog Front End (per channel)
- **Transimpedance amplifier**: AD8648 op-amp, 500 kΩ feedback resistor, 3.3 pF compensation capacitor (bandwidth limiting / stability)
- **Summing stage**: inverting adder, 2 kΩ input resistors, 1 kΩ feedback resistor (gain < 1 to avoid saturation with multiple active channels), AC-coupled inputs (100 nF) to remove per-channel DC offset
- Reference biasing (`Vmean`, `Vbias`) centers signals for single-supply operation

### Microcontroller
- **STM32F767** (ARM Cortex-M7) — PWM generation, multi-channel ADC, DSP/FFT capability
- ADC sample rate: **100 kHz** (TIM1-triggered), driven by DMA
- TIM3 used for hyperperiod scheduling (20 Hz cycle) — ADC/DMA are powered down while FFT is computed to save energy
- Timers 10/11/13/14 generate the four PWM channels (50% duty cycle)

### PCB (KiCad)
- 2-layer board (top: components + signal traces, bottom: GND plane)
- SMD-only except connectors, min. trace width 0.25 mm, no vias on critical signals
- LEDs/photodiodes connected via ribbon cable
- Built as an add-on to an existing shield (template-constrained connector positions)

## Simulation (LTSpice)

- Photodiode modeled as a current source (15 µA) + diode, with an optional disturbance current source for ambient-light/noise studies
- Transimpedance and adder stages simulated and verified in time and frequency domain
- FFT of the summed output confirms clear, separated peaks at 15/20/25/30 kHz, with expected PWM harmonics at higher frequencies not overlapping the fundamentals

## Firmware

- Built with **STM32CubeIDE**; peripheral setup via `.ioc` (System Core, Analog, Timers)
- System clock: 200 MHz
- ADC conversion triggered by TIM1 interrupt, transferred via DMA
- `HAL_TIM_PeriodElapsedCallback` starts the ADC/DMA sampling window each hyperperiod
- FFT: `arm_rfft_fast_f32` (512-point) + `arm_cmplx_mag_f32` → 256 magnitude bins (CMSIS-DSP)
- Relevant bins for 15/20/25/30 kHz: indices **77, 102, 128, 154**
- Each bin is compared against a threshold to drive the corresponding status LED

### Extended mode — finger-position sensing

Reinterprets the 4-channel coverage pattern as a continuous position value. Smoothed via a 4-sample ring-buffer moving average, then combined into a weighted centroid. 

The resulting value drives both a PWM duty cycle (status LED brightness) and a DAC output, letting the array act as a swipe-based analog controller (e.g. VCO frequency control).
