---
name: adk-go-agent
description: "Guide for building AI agents in Go using adk-go framework. Use when creating agents with Gemini or OpenAI models, implementing tools, running agents with streaming, or managing conversation sessions."
Triggers: "Go agent development, adk-go, llmagent, google.golang.org/adk."
---

# ADK-Go Agent Development

## Quick Start

### 1. Install Dependency

```bash
go get google.golang.org/adk
```

### 2. Create Model

#### Gemini

```go
import (
    "google.golang.org/adk/model/gemini"
    "google.golang.org/genai"
)

// Use lock if changing base URL (genai lacks API for this)
envLock.Lock()
if baseURL != "" {
    os.Setenv("GOOGLE_GEMINI_BASE_URL", baseURL)
} else {
    os.Unsetenv("GOOGLE_GEMINI_BASE_URL")
}

llmModel, err := gemini.NewModel(ctx, modelName, &genai.ClientConfig{
    APIKey: apiKey,
})

os.Unsetenv("GOOGLE_GEMINI_BASE_URL")
envLock.Unlock()
```

#### OpenAI

```bash
go get github.com/byebyebruce/adk-go-openai
```

```go
import (
    openai "github.com/byebyebruce/adk-go-openai"
    go_openai "github.com/sashabaranov/go-openai"
)

cfg := go_openai.DefaultConfig(os.Getenv("OPENAI_API_KEY"))
if baseURL := os.Getenv("OPENAI_BASE_URL"); baseURL != "" {
    cfg.BaseURL = baseURL
}
model := openai.NewOpenAIModel(modelName, cfg)
```

### 3. Wrap with RetryLLM

Always wrap model with RetryLLM for resilience. See [references/retry_llm.md](references/retry_llm.md) for full implementation.

```go
retryLLM := &RetryLLM{
    llm:        llmModel,
    maxRetries: 3,
}
```

### 4. Create Tools

```go
import "google.golang.org/adk/tool/functiontool"

type SetTitleInput struct {
    Title string `json:"title"`
}

type SetTitleOutput struct{}

setTitleTool, _ := functiontool.New(functiontool.Config{
    Name:        "set_title",
    Description: "set the title of the project",
}, func(ctx tool.Context, input SetTitleInput) (SetTitleOutput, error) {
    // Implementation
    return SetTitleOutput{}, nil
})
```

### 5. Create Agent

```go
import (
    "google.golang.org/adk/agent/llmagent"
    "google.golang.org/adk/tool"
    "google.golang.org/genai"
)

agent, err := llmagent.New(llmagent.Config{
    Name:        "my-agent",
    Model:       retryLLM,
    Description: "Agent description",
    Instruction: "System prompt - describe tools usage here",
    Tools:       []tool.Tool{setTitleTool},
    GenerateContentConfig: &genai.GenerateContentConfig{
        MaxOutputTokens: 4096,
    },
})
```

**Important**: Describe tool usage in `Instruction` (system prompt).

### 6. Run Agent

```go
import (
    adkagent "google.golang.org/adk/agent"
    "google.golang.org/adk/runner"
    "google.golang.org/adk/session"
)

// Create session service
sessionService := session.InMemoryService()

// Create runner
r, err := runner.New(runner.Config{
    AppName:        "my-app",
    Agent:          agent,
    SessionService: sessionService,
})

// Run with streaming
runConfig := adkagent.RunConfig{
    StreamingMode: adkagent.StreamingModeSSE,
}

fullResponse := ""
for event, err := range r.Run(ctx, userID, sessionID, userContent, runConfig) {
    if err != nil {
        break
    }
    if event == nil {
        continue
    }
    if event.Content != nil {
        for _, part := range event.Content.Parts {
            if part.Text != "" && event.Partial {
                fullResponse += part.Text
            }
        }
    }
}
```

### 7. Clean Up Session

For single-turn conversations, delete session after use:

```go
err := sessionService.Delete(ctx, &session.DeleteRequest{
    AppName:   "my-app",
    UserID:    userID,
    SessionID: sessionID,
})
```

## Required Imports Summary

```go
import (
    adkagent "google.golang.org/adk/agent"
    "google.golang.org/adk/agent/llmagent"
    adkmodel "google.golang.org/adk/model"
    "google.golang.org/adk/model/gemini"
    "google.golang.org/adk/runner"
    "google.golang.org/adk/session"
    "google.golang.org/adk/tool"
    "google.golang.org/adk/tool/functiontool"
    "google.golang.org/genai"
)
```

## Resources

- [references/retry_llm.md](references/retry_llm.md) - Complete RetryLLM implementation with exponential backoff