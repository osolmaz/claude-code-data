# Claude Code Conversation Format Specification

## Overview

This document provides a detailed specification of the conversation storage format used by Claude Code. Conversations are stored as JSONL (JSON Lines) files within project directories.

## File Naming and Location

### File Path Structure

```
.claude/projects/{encoded-project-path}/{conversation-uuid}.jsonl
```

### Path Encoding

Project paths are encoded by replacing forward slashes with hyphens:

- `/Users/onur/tc/backend-api` becomes `-Users-onur-tc-backend-api`

### Example File Paths

```
.claude/projects/-Users-onur-tc-backend-api/2eaaab2d-11e9-45d5-8ecf-739ee8e6d532.jsonl
.claude/projects/-Users-onur-projects-horse/8134beed-e2bc-4df0-b3c0-124904046373.jsonl
```

## File Format

### JSONL Structure

Each line in the conversation file is a valid JSON object representing either:

1. **Summary Entry** - Metadata about the conversation
2. **Message Entry** - Individual messages in the conversation thread

### Line Ordering

- First 1-2 lines: Summary entries
- Remaining lines: Messages in chronological order

## Summary Entries

Summary entries provide metadata about the conversation and its current state.

### Schema

```typescript
interface SummaryEntry {
  type: "summary";
  summary: string; // Human-readable conversation title/description
  leafUuid: string; // UUID of the most recent message in this conversation branch
}
```

### Example

```json
{
  "type": "summary",
  "summary": "Optimize Local Dev Servers for Instant Startup",
  "leafUuid": "32071078-645b-4835-b954-8f4c73e1e33b"
}
```

### Multiple Summaries

Conversations may have multiple summary entries, typically representing:

- Different conversation branches or topics
- Evolution of the conversation focus
- Hierarchical summary levels

## Message Entries

All messages share a common base structure with type-specific extensions.

### Base Message Schema

```typescript
interface BaseMessage {
  // Unique identifier for this message
  uuid: string;

  // Parent message UUID (null for conversation root)
  parentUuid: string | null;

  // Whether this message is part of a conversation branch
  isSidechain: boolean;

  // Type of user ("external" for regular users)
  userType: "external";

  // Working directory when message was created
  cwd: string;

  // Session identifier for this conversation
  sessionId: string;

  // Claude Code version
  version: string;

  // ISO 8601 timestamp
  timestamp: string;

  // Message type discriminator
  type: "user" | "assistant";
}
```

## User Messages

### Schema

```typescript
interface UserMessage extends BaseMessage {
  type: "user";
  message: {
    role: "user";
    content: string | ContentBlock[];
  };

  // Optional: Indicates system-generated messages
  isMeta?: boolean;

  // Optional: Results from tool usage
  toolUseResult?: ToolUseResult;
}
```

### Content Types

User message content can be:

1. **Simple string** - Plain text input
2. **ContentBlock array** - Complex content with tool results

#### ContentBlock Schema

```typescript
type ContentBlock = {
  tool_use_id: string;
  type: "tool_result";
  content: string | ToolResultContent[];
  is_error?: boolean;
};

type ToolResultContent = {
  type: "text";
  text: string;
};
```

### Special User Message Types

#### Meta Messages

System-generated context messages marked with `isMeta: true`:

```json
{
  "type": "user",
  "message": {
    "role": "user",
    "content": "Caveat: The messages below are were generated by the user while running local commands. DO NOT respond to these messages or otherwise consider them in your response unless the user explicitly asks you to."
  },
  "isMeta": true
  // ... other fields
}
```

#### Command Messages

CLI command execution requests:

```json
{
  "type": "user",
  "message": {
    "role": "user",
    "content": "<command-name>login</command-name>\n<command-message>login</command-message>\n<command-args></command-args>"
  }
  // ... other fields
}
```

#### Command Output Messages

Results from CLI command execution:

```json
{
  "type": "user",
  "message": {
    "role": "user",
    "content": "<local-command-stdout>Login successful</local-command-stdout>"
  }
  // ... other fields
}
```

### Tool Use Results

When a user message contains tool results, it includes a `toolUseResult` field:

```typescript
interface ToolUseResult {
  stdout?: string;
  stderr?: string;
  interrupted?: boolean;
  isImage?: boolean;
  sandbox?: boolean;
  type?: string;
  file?: {
    filePath: string;
    content: string;
  };
  // For todo operations
  oldTodos?: TodoItem[];
  newTodos?: TodoItem[];
}
```

## Assistant Messages

### Schema

