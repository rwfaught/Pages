---
layout: default
title: "Qwen3.8-27B on a 5090 Laptop: Q4 vs NVFP4"
description: "A practical investigation of Qwen3.8-27B performance, context scaling, VRAM use, and coding-agent behavior on a 24 GB RTX 5090 Laptop GPU."
---

# Follow-up: 

I went down the Qwen3.8 5090 rabbit hole and I think I finally understand what I was seeing

I said I was going to dig into this because my results on a 5090 laptop looked way different from yours, and I think I finally have enough data to explain most of it. This mostly started with your earlier llama.cpp numbers. I’m not treating my laptop as a clean hardware A/B against everything in this newer vLLM/NInfer/SGLang post because obviously it isn’t: different GPU class, different OS in the earlier run, different weights, different runtime settings, different cache setup, etc. What I wanted to know was why the difference looked so enormous, and whether my llama.cpp setup was leaving a bunch of performance on the table.

The short version is that yes, I was leaving performance on the table, but the biggest apparent disagreement between our results turned out to be me comparing two different throughput measurements. Then I tested the quantization/model representation directly on my own machine, and after that I went a step further and tested whether the faster NVFP4 version was actually worse at repository analysis or generating new code. That last part turned out to be the most interesting part of this whole thing.

## TL;DR

- The crazy-looking difference in context scaling was mostly a metric mismatch. I was looking at separated llama.cpp decode TG while your figures were output tokens / total wall time including prefill. Once I measured mine the same way, our normalized context-degradation curves were almost identical.

- On my own 24GB 5090 Laptop, changing from UD-Q4_K_XL to NVFP4-MTP-LOW while holding the rest of the important runtime config constant gave me about 25–29% more decode throughput, 50%+ more prompt-processing throughput, and roughly 2.3 GiB more free VRAM at 128K.

- That explains a large part of why your original llama.cpp result was so much faster than mine. Once I did that, the remaining desktop-5090 advantage stopped looking mysterious: my laptop gets about 86.5 tok/s at depth 0 with NVFP4/MTP@2 versus the roughly 115–128 tok/s territory around the desktop results I was comparing against. A 1.3–1.5x remaining hardware-class difference is not difficult to believe.

- I then did matched Q4 vs NVFP4 agent tests because raw tok/s is not really the thing I care about. Both successfully analyzed a fairly complicated repository, and then both solved a new coding task and passed 16/16 hidden tests they never saw.

- The weird part is that Q4 was slower per token but faster as an agent. It solved the coding task in about 210 seconds versus 279 seconds for NVFP4 because it used fewer actions and generated a lot less reasoning/output. So at least from what I have tested so far, I do not see evidence that NVFP4 causes some huge coding-quality collapse, but Q4 still looks more disciplined and efficient in the agent loop.

## My setup

For the matched tests I kept the important runtime variables fixed and changed the model representation. I deliberately stayed at MTP@2 for the matched runs because I had already tested the draft-window question separately and did not want to keep moving multiple variables at once.

| Metric | My machine |
| --- | --- |
| GPU | RTX 5090 Laptop GPU, 24 GB |
| OS | Ubuntu 26.04 |
| Runtime | llama.cpp |
| Build during these tests | b10423 / a94d563ed |
| Context | 131,072 |
| KV | Q8_0 K + V |
| MTP | draft MTP, n-max 2 |
| Parallel | 1 |
| Seed | 42 |
| Reasoning | Medium |
| Models | Qwen3.8-27B UD-Q4_K_XL and NVFP4-MTP-LOW |

Your earlier llama.cpp setup was materially different: desktop 5090 32GB, NVFP4-MTP-LOW, Q8 KV, MTP@4, about 192K context allocation, batch/ubatch 512, unified KV and a large RAM prompt cache. So my original Q4 numbers were never actually apples-to-apples with yours in the first place.

## The first thing I got wrong: the context curve

