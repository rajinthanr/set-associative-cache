# Set-Associative Cache

A 4-way set-associative, write-back data cache in Verilog, with random (LFSR) replacement and a
non-blocking miss path built on four miss status holding registers (MSHRs). It includes a
testbench with 10 test scenarios, a Quartus project targeting the Cyclone IV E FPGA
on a Terasic DE0-Nano, and a LaTeX report.

## Design parameters

All values are `localparam`s in `verilog/cache.v`.

| Parameter | Value |
|---|---|
| CPU address | 20-bit byte address (1 MB main memory) |
| CPU data | 32-bit; byte, half-word and word accesses (`cpu_req_size`) |
| Cache size | 1 KB of data (8 sets x 4 ways x 32 B) |
| Line size | 32 bytes (256 bits), 8 words |
| Sets / ways | 8 sets (`NUM_SETS`), 4 ways (`ASSOC`), 32 lines |
| Address split | tag `[19:8]` (12 bits), set `[7:5]`, word `[4:2]`; `[1:0]` unused |
| Memory interface | 15-bit line address, 256-bit line read and write |
| MSHRs | 4 (`NUM_MSHR`) |
| Replacement | random: low 2 bits of a 16-bit Fibonacci LFSR (taps 16, 14, 13, 11; seed `0xACE1`) |
| Write policy | write-back with per-line dirty bits; write-allocate on a write miss |

The brief in the report asked for 64 KB; the RTL uses 1 KB to keep synthesis fast. 64 KB would
need `NUM_SETS = 512`, `SET_BITS = 9`, `TAG_BITS = 6`, `NUM_LINES = 2048`, and a wider
`mshr_victim` (6 bits, enough for 64 lines).

## Operation

- **Hit.** All four tags in the set are compared in parallel. A hit is answered from the same
  clock edge that samples the request (`cpu_resp_valid`, `cpu_resp_hit = 1`). A write hit merges
  the byte, half-word or word into the line and sets its dirty bit.
- **Miss.** A free MSHR records the block address, set, word offset, read/write, size, write
  data and the victim way. If the victim is valid and dirty, its line and address are copied into
  the MSHR and a write-back is issued before the fetch. When the line arrives it is installed;
  for a write miss the store data is merged first and the line is marked dirty. The response
  carries `cpu_resp_hit = 0`.
- **Non-blocking.** `cpu_req_ready` stays high while at least one MSHR is free, so hits are
  serviced while misses are outstanding (hit-under-miss) and further misses are accepted
  (miss-under-miss). When all four MSHRs are in use the cache drops `cpu_req_ready` until one
  frees. A miss to a block that already has an MSHR is not merged: the request is accepted and
  discarded, and the CPU has to issue it again.
- **Replacement.** The LFSR steps every clock; the victim is `set * 4 + lfsr[1:0]`.

## Known limitations

Found by reading the RTL and confirmed in simulation; none is fixed in this repository.

- The write-back address is built as `{tag, set, 2'b00}` and truncated to 15 bits, so dirty
  lines are written to the wrong memory line (the correct value is `{tag, set}`). In the testbench,
  the line for `0x00200` (block `0x010`) is written back to block `0x040`.
- The memory interface has no ready signal or transaction ID. An MSHR issues as soon as the
  previous request cycle ends, and a memory response completes every MSHR that has issued.
  With both memory models in this repository, which serve one request at a time, outstanding
  misses therefore receive the same line: after test 10 the lines for `0xD000` and `0xE000` hold
  the data of `0x9000`. Miss-under-miss is only correct with one miss in flight.
- `cpu_req_ready` is registered, so it falls only after a request has found every MSHR busy;
  that request is dropped without a response.
- Responses carry no request ID, so a hit under a miss returns ahead of the older miss.
- Byte and half-word writes ignore address bits `[1:0]` and always land in the low bytes of the
  addressed word. Reads always return the whole word.
- The victim is chosen at random even when an invalid way exists, and two misses to the same
  set can pick the same way.

## Verification

