# xtx fork status (2026-11-19)

Our fork of Niko1221/Strata for 2x RX 7900 XTX (gfx1100), Ryzen 5 5600 (Zen 3, 6C/12T, AVX2, no AVX-512; swapped in for the 3600 on 2026-11-20),
64 GB DDR4-3200 (~54 GiB available), system ROCm 7.2.4, kernel 6.8. Base decision (user, 11-19):
upstream v0.2.16 + selectively ported nicejunjie/Strata-rdna3 patches (not a rebase of the rdna3 fork).

## Layout
- Repo: ~/workspace/dev/strata-fork (origin = kimtk44/Strata, upstream = Niko1221/Strata, nj = nicejunjie/Strata-rdna3, blobless clone: avoid `git log -S`)
- Branches: `xtx` = v0.2.16 (64524fb9) · `xtx-f04-pcie-off` = F04 + hardening (23cb6c7d, 9f6b0b6b) · `xtx-f02-rdna-decode` = F02/F03 ported on top of F04 (4e8b013d, 0bd340a3; built, not yet run; worktree ~/workspace/dev/strata-fork-f02)
- Build: `cmake -S engine -B build-hip -G Ninja -DCMAKE_BUILD_TYPE=Release -DSTRATA_ENABLE_HIP=ON -DSTRATA_ENABLE_CUDA=OFF -DCMAKE_HIP_ARCHITECTURES=gfx1100 -DCMAKE_PREFIX_PATH=/opt/rocm -DCMAKE_HIP_COMPILER=/opt/rocm/llvm/bin/clang++` (needs the prefix/compiler flags; engine reports 0.1.62)
- Setup config isolation: `XDG_CONFIG_HOME=$PWD/.xdg` (keeps the old v0.1.41 install in ~/workspace/dev/playground/strata untouched). `./setup.sh --check --backend hip`: IQ3_S needs ~84 GB RAM+VRAM -> fits.
- Model: ISTA-DASLab GSQ-RCO IQ3_S -> ~/llm/models/strata-data/models/IQ3_S/ (2 shards, 51.05 + 26.82 GiB; sha256 to check against HF lfs oid). Pack to build fresh with v0.2.16 `terminal/tools/iq_pack.py --experts-bin` (v0.2.16 rejects pre-v2 packs).
- Patch inventory: patch-inventory-2026-11-19.md (25 families, step-0 config, port order).

## F04 (done, built, reviewed)
`--pcie-frac 0` skips the PCIe share path in every verify window incl. layer-split stages and pipelined odd windows
(fork: IQ3 dual 123.0 -> 136.2, IQ4 114.0 -> 124.1). Guards: no auto share, no peer device, no ADAPT_FETCH,
set_pcie_balance forces the path on, request pcie_frac held at 0 (one-time warning), and GpuPlanSink::pcie_off makes
the pool itself plan no PCIe groups. A/B in one binary: `STRATA_PCIE_OFF=0`. Independent Fable review: sound;
hardening applied. Not hooked: the one-shot `--spec` program path (keeps the full graph).

## Step-0 config notes (from inventory)
- Use hipBLASLt table gfx1100-hipblaslt-100202.txt (system ROCm 7.2.4), not the fork's 100200.
- STRATA_PF_F16=1 (value 2 is read as off upstream). `--tensor-parallel` does not exist upstream.
- Retune for 64 GB: --pool-workers 6, PREFILL_CPU_SHARE unset or auto, --conversation-cache-mib 0-2048. Chosen on the Zen 2 3600;
  the 5600 has the same core count, so 6 stays, but CPU-side numbers (PREFILL_CPU_SHARE, pool timing) were not re-measured on Zen 3.
- Korean user's "main optimized" four: hipBLASLt table + --pcie-frac 0 + STRATA_GDN_CHUNKED=1 + --adapt-every 1 (decode 64 -> 71, read 952 -> 1,790 on one 7900 XTX).

