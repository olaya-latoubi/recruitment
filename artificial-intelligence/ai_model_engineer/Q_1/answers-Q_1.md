
**Q1.1 — Re-identification for selective anonymization**
YOLO already gives you the face crops. You don't need another full-frame pass just run each crop through a lightweight embedding model, compare against a pre-enrolled gallery with cosine similarity, and decide: known identity that should stay visible, or anonymize.
MobileNetV3-Small is a reasonable starting point. The gallery stores embeddings, not raw images, which reduces storage. That said, face embeddings are still biometric data they need proper access control and retention policies, not just a folder on a server.
Latency depends on hardware. Benchmark it before assuming it's fine.

---

**Q1.2 — Depth-based background blurring**

Run a monocular depth estimator to get a per-pixel depth map, threshold it to separate foreground from background, blur the background, keep the foreground. MiDaS is a reasonable starting point.
You don't need to run depth estimation every frame. Every N frames with temporal smoothing between estimates works fine  what N should be depends on the target hardware and how much flicker is acceptable. Benchmark it.
If the hardware supports parallel execution, depth estimation and face detection can run simultaneously. And face bounding boxes from the detector should always override the depth mask — if the depth model misclassifies a face as background, the detector catches it.

---

**Q1.3 — Improving accuracy on dark skin tones and bald heads**

Before changing anything, I'd do a structured error analysis. Break down detection performance by skin tone, hair coverage, illumination, pose, occlusion. Precision, recall, false-negative rate per condition. Without that baseline you're guessing what to fix.
If specific conditions are underrepresented in training, add data for those conditions specifically and augment — illumination, exposure, contrast, color shifts. Targeted, not random.
The thing people miss: overall mAP going up is not enough. If false-negative rate on dark skin tones stays high, the model got worse on the thing that actually matters. Measure subgroup performance separately, every time.

---

**Q1.4 — Added bald data, accuracy still not improving**

More data doesn't automatically fix anything. A few places to look:
Annotation quality first. Bald heads at difficult angles often get inconsistent bounding boxes. Check the new annotations, measure annotator agreement where possible.
If the new examples are still a tiny fraction of the full dataset, they won't move the needle. Test targeted oversampling or loss reweighting.
Distribution mismatch is easy to miss. High-quality frontal bald-head images won't help if failures happen in low-light side-view conditions. Check whether the new data actually matches where the model fails.
Also verify the basics before assuming it's a data problem: is the new data actually in the training split? Are labels and preprocessing consistent? Is there subject-level overlap between training and validation? I've seen all of these cause exactly this symptom.
Start with error analysis on the new failed examples before touching the training strategy.

---

**Q1.5 — MLOps infrastructure**
This system processes biometric data, so the infrastructure needs to answer one question at any point: what exactly ran in production on this date, and why.
DVC for dataset versioning, MLflow for experiment tracking and model registry. Every training run ties to a specific data snapshot. That's the audit trail.
Docker for deployment. Detector and ReID model in separate containers  not for elegance, but because they change at different rates. The detector gets retrained when new edge cases appear; the ReID model changes when the gallery or embedding space changes. Coupling them means redeploying everything when you only needed to touch one. Kubernetes and Triton only if the scale or latency actually requires them.
Prometheus and Grafana for infrastructure monitoring. For drift, track detection confidence distributions on live frames over time  a shift there usually shows up before failures become visible in downstream metrics.
Every production model needs a traceable record: code version, data version, training config, disaggregated evaluation results. Build that into the pipeline from day one. Reconstructing it retroactively for an audit is painful.

---
**Q1.6 — End-user experience**
The core problem with most anonymization systems is that users can't see what the system is doing. If a face gets missed and nobody flagged it, nobody knows until it's a compliance issue.
Real-time preview matters for that reason. The blur needs to be visible live, not in a review queue after the fact.
Adjustable blur strength because different contexts need different levels. A broadcaster and an internal security team don't have the same requirements. Operators should configure this without touching code.
When confidence is low — bad angle, occlusion, poor lighting — the system should say so visibly. Passing a difficult frame through silently is the worst outcome. "Processed with low confidence" is useful. Silent success on a frame the model wasn't sure about is not.
If selective anonymization is in use, gallery management needs to be self-service for authorized operators. If every identity change requires the ML team, the feature stops being used.
Frames with heavy occlusion or degraded quality should be flagged for human review, not treated as successfully anonymized. The system should be honest about what it handled and what it didn't.