This was probably the biggest source of confusion. My original Q4 MTP@2 decode numbers barely moved relative to what I thought I was seeing in your results, so my first reaction was basically: why the hell is his throughput collapsing with context while mine barely moves? The answer is that those were not the same measurements. You explicitly describe yours as generated tokens divided by complete wall time, including prefill. I was comparing that against llama.cpp’s separated decode TG.

| Context depth | My decode TG |
| --- | --- |
| 0 | 67.0 |
| 4K | ~62 |
| 8K | ~64 |
| 16K | ~63 |
| 32K | 56.0 |

| Context depth | Your llama.cpp result |
| --- | --- |
| 0 | 114.3 |
| 4K | 64.4 |
| 8K | 45.4 |
| 16K | 27.9 |
| 32K | 14.6 |

Once I reran mine using the same effective request-throughput calculation, the absolute numbers were still very different, obviously, but the shape changed completely. Mine came out at 36.7, 19.7, 13.9, 8.2 and 4.3 across 0/4K/8K/16K/32K, while yours were 114.3, 64.4, 45.4, 27.9 and 14.6. Normalize each system to its own depth-0 result and the supposed mystery basically disappears:

| Depth | Mine | Yours |
| --- | --- | --- |
| 0 | 100% | 100% |
| 4K | 53.7% | 56.3% |
| 8K | 37.9% | 39.7% |
| 16K | 22.3% | 24.4% |
| 32K | 11.7% | 12.8% |

That was the “oh...” moment. The shapes are basically the same. My actual decode throughput does hold up reasonably well with context, but if you charge increasingly expensive prefill against only 128 generated tokens, of course the effective output-tokens-per-wall-second number falls off a cliff. So there was not some magic context-scaling advantage on my laptop. I was just comparing two different things.

## Then I tested NVFP4 on the laptop

This was the part that explained a lot of the remaining absolute gap. I downloaded the same Qwen3.8-27B-NVFP4-MTP-LOW family and tested it against my UD-Q4_K_XL on the same machine. The files themselves are already pretty different: Q4_K_XL is about 16.692 GiB and NVFP4 LOW is about 14.468 GiB, so NVFP4 is roughly 2.225 GiB smaller on disk. More importantly, that difference carries through to the actual 128K deployment.

| Metric | Q4_K_XL | NVFP4 LOW |
| --- | --- | --- |
| VRAM used | ~23,263 MiB | ~21,001 MiB |
| VRAM free | ~722 MiB | ~2,984 MiB |

That is roughly 2.3 GiB of additional VRAM headroom on a GPU where 2.3 GiB is a big deal. I am already operating close enough to the 24GB limit that this changes what else I can comfortably have running on the machine.

| Depth | Q4 PP | Q4 TG | NVFP4 PP | NVFP4 TG |
| --- | --- | --- | --- | --- |
| 0 | ~1306 | 67.0 | 2041 | 86.5 |
| 32K | ~1211 | 56.0 | 1834 | 69.8 |

So changing the representation alone got me 67.0 → 86.5 tok/s at depth 0, about +29%, and 56.0 → 69.8 at 32K, about +25%. Prompt processing moved from roughly 1306 → 2041 at depth 0 and 1211 → 1834 at 32K, which is around a 50%+ improvement. NVFP4 also did not suddenly lose the advantage as context increased: Q4 retained about 83.6% of its depth-0 decode rate at 32K and NVFP4 retained about 80.7%, so it stayed substantially faster at depth.

## That made the remaining desktop/laptop gap look a lot less weird

With NVFP4 my 5090 Laptop is doing about 86.5 tok/s at depth 0. The desktop NVFP4 llama.cpp results I was comparing against were more in the roughly 115–128 tok/s territory depending on configuration. That puts the laptop at about 68–75% of the desktop result, or something like a 1.3–1.5x desktop advantage. That is a completely different situation from looking at my original Q4 result and wondering why the other machine looked 3x+ faster.

A desktop 5090 has massively more power available, more memory bandwidth, better thermal capacity, 32GB instead of 24GB, etc. There are still runtime/build/config differences mixed in there too, so I am not claiming I isolated a clean “desktop 5090 is exactly X% faster” number. I am just saying I do not see evidence anymore that my llama.cpp setup is mysteriously broken. Once I use NVFP4, what is left looks much more like a normal desktop-vs-laptop hardware difference.

