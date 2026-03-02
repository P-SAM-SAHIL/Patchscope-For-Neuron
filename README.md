# Patchscope-For-Neuron


---

# Patchscope Residual Vector Reactivation Pipeline

This repository implements a **Patchscope-style mechanistic interpretability experiment** on the `gemma-2-2b` model.

The goal is to test whether a residual stream vector that strongly activates a neuron in its original context can **reactivate the same neuron when injected into a new prompt**.

Below is a step-by-step breakdown of how the Version 2 pipeline works and how it maps to the Patchscopes framework.

---

## Step 1  Setup and Initialization

### Model Loading

The script loads the `gemma-2-2b` model using **TransformerLens (`HookedTransformer`)**, which allows precise surgical interventions inside model layers using hooks.

### SAE Loading

A **Sparse Autoencoder (SAE)** trained on Layer 12 residual stream is loaded.
This is used later to translate dense vectors into interpretable features.

---

## Step 2 — Building the Activation Atlas (Profiling)

### Scanning the Dataset

The pipeline iterates through a subset of the `NeelNanda/pile-10k` dataset.

### Monitoring Neurons

For each text sample, it records activations at:

```
Layer 12 → MLP output (hook_mlp_post)
```

### Finding Maximum Activations

For every neuron, the script stores:

* Highest activation value
* Text index
* Token position
* Trigger token

This creates an **Activation Atlas**, mapping neurons to their strongest triggering contexts.

---

## Step 3 — Feature Extraction (Isolating Vector X)

### Target Selection

From the atlas, the neuron with the highest recorded activation is selected
(e.g., Neuron Y = 8526).

### Source Prompt Execution

The model is re-run on the exact text snippet that triggered this neuron.

### Residual Vector Extraction

Right **before Layer 12 processes the token**, the script copies the residual stream vector:

```
hook_resid_pre
```

This copied representation is called **Vector X**.

### Patchscopes Mapping

In Patchscopes terminology, this step extracts a **hidden representation from the source sequence**.

---

## Step 4 — SAE Verification (Interpretability Step)

Vector X is passed through the Sparse Autoencoder.

The SAE decomposes it into human-interpretable features such as:

* Mathematical concepts
* Set-related reasoning
* Structural language features

This helps interpret *what information is encoded inside Vector X*.

---

## Step 5 — Patchscope Intervention (Inject and Observe)

### Target Prompt Setup

A new prompt is defined:

```
"A detailed description of x is that"
```

The token position of `x` is identified as the injection point.

---

### Hook 1 — Injection

A hook is placed at:

```
hook_resid_pre
```

When the model processes token `x`, the hook:

* Replaces the current residual vector
* Injects **Vector X**

This detaches the representation from its original context.

---

### Hook 2 — Observation

A second hook is placed at:

```
hook_mlp_post
```

This records how **Neuron Y** responds to the injected vector in the new context.

---

### Generation Phase

After injection:

* The model continues autoregressive generation
* The patched hidden state is translated into natural language

---

## Patchscopes Framework Mapping

This pipeline directly mirrors Patchscopes methodology:

| Patchscopes Concept     | Implementation                     |
| ----------------------- | ---------------------------------- |
| Source representation   | Extracted residual vector X        |
| Representation transfer | Injection via resid_pre hook       |
| Translation context     | Target prompt with "x"             |
| Interpretation          | Generated text + neuron activation |

---

## Results
--- PART 2: Feature Extraction ---
- Targeting Neuron: 8526
- Max Activation: 9.3750 from token '3'
- Extracted Vector X from Layer 12, Position 94

=== Experiment Results ===
- Original Trigger Token: '3'
- Patched Target Prompt: 'A detailed description of x is that'
- Generated Attributes (Post-Patch): 'it is a set of all the possible outcomes of an experiment.The set of all possible outcomes'

- Neuron 8526 activation during patchscope: 3.1406

--- PART 3: SAE Verification ---
Top 5 Activated SAE Features in Vector X:
 - Feature_315: Activation = 41.0987
   Neuronpedia: https://neuronpedia.org/gemma-2-2b/12-gemmascope-res-16k/315
    - explanation : references to categories or classifications of subjects
 - Feature_6810: Activation = 31.1544
   Neuronpedia: https://neuronpedia.org/gemma-2-2b/12-gemmascope-res-16k/6810
   - explanation : terms related to software licensing and legal disclaimers
 - Feature_2291: Activation = 24.8089
   Neuronpedia: https://neuronpedia.org/gemma-2-2b/12-gemmascope-res-16k/2291
   - explanation : echnical or mathematical symbols and terms
 - Feature_164: Activation = 18.4591
   Neuronpedia: https://neuronpedia.org/gemma-2-2b/12-gemmascope-res-16k/164
   - explantion : references to Wikipedia and its articles
 - Feature_11195: Activation = 17.2775
   Neuronpedia: https://neuronpedia.org/gemma-2-2b/12-gemmascope-res-16k/11195
   - explanation : references to cats





## Why This Design Works

By placing:

* Injection **before** the layer
* Observation **after** the layer

The pipeline enables:

* Causal testing of neuron behavior
* Direct measurement of reactivation
* Natural-language decoding of hidden features



---

## Core Insight

This experiment reveals whether:

> Neurons respond to intrinsic features inside residual vectors rather than only to their original textual context.


