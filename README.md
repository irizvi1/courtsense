<!-- FILL-IN CHECKLIST before making this repo public:
     search for [[ and replace every placeholder with a real number or delete the section.
     Do not publish with any [[ ]] left in the file. -->

# CourtSense

Team-level shooting statistics from a handheld phone recording of an amateur basketball game.

**[[live demo]]** · **[[annotated sample output]]**

---

## The problem

Every commercial basketball analysis system solves the hard vision problem by controlling the camera. Veo, Hudl, Pixellot, and Trace all require an elevated, wide-angle, full-court view from dedicated hardware costing $1,000 or more. HomeCourt runs on a phone but only tracks solo shooting drills, one player at a time.

Nobody handles the footage that actually exists for rec league and high school games: one person in the bleachers with a phone, panning, at an angle, with players occluding each other constantly. This project targets that case specifically and reports honestly on how well it works.

## Results

All numbers below come from **Clip B**, an [[N]]-minute rec league clip that was annotated before development began and not opened until the final week. No tuning was done against it.

| Metric | Result |
|---|---|
| Shot attempt detection, recall | [[0.XX]] |
| Shot attempt detection, precision | [[0.XX]] |
| Shot attempt detection, F1 | [[0.XX]] |
| Make/miss classification accuracy | [[XX%]] |
| Team assignment accuracy | [[XX%]] |
| Processing time per minute of 720p video | [[XX]]s (CPU, ONNX int8) |

Reported team totals against ground truth:

| | Team A attempts | Team A makes | Team B attempts | Team B makes |
|---|---|---|---|---|
| Ground truth | [[X]] | [[X]] | [[X]] | [[X]] |
| System output | [[X]] | [[X]] | [[X]] | [[X]] |

Development numbers on Clip A were [[higher / comparable]], which is expected and is why Clip B exists.

### Which signal made each call

Every shot decision is attributed to the signal that produced it, stored in `shot_events.decided_by`:

| Signal | Share of decisions | Accuracy when used |
|---|---|---|
| Direct rim observation | [[XX%]] | [[XX%]] |
| Trajectory extrapolation | [[XX%]] | [[XX%]] |
| Game-context motion | [[XX%]] | [[XX%]] |
| Consensus | [[XX%]] | [[XX%]] |

[[One or two sentences on what this table shows. For example: trajectory extrapolation carries most of the load because the ball is occluded through the rim on the majority of makes.]]

## Scope

**In scope:** total shot attempts, total makes and misses, per-team attempts, per-team makes, per-team shooting percentage.

**Deliberately out of scope**, each for a technical reason:

- **Court lines, 2PT vs 3PT, and score.** Fitting a homography requires a stable view of known court geometry. A panning handheld camera changes the homography every frame and typically has half the court out of view, so per-frame estimates are unreliable and the errors compound into shot classification. Scoring 2 vs 3 incorrectly is worse than not reporting it.
- **Per-player statistics.** Points, assists, and rebounds all require persistent player identity, which on this footage means jersey number OCR at roughly twenty pixels of text height under motion blur. That is a research problem, not an engineering one.
- **Real-time processing.** Batch only. A clip is uploaded and processed asynchronously.
- **Multi-camera or calibrated views.** The premise of the project is that neither is available.

## How it works

```
[Browser]
    |
    | 1. request presigned upload URL
    v
[FastAPI on App Runner] --> [S3: raw video]
    |                            ^
    | 2. enqueue job             |
    v                            |
[SQS queue + DLQ]                |
    |                            |
    | 3. poll                    |
    v                            |
[Worker on ECS Fargate] ---------+
    |
    | 4. write results + artifacts
    v
[RDS Postgres]  +  [S3: annotated video]
    ^
    | 5. poll status, fetch results
    |
[Browser]
```

Processing a ten-minute clip takes minutes, not milliseconds, so an HTTP request cannot hold it open. The API accepts the job, writes a row, pushes to SQS, and returns immediately. A separate worker service polls the queue and runs the pipeline. This means the API stays responsive while a worker is pinned at full CPU, and the two scale independently.

Because SQS delivers at least once, the worker is idempotent: it checks job status before processing and treats a re-delivered message as a no-op. Visibility timeout is set above maximum observed processing time so a job is not re-delivered mid-run, and repeated failures land in a dead letter queue instead of looping.

Video never passes through the API. The browser uploads directly to S3 with a presigned URL.

### Pipeline

1. **Detection.** YOLOv8 fine-tuned on [[N]] annotated frames from amateur game footage. Classes: player, ball, rim, backboard.
2. **Tracking.** ByteTrack maintains identity across frames so consecutive shots are not merged and a single shot is not double counted. Camera pans break tracks, and ID switches are handled explicitly.
3. **Team assignment.** Player crops are reduced to an HSV color histogram and clustered with KMeans into three groups. The smallest group is dropped, which removes referees.
4. **Trajectory model.** The ball is occluded behind the rim, backboard, and players on most shots at the moment of truth. Rather than guess, the system fits a parabola to the tracked ball positions before occlusion and extrapolates to the rim plane, classifying the outcome from the predicted entry point. [[Describe whether this is the geometric fit or the learned classifier over trajectory features.]]
5. **Game-context signal.** After a made basket, play stops and restarts from the baseline: all ten players move toward one end, there is a pause, and the ball re-enters from out of bounds. After a miss, play continues. Aggregating player motion over the four seconds following a rim event gives a third independent vote. [[Include only if built. Delete this section otherwise.]]
6. **Event assembly.** The three signals are combined into one decision per shot with a confidence score, and the deciding signal is recorded.

## Evaluation methodology

Two clips of amateur game footage were annotated by hand, shot by shot, with timestamp, team, and outcome.

**Clip A** was the development clip and was examined continuously while building. **Clip B** was annotated in advance and then not opened until the pipeline was frozen. Every number in the Results section comes from Clip B on a single run. This matters because a held-out set that gets peeked at during development stops being held out, and most portfolio projects report numbers from the data they tuned against.

Ground truth CSVs for both clips are in [`eval/ground_truth/`](eval/ground_truth/). The evaluation script is [`eval/run_eval.py`](eval/run_eval.py).

## Where it fails

[[Write this as prose with real frequencies, three or four categories, and screenshots. Suggested structure per category: what happens, how often it happened on Clip B, why, and what you would do about it. Categories to check:

- ball fully occluded through the rim
- camera pan loses the rim entirely
- two shots within a few seconds counted as one
- team assignment flips when a player is backlit
- rebound put-back counted as a separate attempt
- referee clustered as a player

Include annotated frames in docs/failures/ and embed them here.]]

## Performance

Inference runs on CPU. The model is converted to ONNX and quantized to int8.

| Model | Time per minute of 720p video |
|---|---|
| YOLOv8n, PyTorch FP32 | [[XXX]]s |
| YOLOv8n, ONNX int8 | [[XX]]s |

[[One sentence on the accuracy cost of quantization, if any.]]

## Stack

Python 3.11, PyTorch, Ultralytics YOLOv8, ByteTrack, OpenCV, ONNX Runtime, FastAPI, Pydantic, SQLAlchemy, Alembic, PostgreSQL. AWS: S3, SQS, App Runner, ECS Fargate, RDS, ECR, CloudWatch, Secrets Manager. Frontend: React, Vite, Tailwind.

## Running locally

```bash
git clone [[repo url]]
cd [[repo]]
cp .env.example .env          # set DB and AWS values
docker compose up --build     # api on :8000, worker, postgres
alembic upgrade head
```

Process a clip without the queue:

```bash
python -m pipeline.run --video path/to/clip.mp4 --out results.json
```

Run the evaluation against a ground truth file:

```bash
python eval/run_eval.py --pred results.json --truth eval/ground_truth/clip_b.csv
```

## API

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/jobs` | Create a job, returns `job_id` and a presigned S3 upload URL |
| GET | `/api/jobs/{id}` | Status: `queued`, `processing`, `done`, `failed` |
| GET | `/api/jobs/{id}/results` | Statistics JSON |
| GET | `/api/jobs/{id}/video` | Presigned URL for the annotated output video |
| GET | `/health` | Health check |

## Future work

Court line detection via a court-keypoint model rather than a Hough transform would make 2PT and 3PT classification viable and is the most valuable missing feature. Per-player statistics would need a jersey number model trained on low-resolution crops. Both are additive: `shot_events` already has room for the columns.

## License

[[MIT, or whatever you pick]]
