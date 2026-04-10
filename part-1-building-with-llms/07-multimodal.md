<!--
  CHAPTER: 7
  TITLE: Multimodal AI
  PART: 1 — Building with LLM APIs
  PHASE: 1 — Get Dangerous
  PREREQS: Chapter 3
  KEY_TOPICS: vision APIs, image analysis, OCR, image generation, DALL-E, audio, Whisper, TTS, multi-input, multimodal messages
  DIFFICULTY: Intermediate
  LANGUAGE: TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 7: Multimodal AI

> **Part 1 — Building with LLM APIs** | Phase 1: Get Dangerous | Prerequisites: Ch 3 | Difficulty: Intermediate | Language: TypeScript

Text in, text out. That's what we've done so far. But the real world is images, audio, documents, screenshots, diagrams, and voice. Modern LLMs don't just process text — they see, hear, and generate across modalities.

This chapter opens the aperture. You'll send images to LLMs for analysis, generate images from text descriptions, transcribe audio to text, synthesize speech from text, and combine multiple modalities in a single request. Each capability unlocks real product features: OCR from receipts, automated image descriptions for accessibility, voice interfaces, UI screenshot review, and more.

The good news: if you can make a text API call (Ch 3), you can do all of this. The APIs follow the same patterns — you just send different content types in the messages array. The Vercel AI SDK normalizes the differences across providers.

### In This Chapter
- Vision: sending images to LLMs for analysis and understanding
- Image generation: creating images from text with DALL-E
- Audio: transcription with Whisper, text-to-speech
- Multi-input: combining text + image + audio in requests
- Practical use cases and trade-offs

### Related Chapters
- **Ch 3 (Your First LLM Call)** — spirals back: text-only calls expand to all modalities
- **Ch 5 (Structured Output)** — combine vision with structured extraction
- **Ch 41 (Image Generation Deep Dive)** — spirals forward to Stable Diffusion, image-to-image, DreamBooth

---

## 1. Vision: Making LLMs See

### 1.1 What Vision Models Can Do

Modern LLMs (GPT-4o, Claude, Gemini) can process images natively. You send an image alongside text, and the model can:

- **Describe** what's in the image
- **Answer questions** about the image content
- **Extract text** (OCR) from photos, screenshots, documents
- **Analyze** charts, graphs, diagrams
- **Compare** multiple images
- **Review** UI designs and screenshots
- **Read** handwriting, receipts, business cards
- **Identify** objects, people's actions, settings

### 1.2 Sending Images with the Vercel AI SDK

Images are sent as part of the `content` array in a user message:

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";
import * as fs from "fs";

async function describeImage(imagePath: string): Promise<string> {
  // Read image as base64
  const imageBuffer = fs.readFileSync(imagePath);
  const base64Image = imageBuffer.toString("base64");
  const mimeType = imagePath.endsWith(".png") ? "image/png" : "image/jpeg";

  const result = await generateText({
    model: openai("gpt-4o-mini"),
    messages: [
      {
        role: "user",
        content: [
          {
            type: "text",
            text: "Describe this image in detail. What do you see?",
          },
          {
            type: "image",
            image: `data:${mimeType};base64,${base64Image}`,
          },
        ],
      },
    ],
  });

  return result.text;
}

// Usage
const description = await describeImage("./screenshot.png");
console.log(description);
```

### 1.3 Images from URLs

You can also pass image URLs directly:

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

const result = await generateText({
  model: openai("gpt-4o-mini"),
  messages: [
    {
      role: "user",
      content: [
        {
          type: "text",
          text: "What product is shown in this image? What's the price?",
        },
        {
          type: "image",
          image: new URL("https://example.com/product-photo.jpg"),
        },
      ],
    },
  ],
});
```

### 1.4 Image from a Buffer

When working with uploaded files or programmatically generated images:

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";
import * as fs from "fs";

const imageBuffer = fs.readFileSync("./chart.png");

