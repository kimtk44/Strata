# Strata patch inventory: nicejunjie/Strata-rdna3 (nj/main 8b01b0c8) vs upstream v0.2.16

Method: merge-base = 8183c6b6 (upstream 0.2.4.1, 2026-10-24), but upstream rewrote history, so `v0.2.16..nj/main` lists 929
non-merge commits of which 758 are upstream commits re-reachable only by hash. nj-own commits = subject not present in v0.2.16
history = **168 commits** (165 Junjie Li, 2 ivanjiang5628, 1 Matthew Lenz), grouped below into 25 families (F01..F25).
Path map: old `src/` -> `engine/src/`, `include/strata/` -> `engine/include/strata/`, `tests/` -> `engine/tests/`,
`serve/` -> `terminal/serve/`, `setup.py`/`tools/` -> `terminal/`, root CMakeLists -> `engine/CMakeLists.txt` + `engine/cmake/`.
"Conflicts" = files conflicting when each commit is cherry-picked ALONE on v0.2.16 with rename detection (a lower bound of
effort: many conflicts are only missing predecessor commits of the same family). Hot-file churn (v0.2.16, last 30 days, both paths):
generate.cpp 461 commits (24,341 lines), prefill.cpp 214 (9,080), expert_source.cpp 54, verify.cpp 81 (5,613), moe_fused_iq.cu 94,
prefill/kernels.cu 33, mtp.cpp 20, arch_defaults.cpp 38. Fork-only new files (tp.cpp, rdna_gr.cu, rdna_mmvq.cu, sdma_copy.cpp,
kfdq.cpp, megakernel_gdn.cu, arena_shm.cpp, host_lock.hpp) do not exist upstream: copying them is conflict-free, only the hooks into hot files hurt.

Our box (read-only checks): 2x Navi31 at 10:00.0 / 13:00.0; sysfs: 10:00.0 negotiated 16 GT/s x8 (bridge 0e:00.0 x8, root port 00:03.1 max x8);
13:00.0 reads 2.5 GT/s x1 at idle (root port 00:03.2 max x8) = almost certainly ASPM idle downshift (inferred; re-read under load).
So both cards are Gen4 x8 capable = same link as the fork's reference box (inferred). `lspci -vv` printed no LnkSta without root; sysfs used.
Ryzen 5 3600 (Zen2, AVX2, no AVX-512), 62 GiB total / 53 avail, system ROCm 7.2.4 (hipcc 7.2.53211, clang 22 roc-7.2.4, hipBLASLt .so 1.2.70204,
libhsa-runtime64 present, rocWMMA 2.2 headers present).

Key finding: **v0.2.16 already contains most of the fork's *prompt-side* kernels as opt-in switches** (HIP_WMMA, SELECT_WMMA, PF_GEMM/PF_PAD,
PA_FAST, HC_UPMIX, PF_HCDOWN, GDN_CHUNKED (gfx11 FP16 WMMA, 16-token chunks), PF_F16=1 gfx11 expert kernels, PF_FUSED/_KQ, QSA_ROWS,
KV_HOST_DMA, ADAPT_OVERLAP, hipBLASLt tuning tables incl. gfx1100-100202 "for system ROCm 7.2.4"). Defaults for gfx1100 are OFF (arch_defaults.cpp only
has gfx12/gfx1151/iGPU tables). What is absent: the whole decode-side RDNA path, TP decode, DP/TP prefill, refill/SDMA, megakernel, kfdq.

## Family table

Gain column quotes fork docs (README.md RDNA3 section, docs/RDNA3_SETTINGS.md) or commit bodies; "unmeasured" = no number for UD-IQ4_XS.
Difficulty: S <300 lines/no hot file, M 300-900 or one hot file, L >900 or several hot files, XL structural.

