# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository contains the **official Raycast API documentation** (in the `docs/` folder) and is used as a reference for developing Raycast extensions.

## Prerequisites

Before developing Raycast extensions, ensure you have:
- **Raycast 1.26.0 or higher** installed
- **Node.js 22.14 or higher** (recommend using [nvm](https://github.com/nvm-sh/nvm))
- **npm 7 or higher**
- Familiarity with **React** and **TypeScript**
- **Signed in** to Raycast (required for extension development commands)

## Common Development Commands

```bash
# Create a new extension (via Raycast UI)
# Open "Create Extension" command in Raycast

# Development with hot-reload
npm run dev
# or: ray develop

# Build for production
npm run build
# or: ray build

# Linting
npm run lint
# or: ray lint

# Auto-fix linting issues
npm run fix-lint
# or: ray lint --fix

# Publish to Raycast Store
npm run publish
```

## Extension Structure

```bash
extension/
├── src/                       # Source files
│   └── index.tsx             # Entry point for commands
├── assets/                   # Icons (512x512px PNG)
├── package.json              # Manifest file
├── tsconfig.json             # TypeScript config
├── eslint.config.js          # ESLint configuration
└── .prettierrc               # Prettier config (printWidth: 120, singleQuote: false)
```

**Important**: `raycast-env.d.ts` is auto-generated from `package.json` - **NEVER EDIT MANUALLY**.

## AI Development

### AI vs. AI Extensions

There are **two ways** to use AI in extensions:

1. **AI APIs** - Use `AI.ask()` to generate content (summaries, translations, etc.) within regular extensions
2. **AI Extensions** - Create tools for natural language interactions in Quick AI, AI Chat, and AI Commands

**Requirements**:
- [Raycast Pro](https://raycast.com/pro) subscription required
- AI Extensions not available on Windows

### AI APIs

Basic usage in "no-view" Commands:

```typescript
import { AI, Clipboard } from "@raycast/api";

export default async function command() {
  const answer = await AI.ask("Suggest 5 jazz songs");
  await Clipboard.copy(answer);
}
```

**Streaming**:

```typescript
import { AI } from "@raycast/api";

export default async function main() {
  let allData = "";
  const answer = AI.ask("Suggest 5 jazz songs");

  answer.on("data", async (data) => {
    allData += data;
    // Stream output incrementally
  });

  await answer;
}
```

**Check access**:

```typescript
import { AI, environment } from "@raycast/api";

if (environment.canAccess(AI)) {
  // User has AI access
}
```

### AI Extensions

AI Extensions allow natural language interactions through **tools** that AI can call.

#### Core Concepts

1. **Tools** - Functions that take input and return values
2. **Instructions** - AI-wide guidance (defined in `package.json` under `ai` key)
3. **Evals** - Integration tests for AI behavior

#### Creating a Tool

```typescript
type Input = {
  /**
   * The first name of the user to greet
   */
  name: string;
};

/**
 * Greet the user with a friendly message
 */
export default function tool(input: Input) {
  return `Hello, ${input.name}!`;
}
```

**Tool with Confirmation** (keep human in loop):

```typescript
import { Tool } from "@raycast/api";

type Input = {
  name: string;
};

export const confirmation: Tool.Confirmation<Input> = async (input) => {
  return {
    message: `Are you sure you want to greet ${input.name}?`,
  };
};

export default function tool(input: Input) {
  return `Hello, ${input.name}!`;
}
```

#### Adding Tools

Add tools via "Manage Extensions" command (`⌥⌘T`) or edit `package.json`:

```json
{
  "tools": [
    {
      "name": "greet",
      "description": "Greet a user by name",
      "inputParameters": {
        "name": "string"
      }
    }
  ]
}
```

#### Instructions

Define AI-wide behavior in `package.json`:

```json
{
  "ai": {
    "instructions": "When you don't know the user's first name, ask for it."
  }
}
```

**Best practices for instructions**:
- Avoid phrases like "You are a ... assistant" (conflicts with other AI Extensions)
- Focus on extension-specific details (e.g., relationships between issues, projects, teams)
- Keep general and non-conflicting

#### AI Files

For long instructions, use separate file (`ai.json`, `ai.yaml`, or `ai.json5`):

```yaml
# ai.yaml
instructions: |
  When you don't know the user's first name, ask for it.
```

#### Evals (Testing)

Define in `package.json` under `ai` key:

```json
{
  "ai": {
    "evals": [
      {
        "input": "@my-extension Greet Thomas",
        "mocks": {
          "greet": "Hello, Thomas!"
        },
        "expected": [
          {
            "callsTool": {
              "name": "greet",
              "arguments": {
                "name": "thomas"
              }
            }
          }
        ]
      }
    ]
  }
}
```

**Expectation types**:
- `includes`: Check response includes substring
- `matches`: Check response matches regex
- `meetsCriteria`: AI-validated criteria
- `callsTool`: Check if specific tool was called (short or long form)

**Example expectations**:

```json
{
  "expected": [
    { "callsTool": "greet" },
    {
      "callsTool": {
        "name": "create-comment",
        "arguments": {
          "body": { "includes": "waiting for design" }
        }
      }
    },
    {
      "not": {
        "callsTool": "create-issue"
      }
    }
  ]
}
```

### AI Best Practices

1. **Use Confirmations** - Keep human in loop for destructive actions
2. **Write Evals** - Test common use-cases and provide user prompts
3. **Describe Input Formats** - Specify ISO 8601 for dates, ID formats, etc.
4. **Teach Parameter Retrieval** - Explain how to get required parameters (e.g., team IDs)
5. **Use JSDoc Comments** - Better tool descriptions = better AI usage

### AI Models

Available models include:
- **OpenAI**: GPT-5, GPT-4.1, GPT-4o, o1, o3 (various variants)
- **Anthropic**: Claude 4.5 Sonnet, Claude 4 Opus, Claude 3.5 Haiku
- **Google**: Gemini 3 Pro, Gemini 2.5 Pro/Flash
- **Others**: Groq, Mistral, Perplexity, xAI Grok, etc.

Specify via `AI.ask` options:

```typescript
await AI.ask("Prompt", {
  model: AI.Model.Anthropic_Claude_4_5_Sonnet,
  creativity: "medium"
});
```

**Creativity levels**: `"none"` | `"low"` | `"medium"` | `"high"` | `"maximum"` | number (0-2)

## Manifest (package.json)

Key properties:

**Extension**:
- `name`*: Unique name (URL-compatible)
- `title`*: Display name in Store
- `description`*: Full description
- `icon`*: 512x512px PNG (`icon.png` and `icon@dark.png` for themes)
- `author`*: Raycast Store handle
- `platforms`*: `["macOS"]`, `["Windows"]`, or both
- `categories`*: e.g., `["Productivity", "Developer Tools"]`
- `commands`*: Array of commands
- `tools`: Array of AI tools
- `ai`: AI configuration (instructions, evals)

**Commands**:
- `name`*: Maps to `src/<name>.{ts,tsx,js,jsx}`
- `title`*: Display name
- `mode`*: `"view"` (has UI), `"no-view"` (no UI), `"menu-bar"`
- `description`*: What the command does
- `interval`: Background interval (e.g., "90s", "1m", "12h", "1d")

## Best Practices

### General

1. **Handle Errors Gracefully**
   - Show Toast with error information
   - Use cached data if network fails
   - Don't disrupt user flow

2. **Show Loading Indicators**
   - Use `isLoading` prop on List, Detail, Form, Grid
   - Render quickly, load data asynchronously

3. **Handle Runtime Dependencies**
   - Check if required apps/CLIs are installed
   - Show helpful messages if missing
   - Optionally hide features if dependency unavailable

### AI-Specific

1. **Use Confirmations** for destructive actions
2. **Write Evals** for common use-cases
3. **Include formatting info** in input JSDoc (ISO dates, ID formats)
4. **Explain parameter retrieval** in descriptions
5. **Avoid conflicting instructions** with other AI Extensions

## Key Documentation Files

- `/Users/elwin/Documents/Codes/raycast_extensions/docs/SUMMARY.md` - Complete documentation index
- `/Users/elwin/Documents/Codes/raycast_extensions/docs/ai/getting-started.md` - AI development overview
- `/Users/elwin/Documents/Codes/raycast_extensions/docs/ai/create-an-ai-extension.md` - AI Extension guide
- `/Users/elwin/Documents/Codes/raycast_extensions/docs/ai/learn-core-concepts-of-ai-extensions.md` - Tools, Instructions, Evals
- `/Users/elwin/Documents/Codes/raycast_extensions/docs/ai/follow-best-practices-for-ai-extensions.md` - AI best practices
- `/Users/elwin/Documents/Codes/raycast_extensions/docs/api-reference/ai.md` - AI API reference
- `/Users/elwin/Documents/Codes/raycast_extensions/docs/api-reference/tool.md` - Tool API reference
- `/Users/elwin/Documents/Codes/raycast_extensions/docs/information/manifest.md` - Complete manifest reference
- `/Users/elwin/Documents/Codes/raycast_extensions/docs/information/best-practices.md` - General best practices
- `/Users/elwin/Documents/Codes/raycast_extensions/docs/basics/create-your-first-extension.md` - First extension tutorial

## Technology Stack

- **Runtime**: Node.js with Raycast API
- **Language**: TypeScript (strict mode, ES2022/ES2023)
- **UI**: React with `@raycast/api` components
- **Build**: `ray build` CLI tool
- **Linting**: ESLint with `@raycast/eslint-config`
- **Formatting**: Prettier (120 char width, double quotes)