const result = await generateText({
  model: openai("gpt-4o"),
  messages: [
    {
      role: "user",
      content: [
        {
          type: "text",
          text: "Analyze this chart. What are the key trends?",
        },
        {
          type: "image",
          image: imageBuffer, // Pass the Buffer directly
        },
      ],
    },
  ],
});
```

---

## 2. Vision Use Cases

### 2.1 OCR: Extracting Text from Images

One of the most immediately useful capabilities:

```typescript
import { generateObject } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";
import * as fs from "fs";

const ReceiptSchema = z.object({
  merchant: z.string().describe("Business name"),
  date: z.string().nullable().describe("Transaction date in YYYY-MM-DD format"),
  items: z.array(z.object({
    name: z.string(),
    quantity: z.number().nullable(),
    price: z.number(),
  })).describe("Line items on the receipt"),
  subtotal: z.number().nullable(),
  tax: z.number().nullable(),
  total: z.number().describe("Total amount paid"),
  paymentMethod: z.string().nullable().describe("Credit card, cash, etc."),
});

async function extractReceipt(imagePath: string) {
  const imageBuffer = fs.readFileSync(imagePath);

  const result = await generateObject({
    model: openai("gpt-4o"),
    schema: ReceiptSchema,
    messages: [
      {
        role: "system",
        content: "Extract receipt data from the image. Use null for anything not visible or unclear.",
      },
      {
        role: "user",
        content: [
          { type: "text", text: "Extract all data from this receipt:" },
          { type: "image", image: imageBuffer },
        ],
      },
    ],
  });

  return result.object;
}

// Usage
const receipt = await extractReceipt("./receipt-photo.jpg");
console.log(`${receipt.merchant}: $${receipt.total}`);
console.log(`Items: ${receipt.items.length}`);
```

Notice we combined vision (Ch 7) with structured output (Ch 5). This is a common and powerful pattern: the LLM sees the image and returns typed data.

### 2.2 UI Screenshot Review

Automatically review UI designs or screenshots:

```typescript
import { generateObject } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";
import * as fs from "fs";

const UIReviewSchema = z.object({
  description: z.string().describe("Brief description of what the UI shows"),
  issues: z.array(z.object({
    severity: z.enum(["critical", "warning", "suggestion"]),
    area: z.string().describe("Which part of the UI"),
    issue: z.string().describe("What's wrong"),
    suggestion: z.string().describe("How to fix it"),
  })),
  accessibility: z.array(z.string()).describe("Potential accessibility concerns"),
  overallScore: z.number().min(1).max(10).describe("Overall UI quality score"),
});

async function reviewUI(screenshotPath: string, context: string = "") {
  const imageBuffer = fs.readFileSync(screenshotPath);

  const result = await generateObject({
    model: openai("gpt-4o"),
    schema: UIReviewSchema,
    messages: [
      {
        role: "system",
        content: `You are a senior UI/UX designer reviewing a screenshot. 
Focus on: layout, typography, color contrast, spacing, alignment, accessibility, and usability.
Be specific and actionable in your feedback.`,
      },
      {
        role: "user",
        content: [
          {
            type: "text",
            text: `Review this UI screenshot.${context ? ` Context: ${context}` : ""}`,
          },
          { type: "image", image: imageBuffer },
        ],
      },
    ],
  });

  return result.object;
}
```

### 2.3 Document Analysis

Analyze documents, slides, or diagrams:

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";
import * as fs from "fs";

async function analyzeDocument(imagePath: string, question: string) {
  const imageBuffer = fs.readFileSync(imagePath);

  const result = await generateText({
    model: openai("gpt-4o"),
    messages: [
      {
        role: "system",
        content: `You analyze documents and images accurately. 
When extracting information, only report what you can clearly see.
If something is unclear or partially visible, say so.`,
      },
      {
        role: "user",
        content: [
          { type: "text", text: question },
          { type: "image", image: imageBuffer },
        ],
      },
    ],
  });

  return result.text;
}

// Usage
const analysis = await analyzeDocument(
  "./architecture-diagram.png",
  "Explain the data flow in this system architecture diagram. What are the potential bottlenecks?"
);
```

