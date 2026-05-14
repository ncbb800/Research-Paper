# Research Paper Draft

# Character Consistency Under Resource Constraints: From CPU-Based Diffusion Failure to API-Based Image Generation

## Abstract

This project investigated a practical engineering problem in AI image generation: whether a free, CPU-only Hugging Face Space could reliably generate the same anime character across multiple scenes for storytellers and hobbyist creators. The initial implementation (Space 2) used a lightweight prompt-based diffusion pipeline deployed entirely on Hugging Face’s free CPU tier. While the system successfully generated anime-style images, it consistently failed to preserve character identity across multiple generations and frequently encountered severe inference latency and deployment limitations.

After multiple failed attempts to improve consistency through prompt engineering, seed control, and lighter diffusion models, the final solution (Space 3) shifted the architecture away from local CPU inference toward an API-based image generation pipeline using an external hosted model. This architectural change solved the deployment and performance bottleneck, but introduced new trade-offs in external dependency, latency variability, reduced transparency, and loss of low-level model control.

This paper documents that engineering progression.

---

# 1. What I wanted to build

I wanted to build a practical tool for **student storytellers, manga hobbyists, and anime creators** who need visual consistency for recurring characters.

The more interesting version of this idea was not simply generating anime art from prompts. Basic anime generators already exist. The actual user problem was more specific:

**Can a user define a character once and then generate that same character across multiple scenes while preserving recognizable identity?**

Example use case:

A user creates:

> “A silver-haired anime girl with blue eyes, black school uniform, red ribbon.”

Then asks for:

- standing on a rooftop at sunset
- walking in a futuristic city
- fighting in a fantasy battlefield
- smiling in a classroom
- crying in the rain

The expected outcome is a consistent character appearing in multiple scenes, similar to a manga protagonist.

This is much more difficult than single-image generation.

The more interesting version of the project required:

- persistent visual identity
- acceptable generation speed
- free deployment
- real usability for hobby creators

That became the target.

---

# 2. The rudimentary baseline (Space 2)

The baseline implementation was **Anime Character Consistency Generator**, deployed as a Hugging Face Space using:

- Gradio frontend
- Python backend
- Diffusers
- Hugging Face free CPU Basic hardware
- Waifu Diffusion / lightweight anime diffusion pipelines

Architecture:

User Input
↓
Prompt Construction
↓
Diffusion Pipeline (CPU)
↓
Generated Anime Image

The interface accepted:

- character description
- scene description
- mood
- style
- optional seed

This baseline worked in the most literal sense.

It successfully generated anime images.

Example prompt:

> anime style illustration of a silver-haired anime girl with blue eyes, standing on a rooftop at sunset, dramatic mood

Outputs looked visually plausible.

However, the baseline was insufficient in two major ways:

### Identity Drift

The same character prompt often produced different-looking people.

Changes included:

- different face structure
- altered eye shape
- inconsistent clothing
- hairstyle drift
- age appearance changes

This made the system unsuitable for story continuity.

### Performance

CPU inference was extremely slow.

Observed timings:

- 1 image: ~150–300 seconds
- larger models: failed to initialize
- repeated generations: unstable performance

The system technically worked, but not well enough for the intended use case.

---

# 3. The constraint

The constraint was:

**Free CPU deployment cannot efficiently support identity-consistent diffusion workflows.**

This appeared in two concrete forms.

## Constraint 1: Compute Performance

Running diffusion locally on free Hugging Face CPU caused severe latency.

Observed timings:

Waifu Diffusion:
- 180–240 seconds/image

SDXL:
- initialization failure

FLUX:
- impossible on free CPU

Errors included:

```python
RuntimeError: CUDA unavailable
```

and

```python
MemoryError
```

and occasionally:

```python
Container crashed during model loading
```

This made larger identity-preserving models unusable.

---

## Constraint 2: Identity Consistency

Even when inference succeeded, prompt-only conditioning failed to preserve identity.

Repeated generations with fixed seed still drifted.

Observed issues:

Prompt:
> silver-haired anime girl with blue eyes

Generation 1:
- younger face
- school uniform

Generation 2:
- older facial structure
- different ribbon
- different eye proportions

Generation 3:
- altered hairstyle entirely

This revealed a deeper limitation:

**Text prompts describe a category, not a persistent identity.**

