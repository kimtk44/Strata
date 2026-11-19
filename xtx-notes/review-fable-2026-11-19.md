# Fable review 2026-11-19: improvement plan (IQ3_S, 2x 7900 XTX)

Read-only review by a Fable subagent. Caller spot-checked 11-19: generate.cpp:11287 (split leaves later stages' cached experts out of the RAM copy), verify.cpp:5561 (pl_finish samples with hist_d_), generate.cpp:14058 (tail checkpoint `!multi_gpu`), auto split K differs per arm (b2np K=22, b4np K=23 - the review had them swapped), "conversation cache off: not enough free RAM" in b4np. All confirmed.

## Top 3
1. RAM: `--resident-experts` (cache-complement copy, works on a layer split) instead of the full 46.84 GiB pinned arena -> ~13-15 GiB RSS (estimate). Pause-subtracted cold prefill ~3150-3280 vs 2541-2667 measured (+20%). Ends reaper kills, brings the conversation cache back. The RAM-short env trio did not help (pauses 6.4-12.5 s with it vs 1.5-2.4 s without; confounded by order). Later, with services moved off: hugetlb pool `vm.nr_hugepages=24000` (arena tries MAP_HUGETLB first) + `vm.compaction_proactiveness=0` - host changes, user-run. Engine option: extend the iGPU-only page-cache give-back gate (generate.cpp:4170-4216) to large arenas.
2. Penalties under the pipeline (~15 lines): drop `hist_n > 0` at generate.cpp:20628; before `V1(A).pl_launch` (21179) upload penalty_rows to d_hist as the serial loop does (21778-21787); separate pl_cal gate for penalised requests. Phase 2: mtp.set_draft_history. Expect thinking-off default 56-69 -> ~67-83. Verify token-exact vs serial with STRATA_IQ_MT_MIN=1, same seed; STRATA_PIPELINE_SWITCH toggles pw in one process.
3. Warm turns re-read 10K tokens (~6.5 s): `--prompt-cache-every 8192` -> ~1.8K re-read (~1.5 s); patch to allow the tail checkpoint on a split (on_stage_chunk 14097-14117); `STRATA_PREFILL_HELP=1` for short tails.

## Measurement fixes
Pin `--layer-split`; temperature 0 or --spec-follow for decode A/B; STRATA_DECODE_TIMING / SPLIT_TIMING / VERIFY_PROFILE / PIPELINE_LOG / SPEC_DEPTH; sweep per request via strata_tune (spec_min_p, pcie_frac, aux_cpus). Pipelined window 33.0 ms vs ideal ~21 ms; serial 41.9 ms vs model 30.7.

## Settings to sweep
spec-min-p 0.6/0.7/0.8; spec 3/5; STRATA_PIPELINE_HIT_MIN=0.35, PIPELINE_LAZY_CHAIN=1, PIPELINE_LOOKUP_RUN=1; STRATA_SPEC_COUPLED=1 (+GUMBEL) serial first; split 24/25/26; kv-resident 65536/98304 or k8v4; vram-reserve-mib 512, adapt-swaps 48, expert-profile-save; prefill auto:16384, calibrate prefill-pipe; pool-workers 5/unset (4 on two cards), host-core sibling; the unapplied gfx11 opt-ins (HIP_WMMA, SELECT_WMMA, PF_GEMM, PF_PAD, PA_FAST, HC_UPMIX, PF_HCDOWN, QSA_ROWS, QFUSE, VERIFY_QDEDUP, KV_HOST_DMA, GDN_*, FUSE_CONVAB, HEAD_MIX_MULTI, FUSE_HITS, FUSE_WAIT) in groups of 3-4.

## F02/F03 order for this model (dense = mixed Q4_K/Q5_K/Q6_K/IQ4_XS, hc = BF16)
RDNA_Q6K -> +PAIR -> GR_RDNA -> ROUTE_PUBLISH -> COMBINE_CPU+WAIT_FUSE (vs FUSE_WAIT/FUSE_HITS) -> GR_RDNA_Q8/A8/INJ_UP/GR_FUSED last, one card (Q8 is lossy here: hc is BF16; GR_FUSED needs bounded-spin fallback first). Skip RDNA_IQ3S (no IQ3_S dense tensors), KQ_ALL (windows ~3.7 rows). DPP_SHFL separate build after GR_RDNA.

## Beyond the fork
Megakernel on the layer-split verifier (L); TP chain F12->F13/F15/F16 (XL, fork 133-147 decode) with REFILL_SDMA=0 on <6.14, watchdog scaled (>=180), F20 queue parking; DP/TP prefill pays under 16K warm (needs full arena -> hugetlb); P2P not worth it for layer split; WMMA decode no; coupled sampling in the chain (M) if acceptance gain shows; size-aware cache fill (frequency per byte).

## Corrections
DP prefill: arena fits (46.84 GiB), headroom is the problem. hc matrices are BF16 in IQ3_S (inventory assumed Q8_0). F04 note stands but rerun with pinned split.
