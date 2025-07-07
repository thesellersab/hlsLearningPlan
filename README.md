# 6-Week Vitis HLS Learning Roadmap

A carefully paced, hands-on plan that turns a second-year CS student into a confident beginner HLS designer. Each week blends core theory, guided labs, and self-checks so you steadily build intuition while documenting every experiment.

## Overview

High-Level Synthesis (HLS) turns familiar C/C++ ideas into FPGA hardware, letting software-oriented developers unlock parallelism without wading straight into Verilog. Over the next six weeks you will:

- Install and license Vitis HLS, then explore the GUI and command-line flows.
- Prototype a “Hello-World” vector-adder, then iterate with pragmas that reveal how loops map to pipelines.
- Grow from single-function kernels to multi-process dataflow designs, culminating in two open-ended stretch projects.

Expect to invest roughly 8hrs each week. Treat your lab notebook as a living log: record goals, changes, synthesis results (latency, LUTs, BRAMs) and what you would try next time. Analysis plus iteration is the heart of hardware design.

## Weekly Learning Table

| Week | Key Concepts / Topics | Hands-On Lab / Project Focus | Recommended Resources (read/watch **before** lab) | Time Commitment | Success Criteria / Mini-Quiz Ideas |
| :-- | :-- | :-- | :-- | :-- | :-- |
| 1 | -  FPGA vs. CPU/GPU basics<br>-  Vitis HLS tool overview<br>-  C sim vs. RTL sim | Build a “Hello-World” **vector addition** kernel from template; run C simulation | -  Vitis HLS User Guide UG1399, “Getting Started” section[^1]<br>-  GitHub “Hello World” example README[^2]<br>-  YouTube intro video (6 min)[^3] | 8hrs | - Describe the difference between C simulation and RTL simulation (1-sentence).<br>- Explain why AXI4 interfaces are needed (multiple-choice). |
| 2 | -  Interfaces: AXI4-Lite, AXI4-MM<br>-  Loop pipelining \& initiation interval (II)<br>-  Lite vs. burst access patterns | Add **pipeline pragma** to inner loop of vector adder; measure II change | -  UG1399 “Pipelining” chapter[^1]<br>-  Quick-Start Guide sections 4–6 (unroll, pipeline)[^4] | 8hrs | - Given a loop body and II=3, compute expected latency for N=64.<br>- Spot which pragma line is wrong in a code snippet. |
| 3 | -  Loop unrolling vs. array partitioning<br>-  Resource trade-offs (LUTs, DSPs, BRAMs)<br>-  Synth → Implementation reports | Duplicate vector adder function, but unroll by factor 2 and partition arrays; compare resource usage | -  UG1399 “Resource Analysis Report”[^1]<br>-  Imperix workflow tutorial resource section[^5] | 8hrs | - Fill in a table: unroll factor vs. latency vs. LUT count (from your run).<br>- Short answer: why does partitioning reduce burst length? |
| 4 | -  Dataflow task-level parallelism<br>-  Streams \& FIFOs<br>-  Testbench design patterns | Refactor code into 4 tasks (read1, read2, add, write) with `#pragma HLS DATAFLOW`; verify functional equivalence | -  GitHub Hello-World example discussion of `dataflow`[^6]<br>-  UG1399 “Task-Level Parallelism”[^7] | 8hrs | - Screenshot or note the new performance report showing task overlap.<br>- Quiz: identify the deadlock cause in a mis-wired stream diagram. |
| 5 | -  Creating reusable IP<br>-  Exporting `.xo` vs. Vivado IP<br>-  Integrating IP into Vivado block design | Export your vector adder as **IP**, import it into a blank Vivado design, and connect to AXI interconnect (no bitstream needed) | -  UG1399 “Exporting RTL” steps[^1]<br>-  XUP Design Flow Lab 1, section 3 (Export IP)[^8] | 8hrs | - Written check-list: four files generated when exporting.<br>- Drag-and-drop challenge: match IP ports to AXI pins in block diagram. |
| 6 | -  Performance vs. area tuning strategy<br>-  Scripted flows with `vitis_hls -f script.tcl`<br>-  Intro to HLS Cosimulation | Automate Week 2–4 experiments with a TCL script; run C/RTL co-sim; generate final project report summarizing observed trade-offs | -  Vitis HLS TCL command cheatsheet in UG1399 appendix[^1]<br>-  Vitis Tutorials “Creating a Project via TCL” page[^9] | 8hrs | - Provide the TCL snippet that sets pipeline unroll factor.<br>- Multiple-choice: which stage does co-sim catch that C sim cannot? |

## Installing \& Licensing Vitis HLS (Windows \& Linux)

1. **Download** the unified installer from AMD Adaptive Computing Downloads[^10].
2. **Select tools**: keep Vitis, Vivado, and Vitis HLS checked by default (Model Composer optional)[^11].
3. **Disk space**: allocate at least 44 GB on Windows or ∼50 GB on Linux[^12].
4. **Run post-install script (Linux only)**:

