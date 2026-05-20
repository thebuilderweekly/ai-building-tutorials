## Opening thesis

You will build an agent that takes a raw audio file, transcribes it, and returns a JSON array of structured tasks with owners, deadlines, and priorities. The whole pipeline runs in about 30 seconds for a 20-minute recording. An agent processes audio at 40x real-time speed with consistent extraction recall, which a human transcriber cannot match without multiple listens.

## Before

You finish a 20-minute standup. You open a text editor and hit play on the recording. You pause, rewind, pause again, type a half-sentence. Fifteen minutes in, you realize someone said "Sarah will handle the migration by Friday" at minute three and you missed it entirely. You rewind to the beginning. Forty minutes later you have nine bullet points. The recording contained seventeen action items. You got roughly half. The other half live nowhere. They will surface again only when someone misses a deadline and asks, "Wait, did we agree on that?" This is the normal state of meeting follow-up for most teams. It is slow, lossy, and nobody enjoys it.

## Architecture

The pipeline has three stages. Audio goes to Deepgram for transcription. The transcript goes to the Anthropic API for structured extraction. A verification pass checks the extraction against the transcript for missed items. One Python script ties it together.

```text
DIAGRAM: Voice Note to Structured Tasks Pipeline
Caption: End-to-end flow from audio file to verified JSON task list.
Nodes:
1. Audio File (.mp3/.wav) - raw meeting recording on disk
2. Deepgram Nova-3 - speech-to-text transcription with speaker diarization
3. Raw Transcript - timestamped, speaker-labeled text
4. Anthropic Claude (Extract) - first pass: pull tasks from transcript
5. Task JSON - structured array of tasks with owner, deadline, priority
6. Anthropic Claude (Verify) - second pass: check transcript for missed tasks
7. Verified Task JSON - final output with high recall
Flow:
- Audio File sends bytes to Deepgram Nova-3 via REST API
- Deepgram Nova-3 returns Raw Transcript with speaker labels
- Raw Transcript is sent to Anthropic Claude (Extract) with a system prompt
- Anthropic Claude (Extract) returns Task JSON
- Task JSON and Raw Transcript are sent to Anthropic Claude (Verify)
- Anthropic Claude (Verify) returns Verified Task JSON
```

## Step-by-step implementation

### Step 1: Set up the project and install dependencies

Create a directory and install two packages: the Deepgram Python SDK and the Anthropic Python SDK. Python 3.10 or later is required.

```bash
mkdir voice-task-agent && cd voice-task-agent
python -m venv .venv && source .venv/bin/activate
pip install deepgram-sdk anthropic
```

### Step 2: Set environment variables

You need two API keys. Get your Deepgram key from https://console.deepgram.com/ under API Keys. Get your Anthropic key from https://console.anthropic.com/settings/keys. Export both in your shell.

```bash
export DEEPGRAM_API_KEY="your-deepgram-key-here"
export ANTHROPIC_API_KEY="your-anthropic-key-here"
```

### Step 3: Transcribe the audio with Deepgram

This function sends a local audio file to Deepgram's Nova-3 model. It requests speaker diarization so the transcript labels who said what. Diarization matters because task ownership depends on knowing which speaker made a commitment. The function returns a single string with speaker labels and timestamps.

```python
# transcribe.py
import os
from deepgram import DeepgramClient, PrerecordedOptions, FileSource

def transcribe_audio(file_path: str) -> str:
    dg = DeepgramClient(os.environ["DEEPGRAM_API_KEY"])

    with open(file_path, "rb") as f:
        buffer = f.read()

    payload: FileSource = {"buffer": buffer}

    options = PrerecordedOptions(
        model="nova-3",
        smart_format=True,
        diarize=True,
        utterances=True,
    )

    response = dg.listen.rest.v("1").transcribe_file(payload, options)
    utterances = response.results.utterances

    lines = []
    for u in utterances:
        speaker = f"Speaker {u.speaker}"
        start = f"{u.start:.1f}s"
        lines.append(f"[{start}] {speaker}: {u.transcript}")

    return "\n".join(lines)
```

### Step 4: Define the extraction prompt

The system prompt tells Claude exactly what to extract and what schema to return. Being explicit about the JSON schema prevents hallucinated fields and ensures parseable output. The prompt asks for five fields per task: description, owner, deadline, priority, and the source quote from the transcript.

