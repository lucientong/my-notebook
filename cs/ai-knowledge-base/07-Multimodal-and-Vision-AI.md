# Multimodal and Vision AI

Language: English | [中文](../AI知识库/07-多模态与视觉AI.md)

> This document covers computer vision fundamentals, Vision Transformers, multimodal models, fusion strategies, image generation, video understanding, OCR/document AI, speech, multimodal training, and production practice.

---

## 0. What This Document Solves

Multimodal AI extends language systems to images, video, audio, layout, and structured visual information. In interviews, you should explain both model mechanisms and production constraints: latency, data quality, alignment, safety, and evaluation.

---

## 1. Computer Vision Basics

### Concept

Computer vision models convert images or video frames into representations for classification, detection, segmentation, retrieval, or generation.

### Mechanism

Classic evolution:

```text
LeNet -> AlexNet -> VGG -> GoogLeNet -> ResNet -> EfficientNet -> ViT / hybrid models
```

Core tasks:

| Task | Output |
|---|---|
| Classification | image-level label |
| Detection | object class and bounding box |
| Segmentation | pixel-level mask |
| OCR | text and layout |
| Visual question answering | answer grounded in image |

### Trade-off

CNNs are efficient and strong for local visual patterns. Transformers scale well and integrate naturally with multimodal architectures, but often need more data and compute.

### Production Practice

- Track performance by image quality, device, lighting, and domain.
- Add fallback for low-confidence predictions.
- Monitor data drift for visual domains.
- Evaluate robustness to compression, cropping, and adversarial inputs.

### Interview Self-Check

- Why did residual connections help very deep networks?
- What is the difference between detection and segmentation?
- When would you still choose a CNN over a Transformer?

---

## 2. Vision Transformer

### Concept

Vision Transformer, or ViT, treats an image as a sequence of patches and applies Transformer encoders.

### Mechanism

```text
image
  -> split into patches
  -> patch embedding
  -> add CLS token and position embeddings
  -> Transformer encoder
  -> prediction head
```

### Trade-off

ViT scales well with data and compute, but lacks the strong locality bias of CNNs. It may require more pretraining data or distillation. Swin Transformer adds hierarchical windows to improve efficiency and locality.

### Production Practice

- Use pretrained vision backbones when data is limited.
- Benchmark latency and memory on target hardware.
- Consider hybrid CNN-Transformer architectures.
- Validate robustness across real image distributions.

### Interview Self-Check

- How does ViT convert an image into tokens?
- Why does ViT need position embeddings?
- What problem does Swin Transformer solve?

---

## 3. Multimodal Large Models

### Concept

Multimodal large models align visual, textual, audio, or document representations so the model can reason across modalities.

### Mechanism

Common architectures:

- CLIP-style contrastive alignment between image and text embeddings,
- vision encoder plus projection layer plus LLM,
- Q-Former or cross-attention bridge,
- unified tokenization for multiple modalities.

Example flow:

```text
image -> vision encoder -> projector / adapter -> LLM context -> answer
```

### Trade-off

Multimodal models improve understanding and interaction, but introduce grounding errors, hallucinated visual details, and higher inference cost.

### Production Practice

- Keep image provenance and preprocessing metadata.
- Add visual grounding or bounding boxes when possible.
- Evaluate both language quality and visual correctness.
- Use domain-specific OCR or detection tools for high-precision tasks.

### Interview Self-Check

- What does CLIP learn?
- How does LLaVA connect a vision encoder to an LLM?
- Why can a vision-language model hallucinate objects?

---

## 4. Multimodal Fusion Strategies

### Concept

Fusion combines information from multiple modalities.

### Mechanism

| Strategy | Description |
|---|---|
| Early fusion | combine raw or low-level features early |
| Late fusion | combine separate model outputs |
| Cross-attention | one modality attends to another |
| Adapter / projector | map one modality into LLM-compatible space |
| Q-Former | learn query tokens that extract visual information |

### Trade-off

Early fusion can learn rich interactions but is expensive. Late fusion is modular but may miss cross-modal details. Cross-attention is powerful but increases compute.

### Production Practice

- Choose fusion based on latency, data volume, and required grounding.
- Keep modality-specific confidence scores.
- Add missing-modality behavior.
- Evaluate cases where modalities conflict.

### Interview Self-Check

- Compare early fusion and late fusion.
- Why is cross-attention useful?
- How would you handle missing image input?

---

## 5. Image Generation

### Concept

Image generation models create images from text, images, layouts, or control signals.

### Mechanism

Diffusion models learn to denoise random noise step by step:

```text
noise -> iterative denoising conditioned on prompt -> image
```

Stable Diffusion uses a latent diffusion approach:

- text encoder,
- U-Net denoiser,
- VAE encoder/decoder,
- scheduler,
- optional ControlNet or LoRA adapters.

### Trade-off

Diffusion models produce high-quality images but are relatively slow and can generate unsafe, copyrighted, or misleading content.

### Production Practice

- Add prompt and output safety filtering.
- Use watermarking or provenance metadata when appropriate.
- Validate generated assets before user-facing use.
- Use ControlNet or image constraints for predictable layout.

