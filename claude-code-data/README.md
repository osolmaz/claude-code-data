# Claude Code Data

A TypeScript library for parsing and analyzing Claude Code conversation files (JSONL format).

## Features

- 📖 Parse Claude Code conversation JSONL files
- 🔍 Type-safe message handling with TypeScript
- 📊 Calculate conversation statistics (cost, tokens, duration)
- 🌳 Build conversation trees from flat message lists
- ✅ Comprehensive test coverage with Vitest
- 🚀 Built with Vite for fast development

## Installation

```bash
npm install
```

## Usage

### Basic Parsing

```typescript
import { parseConversation } from "claude-code-data";

const conversation = await parseConversation("./conversation.jsonl");

console.log(`Found ${conversation.summaries.length} summaries`);
console.log(`Found ${conversation.messages.length} messages`);
```

### Analyzing Conversations

```typescript
import {
  parseConversation,
  calculateConversationStats,
} from "claude-code-data";

const conversation = await parseConversation("./conversation.jsonl");
const stats = calculateConversationStats(conversation);

console.log(`Total cost: $${stats.totalCostUSD.toFixed(4)}`);
console.log(
  `Total tokens: ${stats.totalTokens.input + stats.totalTokens.output}`,
);
console.log(
  `Average response time: ${stats.averageResponseTimeMs.toFixed(0)}ms`,
);
```

### Type Guards

```typescript
import {
  isUserMessage,
  isAssistantMessage,
  hasToolResult,
} from "claude-code-data";

for (const message of conversation.messages) {
  if (isUserMessage(message)) {
    console.log("User:", message.message.content);
  } else if (isAssistantMessage(message)) {
    console.log("Assistant:", message.message.content[0].text);
    console.log("Cost:", message.costUSD);
  }
}
```

### Building Conversation Trees

```typescript
import { buildConversationTree } from "claude-code-data";

const tree = buildConversationTree(conversation.messages);
// Tree structure with parent-child relationships
```

## Development

```bash
# Run tests
npm test

# Run tests with UI
npm run test:ui

# Build
npm run build

# Type checking
npm run build
```

## API Reference

### Types

- `BaseMessage` - Common fields for all messages
- `UserMessage` - User message with optional tool results
- `AssistantMessage` - Assistant response with cost and usage data
- `SummaryMessage` - Conversation summary metadata
- `ConversationEntry` - Union of all message types

### Parser Functions

- `parseConversation(filePath)` - Parse entire conversation file
- `parseAndValidateConversation(filePath)` - Parse with validation
- `readConversationLines(filePath)` - Async generator for streaming

### Analyzer Functions

- `calculateConversationStats(conversation)` - Get comprehensive statistics
- `buildConversationTree(messages)` - Create hierarchical structure
- `getActiveBranch(conversation)` - Extract active conversation path

### Type Guards

- `isUserMessage(message)` - Check if message is from user
- `isAssistantMessage(message)` - Check if message is from assistant
- `isSummaryMessage(entry)` - Check if entry is a summary
- `hasToolResult(message)` - Check if user message has tool results

## Data Format Notes

This parser handles various Claude Code conversation formats:

- Messages may or may not include `costUSD` and `durationMs`
- Summary `leafUuid` may not always correspond to existing messages
- Tool usage appears in both assistant requests and user results
- Parent-child relationships enable conversation branching

## License

MIT