### 2.4 Multiple Images

Send multiple images in a single request for comparison or batch analysis:

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";
import * as fs from "fs";

async function compareDesigns(imagePath1: string, imagePath2: string) {
  const image1 = fs.readFileSync(imagePath1);
  const image2 = fs.readFileSync(imagePath2);

  const result = await generateText({
    model: openai("gpt-4o"),
    messages: [
      {
        role: "user",
        content: [
          {
            type: "text",
            text: "Compare these two UI designs. Which is better and why? Be specific about layout, typography, and usability.",
          },
          { type: "image", image: image1 },
          { type: "image", image: image2 },
        ],
      },
    ],
  });

  return result.text;
}
```

---

## 3. Vision: Practical Considerations

### 3.1 Image Tokens and Cost

Images cost tokens. The exact cost depends on the provider and image resolution:

| Provider | Image Cost | Notes |
|---|---|---|
| OpenAI (GPT-4o) | 85-1,105 tokens per image | Depends on resolution and detail setting |
| OpenAI (GPT-4o-mini) | 2,833-6,438 tokens per image | Higher token count but lower per-token price |
| Anthropic (Claude) | ~1,600 tokens per 1 megapixel | Scales with image size |
| Google (Gemini) | 258 tokens per image | Fixed cost regardless of size |

**Cost optimization tips:**
- Resize images before sending (1024x1024 is usually sufficient)
- Use `detail: "low"` for OpenAI when you don't need fine details
- Send smaller images when possible (crop to the relevant area)
- Use the cheapest model that handles your vision task adequately

### 3.2 Image Detail Setting (OpenAI)

OpenAI lets you control image analysis resolution:

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

// Low detail — faster, cheaper, good for simple images
const result = await generateText({
  model: openai("gpt-4o-mini"),
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "What's in this photo?" },
        {
          type: "image",
          image: imageBuffer,
          // Some providers/SDK versions support providerOptions for detail level
        },
      ],
    },
  ],
});
```

- **Low detail**: Image is resized to 512x512. Fixed ~85 tokens. Good for classification, simple descriptions.
- **High detail**: Image is analyzed at higher resolution. More tokens. Good for OCR, detailed analysis.
- **Auto** (default): The model decides based on input size.

### 3.3 What Vision Models Struggle With

Vision models are impressive but not perfect:

- **Small text**: Tiny text in screenshots may be misread
- **Handwriting**: Varies widely — neat handwriting works, messy doesn't
- **Complex diagrams**: May misinterpret connections in dense flowcharts
- **Counting**: Counting objects in busy scenes is often inaccurate
- **Spatial reasoning**: "Is X to the left of Y?" can be unreliable
- **Text in non-Latin scripts**: Quality varies by language
- **CAPTCHA-like text**: Intentionally distorted text is hard

**Best practice:** Always validate extracted data, especially numbers and names. Don't trust OCR output blindly.

---

## 4. Image Generation

### 4.1 DALL-E API

OpenAI's DALL-E generates images from text descriptions:

```typescript
import OpenAI from "openai";
import * as fs from "fs";

const client = new OpenAI();

async function generateImage(prompt: string): Promise<string> {
  const response = await client.images.generate({
    model: "dall-e-3",
    prompt,
    n: 1,
    size: "1024x1024",
    quality: "standard", // "standard" or "hd"
    style: "natural",    // "natural" or "vivid"
  });

  const imageUrl = response.data[0].url;
  if (!imageUrl) throw new Error("No image URL returned");
  
  return imageUrl;
}

// Usage
const url = await generateImage(
  "A minimalist tech company logo featuring a stylized mountain and a code bracket, blue and white color scheme, clean vector art"
);
console.log("Generated image:", url);
```

### 4.2 Saving Generated Images

