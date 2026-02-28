
# **Full-Stack Developer**

**Evaluation Criteria**

* Clarity + practicality of architecture
* Clean data model + API design
* Async job + storage thinking (uploads, outputs, retries)
* Security basics (auth, access control, safe downloads)
* Cost + scalability tradeoffs (MVP → v1)
* Ability to handle ambiguity + user review flows

---

## **Problem 1:** **Video-to-Notes Platform (Architecture Proposal)**

We have long videos (3–4 hours, 200MB+). We need an automated “summary package” per video: **Summary.md + highlights (timestamps) + screenshots/clips references**, organized per video. [READ MORE ABOUT THE PROJECT](./Video-summary-platform.md)

**Task (No code)**

Create a concise architecture proposal for an MVP.

**Your Solution must include**

* Minimal user flow (3–5 steps)
* High-level architecture diagram (UI, API, DB, worker/queue, storage, AI)
* Job lifecycle (queued → processing → success/failed) + progress reporting
* Data model (tables/entities only)
* API list (8–12 endpoints)

**Your Solution for problem 1:**

You need to put your solution here.

# ✅ **User Flow (Line by Line, Very Simple)**

1. User places all videos inside the folder: **`input/videos/`**.
2. User runs the batch command: **`node processFolder.js`**.
3. System scans the folder and finds all video files.
4. For each video, the system **creates a processing job**.
5. Jobs are added to the **job queue (BullMQ)**.
6. Background worker picks the first job from the queue.
7. Worker loads the video and extracts metadata (duration, size).
8. Worker runs **Whisper** to generate the transcript.
9. Worker sends transcript to **LLM** to generate summary + highlights.
10. Worker uses **FFmpeg** to extract clips for each highlight.
11. Worker uses **FFmpeg** to take screenshots at highlight timestamps.
12. Worker builds **Summary.md** with all content + asset links.
13. Worker writes final output to **`output/<video_name>/`**.
14. Job is marked **complete** in the system.
15. User goes to the **output folder**, opens **Summary.md**, and views everything.
---

High level Design

 ```text
High level Design

 ┌─────────────────────────────┐
 │        input/videos/        │
 │     Local video folder      │
 └─────────────────────────────┘
              │
              ▼
 ┌─────────────────────────────┐
 │     Batch Controller        │
 │      processFolder.js       │
 │       - Create jobs         │
 └─────────────────────────────┘
              │
              ▼
 ┌─────────────────────────────┐
 │     Job Queue (BullMQ)      │
 │  video_processing_queue     │
 └─────────────────────────────┘
              │
              ▼
 ┌─────────────────────────────┐
 │   Worker (Node.js)          │
 │  - FFmpeg                   │
 │  - Whisper                  │
 │  - AI Summary               │
 │  - Screenshots              │
 └─────────────────────────────┘
              │
              ▼
 ┌─────────────────────────────┐
 │       output/<video>/       │
 │  Summary.md + assets/       │
 └─────────────────────────────┘
```

 JOB LIFECYCLE (Video → Notes Platform)

(Queued → Processing → Success / Failed)

1. Job Created

User uploads a video or system scans the folder.

Backend creates a job record in DB:

videoPath

status = "queued"

progress = 0%

jobId generated

Job is pushed into the BullMQ queue.

2. Queued

Job sits in the queue waiting for a worker.

Multiple jobs can be queued at once.

No CPU work happens here.

Status remains: queued.

3. Worker Picks Job

Background worker listens to the queue.

When a job is available:

Worker “locks” the job (only one worker can process it).

Worker updates status: processing.

Worker updates progress: 5%.

4. Processing Stage 1 — Load & Prepare

Worker loads the video file.

Extracts metadata: duration, resolution, size.

Creates output folder structure:

/output/<video_name>/

Progress → 10%.

5. Processing Stage 2 — Transcription (Whisper)

Worker runs Whisper (OpenAI / local whisper.cpp).

Generates full transcript (can take 10–40 min for long videos).

Transcript saved to:

/output/<video_name>/transcript.txt

Progress → 60%.

6. Processing Stage 3 — Summary + Highlights (LLM)

Worker sends transcript chunks to LLM.

LLM returns:

high-level summary

highlight bullets with timestamps

key takeaways

chapter breakdown

Worker saves as JSON.

Progress → 75%.

7. Processing Stage 4 — Screenshots

For each highlight timestamp:

worker runs FFmpeg to capture a frame

Stored at:

screenshots/shot_1.png etc

Progress → 90%.

8. Processing Stage 5 — Clips

Worker cuts short video clips around highlight timestamps

example: 10-second clip around each timestamp

Stored at:

clips/clip_1.mp4

Progress → 95%.

9. Processing Stage 6 — Build Summary.md

Worker composes Markdown:

metadata

summary

highlights

links to clips & screenshots

Writes to:

/output/<video_name>/Summary.md

Progress → 100%.

