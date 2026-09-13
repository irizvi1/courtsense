# Success Criteria

Committed 2026-09-13, before any results were produced.

These thresholds are set in advance so that outcomes cannot be
rationalized after the fact. Missing them is a finding to be
explained, not a failure to be hidden.

All final numbers come from Clip B, the held-out clip, on a single
run. Clip A numbers are development only and are reported separately.

## Week 1 gate: ball detectability

Measured with pretrained YOLOv8n on shot windows only.

- Proceed as planned: ball detected in >60% of sampled frames
- Proceed, fine-tuning is load-bearing: 30-60%
- Source better footage or pivot: <30%, or median ball width <10px

## Shot attempt detection

- Minimum viable: recall >= 0.75, precision >= 0.70
- Good: recall >= 0.80, precision >= 0.80
- Excellent: recall >= 0.90, precision >= 0.88

Precision and recall trade off directly against the detection
confidence threshold, so both are reported together with F1 and
the threshold used.

## Make/miss classification

- Minimum viable: >= 72% accuracy
- Good: >= 80% accuracy
- Excellent: >= 88% accuracy

Rec league field goal percentage is roughly 35-45%, so always
predicting "miss" scores about 58-65% while doing nothing. Accuracy
is reported alongside that majority-class baseline and a confusion
matrix, since raw accuracy hides a model that gets misses right and
makes wrong.

## Team assignment

- Minimum viable: >= 80% accuracy
- Good: >= 85% accuracy
- Excellent: >= 92% accuracy

## Inference latency

Reported as a ratio, since absolute seconds depend on hardware and
frame sampling rate.

- Minimum viable: 1.8x speedup, PyTorch FP32 to ONNX int8
- Good: 2.5x
- Excellent: 3.5x

Quantization's cost in detection accuracy is measured and reported.
A speedup with an unmeasured accuracy cost is not a result.

## Out of scope for v1

Court line detection, 2PT/3PT classification, team score, per-player
statistics, real-time processing, multi-camera input. Each is
excluded for a stated technical reason in the README.