```typescript
import OpenAI from "openai";
import * as fs from "fs";

const client = new OpenAI();

async function generateAndSave(prompt: string, outputPath: string) {
  // Request base64 instead of URL
  const response = await client.images.generate({
    model: "dall-e-3",
    prompt,
    n: 1,
    size: "1024x1024",
    response_format: "b64_json",
  });

  const base64Data = response.data[0].b64_json;
  if (!base64Data) throw new Error("No image data returned");

  // Save to file
  const buffer = Buffer.from(base64Data, "base64");
  fs.writeFileSync(outputPath, buffer);
  console.log(`Image saved to ${outputPath}`);

  // Also get the revised prompt (DALL-E 3 often rewrites your prompt)
  const revisedPrompt = response.data[0].revised_prompt;
  console.log("Revised prompt:", revisedPrompt);
}
```

### 4.3 Image Generation Parameters

| Parameter | Options | What It Does |
|---|---|---|
| `model` | `dall-e-3`, `dall-e-2` | DALL-E 3 is higher quality, understands prompts better |
| `size` | `1024x1024`, `1024x1792`, `1792x1024` | Output dimensions (DALL-E 3) |
| `quality` | `standard`, `hd` | HD has more detail, costs 2x |
| `style` | `natural`, `vivid` | Vivid = more dramatic, natural = more photographic |
| `n` | 1 (DALL-E 3), 1-10 (DALL-E 2) | Number of images to generate |

**Pricing (DALL-E 3):**
- Standard 1024x1024: $0.040 per image
- HD 1024x1024: $0.080 per image
- Standard 1792x1024: $0.080 per image
- HD 1792x1024: $0.120 per image

### 4.4 Image Generation Use Cases

**When to generate images:**
- Product mockups and prototypes
- Custom illustrations for content
- Placeholder images during development
- Creative exploration and brainstorming
- Personalized content (user-specific illustrations)

**When NOT to generate images:**
- When you need exact, pixel-perfect output (logos, icons — use a designer)
- When brand consistency matters (style varies between generations)
- When factual accuracy matters (DALL-E can generate incorrect text, wrong counts)
- When stock photos would suffice (cheaper, faster, more consistent)
- When you need the same image twice (generation is non-deterministic)

### 4.5 Image Editing

DALL-E can also edit existing images:

```typescript
import OpenAI from "openai";
import * as fs from "fs";

const client = new OpenAI();

async function editImage(
  imagePath: string,
  maskPath: string,
  prompt: string
) {
  const response = await client.images.edit({
    model: "dall-e-2", // Edit only available with DALL-E 2
    image: fs.createReadStream(imagePath),
    mask: fs.createReadStream(maskPath), // Transparent area = area to edit
    prompt,
    n: 1,
    size: "1024x1024",
  });

  return response.data[0].url;
}

// Usage: replace the background of a product photo
const editedUrl = await editImage(
  "./product.png",
  "./background-mask.png",  // Transparent where background should change
  "A clean, white studio background with soft lighting"
);
```

---

## 5. Audio: Transcription and Speech

### 5.1 Whisper: Speech-to-Text

OpenAI's Whisper model transcribes audio to text:

```typescript
import OpenAI from "openai";
import * as fs from "fs";

const client = new OpenAI();

async function transcribeAudio(audioPath: string): Promise<string> {
  const transcription = await client.audio.transcriptions.create({
    model: "whisper-1",
    file: fs.createReadStream(audioPath),
    language: "en", // Optional: specify language for better accuracy
  });

  return transcription.text;
}

// Usage
const text = await transcribeAudio("./meeting-recording.mp3");
console.log("Transcription:", text);
```

### 5.2 Whisper with Timestamps

Get word-level or segment-level timestamps:

