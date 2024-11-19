# Benchmarking kernel CPU vulnerabilities mitigations and HyperThreading in a Machine Learning CPU-bound workload.

date: 2024-11-19

Three identical metal servers at a major host provider were tested for Machine Learning (ML) workloads that involve 100% CPU utilization. The servers were live systems with similar histories and reliably identical hardware and software. The workload involved running ML code based on a forked Quake 3 game engine specialized in such workloads (ioq3sim). Ioq3sim is designed so it is single-threaded and fully utilizes a single core. It is usually run in parallel so that each core can accommodate one instance and the CPU is utilized at 100% capacity.

Servers sported Xeon E-2386G CPUs and were bare Debian Bookworm (12.8) with kernel 6.1.0-27-amd64 and the latest packages updates through APT. The only active services were SSH and Prometheus node code. The servers were rebooted between each round of benchmarks and not interacted with for the entire duration of each round.

The workload was actual ML experiments, albeit largely shortened, so they're representative of a normal workload but not of the full process. The same seed was used in every experiment to control variance and the ML results were confirmed to be stricly identical.

Three configurations were benchmarked: baseline (base), CPU vulnerabilities mitigations off (VULN), and HyperThreading off (HT). Each server had its configuration verified prior to starting a round, by enumerating the available CPUs and their functionality through `lscpu` and `cat /proc/cpuinfo`.

CPU vulnerabilities mitigations were disabled by loading the kernel with the `mitigations=off` parameter. HyperThreading was disabled through `/sys/devices/system/cpu/smt/control`.

The benchmarks were run in a 'stripped' fashion - i.e. during a round, no two servers ran the same configuration.

|  | A | B | C |
| - | - | - | - |
| round 0 | base | VULN | HT |
| round 1 | VULN | HT | base |
| round 2 | HT | base | VULN |

Table: execution schedule

Each server ran 51 benchmarks in a row for each configuration. Each benchmark consisted of 6 runs executed in parallel - 1 per core - and each of these runs stopped at the 1000th epoch. This resulted in about 12h of wall time per round. The CPU systematically waited for the 6 runs to complete before starting a new set of 6 runs, resulting in downtime as some cores would typically finish early. This results in a higher mean framerate at the tail end of the runs, as resources were being freed and made available to the remaining busy cores.

A typical framerate for this workload is within the 300k fps range. Each frame is 8 ms of game time, resulting in a base rate of over 40 minutes of game time per second.

The following kernel mitigations were reported by `lscpu`:

```
Vulnerabilities:
  Gather data sampling:   Mitigation; Microcode
  Itlb multihit:          Not affected
  L1tf:                   Not affected
  Mds:                    Not affected
  Meltdown:               Not affected
  Mmio stale data:        Mitigation; Clear CPU buffers; SMT disabled
  Reg file data sampling: Not affected
  Retbleed:               Mitigation; Enhanced IBRS
  Spec rstack overflow:   Not affected
  Spec store bypass:      Mitigation; Speculative Store Bypass disabled via prctl
  Spectre v1:             Mitigation; usercopy/swapgs barriers and __user pointer sanitization
  Spectre v2:             Mitigation; Enhanced / Automatic IBRS; IBPB conditional; RSB filling; PBRSB-eIBRS SW sequence; BHI SW loop, KVM SW loop
  Srbds:                  Not affected
  Tsx async abort:        Not affected
```

## Results

No significant differences were found between these configurations. Framerates were found to be highly consistent. Interestingly, framerates significantly differed from one server to the next. They also showed a multimodal distribution within each server, with 3 modes, presumably corresponding to different CPU states.

| | base | VULN | HT |
| - | - | - | - |
| A | 330794 | 330549 | 330497 |
| B | 326085 | 326474 | 326391 |
| C | 321557 | 321702 | 321704 |

Table: mean FPS per server, per configuration, over epochs 200 to 800.

The head and tail ends of the runs were discounted in order to remove inconsistencies induced by tail concurrency (as some cores finish early) and possible interference from Turbo Boost upon restarting benchmarks, though framerates over these epochs were found to be consistent as well nonetheless.

It was observed that framerate varies with epochs, presumably because the agents' behavior and positions affect collision detection within the game world, which accounts for the large majority of the consumed CPU time according to code profiling through Intel VTune. Because each run used the exact same seed, these variations in framerate were highly consistent across benchmarks.

Interestingly server C with HyperThreading off deviated from the other results at key moments, particularly the tail end epochs when framerate bumps up due to some cores finishing early. This was only seen with server C and it is unknown what caused this.

Overall, manipulating kernel mitigations of CPU vulnerabilities and HyperThreading might not be worth pursuing on modern hardware, for this type of CPU-heavy workloads.


 ![Server A: mean FPS over epochs](/241119_hyperthreading_cpu_mitigation_benchmark/fps_A.png)

 ![Server B: mean FPS over epochs](/241119_hyperthreading_cpu_mitigation_benchmark/fps_B.png)

 ![Server C: mean FPS over epochs](/241119_hyperthreading_cpu_mitigation_benchmark/fps_C.png)

 ![Server A: framerate distribution @epoch 600](/241119_hyperthreading_cpu_mitigation_benchmark/kde_600_A.png)

 ![Server B: framerate distribution @epoch 600](/241119_hyperthreading_cpu_mitigation_benchmark/kde_600_B.png)

 ![Server C: framerate distribution @epoch 600](/241119_hyperthreading_cpu_mitigation_benchmark/kde_600_C.png)