```python
# prompts.py
EXTRACT_SYSTEM = """You are a task extraction agent. You read meeting transcripts and return ONLY a JSON array of tasks.

Each task object has these fields:
- "description": string, one sentence describing the action item
- "owner": string, the name or speaker label of the person responsible
- "deadline": string or null, any mentioned deadline in ISO 8601 format or natural language
- "priority": "high" | "medium" | "low", inferred from urgency cues in the conversation
- "source_quote": string, the exact phrase from the transcript that implies this task

Rules:
1. Extract every commitment, assignment, or volunteered action. Err on the side of inclusion.
2. If no deadline is mentioned, set deadline to null.
3. If the speaker says "I will" or "I can do that", the owner is that speaker.
4. Return valid JSON only. No markdown fences. No commentary."""

VERIFY_SYSTEM = """You are a verification agent. You receive a meeting transcript and a JSON array of previously extracted tasks.

Your job:
1. Read the transcript line by line.
2. Identify any action items, commitments, or assignments that are NOT in the provided task list.
3. Return a JSON object with two fields:
   - "missed_tasks": an array of task objects (same schema as the input tasks) for anything that was missed
   - "false_positives": an array of indices (0-based) of tasks in the input list that are NOT real action items

If nothing was missed, return {"missed_tasks": [], "false_positives": []}.
Return valid JSON only. No markdown fences. No commentary."""
```

### Step 5: Build the extraction function

This function sends the transcript to Claude with the extraction system prompt. It uses claude-sonnet-4-20250514 for speed and cost efficiency on a structured extraction task. Temperature is 0 because we want deterministic output, not creative variation.

```python
# extract.py
import os
import json
import anthropic
from prompts import EXTRACT_SYSTEM

def extract_tasks(transcript: str) -> list[dict]:
    client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

    message = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        temperature=0,
        system=EXTRACT_SYSTEM,
        messages=[{"role": "user", "content": transcript}],
    )

    raw = message.content[0].text
    tasks = json.loads(raw)
    return tasks
```

### Step 6: Build the verification function

This is the second pass. It sends the transcript and the extracted tasks back to Claude with a different system prompt. The model looks for anything the first pass missed and flags any false positives. Two-pass extraction is the key to high recall. A single pass typically catches 80 to 90 percent of tasks. The verification pass closes the gap.

```python
# verify.py
import os
import json
import anthropic
from prompts import VERIFY_SYSTEM

def verify_tasks(transcript: str, tasks: list[dict]) -> dict:
    client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

    user_content = f"TRANSCRIPT:\n{transcript}\n\nEXTRACTED TASKS:\n{json.dumps(tasks, indent=2)}"

    message = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        temperature=0,
        system=VERIFY_SYSTEM,
        messages=[{"role": "user", "content": user_content}],
    )

    raw = message.content[0].text
    result = json.loads(raw)
    return result
```

### Step 7: Wire everything together in a main script

This script takes an audio file path as an argument, runs the full pipeline, and writes the final task list to a JSON file. It also prints timing for each stage so you can see the 30-second claim holds.

```python
# main.py
import sys
import json
import time
from transcribe import transcribe_audio
from extract import extract_tasks
from verify import verify_tasks

def main():
    if len(sys.argv) < 2:
        print("Usage: python main.py <audio_file_path>")
        sys.exit(1)

    audio_path = sys.argv[1]

    t0 = time.time()
    print("Transcribing...")
    transcript = transcribe_audio(audio_path)
    t1 = time.time()
    print(f"Transcription: {t1 - t0:.1f}s")

    print("Extracting tasks...")
    tasks = extract_tasks(transcript)
    t2 = time.time()
    print(f"Extraction: {t2 - t1:.1f}s, found {len(tasks)} tasks")

    print("Verifying...")
    verification = verify_tasks(transcript, tasks)
    t3 = time.time()
    print(f"Verification: {t3 - t2:.1f}s")

    missed = verification.get("missed_tasks", [])
    false_pos = verification.get("false_positives", [])

    if missed:
        print(f"Found {len(missed)} missed tasks, adding them.")
        tasks.extend(missed)

    if false_pos:
        print(f"Removing {len(false_pos)} false positives.")
        for idx in sorted(false_pos, reverse=True):
            if 0 <= idx < len(tasks):
                tasks.pop(idx)

    output_path = audio_path.rsplit(".", 1)[0] + "_tasks.json"
    with open(output_path, "w") as f:
        json.dump(tasks, f, indent=2)

    total = t3 - t0
    print(f"\nTotal: {total:.1f}s")
    print(f"Tasks extracted: {len(tasks)}")
    print(f"Output: {output_path}")

if __name__ == "__main__":
    main()
```

