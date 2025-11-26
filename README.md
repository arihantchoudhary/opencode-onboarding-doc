# OpenCode Development Onboarding Guide

A complete guide to getting started with OpenCode development and adding custom AI models.

## Table of Contents

1. [Getting Started with OpenCode](#getting-started-with-opencode)
2. [Understanding OpenCode Architecture](#understanding-opencode-architecture)
3. [Adding Custom Models (e.g., Cerebras)](#adding-custom-models)
4. [Building CLI Tools](#building-cli-tools)
5. [Testing Your Changes](#testing-your-changes)
6. [Contributing](#contributing)

## Getting Started with OpenCode

### What is OpenCode?

OpenCode is an AI-powered coding assistant that integrates with various AI models to help developers write, debug, and understand code.

### Prerequisites

- Node.js (v18 or higher)
- npm or yarn
- Git
- Basic understanding of TypeScript
- API keys for the models you want to use

### Initial Setup

```bash
# Clone the OpenCode repository
git clone https://github.com/your-org/opencode.git
cd opencode

# Install dependencies
npm install

# Build the project
npm run build

# Run in development mode
npm run dev
```

## Understanding OpenCode Architecture

### Key Components

1. **Model Providers** - Handle communication with different AI services
2. **CLI Interface** - Command-line tool for user interaction
3. **Configuration** - Settings and API key management
4. **Tools** - Various utilities and helpers

### Directory Structure

```
opencode/
├── src/
│   ├── models/          # Model provider implementations
│   ├── cli/             # CLI tool logic
│   ├── config/          # Configuration management
│   ├── tools/           # Helper tools and utilities
│   └── types/           # TypeScript type definitions
├── tests/               # Test files
└── package.json
```

## Adding Custom Models

### Example: Adding Cerebras Model Support

Here's a step-by-step guide to add a new model provider like Cerebras:

#### Step 1: Create Model Provider File

Create a new file in `src/models/cerebras.ts`:

```typescript
import { ModelProvider } from '../types/model';

export class CerebrasProvider implements ModelProvider {
  private apiKey: string;
  private baseUrl: string = 'https://api.cerebras.ai/v1';

  constructor(apiKey: string) {
    this.apiKey = apiKey;
  }

  async chat(messages: any[], options?: any) {
    // Implement chat completion logic
    const response = await fetch(`${this.baseUrl}/chat/completions`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        model: options?.model || 'llama3.1-8b',
        messages: messages,
        temperature: options?.temperature || 0.7,
        max_tokens: options?.maxTokens || 2048
      })
    });

    return await response.json();
  }

  async streamChat(messages: any[], options?: any) {
    // Implement streaming logic if supported
    // Similar to chat() but with streaming support
  }
}
```

#### Step 2: Register the Model Provider

Update `src/models/index.ts`:

```typescript
import { CerebrasProvider } from './cerebras';
import { OpenAIProvider } from './openai';
import { AnthropicProvider } from './anthropic';

export const MODEL_PROVIDERS = {
  'openai': OpenAIProvider,
  'anthropic': AnthropicProvider,
  'cerebras': CerebrasProvider,  // Add your new provider
};

export function createModelProvider(provider: string, apiKey: string) {
  const Provider = MODEL_PROVIDERS[provider];
  if (!Provider) {
    throw new Error(`Unknown provider: ${provider}`);
  }
  return new Provider(apiKey);
}
```

#### Step 3: Update Configuration

Update `src/config/models.ts` to include Cerebras models:

```typescript
export const AVAILABLE_MODELS = {
  // ... existing models
  cerebras: {
    'llama3.1-8b': {
      contextWindow: 8192,
      maxTokens: 2048,
      pricing: { input: 0.10, output: 0.10 } // per 1M tokens
    },
    'llama3.1-70b': {
      contextWindow: 8192,
      maxTokens: 2048,
      pricing: { input: 0.60, output: 0.60 }
    }
  }
};
```

#### Step 4: Add Configuration Options

Update your config file to accept Cerebras API key:

```typescript
// src/config/index.ts
export interface Config {
  openaiApiKey?: string;
  anthropicApiKey?: string;
  cerebrasApiKey?: string;  // Add this
  defaultProvider?: string;
  defaultModel?: string;
}
```

#### Step 5: Update CLI Commands

Add Cerebras option to CLI commands in `src/cli/configure.ts`:

```typescript
export async function configureModel() {
  const provider = await select({
    message: 'Select model provider:',
    choices: [
      { name: 'OpenAI', value: 'openai' },
      { name: 'Anthropic', value: 'anthropic' },
      { name: 'Cerebras', value: 'cerebras' },  // Add this
    ]
  });

  const apiKey = await input({
    message: `Enter your ${provider} API key:`
  });

  // Save configuration
  saveConfig({ [`${provider}ApiKey`]: apiKey });
}
```

## Building CLI Tools

### Creating a Simple CLI Tool

Here's how to create a CLI tool for Cerebras:

```typescript
// src/cli/cerebras-tool.ts
import { Command } from 'commander';
import { CerebrasProvider } from '../models/cerebras';
import { getConfig } from '../config';

export function createCerebrasCommand() {
  const command = new Command('cerebras');

  command
    .description('Use Cerebras models')
    .option('-m, --model <model>', 'Model to use', 'llama3.1-8b')
    .option('-p, --prompt <prompt>', 'Prompt to send')
    .action(async (options) => {
      const config = await getConfig();

      if (!config.cerebrasApiKey) {
        console.error('Cerebras API key not configured. Run: opencode configure');
        process.exit(1);
      }

      const provider = new CerebrasProvider(config.cerebrasApiKey);

      const response = await provider.chat([
        { role: 'user', content: options.prompt }
      ], { model: options.model });

      console.log(response.choices[0].message.content);
    });

  return command;
}
```

### Register the CLI Command

Add to main CLI file `src/cli/index.ts`:

```typescript
import { createCerebrasCommand } from './cerebras-tool';

const program = new Command();

program
  .name('opencode')
  .description('AI-powered coding assistant')
  .version('1.0.0');

// ... other commands
program.addCommand(createCerebrasCommand());

program.parse();
```

## Testing Your Changes

### Unit Tests

Create test file `tests/cerebras.test.ts`:

```typescript
import { CerebrasProvider } from '../src/models/cerebras';

describe('CerebrasProvider', () => {
  it('should create provider with API key', () => {
    const provider = new CerebrasProvider('test-key');
    expect(provider).toBeDefined();
  });

  it('should make chat completion request', async () => {
    const provider = new CerebrasProvider(process.env.CEREBRAS_API_KEY!);
    const response = await provider.chat([
      { role: 'user', content: 'Hello' }
    ]);
    expect(response).toBeDefined();
  });
});
```

### Manual Testing

```bash
# Build your changes
npm run build

# Test the CLI
./dist/cli.js cerebras --prompt "Write a hello world in Python"

# Or if installed globally
opencode cerebras --prompt "Write a hello world in Python"
```

## Development Workflow

### 1. Set Up Development Environment

```bash
# Install in development mode
npm install

# Watch for changes
npm run dev
```

### 2. Make Your Changes

- Add new model provider
- Update configuration
- Add CLI commands
- Write tests

### 3. Test Locally

```bash
# Run tests
npm test

# Build the project
npm run build

# Test CLI locally
node dist/cli.js <your-command>
```

### 4. Submit Changes

```bash
git add .
git commit -m "Add Cerebras model support"
git push origin feature/cerebras-support
```

## Common Patterns

### Error Handling

```typescript
try {
  const response = await provider.chat(messages);
  return response;
} catch (error) {
  if (error.response?.status === 401) {
    throw new Error('Invalid API key');
  } else if (error.response?.status === 429) {
    throw new Error('Rate limit exceeded');
  }
  throw error;
}
```

### Environment Variables

```typescript
// .env file
CEREBRAS_API_KEY=your_api_key_here
OPENAI_API_KEY=your_openai_key

// Loading in code
import dotenv from 'dotenv';
dotenv.config();

const apiKey = process.env.CEREBRAS_API_KEY;
```

### Configuration Management

```typescript
import fs from 'fs';
import path from 'path';

const CONFIG_PATH = path.join(os.homedir(), '.opencode', 'config.json');

export function saveConfig(config: Partial<Config>) {
  const existing = loadConfig();
  const updated = { ...existing, ...config };
  fs.writeFileSync(CONFIG_PATH, JSON.stringify(updated, null, 2));
}

export function loadConfig(): Config {
  if (!fs.existsSync(CONFIG_PATH)) {
    return {};
  }
  return JSON.parse(fs.readFileSync(CONFIG_PATH, 'utf-8'));
}
```

## Tips and Best Practices

1. **Follow TypeScript Best Practices** - Use proper types and interfaces
2. **Handle Errors Gracefully** - Provide clear error messages
3. **Add Tests** - Write unit tests for new functionality
4. **Document Your Code** - Add comments and JSDoc
5. **Follow Existing Patterns** - Look at how other providers are implemented
6. **Test with Real API Keys** - Make sure your integration actually works
7. **Handle Rate Limits** - Implement retry logic and rate limiting
8. **Support Streaming** - If the API supports it, implement streaming responses

## Useful Resources

- [OpenCode GitHub Repository](https://github.com/your-org/opencode)
- [TypeScript Documentation](https://www.typescriptlang.org/docs/)
- [Commander.js (CLI framework)](https://github.com/tj/commander.js)
- API Documentation for your model provider (e.g., Cerebras, OpenAI, etc.)

## Getting Help

- Open an issue on GitHub
- Join our Discord community
- Check existing documentation and examples

## Contributing

We welcome contributions! Please:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests
5. Submit a pull request

## License

MIT License - see LICENSE file for details
