# DeepVariant / DeepSomatic — Extended Fork
---

## Overview

This fork extends Google's [DeepVariant](https://github.com/google/deepvariant) and [DeepSomatic](https://github.com/google/deepsomatic) pipelines with several enhancements:

1. **Enriched VCF Output for Somatic Variant Calling** — Variant allele frequency (VAF) information from the normal (non-tumor) sample and CNN class probabilities are now emitted as standard VCF FORMAT fields.
2. **Selective Layer Fine-Tuning (Keras)** — A new utility allows freezing all layers except user-specified ones, enabling supervised fine-tuning on individual network layers to improve sensitivity for low-frequency somatic variants.

---

## My Contributions (Commit Details)

### 1. Non-Target Sample Allele Frequency in VCF Output
**Files:** `deepvariant/variant_calling_multisample.cc`, `deepvariant/variant_calling_multisample.h`, `deepvariant/dv_vcf_constants.py`, `deepvariant/postprocess_variants.py`

Added new VCF FORMAT fields so that every output `.vcf` record now includes allele information from the non-tumor (normal) sample alongside the tumor sample:

| FORMAT Field | Type    | Description |
|:-------------|:--------|:------------|
| `NVAF`       | Float   | Variant allele fraction in non-target (normal) samples |
| `NVAD`       | Integer | Variant allele depth in non-target (normal) samples |
| `NVDP`       | Integer | Total read depth in non-target (normal) samples |

**Implementation highlights:**
- **C++ core** (`variant_calling_multisample.cc`): Implemented `AddNonTargetAlleleFrequencies()`, which iterates over all non-target samples, aggregates their allele counts, computes per-alt-allele VAF and depth, and writes `NVAF`, `NVAD`, and `NVDP` into the VCF `VariantCall` using Nucleus `SetInfoField`.
- **VCF header** (`dv_vcf_constants.py`): Registered all new FORMAT fields with proper VCF spec metadata (Number, Type, Description).
- **Post-processing** (`postprocess_variants.py`): Added `NVAF` and `NVAD` to the alt-allele-indexed format field set so they are correctly re-indexed during VCF post-processing.

### 2. Prediction Probability Output (Artifact vs. Germline vs. Somatic)
**Files:** `deepvariant/variant_calling_multisample.cc`, `deepvariant/variant_calling_multisample.h`, `deepvariant/dv_vcf_constants.py`, `deepvariant/postprocess_variants.py`

The CNN's three-class softmax probabilities are now written directly into the VCF output, enabling transparent downstream filtering and analysis:

| FORMAT Field | Type  | Description |
|:-------------|:------|:------------|
| `P_REF`      | Float | Probability the variant is reference (artifact) |
| `P_GERMLINE` | Float | Probability the variant is germline |
| `P_SOMATIC`  | Float | Probability the variant is a true somatic variant |

**Implementation highlights:**
- **C++** (`variant_calling_multisample.cc`): Added static method `VariantCaller::AddPredictionProbabilities()` that writes the three-class probabilities into FORMAT fields, with zero-padding when fewer than three predictions are available.
- **Python** (`postprocess_variants.py`): Injected probability fields (`P_REF`, `P_GERMLINE`, `P_SOMATIC`) during the `add_call_to_variant` step using Nucleus `struct_utils`.
- **Header** (`dv_vcf_constants.py`): Registered all three probability FORMAT fields.

### 3. Selective Layer Fine-Tuning in Keras
**File:** `deepvariant/keras_modeling.py`

Added `set_trainable_layers()` — a utility function that enables **supervised fine-tuning on any user-specified subset of network layers** while freezing all others:

```python
def set_trainable_layers(
    model: tf.keras.Model,
    trainable_layer_names: Optional[Sequence[str]] = None,
) -> tf.keras.Model:
```
---

## Tech Stack

- **Languages:** Python, C++
- **ML Framework:** TensorFlow / Keras
- **Infrastructure:** Docker, AWS (EC2, S3), Github CI/CD
- **Genomics:** VCF, BAM/CRAM, Nucleus, htslib

---

## Base Repository

This fork is based on [google/deepvariant v1.9](https://github.com/google/deepvariant). The upstream README and documentation are preserved below for reference.

---

*The remainder of this README is the original DeepVariant documentation from Google.*

---

<img src="docs/images/dv_logo.png" width=50% height=50%>

[![release](https://img.shields.io/badge/release-v1.9-green?logo=github)](https://github.com/google/deepvariant/releases)

DeepVariant is a deep learning-based variant caller that takes aligned reads (in
BAM or CRAM format), produces pileup image tensors from them, classifies each
tensor using a convolutional neural network, and finally reports the results in
a standard VCF or gVCF file.

*See the [original DeepVariant documentation](https://github.com/google/deepvariant) for full usage instructions, case studies, and citation information.*

## License

[BSD-3-Clause license](LICENSE)

## Disclaimer

This is not an official Google product.

NOTE: the content of this research code repository (i) is not intended to be a
medical device; and (ii) is not intended for clinical use of any kind, including
but not limited to diagnosis or prognosis.
