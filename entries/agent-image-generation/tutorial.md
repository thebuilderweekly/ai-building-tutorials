## Opening thesis

You will build an AI agent that generates blog post images through Replicate, scores each output with the Anthropic API, and discards anything below a quality threshold. A naive image generation agent produces 100 mediocre images at $5 total cost; a properly-prompted agent with output verification produces 10 great images at the same cost. The agent beats a human at this task because it applies a consistent rubric to every image in seconds, with zero taste fatigue.

## Before

You have a content pipeline. Every Monday your agent generates hero images for five blog posts. It calls a diffusion model on Replicate with a one-line prompt like "a colorful illustration of cloud computing." The model returns something. Sometimes it is good. Mostly it is not: mangled text, wrong aspect ratio, skin tones that look radioactive, objects floating in white voids. So you run it again. And again. You end up with 80 to 100 generations per week. At roughly $0.05 per image, that is $4 to $5 a week for a folder of mostly unusable PNGs. You scroll through the folder, pick the five least bad ones, crop them manually, and move on with your life. The waste is not dramatic. It is grinding. It adds up. And it never gets better on its own.

## Architecture

The system has four components. An orchestrator script coordinates the loop. It calls the Anthropic API to build structured prompts from blog post metadata. It sends each prompt to Replicate's SDXL endpoint. Then it sends the resulting image back to the Anthropic API for a quality score. Only images that pass a threshold get saved.

```text
DIAGRAM: Image generation agent with quality gate
Caption: Full loop from blog metadata to verified output images
Nodes:
1. Orchestrator (Python script) - coordinates the full pipeline
2. Anthropic API (Claude) - builds structured prompts and scores output images
3. Replicate API (SDXL) - generates images from prompts
4. Local filesystem - stores only passing images
Flow:
- Orchestrator sends blog post title and description to Anthropic API
- Anthropic API returns a structured image prompt with style constraints
- Orchestrator sends the structured prompt to Replicate API
- Replicate API returns a generated image URL
- Orchestrator sends the image URL to Anthropic API for quality scoring
- Anthropic API returns a numeric score and pass/fail verdict
- Orchestrator saves passing images to local filesystem, discards failures
- Orchestrator retries with a modified prompt if the image fails (up to 3 attempts)
```

## Step-by-step implementation

### 1. Set up environment and dependencies

You need Python 3.10 or later, a Replicate account, and an Anthropic account. Get your Replicate API token from https://replicate.com/account/api-tokens. Get your Anthropic API key from https://console.anthropic.com/settings/keys. Set both as environment variables.

```bash
export REPLICATE_API_TOKEN="your_replicate_token_here"
export ANTHROPIC_API_KEY="your_anthropic_key_here"
pip install replicate anthropic requests
```

### 2. Define the blog post input data

The agent needs structured input. Each blog post has a title, a one-sentence description, and a desired mood. This gives the prompt builder enough material to work with. Save this as `posts.json`.

```json
[
  {
    "slug": "zero-downtime-deploys",
    "title": "Zero-Downtime Deploys with Rolling Updates",
    "description": "A technical guide to deploying without interrupting live traffic.",
    "mood": "calm, technical, precise"
  },
  {
    "slug": "rate-limiting-apis",
    "title": "Rate Limiting Your API Without Annoying Your Users",
    "description": "Practical patterns for throttling requests while keeping the developer experience smooth.",
    "mood": "friendly, structured, clear"
  }
]
```

### 3. Build the prompt generator with Claude

A one-line prompt produces generic images. A structured prompt with explicit style constraints, negative instructions, and composition guidance produces usable images. The agent calls Claude to turn blog metadata into a detailed image prompt.

```python
import anthropic
import os
import json

client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

def build_image_prompt(post: dict) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=300,
        messages=[
            {
                "role": "user",
                "content": (
                    f"Write a detailed image generation prompt for a blog hero image.\n"
                    f"Blog title: {post['title']}\n"
                    f"Blog description: {post['description']}\n"
                    f"Desired mood: {post['mood']}\n\n"
                    f"Requirements for the prompt you write:\n"
                    f"- Specify a concrete visual scene, not abstract concepts\n"
                    f"- Include lighting direction and color palette (3 to 4 colors)\n"
                    f"- Specify aspect ratio as 16:9 landscape\n"
                    f"- Include 'no text, no watermarks, no logos' as a negative constraint\n"
                    f"- Keep it under 120 words\n"
                    f"- Return only the prompt, no commentary"
                ),
            }
        ],
    )
    return response.content[0].text.strip()
```

