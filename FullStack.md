
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

# ✅ User Flow (Line by Line, Very Simple)

1. User places all videos inside the folder: `input/videos/`.
2. User runs the batch command: `node processFolder.js`.
3. System scans the folder and finds all video files.
4. For each video, the system creates a processing job.
5. Jobs are added to the job queue (BullMQ).
6. Background worker picks the first job from the queue.
7. Worker loads the video and extracts metadata (duration, size).
8. Worker runs **Whisper** to generate the transcript.
9. Worker sends transcript to LLM to generate summary + highlights.
10. Worker uses FFmpeg to extract clips for each highlight.
11. Worker uses FFmpeg to take screenshots at highlight timestamps.
12. Worker builds Summary.md with all content + asset links.
13. Worker writes final output to `output/<video_name>/`.
14. Job is marked complete in the system.
15. User goes to the output folder, opens Summary.md, and views everything.
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

1️⃣ Minimal User Flow (5 Steps)

Connect LinkedIn
User logs in → platform stores OAuth token.

Define Persona & Tone
User selects writing style (Professional, Humorous, Founder-style, etc.)

Provide Topics/Ideas
User enters themes like “AI”, “Career Tips”, “Remote Work”.

Generate 3 Draft Posts
Backend → LLM generates 3 high-quality LinkedIn post drafts using persona + topics.

Approve → Schedule → Auto-Post
User picks one draft → selects date/time → system posts automatically using LinkedIn API.


2. High-Level Architecture (MERN + Worker + LLM + LinkedIn API)

┌──────────────────────────┐
│        React UI           │
│ (Connect → Persona → Drafts) │
└─────────────┬────────────┘
              │  HTTP
              ▼
     ┌────────────────────┐
     │     Express API     │
     │ - LinkedIn OAuth    │
     │ - Draft Generation  │
     │ - Approval Flow     │
     │ - Scheduling APIs   │
     └───────────┬────────┘
                 │
                 ▼
        ┌──────────────────────┐
        │       MongoDB         │
        │ Users, Personas,      │
        │ Drafts, Schedules,    │
        │ PostLogs              │
        └───────────┬──────────┘
                    │
                    ▼
    ┌────────────────────────────────┐
    │     LLM Service (OpenAI/Groq)  │
    │ - Post generation              │
    │ - Tone + persona enforcement   │
    └───────────────┬────────────────┘
                    │
                    ▼
       ┌──────────────────────────┐
       │  Background Scheduler     │
       │   (Node + BullMQ/Cron)   │
       │ - Polls due posts        │
       │ - Publishes via LinkedIn │
       └──────────────┬──────────┘
                      │
                      ▼
           ┌────────────────────┐
           │   LinkedIn API     │
           │  (Post Publishing) │
           └────────────────────┘
           
 3. Where to Store Prompts (Provided by GenAI Team)
 
 | Field     | Type     | Description                           |
| --------- | -------- | ------------------------------------- |
| _id       | ObjectId | Identifier                            |
| name      | String   | e.g., "founder_post", "career_advice" |
| template  | String   | Full prompt text with variables       |
| version   | Number   | Version control                       |
| createdAt | Date     | Timestamp                             |


4. Data Model
User
| Field               | Type     | Description |
| ------------------- | -------- | ----------- |
| _id                 | ObjectId | User ID     |
| name                | String   | User's name |
| email               | String   | Email       |
| linkedinAccessToken | String   | OAuth token |
| createdAt           | Date     | Timestamp   |

Persona
| Field          | Type     | Description                                    |
| -------------- | -------- | ---------------------------------------------- |
| _id            | ObjectId | Persona ID                                     |
| userId         | ObjectId | Linked to User                                 |
| tone           | String   | e.g., “professional”, “funny”, “founder-style” |
| writingStyle   | String   | Style description                              |
| targetAudience | String   | e.g., students, founders                       |
| createdAt      | Date     | Timestamp                                      |


Draft
| Field         | Type     | Description          |
| ------------- | -------- | -------------------- |
| _id           | ObjectId | Draft ID             |
| userId        | ObjectId | Owner                |
| personaId     | ObjectId | Used persona         |
| topic         | String   | Topic given by user  |
| draft1        | String   | Post draft 1         |
| draft2        | String   | Post draft 2         |
| draft3        | String   | Post draft 3         |
| approvedDraft | String   | Final selected draft |
| status        | String   | generated / approved |
| createdAt     | Date     | Timestamp            |

