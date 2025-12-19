# Advanced Background Removal Guide

This guide covers advanced scenarios for background removal including batch processing, URL-based inputs, and integration with other workflows.

## URL-Based Input

Skip the upload step when your image is already hosted online:

```bash
bun run - << 'EOF'
import { fal } from "@fal-ai/client";
import { writeFileSync } from "fs";

const imageUrl = "https://example.com/your-image.jpg";

const result = await fal.subscribe("fal-ai/birefnet/v2", {
  input: {
    image_url: imageUrl,
    model: "General Use (Light)",
    operating_resolution: "1024x1024",
    output_format: "png"
  },
  logs: true,
  onQueueUpdate: (update) => {
    if (update.status === "IN_PROGRESS" && update.logs) {
      update.logs.map((log) => log.message).forEach(console.log);
    }
  },
});

const response = await fetch(result.data.image.url);
const buffer = Buffer.from(await response.arrayBuffer());

writeFileSync("output-nobg.png", buffer);
console.log("Saved: output-nobg.png");
EOF
```

## Batch Processing

Process multiple images in a directory:

```bash
bun run - << 'EOF'
import { fal } from "@fal-ai/client";
import { readFileSync, writeFileSync, readdirSync } from "fs";
import { basename, dirname, join, extname } from "path";

const inputDir = "INPUT_DIRECTORY";
const extensions = [".jpg", ".jpeg", ".png", ".webp"];

const files = readdirSync(inputDir)
  .filter(f => extensions.includes(extname(f).toLowerCase()));

console.log(`Found ${files.length} images to process`);

for (const file of files) {
  const imagePath = join(inputDir, file);
  console.log(`\nProcessing: ${file}`);

  try {
    const imageBuffer = readFileSync(imagePath);
    const mimeType = file.endsWith(".png") ? "image/png" : "image/jpeg";

    const uploadFile = new File([imageBuffer], file, { type: mimeType });
    const imageUrl = await fal.storage.upload(uploadFile);

    const result = await fal.subscribe("fal-ai/birefnet/v2", {
      input: {
        image_url: imageUrl,
        model: "General Use (Light)",
        operating_resolution: "1024x1024",
        output_format: "png"
      },
    });

    const response = await fetch(result.data.image.url);
    const buffer = Buffer.from(await response.arrayBuffer());

    const nameWithoutExt = basename(file, extname(file));
    const outputPath = join(inputDir, `${nameWithoutExt}-nobg.png`);

    writeFileSync(outputPath, buffer);
    console.log(`Saved: ${outputPath}`);
  } catch (error) {
    console.error(`Failed to process ${file}:`, error.message);
  }
}

console.log("\nBatch processing complete!");
EOF
```

Replace `INPUT_DIRECTORY` with the path to your image folder.

## Model Selection Guide

Choose the right model for your use case:

### General Use (Light)
Best for:
- Product photos on solid backgrounds
- Simple subjects with clear edges
- Fast processing needs

### General Use (Heavy)
Best for:
- Complex backgrounds
- Multiple subjects
- Higher quality requirements

### Portrait
Best for:
- Headshots and portraits
- Full-body photos
- Human subjects with clothing

### Matting
Best for:
- Hair and fine details
- Fur and fuzzy edges
- Semi-transparent elements

## Quality Optimization

### High Quality Processing

For maximum quality, combine Portrait model with refinement:

```bash
bun run - << 'EOF'
import { fal } from "@fal-ai/client";
import { readFileSync, writeFileSync } from "fs";
import { basename, dirname, join } from "path";

const imagePath = "INPUT_IMAGE_PATH";
const imageBuffer = readFileSync(imagePath);
const fileName = basename(imagePath);
const mimeType = imagePath.endsWith(".png") ? "image/png" : "image/jpeg";

const file = new File([imageBuffer], fileName, { type: mimeType });
const imageUrl = await fal.storage.upload(file);

const result = await fal.subscribe("fal-ai/birefnet/v2", {
  input: {
    image_url: imageUrl,
    model: "Matting",
    operating_resolution: "2048x2048",
    refine_foreground: true,
    output_format: "png"
  },
  logs: true,
  onQueueUpdate: (update) => {
    if (update.status === "IN_PROGRESS" && update.logs) {
      update.logs.map((log) => log.message).forEach(console.log);
    }
  },
});

const response = await fetch(result.data.image.url);
const buffer = Buffer.from(await response.arrayBuffer());

const dir = dirname(imagePath);
const nameWithoutExt = basename(imagePath, basename(imagePath).match(/\.[^.]+$/)?.[0] || "");
const outputPath = join(dir, `${nameWithoutExt}-nobg-hq.png`);

writeFileSync(outputPath, buffer);
console.log("Saved:", outputPath);
EOF
```