## Two-card findings (web research 11-19, verified items marked)
- Our box (verified): ReBAR on (BAR0 32G both), KFD p2p_links present between the two GPUs (type 2, weight 40), PSU 1200 W Platinum (dual full load ~720 W measured earlier).
- v0.2.14 trims each stage to its own dense weights; v0.2.16 merged pipelined two-card decode (#5860/#5862/#5864).
- `--peer-device` can fault on the first request (~4% of fresh starts, release notes) -> not used.
- Closest hardware report: Strata #6202 (5700X AM4, 64 GB DDR4, x8+x8, 2x 7800 XT): IQ3_S full RAM+VRAM residency, prompt 941 (4K) -> 2,494 (128K), decode 53-56. If RAM runs short: HSA_USERPTR_FOR_PAGED_MEM=0, STRATA_RESIDENT_PIN=0, GPU_PINNED_MIN_XFER_SIZE=1048576 (else 5-37 s pauses).
- Strata #4136 (2x 7900 XTX, one on chipset x4): IQ3_S 262K, layer-split 26, pipeline-windows 2, trim-stage-weights, spec-min-p 0.7 -> decode 66-71, prefill 1,781-2,168. --batch / PARALLEL=2 disables pipelined decode and evicts ~40% of the cache.
- Avalon two AMD cards: #55 decode 84-97 on 2x 7900 XTX x8/x8 (kernel 6.14); #20 two cards slower than one in layer-device mode; #26 host-sync limit open.
- rdna3 fork TP issues on our stack: #14 STRATA_REFILL_SDMA must be 0 on kernel 6.8 (else 30 s stalls; with it off decode 133-147), #16 raise STRATA_WATCHDOG_S to ~180 for huge prompts, #17 cards stuck at 100 W after first big request, #18 megakernel stall. Port TP after the layer-split baseline, applying these fixes as defaults.
- DP prefill (upstream PR #5002 = fork F10) needs every expert in RAM -> likely not on 64 GB with IQ3_S.

## Community findings 11-23 (r/ROCm 1y0n9zg, "PART 2: 2x RX 7900 XTX + Qwen3.8-Flash-Next", u/fintip, 2x XTX x8/x8, 5800X, 64 GB)
Not verified on our box; one tester, paired coding-bug bench with hidden tests (~8,000 scored runs).
- **rdna3 fork NaN (#879) = the fork's refill settings**, not the ROCm runtime (same NaN rate on ROCm 7.2.4 and 10.1). With its 10 refill
  settings removed: 0 NaN in ~300 agent runs. Same machinery as fork issues #3/#14. Exact key(s) and a fork PR pending.
  -> When porting TP (F12+), bring the refill settings in default-off and enable one at a time; this extends the #14 REFILL_SDMA=0 note above.
- **rdna3 fork is a few points weaker on the hardest problems**: upstream 193 vs fork 183 of 222 paired, disagreements re-run fresh
  22 vs 16, ~2% chance of luck. Still ~16% less wall time per run. With all fork tuning switches off (TP kept), decode fell 23% to about
  upstream speed, so the speed (and likely the quality cost) sits in the tuning switches, not TP. Which switch is still being bisected.
  15 fork prompt-kernel switches on upstream 0.2.13.1: bench prompt reads about 2x, no agent wall-time gain, 18 vs 21 solved (noise).
- Upstream 0.2.16 (our base): prompt read +11-32%, decode about flat, no quality cost seen (22/24, same as 0.2.13.1). TAIL_SKIP is off on
  gfx1100 (on by default for CUDA and RDNA4 builds); ROUTE_PRIOR is opt-in.
- **0.2.19 trap on 64 GB**: its RDNA3 defaults raise prompt chunks to 32K only if the cache loan + 6 GiB RAM stay free (RAM guard #2090).
  With Flash-Next IQ3_XXS on 64 GB that never holds, so it silently stays at 8K: log shows "gfx11: prompt chunks stay at 8192".
  `STRATA_GFX11_CHUNK=1` skips the guard (1 XTX: 1,511 -> 1,851 t/s at 16K, 1,430 -> 2,550 at 58K; free RAM stayed >= 3.95 GiB).
  Upstream issue Niko1221/Strata#6687. Check this on any upgrade past 0.2.16.
- Quant plateau: IQ3_XXS, IQ3_S, UD-IQ4_XS, UD-Q4_K_XL and EXL3 4.05 all within noise on agentic coding; only IQ1_M clearly loses.
- EXL3 fork: `--conversation-cache-mib` defaults to 0, so alternating sub-agent conversations re-read the full prompt (~30 s vs ~2 s with 16 GB).

## Next
1. DONE 11-19: IQ3_S verified, packed, first 5 arms measured -> xtx-notes/bench-2026-11-19-iq3s.md
2. gpu-lock both cards (user OK'd now), empty llama-swap, measure: 1 card upstream default / 1 card step-0 / 2-card layer split F04 off vs on.
3. F02/F03 A/B (all env switches default off; build-time `-DSTRATA_DPP_SHFL=ON` separate dir). Order: all off = token-match baseline -> `STRATA_RDNA_Q6K=1` (+PAIR/IQ3S/KQ_ALL) -> `STRATA_GR_RDNA=1` (+`_Q8` look for "BF16 copies skipped", +`_A8`) -> `STRATA_GR_INJ_UP=1` -> `STRATA_GR_FUSED=1` on 1 card, no `--pipeline-windows 2` (grid barrier has no timeout, needs 168 blocks resident; fork saw a layer-26 hang) -> `STRATA_ROUTE_PUBLISH=1`, `STRATA_COMBINE_CPU=1` (+`STRATA_WAIT_FUSE=1`, `STRATA_MMVF_ROWS=1`), compare vs upstream `STRATA_FUSE_WAIT`/`FUSE_HITS`. Caveats: RDNA kernels change summation order (not bitwise), r8 hc copies not in the VRAM estimate and pure extra on layer split. Then F05/F09, F10 check, TP chain (F12 -> F13/F15/F16) with the fixes above.

## Production (2026-11-20)
- llama-swap entry `L-strata-flash-next-iq3s` (user applied patch + restart ~00:20 KST; backup llama-swap.yaml.bak-20261120-pre-strata-iq3s; vault snapshot 229007b3).
- Files ~/llm/strata-xtx/: strata-xtx-combined-be4a6e0c (branch xtx-combined), strata-iq3s.json (c2 + RDNA_Q6K/PAIR/GR_RDNA + spec-min-p 0.7, split 23), run-strata.sh (TERM stops server + engine; rehearsed: 1 s clean exit, VRAM freed).
- Smoke via :8080: cold load 67 s, decode 76 tok/s, warm instant, /unload frees both cards. Not in any matrix set (evicts / is evicted by every other model).
- **Quality caveat (11-23)**: production turns on RDNA_Q6K/PAIR/GR_RDNA, i.e. the fork tuning-switch family the community bench above
  ties to a 2-5 point loss on hard problems. Our only quality evidence is fork-bench-1119 longctx QA (6 questions, every arm 12/12),
  which is at ceiling and cannot see a gap of that size. Before relying on it for hard coding work, run a paired check on harder tasks:
  same prompts, production config vs the same binary with all RDNA switches off.
- Server code runs from ~/workspace/dev/strata-fork/terminal: do not switch that checkout's branch casually.