## But I care more about coding-agent performance than tok/s

This is where I ended up going much farther than I originally intended. NVFP4 clearly won the raw performance comparison on my machine, but I use this model for local coding/agent work, so 29% more tok/s does not help me much if it makes worse decisions, wanders around the repository longer, breaks tool calls, hallucinates implementation details, or writes worse code. So I did two matched quality tests with the same machine, seed, Medium reasoning setting, context and agent setup. Only Q4 versus NVFP4 changed.

## Test 1: repository forensics

I gave both models a read-only agent task reconstructing a fairly complicated code-patching control loop in my Orchestrator repository. This was not “find a function and tell me what it does.” The model had to navigate a lot of source and docs, reconstruct the lifecycle across multiple modules, distinguish automatic steps from operator gates, identify what verification did and did not establish, run the target tests, and support the report with exact source citations. Both completed it successfully, but their behavior was noticeably different.

| Metric | Q4 | NVFP4 |
| --- | --- | --- |
| Time | 272 sec | 305 sec |
| Actions | 37 | 40 |
| Completion tokens | 7,218 | 12,688 |
| Reasoning chars | 4,052 | 19,445 |
| Turns | 38 | 41 |
| Strict single-JSON tool turns | 4 | 14 |

NVFP4 was substantially better at obeying the exact JSON tool protocol in that particular run: 34.1% strict versus only 10.5% for Q4. On the actual analysis, both got the architecture broadly right. After auditing the more aggressive factual claims against the source, I gave Q4 a narrow advantage in epistemic precision. NVFP4 made a few statements that were too broad, for example calling the mutation point the “only point that writes files” when earlier stages also write artifacts, and describing the new control loop as more decoupled from the old patch spine than it really is even though it deliberately bridges into the existing Phase-99 apply engine. Nothing catastrophic; it clearly understood the system. Q4 was just a little more conservative about saying exactly what the evidence established, and NVFP4 used a lot more reasoning to get there.

## Test 2: new code, with hidden tests

For this one I wanted something neither model could solve by just reading an existing implementation. I started both from pristine worktrees at the same exact commit and created a new requirement around the existing evidence_link.py contract: extend source_locator so it could support a new immutable structured locator object while remaining backward-compatible with the existing string/None forms. The task required normalization, validation, serialization, immutability, stable machine-readable errors, backward compatibility and downstream regression safety.

Both models could see the same public specification and repository. The important part is that I wrote a separate 16-test hidden acceptance suite and kept it completely outside the model-visible repository. The models could run the normal public tests and add their own tests, but they could not see the hidden grader. Once each model said it was finished, its patch was frozen and only then did I run the hidden tests externally.

| Metric | NVFP4 | Q4 |
| --- | --- | --- |
| Public regression tests | pass | pass |
| Hidden tests | 16/16 | 16/16 |
| Change scope | pass | pass |
| Finalized normally | yes | yes |

So as far as this coding task goes, NVFP4 absolutely did not show some obvious quality collapse. It wrote working new code that satisfied every hidden behavior I had specified before it saw the task result. Q4 did the same. But this is where things got weird.

## Q4 was slower at generating tokens and still finished the coding task faster

| Metric | NVFP4 | Q4 |
| --- | --- | --- |
| Hidden tests | 16/16 | 16/16 |
| Functional success | yes | yes |
| Actions | 24 | 15 |
| Wall time | 279.0 sec | 209.6 sec |
| Completion tokens | 19,324 | 12,195 |
| Reasoning chars | 48,823 | 23,671 |
| Strict JSON turns | 16 | 14 |
| Parser-recovered turns | 9 | 2 |
| Hard parse errors | 0 | 1 |
| Tool errors | 2 | 1 |