Schedule
| Field       | Type     | Description               |
| ----------- | -------- | ------------------------- |
| _id         | ObjectId | Schedule ID               |
| userId      | ObjectId | Linked to User            |
| draftId     | ObjectId | Approved draft            |
| scheduledAt | Date     | When to publish           |
| status      | String   | pending / posted / failed |

5.PostLog
| Field        | Type     | Description         |
| ------------ | -------- | ------------------- |
| _id          | ObjectId | Identifier          |
| userId       | ObjectId | User                |
| draftId      | ObjectId | Posted content      |
| postedAt     | Date     | Actual posting time |
| responseId   | String   | LinkedIn post ID    |
| status       | String   | success / error     |
| errorMessage | String   | If failed           |

5. Api endpoints
| Method   | Endpoint                | Description                  |
| -------- | ----------------------- | ---------------------------- |
| **GET**  | /api/auth/linkedin      | Redirect to LinkedIn OAuth   |
| **GET**  | /api/auth/callback      | Store access token           |
| **POST** | /api/persona            | Create/Update persona        |
| **GET**  | /api/persona            | Fetch persona                |
| **POST** | /api/drafts/generate    | Generate 3 drafts using LLM  |
| **GET**  | /api/drafts/:id         | Get drafts                   |
| **POST** | /api/drafts/:id/approve | Approve a draft              |
| **POST** | /api/schedule           | Schedule a post              |
| **GET**  | /api/schedule           | List scheduled posts         |
| **POST** | /api/post/run           | Force manual posting         |
| **POST** | /api/post/webhook       | Scheduler posts via LinkedIn |
| **GET**  | /api/logs               | Post logs                    |

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

1. User Flow (Step-by-step)
1. Upload Template

User selects a .docx file

System extracts placeholders → shows detected fields

2. Field Review

User sees:

field name

field type (text/date/number/options)

required?

validation rules

User edits/approves schema

3. Single Generate

User inputs values

System generates DOCX + PDF

Downloads instantly

4. Bulk Generate

User uploads CSV/Google Sheet

System validates rows

Creates bulk job

Worker processes rows one by one

Produces:

ZIP of all generated docs

Bulk report: success/fail reason

User downloads ZIP + report

2. High Level Design
   
 ┌────────────────────────┐
 │      React Frontend     │
 │ Upload → Review → Run   │
 └─────────────┬──────────┘
               │ REST API
               ▼
 ┌────────────────────────────────────┐
 │            Express API             │
 │ - Upload template                  │
 │ - Extract fields                   │
 │ - Validate CSV                     │
 │ - Create single/bulk jobs          │
 │ - Fetch results                    │
 └─────────────┬──────────────────────┘
               │
               ▼
 ┌────────────────────────────────────┐
 │             MongoDB                │
 │ Templates, Fields, BulkRuns,       │
 │ RowResult, Artifacts               │
 └─────────────┬──────────────────────┘
               │
               ▼
 ┌────────────────────────────────────┐
 │ Background Worker (BullMQ + Node)  │
 │ - Parse template                   │
 │ - Render DOCX                      │
 │ - Convert to PDF                   │
 │ - Validate CSV rows                │
 │ - Generate ZIP                     │
 │ - Save artifacts                   │
 └─────────────┬──────────────────────┘
               │
               ▼
 ┌────────────────────────────────────┐
 │       Local Storage (/data)        │
 │ template.docx                      │
 │ generated/<jobId>/<files>          │
 │ zip bundles, PDFs                  │
 └────────────────────────────────────┘

3.
QUEUED → VALIDATING_ROWS → PROCESSING_ROWS → GENERATING_ZIP → SUCCESS
     OR
                                     └──► FAILED (reason)

4.   Data Model (Database Schema)
   Template
{
  _id,
  name,
  filePath,
  uploadedBy,
  createdAt
}

TemplateField
{
  _id,
  templateId,
  fieldName,              // {{name}}
  type,                   // text, number, date, enum
  required,               // boolean
  validation,             // regex, min/max
  options                 // for enum
}