10. Success

Worker updates job in DB:

status = "success"

progress = 100%

completedAt = timestamp

Queue marks job as completed.

11. Failed

If any error happens:

status = "failed"

errorMessage = stacktrace

Saved in logs for debugging.

User can retry using a “retry job” endpoint.

1. VideoJob

Represents each video processed.

VideoJob {
  _id: ObjectId,
  jobId: String,               // queue job id
  videoName: String,           // original filename
  videoPath: String,           // local path
  outputPath: String,          // /output/<videoName>/
  status: "queued" | "processing" | "success" | "failed",
  progress: Number,            // 0–100
  duration: Number,            // seconds
  createdAt: Date,
  updatedAt: Date,
  errorMessage: String | null
}
2. Transcript

Stores full Whisper transcription.

Transcript {
  _id: ObjectId,
  jobId: String,
  text: String,
  createdAt: Date
}
3. SummaryMeta
| Field      | Type     | Description               |
| ---------- | -------- | ------------------------- |
| _id        | ObjectId | Primary key               |
| jobId      | String   | Reference to VideoJob     |
| summary    | String   | Short summary paragraph   |
| highlights | Array    | List of highlight objects |
| takeaways  | Array    | Key bullet points         |
| createdAt  | Date     | Timestamp                 |


4. AssetIndex

Index of all generated assets per video.

AssetIndex {
  _id: ObjectId,
  jobId: String,
  screenshots: [String],  // list of screenshot paths
  clips: [String],        // list of clip paths
  summaryFile: String     // /output/.../Summary.md
}

5. Api End Points

| Method          | Endpoint                    | Description                          |
| --------------- | --------------------------- | ------------------------------------ |
| POST            | /api/videos/upload          | Upload video and create job          |
| POST            | /api/videos/submit          | Submit local video path              |
| GET             | /api/jobs                   | List all jobs                        |
| GET             | /api/jobs/:jobId            | Get job details                      |
| GET             | /api/jobs/:jobId/progress   | Get job progress only                |
| POST            | /api/jobs/:jobId/retry      | Retry failed job                     |
| GET             | /api/jobs/:jobId/transcript | Get transcript                       |
| GET             | /api/jobs/:jobId/summary    | Get summary + highlights             |
| GET             | /api/jobs/:jobId/assets     | Get list of assets                   |
| GET             | /api/jobs/:jobId/download   | Download ZIP output                  |
| DELETE          | /api/jobs/:jobId            | Delete job and files                 |
| POST (optional) | /api/admin/scan-folder      | Create jobs for all videos in folder |

---

## **Problem 2:** **LinkedIn Automation Platform (Architecture + Prompt Spec)**

User connects LinkedIn, defines persona/tone, provides topics. System generates **3 post drafts**, user approves, schedules, and posts automatically. [READ MORE ABOUT THE PROJECT](./linkedin-automation.md)

**Task (No code)**

Provide:

1. **Architecture proposal** for MVP (auth, scheduling, approvals, posting).
2. How to store prompts provide by GenAI team

**Your Solution must include**

* Minimal flow: connect → persona → generate → approve → schedule → post
* Architecture blocks + key integrations (LinkedIn, LLM, scheduler)
* Data model (User, Persona, Draft, Schedule, PostLog)
* API list (8–12 endpoints)

**Your Solution for problem 2:**

You need to put your solution here.

---

## **Problem 3:** **DOCX Template → Bulk DOCX/PDF Generator Architecture**

Users upload Word templates, system detects editable fields, supports **single fill** and **bulk generation via CSV/Sheet**, exports DOCX/PDF, provides ZIP + report. [READ MORE ABOUT THE PROJECT](./docs-template-output-generation.md)

**Task (No code)**

Provide MVP architecture + LLM prompt spec for:

* Template field detection
* Field schema generation (types, required, validations)

**Your Solution must include**

* Flow: upload template → field review → single generate → bulk generate
* Architecture blocks (template parser, worker, storage, export service)
* Data model (Template, TemplateField, BulkRun, RowResult, Artifact)
* Bulk report format (success/fail + reason)

**Your Solution for problem 3:**

You need to put your solution here.

---

## **Problem 4:** **Character-Based Video Series Generator (Architecture Proposal)**

User defines characters once (image + personality + relationships). For each episode, user provides a short story/situation. System outputs an “episode package” (script, scenes, assets list, voiceover plan) and optionally a final video.  [READ MORE ABOUT THE PROJECT](./char-based-video-generation.md)

**Task (No code)**

Create a small architecture proposal for MVP.

**Your Solution must include**

* Data model for Character, Relationship, Episode, Scene, Asset
* Pipeline flow: story → scene breakdown → dialogues → asset plan → render plan
* Consistency strategy (character memory, style guide, asset reuse)
* MVP scope vs v1 scope (what you would ship first)

**Your Solution for problem 4:**

You need to put your solution here.