### 4. Generate an image on Replicate

This function sends the structured prompt to Replicate's SDXL model and returns the output image URL. The `wait` method blocks until the prediction finishes, which simplifies the orchestration loop. For production use, you would switch to webhooks.

```python
import replicate

def generate_image(prompt: str) -> str:
    output = replicate.run(
        "stability-ai/sdxl:7762fd07cf82c948538e41f63f77d685e02b063e37e496e96eefd46c929f9bdc",
        input={
            "prompt": prompt,
            "negative_prompt": "text, watermark, logo, blurry, distorted, low quality, overexposed",
            "width": 1344,
            "height": 768,
            "num_inference_steps": 30,
            "guidance_scale": 7.5,
        },
    )
    if isinstance(output, list) and len(output) > 0:
        return str(output[0])
    return str(output)
```

### 5. Score the image with Claude's vision

This is the quality gate. The agent sends the generated image to Claude and asks for a structured quality assessment. Claude returns a JSON object with a score from 1 to 10 and a pass/fail verdict. The scoring rubric is explicit: composition, relevance to the blog topic, visual coherence, absence of artifacts.

```python
import base64
import requests

def score_image(image_url: str, post: dict) -> dict:
    image_data = requests.get(image_url).content
    b64_image = base64.standard_b64encode(image_data).decode("utf-8")

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=200,
        messages=[
            {
                "role": "user",
                "content": [
                    {
                        "type": "image",
                        "source": {
                            "type": "base64",
                            "media_type": "image/png",
                            "data": b64_image,
                        },
                    },
                    {
                        "type": "text",
                        "text": (
                            f"Score this blog hero image for the post titled '{post['title']}'.\n"
                            f"Rate it on four criteria, each 1 to 10:\n"
                            f"1. Composition: is the layout balanced and visually clear?\n"
                            f"2. Relevance: does it relate to '{post['description']}'?\n"
                            f"3. Coherence: are there visual artifacts, mangled objects, or unnatural elements?\n"
                            f"4. Mood match: does it feel '{post['mood']}'?\n\n"
                            f"Return ONLY a JSON object with keys: composition, relevance, coherence, mood_match, average, pass.\n"
                            f"Set pass to true if average >= 7, false otherwise.\n"
                            f"No commentary. Just the JSON."
                        ),
                    },
                ],
            }
        ],
    )
    raw = response.content[0].text.strip()
    if raw.startswith("```"):
        raw = raw.split("\n", 1)[1].rsplit("```", 1)[0]
    return json.loads(raw)
```

### 6. Build the orchestration loop

The orchestrator ties everything together. For each blog post, it builds a prompt, generates an image, scores it, and retries up to three times if the image fails. On retry, it appends the failure reason to the prompt so the next generation avoids the same mistake.

```python
import pathlib

def run_pipeline(posts_file: str = "posts.json", output_dir: str = "output"):
    pathlib.Path(output_dir).mkdir(exist_ok=True)
    with open(posts_file) as f:
        posts = json.load(f)

    results = []
    for post in posts:
        print(f"\nProcessing: {post['slug']}")
        base_prompt = build_image_prompt(post)
        prompt = base_prompt
        passed = False

        for attempt in range(1, 4):
            print(f"  Attempt {attempt}: generating image...")
            image_url = generate_image(prompt)
            print(f"  Scoring image...")
            score = score_image(image_url, post)
            print(f"  Score: {score}")

            if score.get("pass", False):
                image_data = requests.get(image_url).content
                out_path = pathlib.Path(output_dir) / f"{post['slug']}.png"
                out_path.write_bytes(image_data)
                print(f"  Saved to {out_path}")
                results.append({"slug": post["slug"], "attempts": attempt, "score": score})
                passed = True
                break
            else:
                lowest = min(score, key=lambda k: score[k] if isinstance(score[k], (int, float)) else 999)
                prompt = base_prompt + f" Improve {lowest}. Avoid the issues from the previous attempt."

        if not passed:
            print(f"  Failed after 3 attempts. Skipping {post['slug']}.")
            results.append({"slug": post["slug"], "attempts": 3, "score": score, "failed": True})

    with open(pathlib.Path(output_dir) / "results.json", "w") as f:
        json.dump(results, f, indent=2)
    print(f"\nDone. Results saved to {output_dir}/results.json")