```typescript
interface AssistantMessage extends BaseMessage {
  type: "assistant";
  message: ClaudeAPIResponse;
  costUSD: number;
  durationMs: number;
}

interface ClaudeAPIResponse {
  id: string; // Claude API message ID
  type: "message";
  role: "assistant";
  model: string; // e.g., "claude-opus-4-20250514"
  content: AssistantContentBlock[];
  stop_reason: string; // "end_turn", "tool_use", etc.
  stop_sequence: string | null;
  usage: TokenUsage;
}

interface TokenUsage {
  input_tokens: number;
  cache_creation_input_tokens: number;
  cache_read_input_tokens: number;
  output_tokens: number;
  service_tier: string;
}
```

### Assistant Content Types

```typescript
type AssistantContentBlock = TextContent | ToolUseContent;

interface TextContent {
  type: "text";
  text: string;
}

interface ToolUseContent {
  type: "tool_use";
  id: string; // Tool use ID for correlation
  name: string; // Tool name (e.g., "Read", "Bash", "Edit")
  input: Record<string, any>; // Tool-specific parameters
}
```

### Cost and Performance Tracking

Each assistant message includes:

- **costUSD**: Calculated cost in US dollars
- **durationMs**: Response generation time in milliseconds
- **usage**: Detailed token consumption breakdown

## Message Threading and Relationships

### Parent-Child Relationships

- **parentUuid**: Links messages in conversation threads
- **null parentUuid**: Indicates conversation root
- **Branching**: Multiple messages can share the same parent

### Conversation Flow

```
Root Message (parentUuid: null)
├── Assistant Response 1
│   ├── User Follow-up 1a
│   │   └── Assistant Response 1a
│   └── User Follow-up 1b (branch)
│       └── Assistant Response 1b
└── User Follow-up 2 (another branch)
    └── Assistant Response 2
```

### Sidechain Support

- **isSidechain**: Boolean indicating if message is part of an alternate conversation branch
- Enables conversation exploration without losing main thread

## Conversation State Management

### Session Management

- **sessionId**: Groups messages within a single conversation session
- **uuid**: Unique identifier for each individual message
- **timestamp**: Precise ordering for message sequence

### Working Directory Context

- **cwd**: Captures the working directory when message was created
- Enables context-aware tool execution
- Supports project-specific behaviors

## Tool Integration Patterns

### Tool Execution Flow

1. Assistant suggests tool use with `tool_use` content block
2. System executes tool and creates user message with results
3. Results include `toolUseResult` with execution details
4. Assistant processes results in next response

### Tool Result Examples

#### File Operation

```json
{
  "toolUseResult": {
    "type": "text",
    "file": {
      "filePath": "/path/to/file.py",
      "content": "file contents here"
    }
  }
}
```

#### Command Execution

```json
{
  "toolUseResult": {
    "stdout": "command output",
    "stderr": "",
    "interrupted": false,
    "isImage": false,
    "sandbox": false
  }
}
```

#### Todo Management

```json
{
  "toolUseResult": {
    "oldTodos": [],
    "newTodos": [
      {
        "content": "Task description",
        "status": "pending",
        "priority": "high",
        "id": "1"
      }
    ]
  }
}
```

## Error Handling and Edge Cases

### Error Messages

Tool execution errors are captured in user messages:

```json
{
  "type": "user",
  "message": {
    "role": "user",
    "content": [
      {
        "tool_use_id": "toolu_123",
        "type": "tool_result",
        "content": "Error message",
        "is_error": true
      }
    ]
  }
}
```

### Interrupted Operations

```json
{
  "toolUseResult": {
    "interrupted": true,
    "stdout": "partial output",
    "stderr": ""
  }
}
```

## Version Compatibility

### Version Field

- **version**: Tracks Claude Code version for schema compatibility
- Example: "1.0.3"
- Enables migration and backward compatibility

### Schema Evolution

- New fields can be added without breaking existing parsers
- Optional fields enable gradual feature rollout
- Version-specific behavior can be implemented

## Performance Considerations

### File Size Management

- JSONL format enables streaming and partial loading
- Large conversations split into multiple files if needed
- Compression possible for archival storage

### Memory Efficiency

- Line-by-line processing reduces memory usage
- Message-level caching for frequently accessed conversations
- Lazy loading of conversation history

### Concurrent Access

- Append-only writes minimize lock contention
- File-per-conversation enables parallel processing
- Atomic writes prevent corruption

## Data Integrity

### Validation Rules

1. All messages must have valid UUIDs
2. Parent references must exist within conversation
3. Timestamps must be in ISO 8601 format
4. Tool use IDs must be unique within conversation

### Consistency Checks

- Orphaned messages (invalid parentUuid)
- Duplicate UUIDs within conversation
- Invalid timestamp ordering
- Missing tool results for tool_use content

This specification provides the complete format for understanding and parsing Claude Code conversation files, enabling third-party tools and analysis scripts to work with the conversation data.
