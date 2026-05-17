# From CPU Failure to Hosted Inference: Solving Compute Constraints While Exposing Persistent Identity Drift in AI Image Generation

## Abstract

This project explored a practical engineering problem in AI image generation: whether a freely deployable web application could generate visually consistent images of the same subject across multiple scenes and artistic styles.

The original goal was to build a tool for storytellers, designers, and student creators who want to define a subject once and then generate that same subject in different settings. The baseline implementation (Space 2) ran a local diffusion pipeline on Hugging Face’s free CPU infrastructure. While it successfully generated images, inference was extremely slow, larger models failed to run, and prompt-only generation produced severe identity drift.

Several incremental fixes were attempted, including prompt engineering, deterministic seed control, larger hosted models, and identity-conditioning research. Hosted APIs introduced new engineering constraints, including quota failures and provider compatibility issues. Eventually, the successful architectural move was to shift heavy inference to hosted image generation using fal.ai-hosted FLUX models through a Hugging Face Space frontend.

This solved the compute and deployment problem, dramatically improved image quality, and allowed multi-style image generation. However, a deeper limitation remained: even strong hosted text-to-image systems do not reliably preserve persistent subject identity across different scenes.

This paper documents that engineering progression.

---

# 1. What I wanted to build

I wanted to build a practical AI image generation tool for:

- storytellers
- concept artists
- student creators
- game designers
- hobby visual creators

The user problem was not simply "generate an image."

That problem is already solved by many public tools.

The more interesting version was:

**Can a user define a subject once, then generate that same recognizable subject across multiple scenes and artistic styles?**

Example:

A user defines:

> A silver-haired young woman with sharp blue eyes, black jacket, red scarf, and calm expression.

Then asks for:

- standing in futuristic Tokyo at night
- walking through a rainy cyberpunk alley
- painted as an oil portrait
- rendered as anime
- transformed into fantasy concept art
- placed in a medieval castle

The expectation is not just visual similarity in style, but persistent identity.

That requires:

- subject consistency
- multi-style flexibility
- acceptable latency
- free or low-cost deployment
- practical usability

This was significantly harder than basic text-to-image generation.

---

# 2. The rudimentary baseline (Space 2)

The baseline implementation was **Anime Character Consistency Generator**, deployed entirely on Hugging Face’s free CPU tier.

Architecture:

User Input  
↓  
Prompt Construction  
↓  
Local Diffusion Pipeline (CPU)  
↓  
Generated Image

Technology stack:

- Hugging Face Spaces
- Gradio
- Python
- Diffusers
- PyTorch
- Waifu Diffusion / lightweight anime diffusion models

The interface accepted:

- character description
- scene description
- mood
- style
- optional seed

Example prompt:

> anime-style silver-haired girl with blue eyes standing on a rooftop at sunset

The baseline technically worked.

It generated anime images successfully.

However, it failed in two important ways.

## Identity Drift

Repeated generations with the same character description produced visibly different faces.

Observed changes:

- altered facial structure
- different eye proportions
- inconsistent hair shape
- outfit drift
- age changes
- expression distortion

This made the system unsuitable for story continuity.

---

## Performance Constraints

Running diffusion locally on free CPU was extremely slow.

Observed timings:

- 150–300 seconds per image
- occasional container instability
- repeated slow cold starts

Larger models failed entirely.

Examples:

SDXL:
failed to initialize

FLUX:
unusable locally

Errors included:

```python
RuntimeError: CUDA unavailable
```

```python
MemoryError
```

```python
Container crashed during model loading
```

The baseline worked, but not at a usable engineering level.

---

# 3. The constraint

The main constraint was:

**Free CPU deployment could not support practical identity-consistent image generation.**

This appeared in several forms.

---

## Constraint 1: Local Compute Limits

Local inference on free CPU was too slow.

Observed:

Waifu Diffusion:
180–240 sec/image

Larger models:
complete failure

The local environment simply could not support modern image generation workloads.

---

## Constraint 2: Prompt-Only Identity Failure

Even when generation succeeded, prompt-only conditioning did not preserve subject identity.

Repeated generations changed:

- face shape
- hairstyle
- facial age
- eye structure
- clothing

This revealed:

**Text prompts describe a category, not a persistent identity.**

---

## Constraint 3: Hosted API Access Limits

I attempted to solve compute limits by moving inference to Gemini image generation.

This introduced a new failure:

```txt
429 RESOURCE_EXHAUSTED
Quota exceeded
free tier limit: 0
```

Meaning:

the architecture was valid, but API access was unavailable under free-tier limits.

---

## Constraint 4: Provider Compatibility Failures

I then moved to Hugging Face hosted inference.

My first FLUX implementation failed with:

```txt
410 Gone
The requested model is deprecated and no longer supported by provider hf-inference
```

Meaning:

model selection alone was insufficient.

Provider compatibility mattered.

---

# 4. What I tried first

## Attempt 1: Prompt Engineering

I increased prompt specificity.

Instead of:

> young woman

I used:

> silver-haired young woman with sharp blue eyes, red scarf, black jacket, calm expression

Result:

slightly better consistency

But identity drift remained.

The model still reinterpreted the prompt every time.

---

## Attempt 2: Fixed Seed Control

I introduced deterministic seed control.

Expectation:

Same prompt + same seed = same subject

Reality:

This helped reproducibility only when everything else remained unchanged.

Changing the scene still caused divergence.

Seed locking improved repeatability, not persistent identity.

---

## Attempt 3: Larger Local Models

I attempted:

- SDXL
- FLUX
- Juggernaut

Result:

better theoretical capability

Actual outcome:

deployment failure