if __name__ == "__main__":
    run_pipeline()
```

### 7. Run the pipeline

Execute the script. It processes each post, generates images, scores them, and saves only passing results.

```bash
python -c "from pipeline import run_pipeline; run_pipeline()"
```

Wait. That assumes you named the file `pipeline.py`. If you pasted the code into one file, name it `pipeline.py` and run it directly.

```bash
python pipeline.py
```

## Breakage

If you skip the quality scoring step, the agent becomes a blind generator. It fires prompts at Replicate and saves every result. You end up back where you started: a folder of 10 to 20 images per post, most of them unusable. The structured prompt helps, but it does not guarantee good output. Diffusion models are stochastic. Without a verification step, there is no feedback loop, no retry logic, and no way to improve across attempts. You spend the same money and still pick winners by hand.

```text
DIAGRAM: Pipeline without quality gate
Caption: The failure mode when scoring is removed
Nodes:
1. Orchestrator (Python script) - generates images but cannot evaluate them
2. Anthropic API (Claude) - builds prompts only, never sees outputs
3. Replicate API (SDXL) - generates images from prompts
4. Local filesystem - stores every image, good or bad
Flow:
- Orchestrator sends blog metadata to Anthropic API for prompt building
- Anthropic API returns a structured prompt
- Orchestrator sends the prompt to Replicate API
- Replicate API returns an image URL
- Orchestrator saves the image to filesystem without evaluation
- No retry, no feedback, no filtering
- Human must manually review all outputs
```

## The fix

The fix is the `score_image` function from Step 5 and the retry loop from Step 6. Those are already in the code above. The scoring function closes the loop. It turns a one-shot generation into an iterative process where each failure informs the next attempt. Here is the core of the fix isolated, showing how the retry modifies the prompt based on the weakest scoring dimension:

```python
for attempt in range(1, 4):
    image_url = generate_image(prompt)
    score = score_image(image_url, post)

    if score.get("pass", False):
        image_data = requests.get(image_url).content
        out_path = pathlib.Path(output_dir) / f"{post['slug']}.png"
        out_path.write_bytes(image_data)
        break
    else:
        lowest = min(score, key=lambda k: score[k] if isinstance(score[k], (int, float)) else 999)
        prompt = base_prompt + f" Improve {lowest}. Avoid the issues from the previous attempt."
```

The key mechanic: the agent identifies which dimension scored lowest (composition, relevance, coherence, or mood match) and appends a corrective instruction to the prompt. The next generation is not random. It is directed.

## Fixed state

```text
DIAGRAM: Complete pipeline with quality gate and retry
Caption: The full system with scoring, filtering, and iterative improvement
Nodes:
1. Orchestrator (Python script) - coordinates generation, scoring, and retry
2. Anthropic API (Claude) - builds structured prompts and scores output images
3. Replicate API (SDXL) - generates images from prompts
4. Local filesystem - stores only images that score 7 or above
5. Results log (results.json) - records attempt counts and scores for cost tracking
Flow:
- Orchestrator sends blog metadata to Anthropic API for prompt construction
- Anthropic API returns a detailed, constrained image prompt
- Orchestrator sends the prompt to Replicate API
- Replicate API returns a generated image URL
- Orchestrator sends the image to Anthropic API for multi-dimension scoring
- Anthropic API returns a score object with pass/fail verdict
- If pass: Orchestrator saves the image and logs the result
- If fail: Orchestrator modifies the prompt using the lowest-scoring dimension
- Orchestrator retries up to 3 times per post, then moves on
- Only verified images reach the filesystem
```

## After

You have a content pipeline. Every Monday your agent generates hero images for five blog posts. It builds a structured prompt from each post's metadata, generates an image on Replicate, scores it against a four-dimension rubric using Claude's vision, and retries with a corrective instruction if the image fails. Most posts pass on the first or second attempt. You spend $0.50 to $1.00 per week instead of $4 to $5. The output folder contains five images, all usable. You do not scroll through rejects. You do not crop manually. The agent's quality bar is consistent at 3 AM on a Tuesday, which is something you cannot say about your own eyes after 80 thumbnails.

## Takeaway

The pattern is: generate, verify, retry with feedback. It applies to any agent task where the output is nondeterministic. Code generation, data extraction, email drafting. If your agent produces output but never checks it, you have built an expensive random number generator. Close the loop.