# RetryLLM Implementation

Complete implementation of RetryLLM wrapper with exponential backoff.

```go
package yourpackage

import (
	"context"
	"iter"
	"time"

	adkmodel "google.golang.org/adk/model"
)

// RetryLLM wraps an LLM with automatic retry on errors
type RetryLLM struct {
	llm        adkmodel.LLM
	maxRetries int
}

// NewRetryLLM creates a new RetryLLM wrapper
func NewRetryLLM(llm adkmodel.LLM, maxRetries int) *RetryLLM {
	return &RetryLLM{
		llm:        llm,
		maxRetries: maxRetries,
	}
}

// Name returns the LLM name
func (r *RetryLLM) Name() string {
	return r.llm.Name()
}

// GenerateContent generates content with automatic retry on errors
func (r *RetryLLM) GenerateContent(ctx context.Context, req *adkmodel.LLMRequest, stream bool) iter.Seq2[*adkmodel.LLMResponse, error] {
	return func(yield func(*adkmodel.LLMResponse, error) bool) {
		for attempt := 0; attempt <= r.maxRetries; attempt++ {
			if attempt > 0 {
				// Exponential backoff: 2s, 4s, 8s...
				waitTime := (1 << uint(attempt-1)) * 2
				time.Sleep(time.Duration(waitTime) * time.Second)
			}

			hasError := false
			var lastError error

			for resp, err := range r.llm.GenerateContent(ctx, req, stream) {
				if err != nil {
					lastError = err
					hasError = true

					if attempt < r.maxRetries {
						// More retries available, break to retry
						break
					} else {
						// Last attempt, pass error to caller
						if !yield(resp, err) {
							return
						}
					}
				} else {
					// Success, pass response to caller
					if !yield(resp, nil) {
						return
					}
				}
			}

			// No error, success
			if !hasError {
				return
			}

			// Reached max retries
			if attempt >= r.maxRetries {
				_ = lastError // Can log here if needed
				return
			}
		}
	}
}
```

## Usage

```go
llmModel, err := gemini.NewModel(ctx, modelName, &genai.ClientConfig{
    APIKey: apiKey,
})
if err != nil {
    return err
}

retryLLM := NewRetryLLM(llmModel, 3)

// Use retryLLM when creating agent
agent, err := llmagent.New(llmagent.Config{
    Model: retryLLM,
    // ...
})