```typescript
import OpenAI from "openai";
import * as fs from "fs";

const client = new OpenAI();

async function transcribeWithTimestamps(audioPath: string) {
  const transcription = await client.audio.transcriptions.create({
    model: "whisper-1",
    file: fs.createReadStream(audioPath),
    response_format: "verbose_json",
    timestamp_granularities: ["word", "segment"],
  });

  return transcription;
}

// Returns segments like:
// {
//   segments: [
//     { start: 0.0, end: 2.5, text: "Hello everyone, welcome to the meeting." },
//     { start: 2.5, end: 5.1, text: "Today we'll discuss the Q4 roadmap." },
//     ...
//   ],
//   words: [
//     { word: "Hello", start: 0.0, end: 0.3 },
//     { word: "everyone", start: 0.3, end: 0.8 },
//     ...
//   ]
// }
```

### 5.3 Transcription Use Cases

```typescript
import OpenAI from "openai";
import { generateObject } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";
import * as fs from "fs";

const oaiClient = new OpenAI();

// Transcribe a meeting and extract action items
async function processMeetingRecording(audioPath: string) {
  // Step 1: Transcribe
  const transcription = await oaiClient.audio.transcriptions.create({
    model: "whisper-1",
    file: fs.createReadStream(audioPath),
  });

  // Step 2: Extract structured data from transcription
  const result = await generateObject({
    model: openai("gpt-4o-mini"),
    schema: z.object({
      summary: z.string().describe("2-3 sentence meeting summary"),
      participants: z.array(z.string()).describe("People mentioned in the meeting"),
      actionItems: z.array(z.object({
        task: z.string(),
        owner: z.string().nullable(),
        deadline: z.string().nullable(),
      })),
      decisions: z.array(z.string()).describe("Key decisions made"),
      followUpDate: z.string().nullable().describe("Next meeting date if mentioned"),
    }),
    messages: [
      {
        role: "system",
        content: "Extract meeting information from this transcript. Use null for anything not mentioned.",
      },
      {
        role: "user",
        content: `Meeting transcript:\n\n${transcription.text}`,
      },
    ],
  });

  return {
    transcript: transcription.text,
    ...result.object,
  };
}
```

### 5.4 Whisper Limitations

- **Max file size:** 25 MB (split longer recordings)
- **Supported formats:** mp3, mp4, mpeg, mpga, m4a, wav, webm
- **Accuracy:** Excellent for clear English; degrades with heavy accents, background noise, multiple speakers talking simultaneously
- **Cost:** $0.006 per minute of audio
- **No real-time:** Whisper processes complete audio files, not streams. For real-time transcription, look at Deepgram or AssemblyAI.

### 5.5 Text-to-Speech (TTS)

Generate spoken audio from text:

```typescript
import OpenAI from "openai";
import * as fs from "fs";

const client = new OpenAI();

async function textToSpeech(
  text: string,
  outputPath: string,
  voice: "alloy" | "echo" | "fable" | "onyx" | "nova" | "shimmer" = "nova"
) {
  const response = await client.audio.speech.create({
    model: "tts-1",      // "tts-1" (faster) or "tts-1-hd" (higher quality)
    voice,
    input: text,
    speed: 1.0,           // 0.25 to 4.0
    response_format: "mp3", // mp3, opus, aac, flac, wav, pcm
  });

  const buffer = Buffer.from(await response.arrayBuffer());
  fs.writeFileSync(outputPath, buffer);
  console.log(`Audio saved to ${outputPath}`);
}

// Usage
await textToSpeech(
  "Hello! Welcome to our AI-powered customer support. How can I help you today?",
  "./welcome.mp3",
  "nova"
);
```

### 5.6 TTS Voices

| Voice | Description | Best For |
|---|---|---|
| `alloy` | Neutral, balanced | General purpose |
| `echo` | Warm, conversational | Podcasts, narration |
| `fable` | Expressive, storytelling | Creative content |
| `onyx` | Deep, authoritative | Professional content |
| `nova` | Friendly, upbeat | Customer-facing |
| `shimmer` | Clear, precise | Technical content |

**Pricing:** $0.015 per 1,000 characters (tts-1) or $0.030 per 1,000 characters (tts-1-hd).

### 5.7 TTS Use Cases

- **Accessibility:** Make your content available to visually impaired users
- **Voice interfaces:** Build voice-enabled chatbots
- **Content creation:** Auto-generate podcast summaries, audio newsletters
- **Alerts:** Spoken notifications for critical events
- **Language learning:** Generate pronunciation examples