| ID | Family (commits) | Key commits | Code lines (+/-) | Env / flags introduced | In v0.2.16? | Config-only? | Measured effect (quoted) | Port difficulty | Our-box relevance |
|---|---|---|---|---|---|---|---|---|---|
| F01 | xyzzing WMMA attn+QSA select import, aibox HIP bits (K-quant MMQ HIP, MMVQ wave, peer transport) (4) | b77dfff2 b72eaccf c5c6ac36 3b0c8553 | +1258/-62 | HIP_WMMA, SELECT_WMMA, STRATA_ROCWMMA_INCLUDE_DIR, MMQ_KQUANTS(cmake), PEER_TRANSPORT, RDNA_MMVQ_WAVE | **Partly**: HIP_WMMA + SELECT_WMMA exist, gfx11 builtin-based (no rocWMMA). K-quant MMQ: STRATA_MMQ_KQ exists, cmake option MMQ_KQUANTS absent. peer transport/MMVQ_WAVE absent | Yes for WMMA switches | README: Q4 2-GPU 32K ~4,170 -> ~4,770 with MMQ_KQUANTS; "IQ3/IQ4 are unaffected" | S (skip: not needed for IQ4) | rocWMMA NOT needed if upstream builtin version used. Skip peer transport (host copy default) |
| F02 | gfx1100 decode RDNA GEMV / hyper-conn read / DPP / pcie-off (10) | 3557322d e25a7b45 898f0edc f7fc223b 15cbf0c5 404247f8 9ea97232 | +1600/-100; new rdna_gr.cu, rdna_mmvq.cu | GR_RDNA, GR_RDNA_Q8, GR_RDNA_A8, RDNA_Q6K, RDNA_PAIR, RDNA_IQ3S, DPP_SHFL, POOL_CORES, PCIE_OFF | Absent (POOL_CORES, EXP_MODE exist) | No | f7fc223b: "Q6_K heads and dense projections 10-30% faster at T 2-6" (kernel level); no e2e number for IQ4 | L (20 conflict files; verify.cpp x4, mtp.cpp, fused_gr.cu) | Relevant: UD-IQ4_XS has Q6_K heads + Q8_0 hc; decode is GPU+CPU mixed so e2e smaller (inferred). DPP shuffles are RDNA3-specific, ROCm-agnostic |
| F03 | Decode launch fusion (9) | 9ae4a687 b4a32631 98ab70c2 88ab4687 84b27e89 b165b84d 413c41df | +572/-98 | GR_FUSED, GR_INJ_UP, ROUTE_PUBLISH, WAIT_FUSE, COMBINE_CPU, QFUSE(exists), VERIFY_QDEDUP(exists) | Absent except QFUSE/QDEDUP/MMVF | Partly (QFUSE/QDEDUP now) | 88ab4687 "IQ3 single 32K: 23.04 -> 22.43 ms/window" (QFUSE+COMBINE_CPU+QDEDUP); 98ab70c2 "21.85 -> 21.60"; b4a32631 "21.60 -> 21.51"; 84b27e89 "20.69 -> 20.54" => ~ -1..-3% each | M-L; depends on F02 (rdna_gr.cu) | Small stacked gains, all GPU-side |
| F04 | `--pcie-frac 0` on layer split (1) | deafb0d3 | +6/-1 generate.cpp | (uses --pcie-frac 0, STRATA_PCIE_OFF) | Flag exists; the verify-window skip (set_pcie_off) is absent | 1-card: flag works as config. 2-card needs patch | "IQ4 114.0 -> 124.1" dual suite (+8.9%), IQ3 123.0 -> 136.2 | **S** (6 lines + F02's 3557322d verify.cpp skip) but generate.cpp is hottest file | HIGH for 2-card layer split, cheapest win |
| F05 | Long-context decode top-k + QSA (6) | 6ca3071e c4b18079 c3b16b11 d6f659d4 cfccd843 f7d7e996 | +566/-33 | TOPK_SPLIT_MIN, TOPK_SPLIT_EAGER, PA_KLDS, SCORES_LANE, QSA_ROWS(exists) | Absent (QSA_ROWS, SCORES_THREAD exist) | No | c4b18079 "Q4 1-GPU 256K 14.64 -> 13.46 ms GPU/window"; README "256K decode 5-9% faster per window"; c3b16b11 "qsa attn 1102 -> 993 ms at 32K" | M (verify.cpp/qsa_select.cu) | Only >=98K contexts. Medium-low |
| F06 | GDN chunked prefill on FP16 matrix cores (3) | 46a4f61d 8677a924 fcfb7cd4 | +848/-26 | GDN_CHUNKED, GDN_CHUNK_GDV=0, GDN_WMMA_SPLIT, GDN_CHUNK_SKIP | **Yes** (gfx1100-gfx1151, 16-token WMMA chunks, FP16/FP32 acc, fused norm). GDV/own-scratch knobs absent | **Yes**: STRATA_GDN_CHUNKED=1 | 8677a924 "Recurrence 893 -> 378 ms per 32K single-GPU IQ3 prompt (pp +3%)"; upstream measured only gfx1151 (+1.5-2%) | S if skipped; M to port 8677a924 | Step 0. Upstream kernel numerically same family (inferred) |
| F07 | FP16 pair expert kernels, PF_GROUP_RES (5) | 609c1737 3f2393df 1f4b77d7 1799c0a3 b8591de1 | +904/-50 | PF_F16=2 (128-token B tile via LDS), NH_COLSKIP, PF_F16_OCC*, PF_GROUP_RES, PF_F16_CHECK | **Partly**: PF_F16=1 kernels exist (moe_f16_common.cuh/moe_f16_kq.cu); **value 2 is NOT understood upstream (requested() tests v[0]=='1')**; NH_*/OCC/GROUP_RES absent | PF_F16=1 yes; rest no | 609c1737 "IQ3 single 32K expert GEMM 3.02 -> 2.63 s"; 1f4b77d7 "2-GPU prefill +0.5..+2%, 1-GPU unchanged (copy-bound)"; b8591de1 "short prompts +2-3%" | M-L (moe_fused_iq.cu hot, 94 commits/30d) | Medium-low after PF_F16=1 |
| F08 | K-quant MMQ default-on (1) | fea8652e | +4/-1 | STRATA_MMQ_KQ | Exists (gfx12-gated default) | Yes (=1) | "UD-Q4_K_XL 1K 476 -> 685 tok/s" - Q4 only | S | Not for IQ4_XS |
| F09 | Prompt HC kernels (2) | 3dbb4051 84248bcc | +98/-8 | HC_UPMIX_NT, HC_WN4 | HC_UPMIX exists; the 512-thread upmix/vectorised write absent | HC_UPMIX=1 yes | unmeasured (register-pressure fix) | S (kernels.cu) | Low |
| F10 | Dual-GPU data-parallel prompt path (14) | f60e317b 7ee3bc4f 90ea76cd 49a5eebb a5cb144b 0590d01c 866c1544 791cca4b | +1093/-88 | DP_PREFILL, DP_MIN, DP_PIPE_CHUNK, PIPE_HALF_*, DP_EARLY_REFILL, PREFILL_AUTO_MAX(exists) | Absent | No | f60e317b "IQ4 dual 32K +25%, 128K +17%"; 7ee3bc4f "8K +9%, 16K +14%" | L (generate.cpp x7, prefill.cpp x5 conflicts) | 2-card only; HIGH value, needs PCIe peer/host copies |
| F11 | Layer-split stage help on fused int8 kernels (3) | 66eb39d2 acbb0706 02602e15 | +370/-26 | PREFILL_HELP(exists), HELP_FRAC(exists), BATCH_PIECE, SPEC_CUM | Switch exists; fused-kernel help support unverified | STRATA_PREFILL_HELP=1 try | 66eb39d2 "IQ4 4K 1627 -> 2255, 8448 2156 -> 2674" | M | 2-card; verify first whether upstream help covers PF_FUSED (open Q) |
| F12 | Tensor-parallel decode + VRAM aliasing (16) | 6743454e 6820bac2 21910f80 a02c365a d21bdf12 b0ddf9ed b5d1388f d4ee9e6e | +2615/-344; new tp.cpp | --tensor-parallel 2 (engine REJECTS: "unknown argument"), TP_*, SHARE_DP, ALIAS_MAP | Absent | No | README: UD-IQ4_XS 2 GPUs decode 127.8 (EN) / 167.1 (code) at 32K; commits: IQ4 TP suite 135.1 vs layer-split 124.1 (+9%, inferred cross-date); 6820bac2 "18.4 ms/window (110 tok/s)" first M2 | XL (verify.cpp x11, generate.cpp x10, tp.cpp) | 2-card decode; largest decode win but largest cost |
| F13 | Megakernel GDN dense chain (6) | 4d1ab01c ecc8a3ae f59a2c2c a90e1fb4 | +1002/-155 | MEGAKERNEL | Absent | No | a90e1fb4 "UD-IQ4_XS 2-GPU +23..27% ms/window" (vs without megakernel, TP only); 16.72 vs 16.88 ms (ecc8a3ae) | L; **requires F12** (TP verifiers only) | After F12 |
| F14 | MTP drafter opt-ins (6) | fc8f8233 c80244aa c9bb9d4e b1c9c9f2 877bf971 | +406/-64 | MTP_QFUSE, MTP_MULTICOPY, MTP_GR_R8, MTP_DRAFT_ALL; (HIP_PRIO, STAGE0, TOP2, PROFILE exist) | Partly | HIP_PRIO/STAGE0 yes | unmeasured individually | M; GR_R8 needs F02 | Low |
| F15 | Late/early refill, SDMA via HSA, TP hand-over (10) | 6d1baae1 2cc62cb8 29ab4948 5077ba88 d239573b 551d01e6 | +867/-72; new sdma_copy.cpp | REFILL_* (~20), HSA_COPY, TP_PRECOPY, TP_BACK_SKIP | Absent | No | 6d1baae1 "TP 2-GPU IQ3 128K wall@256 22.3 -> 21.85 s" | L; requires F10/F12; needs libhsa-runtime64 (present) | After F10/F12 |
| F16 | TP prompt path + routed walk (7) | 0f8e76ef 6987d7bb fa5c621d af0424fb 9ef1ab50 | +2059/-240 | TP_PREFILL(_MIN/_MAX/_CHUNK), FUSED_ROUTED, PF_NO_GATED | Absent | No | 6987d7bb "IQ4 8K 3375 -> 4440, 12K 3247 -> 5113"; README "UD-IQ4_XS +26% / +17.5% at 12K/16K vs rg24" | XL; requires F10+F12+F15 | Late |
| F17 | Adaptive tier tweaks (3) | fc7293d1 98031db9 3e709a06 | +149/-7 | ADAPT_MIN_USE, ADAPT_SDMA(not a win) | ADAPT_OVERLAP/LAG/NOWAIT exist | --adapt-every 1 is config | "not a win" (own text) | S | skip |
| F18 | Arena in shm across starts, O_DIRECT arena load (2) | af94f272 98d8c36f | +1295/-29 | ARENA_SHM | Upstream has its own unbuffered tier (#286, STRATA_UNBUFFERED_LOAD) | partial | "72 GB Q4 arena loads ~1.9 vs ~1.0 GiB/s" | M | Low (start-up only) |
| F19 | CPU kernels (4) | d83ecfcc dc6c8edd 309f0c8f 53deacec | +464/-19 | IQ3S_1T, AVX512_SAFE_INIT, CACHE_PLACEMENT | Absent | No | d83ecfcc "UD-IQ4_XS 1 GPU CPU expert time -10%, window -2.4%" - **needs AVX-512 VBMI (vpermt2b): useless on Zen2**. dc6c8edd is an AVX2-safety fix for global initialisers | S | **Skip d83ecfcc/53deacec**. Check dc6c8edd failure mode applies (upstream iq_avx512.cpp lacks those tables, so likely moot) |
| F20 | Idle-power KFD queue parking (5) | e2151f1b 6677f039 e4a46c46 | +286/-15; new kfdq.cpp (interposes ioctl) | KFDQ* | Absent | No | "idle 98 + 91 W -> 5 + 13 W" | M; ROCr/kfd_ioctl-struct sensitive | Only if GPU_MAX_HW_QUEUES=8 (TP). Not needed for layer split |
| F21 | Host-memory/mlock/stall robustness, TP/DP hand-off buffers (21) | 86de9791 1097d7fa 18f67ee3 a5b58a29 0ccf2a1c | +666/-140 | PF_HOST_FLOOR, TEST_* | Absent | No | unmeasured (stability) | M-L; mostly tied to F10/F16 | Defer; mlock part (86de9791) maybe standalone |
| F22 | Review fixes / gfx1100 tuned defaults (6) | 34683c4b ae17cf7e a6c16308 b75ef924 e1dad2dc | +271/-37 | GFX1100_DEFAULTS | Absent | n/a | presence_penalty 0: "16K code decode 177.2 vs 157.1" (model-card sampling, a request/config setting) | S-M | `presence_penalty: 0` is config-only |
| F23 | Prefill diagnostics (7) | 359a1060 839f3769 88686782 9e598b1c | +127/-14 | PF_TRACE, PREFILL_SLOW_MS | PREFILL_TIMING exists | partial | none | S | optional |
| F24 | Docs, rdna3 repro kit, README (18) | | | | n/a | n/a | n/a | drop | drop |
| F25 | Fix: one-token commit under QFUSE (1) | 3c1e08e1 | +3/-1 | | N/A on HIP: v0.2.16 keeps the commit graph unless ONE_TOKEN_COMMIT=1 | n/a | | skip | skip (do not set ONE_TOKEN_COMMIT=1) |

Reference overall: README "Against stock Strata on the same card (UD-IQ4_XS, 32K): prefill 1,151 -> 3,455 tok/s on one GPU and 1,727 -> 7,060 on two";
UD-IQ4_XS decode 1 GPU 94.2/91.9/89.0 (English 32K/128K/256K), 122.3/120.6/117.3 (code); 2 GPU 127.8/126.8/129.9 (English), 167.1/159.1/158.4 (code).
Fork box: Ryzen 9 9950X3D2, 128 GB DDR5, PCIe 4.0 x8 per card. Ours has far weaker CPU and RAM: CPU-bound parts (expert math for non-resident experts) will not reach those numbers (inferred).

## Step 0: config-only on v0.2.16 (UD-IQ4_XS)

Config-key check: `checkRunConfig` (app/host/server/run-config-check.mjs) run on both fork JSONs with @GPUS@ substituted: 0 issues (all top-level keys incl.
layer_split, engine_silence_s, lib_dirs, vision, sampling are known). It does NOT inspect `args`/`env`, so:

Rejected / ignored by v0.2.16:
- args: `--tensor-parallel 2` -> engine exits "unknown argument" (generate.cpp:4117). Remove. All other args in both JSONs are parsed (verified).
- `"exe": "@STRATA@/engine/strata"` is the fork layout; point to the v0.2.16 build output (check build dir).
- env keys with no reader in v0.2.16 (silently ignored, drop for clarity): GPU_MAX_HW_QUEUES is a HIP runtime var (still honoured), STRATA_PEER_TRANSPORT, RDNA_MMVQ_WAVE, GR_RDNA, GR_RDNA_Q8, GR_RDNA_A8, RDNA_Q6K, GR_FUSED, GR_INJ_UP, ROUTE_PUBLISH, WAIT_FUSE, COMBINE_CPU, GDN_CHUNK_GDV, PA_KLDS, PF_GROUP_RES, PREFILL_MMQ_TILE_ROWS, TOPK_SPLIT_MIN; 2-GPU only: DP_*, PIPE_HALF_MAX, FUSED_ROUTED/_CPU_FRAC/_MAXT, MEGAKERNEL, MTP_DRAFT_ALL/QFUSE/MULTICOPY/GR_R8, REFILL_*, TP_*, PREFILL_SMALL_RING/MAX.
- **Value trap**: `STRATA_PF_F16=2` (fork) -> upstream only honours `1`; "2" silently turns the F16 kernels OFF. Use 1.
- hipBLASLt table: fork path `tools/hip/gfx1100-hipblaslt-100200.txt` is for TheRock hipBLASLt 1.2.0. On system ROCm 7.2.4 use `terminal/tools/hip/gfx1100-hipblaslt-100202.txt` (docs/AMD_HIP.md: "100202 is for system ROCm 7.2.4"); a version mismatch makes the engine log "hipBLASLt version mismatch" and fall back to plain hipBLAS (safe). Alternative: `STRATA_HIPBLASLT_TABLE=on` (packaged table).
- Hardware-tuned values to change (inferred): `--pool-workers 12` -> 6 (6C/12T); `STRATA_PREFILL_CPU_SHARE=0.6/_MAX 5120` was tuned on 16C Zen5 AVX-512: start with unset/`auto` or 0 and A/B; `--conversation-cache-mib 8192 --conversation-cache-slots 8` can claim 64 GB: use 0-2048 on 53 GiB free; `STRATA_ENGINE_NICE` irrelevant (no iGPU); `--mtp-draft-vocab` zh60k file (238,428 B, `git show nj/main:rdna3/draft_vocab_zh60k.bin`) optional, "+4% English decode".
- `GPU_MAX_HW_QUEUES=8` only with TP + kfdq; on layer split leave default (or 6 on 1 card, "idle-clean" per RDNA3_SETTINGS).

1-card env (accepted by v0.2.16):
```
STRATA_HIPBLASLT_TUNING=<repo>/terminal/tools/hip/gfx1100-hipblaslt-100202.txt
STRATA_HIP_WMMA=1  STRATA_SELECT_WMMA=1  STRATA_PF_GEMM=1  STRATA_PF_PAD=1  STRATA_PA_FAST=1
STRATA_HC_UPMIX=1  STRATA_PF_HCDOWN=1  STRATA_GDN_CHUNKED=1
STRATA_GDN_HEAD=1  STRATA_GDN_PP=1  STRATA_GDN_CONVL2=1  STRATA_GDN_NOY=1  STRATA_FUSE_CONVAB=1  STRATA_HEAD_MIX_MULTI=1
STRATA_PF_FUSED=1  STRATA_PF_FUSED_KQ=1  STRATA_PF_F16=1   # A/B =0 and PF_F16 off: unmeasured on upstream gfx1100
STRATA_QSA_ROWS=1  STRATA_QFUSE=1  STRATA_VERIFY_QDEDUP=1  STRATA_KV_HOST_DMA=1  STRATA_ADAPT_OVERLAP=1
STRATA_SH_STREAM=0  STRATA_STAGER_RING=64  STRATA_PREFILL_RING=192  STRATA_PREFILL_AUTO_MAX=32768
```
args: `--pack ... --native ...UD-IQ4_XS-00001-of-00003.gguf --expert-profile terminal/data/expert-profile.bin --prefill auto --spec 4 --spec-min-p 0.5 --mtp ... --max-context 262144 --kv int8 --ple-io direct --stats --vision --expert-cache auto --ple-row-cache 1048576 --kv-resident 32768 --pcie-frac 0 --vram-reserve-mib 512 --pool-workers 6 --mtp-q4 all --mtp-window 4096 --adapt-every 1 --adapt-swaps 32`; sampling `presence_penalty: 0` optional (model card default is 1.5 upstream).
The Korean user's four switches (table, `--pcie-frac 0`, GDN_CHUNKED, `--adapt-every 1`) are a subset of this.

2-card (layer split, no TP/DP): same env, plus `layer_split: "24"`, `--pipeline-windows 2`, `--adapt-swaps 48`, `STRATA_PREFILL_HELP=1`, `STRATA_PREFILL_STREAM_MIN=480`, `STRATA_PREFILL_LEND_PCT=93`, `STRATA_SPLIT_MTP_BATCH=1`, `STRATA_CKPT_THREADS=1`, `STRATA_CKPT_POOL=1`, `STRATA_ONE_TOKEN_COMMIT=0`, `STRATA_AUX_CPUS=off`, `STRATA_ROUTE_TAIL_SKIP=0`, `STRATA_PREFILL_CPU_SHARE_MAX=1024`; leave GPU_MAX_HW_QUEUES unset; `--vram-reserve-mib 512`. `--pcie-frac 0` is accepted but F04 patch is needed for its full effect on layer split.

## Ranked port plan (gain for UD-IQ4_XS on our box / cost)

0. Step 0 above (no code). Measure prompt + decode, record baseline per switch (fork's own per-switch ablations on upstream do not exist).
1. **F04** `--pcie-frac 0` layer-split skip (+ PCIE_OFF part of F02:3557322d verify.cpp): +8.9% dual decode quoted (IQ4 114.0->124.1); 6+~30 lines; hot generate.cpp but tiny hunk. 2-card only.
2. **F05 PA_KLDS + F09** small kernel patches: low cost, small gain, only if long prompts matter.
3. **F02 + F03** decode RDNA path (rdna_gr.cu, rdna_mmvq.cu, fused launches): 1-3% per switch stacked, plus Q6_K/IQ4_XS multi-column GEMV 10-30% kernel-level; cost L (copy new files, hook verify.cpp/native_*). F03 depends on F02. 1-card and 2-card both. CPU-bound decode on Zen2 dilutes gains (inferred).
4. **F10 DP prefill** (2-card): IQ4 dual 32K +25%, 128K +17%. L, hot generate.cpp/prefill.cpp. Prefer before TP (independent of TP; F10 + F11 are the layer-split prompt side).
5. **F07** FP16 pair kernels / PF_GROUP_RES: only after confirming PF_F16=1 vs 0 gain; M-L in moe_fused_iq.cu.
6. **F12 TP decode** (+~9% decode, 2 cards): XL. Then **F13 megakernel** (+23-27% ms/window on TP), **F15 refill/SDMA**, **F16 TP prefill** (+17-26% at 12-16K), **F20 kfdq** (only if GPU_MAX_HW_QUEUES=8). Chain dependency: F12 -> F13; F12+F10 -> F15 -> F16; F20 only with F12; F21 follows F10/F16.
Skip: F01 (rocWMMA/peer), F08 (Q4 only), F14 mostly, F17, F18, F19 (AVX-512 IQ3 kernel), F22 (except presence_penalty config), F23-F25.

## Toolchain
- v0.2.16 builds against system ROCm 7 by design (setup uses /opt/rocm); ROCm 7.2.4 present with hsa-runtime64, hipBLASLt 1.2.x, rocWMMA 2.2.
- Fork pins TheRock 7.10.0a20251120 + rocWMMA 1ab208f4 (rocm-7.1.1). rocWMMA is used ONLY by the xyzzing QSA select (F01), which v0.2.16 replaces with its own builtin WMMA select: no rocWMMA needed.
- Families F06/F07/F09/F05 use `__builtin_amdgcn_wmma_*` (compiler builtins, gfx1100): fine on clang 22 (roc-7.2.4) (inferred, not built).
- F15 needs libhsa-runtime64 (present). F20 interposes ioctl and matches KFD structs by command number (handles 88 vs 96-byte create args) - should work with system ROCr but unverified.
- I found no code requiring ROCm 7.10-only features; the only version-bound artefact is the hipBLASLt table (use 100202).
- Nothing was built or run (no GPU).

## Open questions
1. Does upstream's PREFILL_HELP already cover PF_FUSED kernels (F11)? Not verified (git log -S too slow).
2. 13:00.0 link under load (idle read x1 Gen1); both cards expected Gen4 x8.
3. Does GDN_CHUNKED upstream (validated only on gfx1151) give the fork's gain on gfx1100? Needs A/B (Korean user's 952->1,790 says yes with the 4 switches).
4. `vision` JSON block keys (exe/mmproj/model/gpu/max_tokens/threads): top-level accepted, inner keys not verified.
5. RAM: UD-IQ4_XS experts ~60 GB per fork doc vs 53 GiB available; 1-card relies on SSD streaming unless 2 cards (2x24 GB VRAM holds most). Not checked on our pack.
6. Whether upstream arch_defaults will ever turn these on for gfx1100 (upstream commits arriving daily; re-diff before each rebase).