## Combining with Image Generation

Remove background from a Nano Banana generated image:

1. First, generate an image with Nano Banana
2. Then remove its background:

```bash
bun run - << 'EOF'
import { fal } from "@fal-ai/client";
import { readFileSync, writeFileSync } from "fs";

const generatedImage = "tmp/generated.png";
const imageBuffer = readFileSync(generatedImage);

const file = new File([imageBuffer], "generated.png", { type: "image/png" });
const imageUrl = await fal.storage.upload(file);

const result = await fal.subscribe("fal-ai/birefnet/v2", {
  input: {
    image_url: imageUrl,
    model: "General Use (Heavy)",
    operating_resolution: "1024x1024",
    output_format: "png"
  },
});

const response = await fetch(result.data.image.url);
const buffer = Buffer.from(await response.arrayBuffer());

writeFileSync("tmp/generated-nobg.png", buffer);
console.log("Saved: tmp/generated-nobg.png");
EOF
```

## Error Handling

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `FAL_KEY not set` | Missing API key | Set `FAL_KEY` environment variable |
| `File not found` | Invalid image path | Check path exists and is readable |
| `Upload failed` | Image too large or network issue | Resize image or retry |
| `Model error` | Invalid model name | Use exact model names from docs |

### Robust Script Template

```bash
bun run - << 'EOF'
import { fal } from "@fal-ai/client";
import { readFileSync, writeFileSync, existsSync } from "fs";
import { basename, dirname, join } from "path";

const imagePath = "INPUT_IMAGE_PATH";

// Validate input
if (!existsSync(imagePath)) {
  console.error("Error: Image not found:", imagePath);
  process.exit(1);
}

if (!process.env.FAL_KEY) {
  console.error("Error: FAL_KEY environment variable not set");
  console.error("Get your key at: https://fal.ai/dashboard/keys");
  process.exit(1);
}

try {
  const imageBuffer = readFileSync(imagePath);
  const fileName = basename(imagePath);
  const mimeType = imagePath.endsWith(".png") ? "image/png" : "image/jpeg";

  console.log("Uploading image...");
  const file = new File([imageBuffer], fileName, { type: mimeType });
  const imageUrl = await fal.storage.upload(file);

  console.log("Processing with BiRefNet...");
  const result = await fal.subscribe("fal-ai/birefnet/v2", {
    input: {
      image_url: imageUrl,
      model: "General Use (Light)",
      operating_resolution: "1024x1024",
      output_format: "png"
    },
    logs: true,
    onQueueUpdate: (update) => {
      if (update.status === "IN_PROGRESS" && update.logs) {
        update.logs.map((log) => log.message).forEach(console.log);
      }
    },
  });

  console.log("Downloading result...");
  const response = await fetch(result.data.image.url);
  const buffer = Buffer.from(await response.arrayBuffer());

  const dir = dirname(imagePath);
  const nameWithoutExt = basename(imagePath, basename(imagePath).match(/\.[^.]+$/)?.[0] || "");
  const outputPath = join(dir, `${nameWithoutExt}-nobg.png`);

  writeFileSync(outputPath, buffer);
  console.log("Success! Saved:", outputPath);

} catch (error) {
  console.error("Failed:", error.message);
  process.exit(1);
}
EOF
```

## Performance Tips

1. **Use Light model** for fast processing when quality is acceptable
2. **1024x1024 resolution** is usually sufficient for web use
3. **Skip logging** for batch processing: remove `logs: true` and `onQueueUpdate`
4. **Process in parallel** for large batches using Promise.all (be mindful of rate limits)

## API Reference

### Input Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `image_url` | string | required | URL of image to process |
| `model` | string | "General Use (Light)" | Model variant |
| `operating_resolution` | string | "1024x1024" | Processing resolution |
| `refine_foreground` | boolean | false | Improve edge quality |
| `output_format` | string | "png" | Output format (always use png) |

### Output

```typescript
interface Result {
  data: {
    image: {
      url: string;
      width: number;
      height: number;
      content_type: string;
      file_name: string;
    }
  }
}
```