---

## 6. Multi-Input: Combining Modalities

### 6.1 Text + Image in One Request

You've already seen this pattern, but let's make it explicit. The `content` array can mix text and images:

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";
import * as fs from "fs";

async function analyzeWithContext(
  imagePath: string,
  textContext: string,
  question: string
) {
  const imageBuffer = fs.readFileSync(imagePath);

  const result = await generateText({
    model: openai("gpt-4o"),
    messages: [
      {
        role: "system",
        content: "You analyze images in the context of the provided information.",
      },
      {
        role: "user",
        content: [
          { type: "text", text: `Context: ${textContext}` },
          { type: "image", image: imageBuffer },
          { type: "text", text: `Question: ${question}` },
        ],
      },
    ],
  });

  return result.text;
}

// Usage: analyze an error screenshot with log context
const analysis = await analyzeWithContext(
  "./error-screenshot.png",
  "The application is a Next.js e-commerce site running on Node.js 20. " +
  "We recently deployed a change to the checkout flow. " +
  "Error logs show: TypeError: Cannot read property 'price' of undefined",
  "Based on the screenshot and the error context, what's likely causing this bug? How would you fix it?"
);
```

### 6.2 Multi-Image Analysis

Send multiple images for comparison, batch processing, or contextual analysis:

```typescript
import { generateObject } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";
import * as fs from "fs";

const ComparisonSchema = z.object({
  differences: z.array(z.object({
    area: z.string().describe("Which part of the UI differs"),
    before: z.string().describe("What it looked like in image 1"),
    after: z.string().describe("What it looks like in image 2"),
    assessment: z.enum(["improvement", "regression", "neutral"]),
  })),
  overallAssessment: z.string(),
  recommendation: z.enum(["ship_it", "needs_changes", "revert"]),
});

async function compareBeforeAfter(beforePath: string, afterPath: string) {
  const before = fs.readFileSync(beforePath);
  const after = fs.readFileSync(afterPath);

  const result = await generateObject({
    model: openai("gpt-4o"),
    schema: ComparisonSchema,
    messages: [
      {
        role: "system",
        content: "Compare two versions of a UI. The first image is 'before', the second is 'after' a change.",
      },
      {
        role: "user",
        content: [
          { type: "text", text: "Before:" },
          { type: "image", image: before },
          { type: "text", text: "After:" },
          { type: "image", image: after },
          { type: "text", text: "Compare these two versions. What changed? Is it better?" },
        ],
      },
    ],
  });

  return result.object;
}
```

### 6.3 Audio + Text Pipeline

Combine transcription with text analysis:

```typescript
import OpenAI from "openai";
import { generateObject } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";
import * as fs from "fs";

const oaiClient = new OpenAI();

const CallAnalysisSchema = z.object({
  sentiment: z.enum(["positive", "negative", "neutral", "mixed"]),
  customerSatisfaction: z.number().min(1).max(10),
  topics: z.array(z.string()),
  escalationNeeded: z.boolean(),
  summary: z.string().max(200),
  agentPerformance: z.object({
    empathy: z.number().min(1).max(5),
    resolution: z.number().min(1).max(5),
    professionalism: z.number().min(1).max(5),
  }),
});

async function analyzeCustomerCall(audioPath: string) {
  // Step 1: Transcribe the call
  const transcription = await oaiClient.audio.transcriptions.create({
    model: "whisper-1",
    file: fs.createReadStream(audioPath),
  });

  // Step 2: Analyze the transcription
  const analysis = await generateObject({
    model: openai("gpt-4o-mini"),
    schema: CallAnalysisSchema,
    messages: [
      {
        role: "system",
        content: `You analyze customer support call transcripts. 
Rate agent performance on empathy, resolution, and professionalism (1-5 each).
Customer satisfaction is your estimate from 1-10 based on the conversation tone and outcome.`,
      },
      {
        role: "user",
        content: `Analyze this customer support call transcript:\n\n${transcription.text}`,
      },
    ],
  });

  return {
    transcript: transcription.text,
    analysis: analysis.object,
  };
}
```

---

## 7. Provider-Specific Vision Capabilities

### 7.1 Anthropic (Claude) Vision

Claude has strong vision capabilities, especially for document analysis:

```typescript
import { generateText } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import * as fs from "fs";