`verilog/tb_cache.v` (top `tb_cache`) drives the cache through a CPU-access task and a
behavioural 1 MB memory model with a 50-cycle latency. Its ten tests cover: hit and miss latency;
byte, half-word and word writes; conflict misses (five blocks in one set); dirty eviction and
write-back; a sequential walk; write-then-read; a thrashing pattern; MSHR read/write misses and
dirty evictions; hit-under-miss; and miss-under-miss with four MSHRs. It prints results rather
than asserting them (only tests 6 and 8 compare read data; the closing banner is fixed text)
and dumps `cache_sim.vcd`.

Run with Icarus Verilog from the repository root:

```sh
iverilog -g2005 -o tb_cache.vvp -s tb_cache verilog/cache.v verilog/tb_cache.v
vvp tb_cache.vvp
gtkwave cache_sim.vcd    # optional
```

Results from Icarus Verilog 12.0 (the latency counter includes the testbench's own handshake
cycles; misses without write-back are 50 memory cycles plus overhead):

| Measurement | Result |
|---|---|
| Hit latency (testbench counter) | 3 cycles |
| Clean miss / miss with dirty write-back | 57 / 111 cycles |
| Test 5, 32 sequential words | 28 hits, 4 misses (87 %) |
| Test 7, 5 blocks thrashing one set, 20 reads | 7 hits, 13 misses (35 %; the report records 40 %, the LFSR makes it timing-dependent) |
| Tests 6 and 8, write then read back | pass |
| Test 9, three hits during an outstanding miss | serviced while the miss was pending |
| Test 10, four misses then a fifth | misses 1-3 took MSHRs (the test 9 miss held the fourth); miss 4 was presented with `cpu_req_ready` high and dropped; miss 5 blocked |

For Questa, `Cache.qsf` sets `tb_cache` as the NativeLink test bench: in Quartus use *Tools > Run
Simulation Tool > RTL Simulation*. This regenerates `simulation/questa/Cache_run_msim_rtl_verilog.do`,
whose committed copy holds absolute paths from the author's machine.

## FPGA build

`Cache.qpf` / `Cache.qsf` target the DE0-Nano's EP4CE22F17C6 (Cyclone IV E) with
`verilog/top_synth.v` as the top level. Open `Cache.qpf` in Quartus Prime Lite and compile. The
`.qsf` has no pin assignments yet: assign `clk_50`, `rst_n`, `sw[3:0]` and `led[7:0]` from the
DE0-Nano user manual before programming a board.

`top_synth` issues one request every 256 cycles of the 50 MHz clock, with a 20-cycle memory stub.

| Input / output | Function |
|---|---|
| `sw[0]` | 0 = reads, 1 = writes |
| `sw[2:1]` | address pattern: sequential, strided, XOR `0x55`, or +256 stride (conflicts in one set) |
| `sw[3]` | `led[7:4]` shows the hit count (0) or the miss count (1), low 4 bits |
| `led[3:0]` | `mem_req_valid`, `cpu_req_ready`, `cpu_resp_valid`, `cpu_resp_hit` |

Apart from `cpu_req_ready`, these are single-cycle pulses, and the counters change far faster
than the eye can follow, so on the board the LEDs show activity rather than readable values.

## Repository layout

```
Cache.qpf, Cache.qsf        Quartus project (device, top level, NativeLink test bench)
verilog/
  cache.v                   cache RTL
  tb_cache.v                testbench (10 tests)
  top_synth.v               DE0-Nano synthesis top with address generator and memory stub
simulation/questa/          Quartus NativeLink Questa script (regenerated by Quartus)
report/
  cache_report.tex          LaTeX source of the design report
  cache_report.pdf          compiled report
LICENSE
```

## Report

[`report/cache_report.pdf`](report/cache_report.pdf) covers the address breakdown, interfaces,
MSHR fields and state machine, the request decision tree, simulation results and scaling. It
describes the intended behaviour; the limitations above are not in it.

## Contributors

Rajinthan Rameshkumar ([@rajinthanr](https://github.com/rajinthanr)): RTL, testbench, Quartus
project, report. [@PravinduG](https://github.com/PravinduG): `cache.v` comments and the
MSHR-conflict handling (commit `6c76a06`).

## Licence

MIT. See [LICENSE](LICENSE).
