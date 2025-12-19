# Background Remover Skill

A Claude Code skill that removes backgrounds from images using FAL.ai's BiRefNet v2 model.

## Setup

1. Install [Bun](https://bun.sh): `npm install -g bun`
2. Get a FAL.ai API key from https://fal.ai/dashboard/keys
3. Set the environment variable: `export FAL_KEY="your-key-here"`

## Usage

Ask Claude Code to remove the background from any image:

```
Remove the background from photo.jpg
```

The output is saved as `photo-nobg.png` with a transparent background.

## Model Options

| Model | Best For |
|-------|----------|
| General Use (Light) | Fast, everyday images |
| General Use (Heavy) | Higher quality |
| Portrait | Human subjects |
| Matting | Complex edges (hair, fur) |

## License

MIT