NVFP4 has about 25–30% higher raw decode throughput, and Q4 still finished the actual coding job about 25% sooner. Q4 used 15 actions instead of 24, generated about 37% fewer completion tokens and roughly half as much reasoning text. That is probably the most useful thing I learned from this whole exercise: raw token throughput is not the same thing as agent throughput. If the faster model decides to think twice as much, inspect more things, take more actions, or spend longer getting to closure, the tok/s advantage can disappear completely. The objective I actually care about is closer to time-to-verified-successful-completion, and on this particular coding task Q4 won that despite losing badly in raw generation speed.

There was even a small implementation-style difference that lined up with what I saw in the repository-analysis test. Both patches passed all hidden tests, so I do not want to exaggerate this, but Q4 took the more conservative backward-compatibility interpretation of an old malformed-mapping case. NVFP4 broadened that old case into the new structured-locator semantics and updated the public test accordingly. Q4 preserved the old behavior unless the mapping actually contained one of the new structured-locator keys. Again, both were functionally valid according to my hidden suite, but Q4’s choice was the more conservative API evolution, which is consistent with the slight precision/closure advantage I saw earlier.

## What I think this means

I am definitely less suspicious of NVFP4 than I was when I started. On a 24GB 5090 Laptop it has some very real advantages: roughly 25–30% faster decode, 50%+ faster prompt processing, and about 2.3 GiB more VRAM headroom. I also now have at least one real novel-code-generation test where it went 16/16 on hidden acceptance tests, so I cannot justify saying the speed comes with some obvious major coding-quality penalty.

But I am also not ready to say “NVFP4 is just Q4 but faster.” At least in these two agent tests, Q4 looked more economical with its reasoning and reached closure more efficiently. On the novel coding task that difference was large enough that Q4 beat NVFP4 in actual wall-clock completion time even though the underlying generation rate is much lower. So for my 24GB machine I think I am going to treat them as two slightly different tools rather than declare an absolute winner. NVFP4 is extremely attractive when VRAM headroom, prompt processing and raw generation speed matter. Q4 still looks like it may be the better profile when I care most about disciplined long-running coding-agent behavior and getting to a verified answer with fewer detours. I need more than one hidden coding task before I would call that a general property of the quantizations, though.

## Recommended starting point on a 24GB 5090 Laptop

Based on everything I have measured so far, my practical llama.cpp starting point for Qwen3.8-27B is currently 128K context, Q8_0 K/V, MTP@2 and Medium reasoning, with a fixed seed when I am benchmarking. I am treating NVFP4-MTP-LOW as the fast/VRAM-efficient profile and UD-Q4_K_XL as the coding-agent/higher-confidence comparison profile for now.

I would not read MTP@2 as me saying 2 is universally optimal. That is just where I froze the matched comparison after doing separate MTP-window testing. I also would not take my 128K choice as the model’s maximum useful context. On 24GB I am balancing context against enough VRAM headroom to actually run the rest of my desktop and agent stack without constantly living on the OOM cliff.

## Caveats

This is still a pretty small experiment. It is one 5090 Laptop, one llama.cpp build family, one repository-analysis task and one hidden-test coding task. The coding result is much stronger evidence to me than looking at benchmark perplexity or eyeballing two answers, but 16/16 on one task does not establish that NVFP4 and Q4 are universally equivalent. Likewise, I am not trying to derive an exact desktop-vs-laptop 5090 multiplier out of this because there are too many other differences.

What I think I can say at this point is that my original comparison with your context curve was wrong because I was comparing decode TG with end-to-end request throughput; my original Q4 model representation was responsible for a meaningful chunk of the absolute performance difference; NVFP4 is a substantially better fit for Blackwell/24GB from a raw performance and memory standpoint; I have not found evidence yet of a large functional coding-quality penalty; and Q4 may still have an agent-efficiency/closure advantage that raw tok/s benchmarks do not capture. That last one is probably what I am most interested in testing next.

I started this expecting to find some stupid llama.cpp flag that explained everything. Instead I mostly found out that your results made sense, I was measuring something different, and model speed gets a lot less straightforward once you put the model inside an agent loop. Which was honestly a much more interesting answer.
