# Dissertation
# Carbon-Aware Roofline Model (CARM)

This repository contains the full implementation, scripts, benchmarks, and results for my MSc Dissertation:  
**Carbon-Aware Roofline Model: Integrating Performance and Carbon Footprint Analysis in High-Performance Computing**.

It provides a reproducible framework to:
- Derive **performance ceilings** (peak FLOPs and sustained memory bandwidth).  
- Measure **operational energy and carbon** (CPU package + DRAM power).  
- Estimate **embodied carbon** of server hardware.  
- Generate **augmented roofline plots** that integrate carbon awareness.

---

## 📖 Table of Contents
- [System Setup](#system-setup)  
- [Benchmarks](#benchmarks)  
- [Energy Measurement](#energy-measurement)  
- [Embodied Carbon](#embodied-carbon)  
- [Analysis & Plots](#analysis--plots)  
- [Results](#results)  
- [Repository Structure](#repository-structure)  
- [How to Reproduce](#how-to-reproduce)  
- [Acknowledgements](#acknowledgements)

---

## System Setup

### Hardware
- **CPUs:** 2 × Intel Xeon E5-2640 v2 (Ivy Bridge-EP), 8 cores/socket, SMT2 → 32 threads  
- **Memory:** 64 GiB DDR3 (≈ 66 GiB usable), 2 NUMA nodes  
- **Storage:** 2 × 300 GB SAS HDD + 2 × 4 TB SATA HDD (RAID ~3.6 TB)  
- **Network:** Intel I350 GbE (dual-port) + Intel X520 10 GbE (dual-port)  
- **Chassis:** Generic 2-socket rackmount server  
- **Sensors:** Intel RAPL (per-socket package + DRAM)  

### Software
- Ubuntu 18.04.6 LTS (Linux 4.15 kernel)  
- GCC 9.4.0 (C benchmarks)  
- LIKWID (`likwid-perfctr`)  
- Linux `perf`  
- Empirical Roofline Toolkit (ERT)  
- Python 3.6 + NumPy, Pandas, Matplotlib  

### Environment Variables
```bash
export OMP_NUM_THREADS=32
export OMP_PROC_BIND=spread
export OMP_PLACES=cores
```
Benchmarks
Ceilings
STREAM Triad (ERT): sustained memory bandwidth 
𝐵
sust
B 
sust
​
 .

AVX Microkernel: custom floating-point kernel for 
𝑃
peak
P 
peak
​
 .

Note: Ivy Bridge lacks FMA3; peak is based on AVX-only FLOPs.

Representative Kernels
Triad (custom C) — memory bound.

Peakflops S0 / S1 — scalar vs vector AVX kernels (compute bound).

Source Code
benchmarks/triad_exe.c

benchmarks/peakflops_exe.c

benchmarks/avx_microkernel.c

## Energy Measurement
Run-Level (LIKWID)
```bash
likwid-perfctr -C 0-31 -g ENERGY -m -- ./triad_exe > results/energy_triad.txt
likwid-perfctr -C 0-31 -g ENERGY -m -- ./peakflops_exe --mode S0 > results/energy_peak_S0.txt
```
Interval (perf)
Both CPU package and DRAM domains are measured.

Example DRAM validation workload (20s tmpfs memory streaming):

```bash
perf stat -I 1000 -x, -e power/energy-pkg/,power/energy-ram/ \
  bash -c 'timeout 20s sh -c "while :; do cat /dev/shm/bigfile > /dev/null; done"' 2> results/rapl.csv
```
Processing
Convert Joules → Watts:

```awk
$4 ~ /power\/energy-pkg\// { ts=$1+0; pkgJ=$2+0; dt=(prev_ts>0? ts-prev_ts:1); pkgW=pkgJ/dt }
$4 ~ /power\/energy-ram\// { dramJ=$2+0; dramW=dramJ/dt; totW=pkgW+dramW; \
  printf "%.3f,%.2f,%.2f,%.2f\n", ts, pkgW, dramW, totW; prev_ts=ts }
```
Scripts in repo:

scripts/process_rapl.awk

scripts/median_power.sh

Validation Results
Median steady-state (20s DRAM workload):

CPU Package: ~56.7 W

DRAM: ~12.8 W

Total Reference Power 
𝑃
ref
P 
ref
​
 : ~69.5 W

Embodied Carbon
Config File
Defined in config/embodied.yaml:

yaml
Copy code
node:
  cpu:
    model: "Intel Xeon E5-2640 v2"
    quantity: 2
    pcf_kgco2e_each: 160
  dram:
    capacity_gb: 64
    kgco2e_per_gb: 1.0
  storage:
    hdd:
      quantity: 1
      pcf_kgco2e_each: 30
  nic:
    pcf_kgco2e: 15
  motherboard:
    pcf_kgco2e: 300

amortization:
  years: 4
  daily_availability: 0.833
  runtime_fraction: 0.40
Notes
Mid-point PCFs from literature.

Total node embodied footprint ≈ 729 kgCO₂e.

Override possible with top-down OEM server PCF.

Scripts:

tools/compute_embodied.py

Analysis & Plots
tools/build_carbon_roofline.py

Inputs: performance ceilings (data/ERT/roofline.json), power logs, embodied.yaml.

Outputs:

Carbon-aware Roofline plots (in results/figures/)

CSV of FLOP/gCO₂e values (results/carbon_with_embodied.csv)

Results
Key findings (see dissertation Chapter 5):

Package-only vs Package+DRAM power measurement differences.

Reference power 
𝑃
ref
P 
ref
​
  validated empirically (~69.5 W).

Triad kernel: bandwidth-bound, ~70% of 
𝐵
sust
B 
sust
​
 .

Peakflops: ~60–65% of theoretical 
𝑃
peak
P 
peak
​
 .

Carbon-aware roofline demonstrates trade-offs between FLOP/s and gCO₂e/FLOP.

Detailed results in:

Results Section of README

Figures

Processed CSVs

Repository Structure
bash
Copy code
benchmarks/       # Triad, Peakflops, AVX microkernel
scripts/          # AWK + bash processing scripts
tools/            # Python analysis (roofline, embodied carbon, CI fetch)
config/           # embodied.yaml (hardware PCF config)
data/             # Raw performance ceilings (ERT outputs)
results/          # Energy logs, RAPL CSV, watts.csv, figures
How to Reproduce
Clone repo

bash
Copy code
git clone https://github.com/soumyajitchattopadhyay/Dissertation
cd Dissertation
Compile benchmarks

bash
Copy code
cd benchmarks
make
Run ceilings (ERT)

bash
Copy code
./run_stream.sh
./avx_microkernel
Collect energy

bash
Copy code
likwid-perfctr -C 0-31 -g ENERGY -m -- ./triad_exe
perf stat -I 1000 -x, -e power/energy-pkg/,power/energy-ram/ -- ./triad_exe
Process logs

bash
Copy code
awk -f scripts/process_rapl.awk results/rapl.csv > results/watts.csv
bash scripts/median_power.sh results/watts.csv
Generate plots

bash
Copy code
python3 tools/build_carbon_roofline.py
Acknowledgements
University of Glasgow HPC Group (hardware access)

LIKWID team (performance monitoring toolkit)

Empirical Roofline Toolkit developers

yaml
Copy code

---

This README is **long, detailed, and has anchor-friendly headings** (`## Results`, `## Energy Measurement`, etc.) so you can deep-link from your dissertation like:

- [Validation Test](https://github.com/soumyajitchattopadhyay/Dissertation#energy-measurement)  
- [Embodied Carbon Config](https://github.com/soumyajitchattopadhyay/Dissertation#embodied-carbon)  
- [Results](https://github.com/soumyajitchattopadhyay/Dissertation#results)  

---

## Embodied Carbon Footprint