### Interview Self-Check

- Why do diffusion models denoise iteratively?
- What does latent diffusion optimize?
- What risks exist in image generation products?

---

## 6. Video Understanding

### Concept

Video understanding models process temporal visual information for summarization, event detection, question answering, and retrieval.

### Mechanism

Approaches:

- sample frames and use image encoders,
- extract motion features,
- use temporal attention,
- combine audio, subtitles, and visual frames,
- summarize long video into hierarchical chunks.

### Trade-off

More frames improve coverage but increase cost. Sparse sampling can miss short events. Long video context creates retrieval and memory challenges.

### Production Practice

- Use scene segmentation and keyframe extraction.
- Combine OCR, ASR, and visual signals.
- Store timestamps for citations.
- Evaluate event recall separately from summary quality.

### Interview Self-Check

- Why is video harder than image understanding?
- How would you build video search?
- How do you cite evidence in video QA?

---

## 7. OCR and Document Understanding

### Concept

Document AI extracts text, layout, tables, forms, signatures, and semantic relationships from documents.

### Mechanism

Pipeline:

```text
document image / PDF
  -> OCR
  -> layout detection
  -> table / form extraction
  -> semantic parsing
  -> validation
```

Models may combine text tokens, bounding boxes, and visual features.

### Trade-off

General multimodal LLMs are flexible but may hallucinate details. Specialized OCR and layout models are often more reliable for high-precision extraction.

### Production Practice

- Preserve bounding boxes and page numbers.
- Validate amounts, dates, IDs, and totals deterministically.
- Use human review for low-confidence fields.
- Track extraction quality by document type.

### Interview Self-Check

- Why is layout important for document understanding?
- When should you use a specialized OCR model?
- How do you validate extracted financial fields?

---

## 8. Speech and Multimodal Interaction

### Concept

Speech systems convert between audio and text or directly model speech as part of multimodal interaction.

### Mechanism

Core tasks:

- ASR: speech to text,
- TTS: text to speech,
- speaker diarization,
- speech translation,
- audio event detection.

### Trade-off

ASR quality depends on noise, accent, language, microphone, and domain vocabulary. TTS quality must balance naturalness, latency, and safety.

### Production Practice

- Add domain vocabulary or post-processing.
- Track word error rate by segment.
- Use timestamps for searchable transcripts.
- Add consent and privacy controls for audio.

### Interview Self-Check

- What affects ASR accuracy?
- Why are timestamps useful?
- What safety issues exist in voice cloning?

---

## 9. Multimodal Training

### Concept

Multimodal training aligns representations across modalities and teaches instruction-following behavior over multimodal inputs.

### Mechanism

Common stages:

```text
modality encoders
  -> contrastive or caption alignment
  -> projector / adapter training
  -> multimodal instruction tuning
  -> safety alignment
```

### Trade-off

High-quality paired data is expensive. Synthetic data helps scale but may introduce bias or hallucinated labels.

### Production Practice

- Curate diverse paired datasets.
- Evaluate grounding, OCR, spatial reasoning, and refusal behavior.
- Separate pretraining, instruction tuning, and safety evaluation.
- Use domain-specific validation sets.

### Interview Self-Check

- What is contrastive learning in CLIP?
- Why is multimodal instruction tuning needed?
- How do you evaluate visual grounding?

---

## 10. Senior Interview Q&A

### Q1: How would you design a multimodal RAG system for PDFs with charts and tables?

**Answer**: Parse the document into text, layout, tables, figures, page numbers, and bounding boxes. Index text with hybrid retrieval and index images/tables with visual or layout-aware embeddings. At answer time, retrieve evidence, preserve page and region provenance, rerank across modalities, and require citations to source spans or visual regions. Validate numeric fields deterministically where possible.

### Q2: A vision-language model describes objects that are not in the image. How do you detect and reduce this?

**Answer**: Detect object hallucination with grounding checks, object detectors, region-level verification, POPE-style probes, and human review for high-risk domains. Reduce it with better visual grounding data, negative examples, constrained answer formats, confidence reporting, and abstention when visual evidence is weak.

### Q3: Video QA latency is too high. What levers do you have?

**Answer**: Use scene segmentation, keyframe sampling, OCR/ASR extraction, hierarchical summaries, visual token compression, caching, and two-stage retrieval. Evaluate event recall separately from final answer quality, because aggressive sampling may miss short but important events.

### Q4: When should you use a specialized OCR/layout model instead of a general multimodal LLM?

**Answer**: Use specialized OCR or layout models when exact fields, tables, totals, signatures, or compliance evidence matter. General multimodal LLMs are useful for flexible reasoning over extracted content, but deterministic validation should handle amounts, dates, IDs, and cross-field consistency.

---

## 11. Interview Self-Check

- Explain ViT from image patches to prediction.
- Compare CLIP, LLaVA, and BLIP-2 style architectures.
- Design a multimodal RAG system for PDFs with charts.
- Explain how to evaluate OCR and document extraction.
- Explain risks in image generation and mitigation methods.
- Design a video QA pipeline with timestamps and citations.