const result = await generateText({
  model: anthropic("claude-sonnet-4-20250514"),
  messages: [
    {
      role: "user",
      content: [
        {
          type: "text",
          text: "Analyze this architecture diagram. List all components and their connections.",
        },
        {
          type: "image",
          image: fs.readFileSync("./architecture.png"),
        },
      ],
    },
  ],
});
```

Claude's vision strengths:
- Excellent at analyzing dense technical diagrams
- Strong OCR, especially for multi-language documents
- Good at following specific instructions about what to extract
- Handles long documents across multiple page images well

### 7.2 Google Gemini Vision

Gemini handles video in addition to images:

```typescript
import { generateText } from "ai";
import { google } from "@ai-sdk/google";
import * as fs from "fs";

const result = await generateText({
  model: google("gemini-2.0-flash"),
  messages: [
    {
      role: "user",
      content: [
        {
          type: "text",
          text: "What's happening in this image?",
        },
        {
          type: "image",
          image: fs.readFileSync("./photo.jpg"),
        },
      ],
    },
  ],
});
```

Gemini's vision strengths:
- 1M token context allows analyzing many images at once
- Native video understanding (send video files directly)
- Competitive image analysis at lower cost with Flash models
- Good multilingual OCR

### 7.3 Choosing a Vision Model

| Task | Best Choice | Why |
|---|---|---|
| Quick image classification | GPT-4o-mini, Gemini Flash | Cheap, fast, good enough |
| Detailed OCR | GPT-4o, Claude | Higher accuracy for text extraction |
| Document analysis | Claude | Excellent instruction following for structured extraction |
| Batch image processing | Gemini Flash | Lowest cost per image |
| Multiple images in context | Gemini | 1M token context fits many images |
| UI review | GPT-4o, Claude | Both strong at design analysis |

---

## 8. Building a Multimodal Chat

### 8.1 Accepting Image Uploads in Chat

Extend the chat from Chapter 6 to handle images:

```typescript
// app/api/chat/route.ts
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: openai("gpt-4o-mini"),
    system: "You are a helpful assistant that can analyze images and answer questions about them.",
    messages,
  });

  return result.toDataStreamResponse();
}
```

```typescript
// app/page.tsx
"use client";

import { useChat } from "@ai-sdk/react";
import { useRef } from "react";