Observed:

- build timeouts
- memory crashes
- container restarts

Conclusion:

local hardware constraints prevented scaling.

---

## Attempt 4: Identity Conditioning Research

I investigated:

- IP-Adapter
- FaceID
- ControlNet reference conditioning

These appeared promising.

However:

they required heavier inference pipelines unsuitable for free CPU deployment.

Conclusion:

theoretically correct, practically infeasible.

---

## Attempt 5: Gemini Hosted Image API

I shifted inference off local hardware.

Expectation:

hosted inference solves compute limits

Actual result:

```txt
429 RESOURCE_EXHAUSTED
```

because the Gemini image model had no usable free-tier quota.

Conclusion:

external APIs solve compute but introduce access dependency.

---

## Attempt 6: Hugging Face Hosted Inference

I switched to Hugging Face hosted inference.

Expectation:

same ecosystem, easier deployment

Actual result:

```txt
410 Gone
```

because FLUX was not supported by the selected provider.

Conclusion:

hosted inference requires provider-model compatibility awareness.

---

# 5. The move that worked (Space 3)

The successful architectural move was:

**Move heavy inference entirely off local hardware and route image generation to hosted fal.ai FLUX inference through a Hugging Face frontend.**

New architecture:

User  
↓  
Hugging Face Gradio frontend  
↓  
Style selection  
↓  
Routing layer  
↓  
fal.ai hosted FLUX inference  
↓  
Generated image

This solved:

- CPU limitations
- local memory failures
- deployment instability
- unacceptable inference times

Results:

Before:

150–300 sec/image

After:

5–20 sec/image

Quality:

dramatically improved

Reliability:

stable hosted inference

This was not a code optimization.

It was a system architecture redesign.

# 5.5 Comparative System Evaluation

The engineering progression can be summarized by comparing the major implementations tested during the project.

| System Version | Deployment Method | Typical Latency | Reliability | Image Quality | Identity Consistency | Major Constraint |
|---------------|------------------|----------------|-------------|--------------|---------------------|------------------|
| Space 2 (Local CPU Diffusion) | Hugging Face free CPU | 150–300 sec/image | Low | Moderate | Poor | Compute limitations, crashes |
| Gemini Hosted API | External hosted API | Not fully testable | Failed | N/A | N/A | 429 RESOURCE_EXHAUSTED quota limits |
| HF Hosted Inference (hf-inference + FLUX attempt) | Hosted inference | Not fully testable | Failed | N/A | N/A | 410 deprecated provider-model mismatch |
| Space 3 (fal.ai + FLUX hosted inference) | Hosted inference via fal.ai | 5–20 sec/image | High | High | Moderate to Poor | Persistent identity drift |

Several patterns became clear.

First, moving inference off local CPU dramatically improved runtime performance and deployment stability.

Second, hosted infrastructure introduced its own engineering risks, including quota limits and provider compatibility constraints.

Third, better image quality did not automatically solve the deeper research problem of identity consistency.

This confirmed that compute performance and subject continuity are related but separate engineering challenges.

---

# 6. What the move cost me

Every engineering fix introduces trade-offs.

This one did too.

---

## External Dependency

The application now depends on:

- fal.ai
- hosted inference availability
- API routing

If the provider fails:

the product fails.

Local inference avoided this dependency.

---

## Rate Limits / Billing Risk

Hosted inference may impose:

- quotas
- throttling
- paid usage

This creates operational constraints.

---

## Reduced Low-Level Control

With local diffusion, I could tune:

- schedulers
- inference steps
- precision
- memory behavior

Hosted systems abstract those away.

Convenience increased.

Control decreased.

---

## Latency Variability

Observed:

best case:
5 sec

worst case:
20–30 sec

Still dramatically better than local CPU.

But less predictable.

---

## Persistent Identity Still Fails

This was the most important remaining trade-off.

Even with strong hosted FLUX inference:

faces still changed across scenes.

Why?

Because the system remained text-to-image.

No persistent identity embeddings were used.

The hosted model improved quality and speed, but not true subject continuity.

---

# 7. What I'd do next

The current system solved deployment.

It did not solve identity persistence.

The next engineering move would be:

## Reference-Based Identity Conditioning

Architecture:

User uploads reference image  
↓  
Identity embedding extraction  
↓  
Conditioned generation  
↓  
Consistent outputs

Candidate tools:

- IP-Adapter
- FaceID
- ControlNet reference pipelines
- image-to-image workflows

This directly addresses the core unresolved problem.

---

## LoRA Personalization

Another possibility:

train lightweight personalized subject models.

Architecture:

subject images  
↓  
LoRA fine-tuning  
↓  
specialized generation

Trade-offs:

- training time
- storage
- complexity
- user onboarding friction

---

# Acknowledgments

This project relied heavily on free and open AI infrastructure.

Tools used:

- Hugging Face Spaces
- Gradio
- Hugging Face hosted inference
- fal.ai
- Hugging Face model hub
- Diffusers
- Python
- PyTorch
- Gemini API experimentation

Without public AI infrastructure, iterative experimentation at this scale would not have been possible.

---

# Conclusion

The biggest lesson from this project was that engineering constraints shape research direction.

The original problem was persistent subject identity in generated imagery.

The first implementation failed because local CPU infrastructure could not support practical image generation workloads.

Incremental prompt-based fixes were insufficient.

Hosted APIs solved deployment and compute constraints, but introduced new dependencies and compatibility failures.

Even after solving infrastructure, the deeper research problem remained:

**high-quality text-to-image generation does not equal persistent identity consistency.**

This is what practical AI engineering looks like:

solve one constraint, expose the next.