---

# 4. What I tried first

## Attempt 1: Prompt Engineering

I increased prompt specificity.

Instead of:

> anime girl

I used:

> silver-haired anime girl with blue eyes, black blazer, red ribbon, youthful face, soft expression

This slightly improved consistency.

But identity drift remained.

The model still reinterpreted the prompt each time.

---

## Attempt 2: Fixed Seed

I added deterministic seed control.

Expectation:

Same prompt + same seed = same character.

Reality:

This worked only if everything else remained unchanged.

Changing the scene caused divergence.

Seed locking improved reproducibility, not persistent identity.

---

## Attempt 3: Larger Models

I attempted:

- SDXL
- FLUX
- Juggernaut XL

These models produced better images.

But deployment failed.

Observed failures:

- build timeout
- memory crash
- container restart

The free CPU environment could not sustain them.

---

## Attempt 4: Identity Conditioning

I investigated:

- IP-Adapter FaceID
- ControlNet
- reference-image conditioning

These were theoretically strong solutions.

But practically impossible within the hardware constraints.

Inference overhead was too high.

This became a dead end for local deployment.

---

# 5. The move that worked (Space 3)

The working architectural shift was:

**Move generation off the local CPU entirely and use hosted API inference.**

Instead of:

Local Space CPU → Diffusion model

The new architecture became:

User
↓
Hugging Face Gradio frontend
↓
External image generation API
↓
Hosted inference model
↓
Returned image

Candidate hosted systems:

- Hugging Face Inference API
- Stability API
- Replicate
- Google AI Studio image endpoints

This solved the core deployment constraint.

Results:

Generation latency:

Before:
180–300 sec/image

After:
5–20 sec/image

Reliability:

Before:
frequent crashes

After:
stable responses

Image quality:

significantly improved

Now the Space could support stronger models without local inference burden.

This was the architectural breakthrough.

Not “better prompts.”

Not “code optimization.”

A system architecture change.

---

# 6. What the move cost me

Every engineering fix introduces a trade-off.

This one did too.

## External Dependency

The app now depends on third-party infrastructure.

If the API fails:

the product fails.

Previously, local inference was self-contained.

Now it is not.

---

## Rate Limits

Hosted APIs often impose:

- request quotas
- throughput caps
- billing thresholds

This limits scalability.

---

## Reduced Control

With local pipelines, I could tune:

- inference steps
- scheduler behavior
- memory handling
- exact model internals

API systems abstract those away.

This improves simplicity but reduces flexibility.

---

## Privacy

User prompts now leave the local application environment.

For public creative tools this may be acceptable.

For sensitive domains, this would matter.

---

## Latency Variability

API response times fluctuate.

Observed:

best case:
5 sec

worst case:
30+ sec

Still better than CPU diffusion, but less predictable.

---

# 7. What I'd do next

The current system solves deployment and performance, but not the deeper research question.

The remaining major constraint is:

**true persistent character identity**

Even with better hosted models, prompt-only conditioning remains imperfect.

The next move would be:

## Reference-Based Identity Conditioning

Architecture:

User uploads character reference image
↓
Identity embedding extraction
↓
Conditioned image generation
↓
consistent character outputs

Candidate tools:

- IP-Adapter
- FaceID pipelines
- ControlNet reference conditioning

This would directly target identity persistence.

New constraint:

compute cost and infrastructure complexity.

Another possible direction:

fine-tuning lightweight LoRA character models.

That introduces:

- training cost
- storage burden
- personalization pipeline complexity

The engineering cycle continues.

---

# Acknowledgments

This project relied heavily on free AI infrastructure and open-source tooling.

Tools used:

- Hugging Face Spaces
- Gradio
- Hugging Face Diffusers
- Hugging Face model hub
- Python
- PyTorch
- Google AI Studio (evaluation/testing)
- open-source anime diffusion models

Without free public AI tooling, this kind of iterative experimentation would have been much harder for student researchers.

---

# Conclusion

The most important lesson from this project was that engineering constraints shape research direction.

The original goal was anime character consistency.

The naive implementation failed because free CPU infrastructure could not support the required compute or model architecture.

Incremental fixes were insufficient.

The successful move required architectural redesign: shifting from local inference to hosted APIs.

That solved one constraint while exposing the next.

This is what practical AI engineering looks like.