BulkRun
{
  _id,
  templateId,
  status,                 // queued, running, success, failed
  totalRows,
  successCount,
  failedCount,
  zipFilePath,
  reportFilePath,
  createdAt
}

{
  _id,
  bulkRunId,
  rowNumber,
  status,                 // success / failed
  reason,                 // null or error
  outputDocPath,
  outputPdfPath
}

5. API Endpoints (10 endpoints)
Template + Fields

POST /template/upload — upload .docx

GET /template/:id/fields — get detected fields

PUT /template/:id/fields — update field schema

Single Generate

POST /generate/single/:templateId — returns docx/pdf

GET /artifact/:id — download file

Bulk Generate

POST /generate/bulk/:templateId — upload CSV → create job

GET /bulk/:id/status — job progress

GET /bulk/:id/zip — download ZIP

GET /bulk/:id/report — download report

DELETE /bulk/:id — cleanup
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

1. Data Model
Character
{
  _id,
  name,
  description,            // personality, traits
  imageRef,               // uploaded image or generated base look
  voiceStyle,             // voice tone, accent (optional)
  styleGuide,             // clothing, color palette
  createdAt
}
Relationship
{
  _id,
  characterA,             // reference to Character
  characterB,             // reference to Character
  relationshipType,       // friends, rivals, mentor, etc.
  notes
}
Episode
{
  _id,
  title,
  storyPrompt,            // user provided short story
  characters,             // selected characters for this episode
  overallSummary,
  status,                 // draft, processing, complete
  createdAt
}
Scene
{
  _id,
  episodeId,
  sceneNumber,
  description,
  dialogues,              // ordered per character
  assetPlanIds,           // list of Asset documents
  voicePlan,              // lines + voice metadata
}
Asset
{
  _id,
  episodeId,
  sceneId,
  type,                   // image / prop / background / motion element
  prompt,                 // generation prompt
  generatedRef,           // link to generated asset
}

2. Pipeline Flow
Step 1 — Story → Episode Setup

User inputs a short story/situation

System:

understands the plot

identifies the characters involved

ensures tone/style matches character definitions

Step 2 — Scene Breakdown

LLM converts story into a structured sequence:

Scene 1: Setup
Scene 2: Conflict
Scene 3: Resolution

Each scene contains:

setting

motivation & intention

emotional beats

camera style (optional)

Step 3 — Dialogues Generation

For each scene:

LLM generates character-specific dialogues

Keeps tone, personality, relationships consistent

Builds a voiceover plan:

character: A
line: "Let's move quickly."
emotion: urgent
voiceStyle: deep calm tone
Step 4 — Asset Plan Generation

For each scene:

background description

props required

character poses

camera framing

transitions

The system reuses existing character base images to keep consistency

Output example:

Scene 2 Assets:
- Background: "dark alley futuristic"
- Pose: "Character A - angry expression"
- Prop: "digital tablet glowing"
Step 5 — Render Plan

Defines the final output:

Which frames to generate

Which images can be reused

Where to animate or pan

Voiceover timing

(For V1) Export as slideshow video with transitions

(Future) Lip-sync + animation

3. Consistency Strategy
A. Character Memory

Store:

personality traits

speaking style examples

does/does-not behaviors

visual style rules
LLM sees a “Character Bible” for every episode.

B. Style Guide

fixed color palette

clothing style

silhouette

face identity embedding (for image model)
Ensures visual consistency across all episodes.

C. Asset Reuse

Assets generated in Episode 1 can be:

reused in Episode 2

reposed

recolored

background extended
This drastically reduces cost and ensures consistency.

D. Relationship Memory

Used to shape dialogues:

If A is mentor of B → A’s tone = guiding, B’s tone = learning
If rivals → sarcastic or competitive tone

4. MVP Scope vs V1 Scope
MVP (What we ship first)

Focus on script + static assets + voice plan.

✔ Character creation (name + image + personality)
✔ Relationship definitions
✔ Input story → scene breakdown
✔ Dialogues generation
✔ Asset prompts (NOT actual video rendering)
✔ Basic voiceover plan
✔ JSON + Markdown episode package
✔ Light-weight image generation (one image per scene)

Output:
A structured episode package folder:


episode-01/
  script.md
  scenes.json
  assets/
    scene1.png
    scene2.png
  voice_plan.json
  
You need to put your solution here.