export default function MultimodalChat() {
  const { messages, input, handleInputChange, handleSubmit, isLoading } =
    useChat();
  const fileInputRef = useRef<HTMLInputElement>(null);

  function handleFormSubmit(e: React.FormEvent) {
    e.preventDefault();

    const files = fileInputRef.current?.files;

    handleSubmit(e, {
      experimental_attachments: files
        ? Array.from(files).map((file) => ({
            name: file.name,
            contentType: file.type,
            url: URL.createObjectURL(file),
          }))
        : undefined,
    });

    // Clear file input
    if (fileInputRef.current) {
      fileInputRef.current.value = "";
    }
  }

  return (
    <div className="flex flex-col h-screen max-w-2xl mx-auto">
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.map((message) => (
          <div key={message.id} className="mb-4">
            <div className="font-medium text-sm text-gray-500 mb-1">
              {message.role === "user" ? "You" : "Assistant"}
            </div>

            {/* Render attachments */}
            {message.experimental_attachments?.map((attachment, i) => (
              <div key={i} className="mb-2">
                {attachment.contentType?.startsWith("image/") && (
                  <img
                    src={attachment.url}
                    alt={attachment.name}
                    className="max-w-sm rounded-lg"
                  />
                )}
              </div>
            ))}

            <div className="whitespace-pre-wrap">{message.content}</div>
          </div>
        ))}
      </div>

      <form onSubmit={handleFormSubmit} className="p-4 border-t">
        <div className="flex gap-2 items-end">
          <label className="cursor-pointer p-2 hover:bg-gray-100 rounded">
            <input
              type="file"
              ref={fileInputRef}
              accept="image/*"
              className="hidden"
              multiple
            />
            <span className="text-2xl">+</span>
          </label>
          <input
            value={input}
            onChange={handleInputChange}
            placeholder="Type a message or attach an image..."
            className="flex-1 p-3 border rounded-lg"
            disabled={isLoading}
          />
          <button
            type="submit"
            disabled={isLoading}
            className="px-6 py-3 bg-blue-600 text-white rounded-lg disabled:opacity-50"
          >
            Send
          </button>
        </div>
      </form>
    </div>
  );
}
```

---

## 9. Cost Comparison Across Modalities

Here's a rough cost comparison to help you budget:

| Modality | Model | Unit | Cost | Notes |
|---|---|---|---|---|
| **Text (input)** | GPT-4o-mini | 1M tokens | $0.15 | ~750K words |
| **Text (output)** | GPT-4o-mini | 1M tokens | $0.60 | |
| **Text (input)** | GPT-4o | 1M tokens | $2.50 | |
| **Text (output)** | GPT-4o | 1M tokens | $10.00 | |
| **Vision** | GPT-4o | per image | $0.003-$0.01 | Depends on resolution |
| **Image generation** | DALL-E 3 | per image | $0.04-$0.12 | Depends on quality/size |
| **Transcription** | Whisper | per minute | $0.006 | |
| **Text-to-Speech** | TTS-1 | 1K chars | $0.015 | |
| **Text-to-Speech** | TTS-1-HD | 1K chars | $0.030 | |

**Example budget: A multimodal customer support app**
- 1,000 text conversations/day: ~$60/month (gpt-4o-mini)
- 200 image analyses/day (receipts, screenshots): ~$60/month (gpt-4o)
- 50 audio transcriptions/day (5 min average): ~$45/month (Whisper)
- Total: ~$165/month

That's surprisingly affordable for a full multimodal AI support system.

---

## 10. Key Takeaways

1. **Vision is just another message type.** Send images as part of the content array alongside text. Same API pattern as text-only calls.

2. **Vision + structured output is powerful.** Combine image analysis with Zod schemas to extract typed data from images (receipts, documents, screenshots).

3. **DALL-E generates, not edits.** Use it for creative generation and mockups. Don't expect pixel-perfect control or consistency between generations.

4. **Whisper handles transcription well.** Great for meeting notes, call analysis, and voice interfaces. Pair it with text analysis for a full pipeline.

5. **TTS is cheap and effective.** At $0.015/1K characters, adding voice output to your app is affordable.

6. **Match the model to the task.** Use cheap models (GPT-4o-mini, Gemini Flash) for simple vision tasks. Use GPT-4o or Claude for complex analysis.

7. **Images cost tokens.** Budget for image token costs. Resize images before sending. Use the lowest detail level that works.

8. **Combine modalities for real power.** Audio transcription into text analysis into structured output into TTS response — each step uses a different modality.

---

## What's Next

That's the end of Part 1. You can now:
- Make LLM calls with proper error handling (Ch 3)
- Write effective prompts (Ch 4)
- Get structured, typed data back (Ch 5)
- Build streaming chat interfaces (Ch 6)
- Work with images, audio, and multiple modalities (Ch 7)

In **Part 2 — Agent Engineering**, you'll take everything from Part 1 and build something more powerful: autonomous agents. Chapter 8 introduces **tool calling** — giving LLMs the ability to execute functions, query databases, and interact with external systems. Chapter 9 wraps that into the **agent loop** — the pattern where LLMs decide what to do, do it, observe the result, and decide what to do next.

You're about to go from "AI features" to "AI agents." Let's go.