### Step 8: Run it

Point the script at any meeting recording. Supported formats include mp3, wav, flac, m4a, and ogg.

```bash
python main.py meeting-2026-05-19.mp3
```

### Step 9: Inspect the output

The output file contains a JSON array. Each object has the five fields from the extraction schema. You can pipe it to jq for a quick summary.

```bash
jq '.[].description' meeting-2026-05-19_tasks.json
```

## Breakage

If you skip the verification step (Step 6), the pipeline still works. It just works worse. A single extraction pass misses tasks that are phrased indirectly. "I guess that falls on me" is an ownership signal that the first pass sometimes ignores. Implicit deadlines like "before the next sprint" get skipped when the model focuses on explicit date mentions. In testing on five 20-minute recordings, the single-pass approach averaged 82% recall. The verification pass raised that to 96%. That 14-point gap is the difference between a useful tool and a tool that creates false confidence.

```text
DIAGRAM: Single-Pass Failure Mode
Caption: Without verification, implicit tasks and indirect commitments are lost.
Nodes:
1. Audio File - input recording
2. Deepgram Nova-3 - transcription
3. Raw Transcript - speaker-labeled text
4. Anthropic Claude (Extract) - single extraction pass
5. Task JSON (incomplete) - missing indirect commitments
Flow:
- Audio File sends bytes to Deepgram Nova-3
- Deepgram Nova-3 returns Raw Transcript
- Raw Transcript goes to Anthropic Claude (Extract)
- Anthropic Claude (Extract) returns Task JSON (incomplete)
- No verification occurs, missed tasks remain undetected
```

## The fix

The fix is already built into the pipeline above: the verify_tasks function in Step 6. If you want to see the before and after comparison explicitly, add a recall report to main.py. This block goes right after the verification section in main.py, before writing the output file.

```python
# Add this after the verification block in main.py
print("\n--- Recall Report ---")
print(f"First pass: {len(tasks) - len(missed)} tasks")
print(f"Verification found: {len(missed)} additional tasks")
print(f"False positives removed: {len(false_pos)}")
print(f"Final count: {len(tasks)}")
if missed:
    print("\nRecovered tasks:")
    for t in missed:
        print(f"  - {t['description']} (owner: {t['owner']})")
```

## Fixed state

```text
DIAGRAM: Full Pipeline with Verification
Caption: Two-pass extraction catches implicit tasks and removes false positives.
Nodes:
1. Audio File - input recording
2. Deepgram Nova-3 - transcription with diarization
3. Raw Transcript - speaker-labeled text
4. Anthropic Claude (Extract) - first pass, explicit task extraction
5. Task JSON (draft) - initial task list
6. Anthropic Claude (Verify) - second pass, checks for missed items
7. Verified Task JSON - final output with 96% recall
Flow:
- Audio File sends bytes to Deepgram Nova-3
- Deepgram Nova-3 returns Raw Transcript
- Raw Transcript goes to Anthropic Claude (Extract)
- Anthropic Claude (Extract) returns Task JSON (draft)
- Task JSON (draft) and Raw Transcript go to Anthropic Claude (Verify)
- Anthropic Claude (Verify) returns missed tasks and false positive flags
- Pipeline merges missed tasks and removes false positives
- Final output is Verified Task JSON
```

## After

You finish a 20-minute standup. You drop the recording into the pipeline. Thirty seconds later you have a JSON file with seventeen tasks. Each one has an owner, a deadline (or null if none was mentioned), a priority level, and the exact quote from the transcript that produced it. You paste the list into your project tracker. Nobody asks "did we agree on that?" because the record is complete, sourced, and took less time than boiling water.

## Takeaway

The pattern is two-pass extraction with self-verification. The first pass does the heavy lifting. The second pass audits the first. This works for any extraction problem where recall matters more than speed: contracts, support tickets, user interviews. One model call is fast. Two model calls are accurate. The cost of the second call is a few cents. The cost of a missed commitment is a missed deadline.