```bash
sudo <install_dir>/Vitis/<release>/scripts/installLibs.sh
```

to pull required distro packages[^11].
5. **Environment setup** every terminal session:

```bash
source <install_path>/Vitis/<ver>/settings64.sh
source /opt/xilinx/xrt/setup.sh
```

Add these to `.bashrc` for convenience[^13].
6. **Launch the IDE**:
    - Windows: Start Menu → Vitis HLS 2025.1 (or `vitis_hls.bat`)[^14].
    - Linux: `vitis_hls` or `vitis --classic` in a terminal[^14].
7. **License**: the basic Vitis HLS feature is node-locked and free; use the “Get Free Vivado/ISE WebPACK License” option in the Xilinx Licensing Client, then point `XILINXD_LICENSE_FILE` to the generated `.lic` file (GUI guides you)[^10].

*Troubleshooting tip:* If the GUI stalls during first launch on Windows, run as Administrator once to let it create `%APPDATA%\Xilinx` folders[^12].

## Starter “Hello-World” Project Outline

### 1. C/C++ Kernel Skeleton

```cpp
// vadd.h
#pragma once
#define VEC_SIZE 1024

void vadd(const int *a,
          const int *b,
          int *c,
          int size);
```

```cpp
// vadd.cpp
#include "vadd.h"

void vadd(const int *a,
          const int *b,
          int *c,
          int size) {
  #pragma HLS INTERFACE m_axi port=a  bundle=gmem0
  #pragma HLS INTERFACE m_axi port=b  bundle=gmem1
  #pragma HLS INTERFACE m_axi port=c  bundle=gmem0
  #pragma HLS INTERFACE s_axilite port=a  bundle=control
  #pragma HLS INTERFACE s_axilite port=b  bundle=control
  #pragma HLS INTERFACE s_axilite port=c  bundle=control
  #pragma HLS INTERFACE s_axilite port=size bundle=control
  #pragma HLS INTERFACE s_axilite port=return bundle=control
  #pragma HLS PIPELINE II=1

  for (int i=0; i<size; ++i) {
      #pragma HLS LOOP_TRIPCOUNT min=VEC_SIZE max=VEC_SIZE
      c[i] = a[i] + b[i];
  }
}
```


### 2. Project Flow (GUI or TCL)

1. **Create project:** *File → New Project* → add `vadd.cpp` and `vadd.h`; set `vadd` as top function[^9].
2. **C Simulation:** click the green play icon; verify pass/fail in console[^13].
3. **C Synthesis:** run *Solution → Run C Synthesis*; examine latency, resources, and II in the report[^1].
4. **Cosimulation (optional Week 6):** *Solution → Run C/RTL Co-simulation*; ensure waveforms match[^13].
5. **Export RTL/IP:** *Solution → Export RTL*; choose Vivado IP or Vitis Kernel (.xo)[^8].

*Command-line shortcut:*

```bash
vitis_hls -f run_hls.tcl
```

where `run_hls.tcl` contains:

```tcl
open_project vadd_prj
set_top vadd
add_files vadd.cpp
add_files -tb vadd_test.cpp
open_solution "sol1"
set_part {xczu7ev-ffvc1156-2-e}
create_clock -period 5
csim_design
csynth_design
export_design -format ip_catalog
exit
```

(Modify `set_part` for your board.)

## Two Stretch Goals for Motivated Learners

- **Pipelined FIR Filter (16-tap):** write an HLS function that performs streaming FIR on AXI4-Stream input; apply `DATAFLOW` plus loop unroll to push II to1. Evaluate DSP block usage and compare against a hand-tuned reference[^2][^4].
- **Optimized Matrix-Multiply (64×64):** start from XUP Lab-1 matrix-mul example[^8]; progressively add block-RAM partitioning, block-tiling, then complete array partitioning to hit sub-1 µs latency on a Zynq MPSoC target[^7]. Document every pragma change and resulting resource trade-offs.


## Tips for Productive Iteration \& Documentation

### Keep a Design Diary

- Date, goal, tool version.
- Code diff or pragma change.
- Key metrics (II, latency, LUTs, BRAM, DSP).
- Observations: did latency drop as expected? why/why not?


### Use `run_[exp].tcl` Scripts

Script each hypothesis so you can re-run on a clean workspace, then commit script plus HTML reports to Git for traceability[^9].

### Celebrate Small Wins

Each synthesis run that meets timing is a milestone. Note what you learned, even if the improvement is tiny. Hardware expertise is built from incremental insights.

## Frequently Asked Questions

### “I changed a pragma but latency didn’t drop!”

Check whether a memory port became the bottleneck; pipelining loops without partitioning arrays often leaves memory bandwidth unaffected[^1].

### “Why does unrolling explode LUT usage?”

Parallel hardware = replicated logic. Use resource viewer and consider instead `PIPELINE II>1` or partial unroll factors[^7].

### “Do I need an FPGA board?”

Not for Weeks 1–4; C/RTL simulation is enough. Week 5 export works even without connecting real hardware. A board becomes essential once you want bitstreams and on-device profiling.

## What’s Next After Week 6?

1. **Hybrid C/RTL:** mix your HLS kernels with hand-coded Verilog for finest-grain control.
2. **Open-Source Front-End:** explore the public Vitis HLS Clang-based front-end on GitHub for research projects[^15].
3. **LLM-Assisted Refactoring:** experiment with automatically converting legacy C to HLS-friendly style using AI techniques[^16].
4. **Advanced Debug:** learn HLS debug overlays to capture runtime waveforms with minimal re-synthesis overhead[^17].

Document every new journey, stay curious, and remember: the fastest hardware iteration loop is a well-designed experiment.

<div style="text-align: center">⁂</div>

[^1]: https://www.xilinx.com/support/documents/sw_manuals/xilinx2022_2/ug1399-vitis-hls.pdf

[^2]: https://xilinx.github.io/Vitis_Accel_Examples/2021.1/html/hello_world.html

[^3]: https://www.youtube.com/watch?v=WdLKrxYbcbs

[^4]: https://github.com/vickyiii/Quick-Start-Guide-for-HLS

[^5]: https://imperix.com/doc/help/xilinx-vitis-hls

[^6]: https://xilinx.github.io/Vitis_Accel_Examples/2022.2/html/hello_world.html

[^7]: https://docs.amd.com/r/en-US/ug1399-vitis-hls/HLS-Programmers-Guide

[^8]: https://xilinx.github.io/xup_high_level_synthesis_design_flow/Lab1.html

[^9]: https://xilinx.github.io/Vitis-Tutorials/2022-1/build/html/docs/Getting_Started/Vitis_HLS/new_project.html

[^10]: https://docs.amd.com/r/en-US/ug1742-vitis-release-notes/Installing-the-Vitis-Software-Platform

[^11]: https://docs.amd.com/r/en-US/ug1400-vitis-embedded/Installing-the-Vitis-Software-Platform

[^12]: https://cgi.cse.unsw.edu.au/~cs4601/22T2/labs/vivado-installation-guide.pdf

[^13]: https://xilinx.github.io/Vitis-Tutorials/2022-1/build/html/docs/Getting_Started/Vitis_HLS/Getting_Started_Vitis_HLS.html

[^14]: https://digilent.com/reference/programmable-logic/guides/vitis-launch?srsltid=AfmBOoq72AVip5drEZ97hn1PiKJJCGunaK5MRz-vgAFzJGYfk2jMN3xV

[^15]: https://community.amd.com/t5/adaptive-computing/opening-a-world-of-possibilities-vitis-hls-front-end-is-now-open/ba-p/558688/page/2

[^16]: https://ieeexplore.ieee.org/document/10691856/

[^17]: https://dl.acm.org/doi/10.1145/3372490

[^18]: https://ieeexplore.ieee.org/document/11044149/

[^19]: https://ieeexplore.ieee.org/document/10622147/

[^20]: https://ieeexplore.ieee.org/document/10764626/

[^21]: https://dl.acm.org/doi/10.1145/3695794.3695815

[^22]: https://dl.acm.org/doi/10.1145/3676849

[^23]: https://onlinelibrary.wiley.com/doi/10.1002/nur.22450

[^24]: https://www.semanticscholar.org/paper/330aecf699b0b465bcf30863c93e6846016b5055

[^25]: https://dl.acm.org/doi/10.1145/3706628.3708861

[^26]: https://xilinx.github.io/Vitis-Tutorials/2021-1/build/html/docs/Getting_Started/Vitis_HLS/Getting_Started_Vitis_HLS.html

[^27]: https://docs.amd.com/r/en-US/ug1701-vitis-accelerated-embedded/Installing-the-Vitis-Software-Platform

[^28]: https://xilinx.github.io/Vitis_Accel_Examples/2019.2/html/hello_world.html

[^29]: https://github.com/Xilinx/Vitis-Tutorials

[^30]: https://github.com/Xilinx/Vitis-HLS-Introductory-Examples

[^31]: https://www.mdpi.com/2079-9292/10/5/532/pdf?version=1614760575

[^32]: http://arxiv.org/pdf/2411.11384.pdf

[^33]: http://arxiv.org/pdf/1805.03648.pdf

[^34]: https://www.mdpi.com/2079-9292/9/12/2024/pdf

[^35]: http://www.hrpub.org/download/20200530/UJEEE4-14915137.pdf

[^36]: http://arxiv.org/pdf/1812.07012.pdf

[^37]: https://www2.ia-engineers.org/conference/index.php/iciae/iciae2016/paper/download/971/652

[^38]: https://arxiv.org/pdf/2501.14631.pdf

[^39]: https://www.mdpi.com/2079-9292/10/14/1746/pdf

[^40]: https://arxiv.org/pdf/2311.08198.pdf


