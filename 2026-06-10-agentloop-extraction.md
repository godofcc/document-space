# AgentLoop 抽取设计与实现文档

## 1. 目标

将 `internal/agent/agent.go` 中 LLM 工具调用循环（tool-use loop）的核心逻辑抽取为独立的 `internal/agentloop` 包，使其可被其他工程项目复用。抽取后的包：

- **零业务依赖**：不知道 code review、diff、comment 是什么
- 只依赖 `internal/llm`（LLM 客户端接口）
- 提供通用的 "LLM + 工具调用" 循环能力
- 包含可选的上下文压缩功能

## 2. 三层架构

```
┌─────────────────────────────────────────────────────────────────┐
│  业务层（各项目自定义）                                            │
│  - agent.go 中的 code_comment 特殊处理                            │
│  - tools.json 定义、Review 模板                                    │
│  - Session 记录、CommentCollector 等                               │
├─────────────────────────────────────────────────────────────────┤
│  agentloop 包（本次抽取）                                         │
│  - RunLoop(): LLM 工具调用主循环                                   │
│  - 上下文压缩（可选）                                               │
│  - ToolProvider 接口                                               │
│  - LoopConfig / LoopResult                                         │
├─────────────────────────────────────────────────────────────────┤
│  llm 包（已独立，无需改动）                                         │
│  - LLMClient, ChatRequest, ChatResponse                           │
│  - Message, ToolCall, ToolDef                                      │
│  - OpenAI / Anthropic 双协议适配                                     │
└─────────────────────────────────────────────────────────────────┘
```

## 3. 依赖关系

```
agentloop → llm（唯一的外部依赖）
业务层 → agentloop + llm + tool + diff + session + ...
```

## 4. 完整实现代码

### 4.1 `internal/agentloop/types.go`

```go
package agentloop

import (
	"context"

	"github.com/open-code-review/open-code-review/internal/llm"
)

// ToolProvider is the interface that all tool implementations must satisfy.
// This is the generic equivalent of tool.Provider, but uses string names
// instead of a typed enum, making it framework-agnostic.
type ToolProvider interface {
	// Name returns the tool name used in tool definitions and LLM tool calls.
	Name() string
	// Execute runs the tool with the given arguments and returns the result string.
	Execute(args map[string]any) (string, error)
}

// CompletionSignal indicates whether the LLM invoked the completion tool
// (e.g. "task_done") and what state it reported.
type CompletionSignal struct {
	// Completed is true when the LLM called the designated completion tool.
	Completed bool
	// State holds the value of the completion parameter (e.g. "DONE" or "FAILED").
	State string
	// Data carries any additional output from the completion tool call.
	Data string
}

// LoopConfig holds configuration for the LLM tool-use loop.
type LoopConfig struct {
	// MaxRounds is the maximum number of LLM request rounds.
	// Each round = one LLM call + corresponding tool executions.
	MaxRounds int

	// MaxEmptyRounds is the maximum number of consecutive rounds where
	// the LLM returns no valid tool calls or all tool results are empty.
	MaxEmptyRounds int

	// EmptyRoundPrompt is the user message injected when the LLM returns
	// no tool calls. Default: "You did not successfully call any tools.
	// Please try again or use the completion tool if finished."
	EmptyRoundPrompt string

	// CompletionTool is the name of the tool that signals task completion.
	// When the LLM calls this tool, the loop exits. Default: "task_done".
	CompletionTool string

	// CompletionStateParam is the parameter name that carries the state
	// value in the completion tool call. Default: "state".
	CompletionStateParam string

	// ToolDefs are the tool definitions sent to the LLM in each request.
	ToolDefs []llm.ToolDef

	// Model is the LLM model name to use.
	Model string

	// MaxTokens is the maximum tokens per LLM request.
	MaxTokens int

	// Compression, if non-nil, enables context compression when token
	// counts exceed thresholds. Set to nil to disable compression.
	Compression *CompressionConfig

	// OnRoundStart is an optional callback invoked before each LLM request.
	// It receives the current round number (0-indexed) and the message history.
	OnRoundStart func(ctx context.Context, round int, messages []llm.Message)

	// OnLLMResponse is an optional callback invoked after each LLM response.
	// Receives the round number, response, and duration.
	OnLLMResponse func(ctx context.Context, round int, resp *llm.ChatResponse, duration float64)

	// OnToolCall is an optional callback invoked for each tool call execution.
	// Receives the tool name, arguments, and result.
	OnToolCall func(ctx context.Context, toolName string, args map[string]any, result string, err error)

	// OnToolResult is an optional callback for each tool result added to messages.
	OnToolResult func(ctx context.Context, toolName string, result string)
}

// CompressionConfig holds settings for context compression.
type CompressionConfig struct {
	// Client is the LLM client used for compression requests.
	Client llm.LLMClient

	// Model is the model name for compression requests.
	Model string

	// MaxTokens is the token budget for the main conversation.
	// Compression triggers when message tokens exceed thresholds.
	MaxTokens int

	// SoftThreshold is the fraction of MaxTokens at which async compression
	// starts in the background. Default: 0.60.
	SoftThreshold float64

	// WarnThreshold is the fraction of MaxTokens at which synchronous
	// compression is forced. Default: 0.80.
	WarnThreshold float64

	// TemplateMessages are the messages used as the compression LLM prompt.
	// Must contain a message with {{context}} placeholder.
	TemplateMessages []llm.Message

	// ContextPlaceholder is the string replaced with conversation context
	// XML in the template. Default: "{{context}}".
	ContextPlaceholder string

	// SummaryTag is the XML tag used to inject the compressed summary
	// back into the user message. Default: "<previous_review_summary>".
	SummaryTag string

	// FrozenZoneSize is the number of leading messages to always preserve
	// during compression. Default: 2 (system + first user).
	FrozenZoneSize int
}

// LoopResult holds the complete result of a tool-use loop execution.
type LoopResult struct {
	// CompletionState is the final state of the loop.
	// Possible values: "DONE", "FAILED", "MAX_ROUNDS", "EMPTY_ROUNDS",
	// "CONTEXT_OVERFLOW", "ERROR".
	CompletionState string

	// Messages is the full conversation history after the loop completes.
	Messages []llm.Message

	// Rounds is the number of LLM request rounds that were executed.
	Rounds int

	// ToolCallCount is the total number of tool calls across all rounds.
	ToolCallCount int

	// CompletionSignal holds the signal from the completion tool, if one
	// was called. Nil if the loop terminated without calling the completion tool.
	CompletionSignal *CompletionSignal

	// TotalInputTokens accumulates input/prompt tokens from all LLM calls.
	TotalInputTokens int64

	// TotalOutputTokens accumulates output/completion tokens from all LLM calls.
	TotalOutputTokens int64

	// TotalTokensUsed accumulates total tokens from all LLM calls.
	TotalTokensUsed int64

	// Err holds the error that caused the loop to terminate, if any.
	Err error
}

// Defaults fills in zero-valued fields with sensible defaults.
func (c *LoopConfig) Defaults() {
	if c.MaxRounds <= 0 {
		c.MaxRounds = 20
	}
	if c.MaxEmptyRounds <= 0 {
		c.MaxEmptyRounds = 3
	}
	if c.CompletionTool == "" {
		c.CompletionTool = "task_done"
	}
	if c.CompletionStateParam == "" {
		c.CompletionStateParam = "state"
	}
	if c.EmptyRoundPrompt == "" {
		c.EmptyRoundPrompt = "You did not successfully call any tools. Please try again or use " + c.CompletionTool + " if finished."
	}
	if c.MaxTokens <= 0 {
		c.MaxTokens = 32768
	}
	if c.Compression != nil {
		if c.Compression.SoftThreshold <= 0 {
			c.Compression.SoftThreshold = 0.60
		}
		if c.Compression.WarnThreshold <= 0 {
			c.Compression.WarnThreshold = 0.80
		}
		if c.Compression.FrozenZoneSize <= 0 {
			c.Compression.FrozenZoneSize = 2
		}
		if c.Compression.ContextPlaceholder == "" {
			c.Compression.ContextPlaceholder = "{{context}}"
		}
		if c.Compression.SummaryTag == "" {
			c.Compression.SummaryTag = "<previous_review_summary>"
		}
	}
}
```

### 4.2 `internal/agentloop/loop.go`

```go
package agentloop

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"sync"
	"sync/atomic"
	"time"

	"github.com/open-code-review/open-code-review/internal/llm"
)

// RunLoop executes the core LLM tool-use loop:
// send messages → receive LLM response → parse tool calls → execute tools →
// append results → repeat until completion or limits reached.
//
// This function is the generic equivalent of Agent.performLlmCodeReview,
// stripped of all code-review-specific logic.
func RunLoop(ctx context.Context, client llm.LLMClient, messages []llm.Message, config LoopConfig, providers []ToolProvider) *LoopResult {
	config.Defaults()

	result := &LoopResult{
		Messages: messages,
	}

	// Build a name → provider lookup map.
	providerMap := make(map[string]ToolProvider, len(providers))
	for _, p := range providers {
		providerMap[p.Name()] = p
	}

	// Compression state.
	var compState *compressionState
	if config.Compression != nil {
		compState = &compressionState{
			cfg: config.Compression,
		}
	}

	roundLimit := config.MaxRounds
	consecutiveEmptyRounds := 0

	for roundLimit > 0 {
		select {
		case <-ctx.Done():
			result.CompletionState = "ERROR"
			result.Err = ctx.Err()
			return result
		default:
		}

		roundLimit--
		result.Rounds++

		if config.OnRoundStart != nil {
			config.OnRoundStart(ctx, result.Rounds-1, result.Messages)
		}

		// Apply any completed async compression before sending the next request.
		if compState != nil {
			compState.tryApply(&result.Messages)
		}

		startTime := time.Now()
		resp, err := client.CompletionsWithCtx(ctx, llm.ChatRequest{
			Model:     config.Model,
			Messages:  result.Messages,
			Tools:     config.ToolDefs,
			MaxTokens: config.MaxTokens,
		})
		duration := time.Since(startTime).Seconds()

		if err != nil {
			result.CompletionState = "ERROR"
			result.Err = fmt.Errorf("LLM completion error: %w", err)
			return result
		}

		// Track token usage.
		if resp.Usage != nil {
			atomic.AddInt64(&result.TotalTokensUsed, int64(resp.Usage.TotalTokens))
			atomic.AddInt64(&result.TotalInputTokens, int64(resp.Usage.PromptTokens+resp.Usage.CacheReadTokens))
			atomic.AddInt64(&result.TotalOutputTokens, int64(resp.Usage.CompletionTokens+resp.Usage.CacheWriteTokens))
		}

		if config.OnLLMResponse != nil {
			config.OnLLMResponse(ctx, result.Rounds-1, resp, duration)
		}

		content := resp.Content()
		calls := resp.ToolCalls()

		// No tool calls: inject reminder and retry.
		if len(calls) == 0 {
			consecutiveEmptyRounds++
			if consecutiveEmptyRounds >= config.MaxEmptyRounds {
				result.CompletionState = "EMPTY_ROUNDS"
				return result
			}
			result.Messages = append(result.Messages, llm.NewTextMessage("user", config.EmptyRoundPrompt))
			if content != "" {
				// Insert the assistant content before the reminder so the
				// conversation flows: assistant → user-reminder.
				result.Messages = append(
					result.Messages[:len(result.Messages)-1],
					llm.NewTextMessage("assistant", content),
					result.Messages[len(result.Messages)-1],
				)
			}
			continue
		}
		consecutiveEmptyRounds = 0

		// Process each tool call.
		var toolResults []llm.Message
		var signals []CompletionSignal

		for _, call := range calls {
			result.ToolCallCount++

			// Check if this is the completion tool.
			if call.Function.Name == config.CompletionTool {
				sig := CompletionSignal{Completed: true}
				// Parse the state parameter from arguments.
				var args map[string]any
				if err := json.Unmarshal([]byte(call.Function.Arguments), &args); err == nil {
					if state, ok := args[config.CompletionStateParam].(string); ok {
						sig.State = state
					}
				}
				sig.Data = content
				signals = append(signals, sig)
				toolResults = append(toolResults, llm.NewToolResultMessage(call.ID, "Task completed successfully."))
				continue
			}

			// Look up the tool provider.
			p, ok := providerMap[call.Function.Name]
			if !ok {
				toolResults = append(toolResults, llm.NewToolResultMessage(call.ID, "Error: Tool not found. The tool you attempted to call does not exist or is not available."))
				continue
			}

			// Parse arguments.
			var args map[string]any
			if err := json.Unmarshal([]byte(call.Function.Arguments), &args); err != nil {
				toolResults = append(toolResults, llm.NewToolResultMessage(call.ID, fmt.Sprintf("Error parsing tool arguments for %s: %v", call.Function.Name, err)))
				continue
			}

			// Execute the tool.
			toolResult, err := p.Execute(args)
			if config.OnToolCall != nil {
				config.OnToolCall(ctx, call.Function.Name, args, toolResult, err)
			}
			if err != nil {
				toolResults = append(toolResults, llm.NewToolResultMessage(call.ID, fmt.Sprintf("Error executing tool %s: %v", call.Function.Name, err)))
			} else {
				toolResults = append(toolResults, llm.NewToolResultMessage(call.ID, toolResult))
			}
			if config.OnToolResult != nil {
				config.OnToolResult(ctx, call.Function.Name, toolResult)
			}
		}

		// Check for completion signal.
		if len(signals) > 0 {
			result.CompletionSignal = &signals[len(signals)-1]
			result.CompletionState = signals[len(signals)-1].State
			if result.CompletionState == "" {
				result.CompletionState = "DONE"
			}
			// Append the final messages including completion tool result.
			if len(calls) > 0 {
				result.Messages = append(result.Messages, llm.NewToolCallMessage(content, calls))
			}
			result.Messages = append(result.Messages, toolResults...)
			return result
		}

		// Append assistant message + tool results to the conversation.
		if len(calls) > 0 {
			result.Messages = append(result.Messages, llm.NewToolCallMessage(content, calls))
		} else if content != "" {
			result.Messages = append(result.Messages, llm.NewTextMessage("assistant", content))
		}
		result.Messages = append(result.Messages, toolResults...)

		// Context compression.
		if compState != nil {
			ok := applyCompressionIfNeeded(ctx, compState, &result.Messages, config.Compression, result)
			if !ok {
				result.CompletionState = "CONTEXT_OVERFLOW"
				return result
			}
		}
	}

	result.CompletionState = "MAX_ROUNDS"
	return result
}

// applyCompressionIfNeeded checks token thresholds and applies compression
// when necessary. Returns false if messages still exceed the hard limit
// after compression (meaning the loop should stop).
func applyCompressionIfNeeded(ctx context.Context, cs *compressionState, messages *[]llm.Message, cfg *CompressionConfig, result *LoopResult) bool {
	warnLimit := int(float64(cfg.MaxTokens) * cfg.WarnThreshold)
	softLimit := int(float64(cfg.MaxTokens) * cfg.SoftThreshold)

	tokenCount := countMessagesTokens(*messages)

	// Hard threshold: synchronous compression.
	if tokenCount > warnLimit {
		cs.cancelPending()
		rebuilt, err := runCompression(ctx, cs, *messages, cfg, result)
		if err != nil {
			log.Printf("[agentloop] compression error: %v", err)
		}
		if rebuilt != nil {
			*messages = rebuilt
		}
	}

	// Soft threshold: trigger async compression.
	tokenCount = countMessagesTokens(*messages)
	if tokenCount > softLimit && cs.pendingJob == nil {
		cs.triggerAsync(ctx, *messages, cfg, result)
	}

	// Final check.
	finalCount := countMessagesTokens(*messages)
	return finalCount < warnLimit
}

// countMessagesTokens estimates the total token count across all messages.
func countMessagesTokens(msgs []llm.Message) int {
	var total int
	for _, m := range msgs {
		total += llm.CountTokens(m.ExtractText())
	}
	return total
}
```

### 4.3 `internal/agentloop/compression.go`

```go
package agentloop

import (
	"context"
	"fmt"
	"strings"
	"sync"
	"sync/atomic"
	"time"

	"github.com/open-code-review/open-code-review/internal/llm"
)

// compressionState manages background (async) and foreground (sync) compression.
type compressionState struct {
	cfg        *CompressionConfig
	pendingJob *compressionJob
	mu         sync.Mutex
}

type compressionJob struct {
	done    chan struct{}
	cancel  context.CancelFunc
	rebuilt []llm.Message
}

// round groups consecutive messages starting with an assistant message
// followed by zero or more tool result messages.
type round struct {
	assistantIdx int
	toolIdxs     []int
}

// partitionResult describes how messages should be split for compression.
type partitionResult struct {
	frozenEnd   int     // always [0:2] — system + first user message
	compressEnd int     // messages[2:compressEnd] can be compressed
	rounds      []round // parsed conversational rounds
	activeCount int     // number of most-recent rounds to preserve
}

// tryApply checks if a background compression job completed and swaps the result.
func (cs *compressionState) tryApply(messages *[]llm.Message) {
	cs.mu.Lock()
	job := cs.pendingJob
	cs.mu.Unlock()

	if job == nil {
		return
	}

	select {
	case <-job.done:
		cs.mu.Lock()
		if cs.pendingJob == job && job.rebuilt != nil {
			*messages = job.rebuilt
			cs.pendingJob = nil
		}
		cs.mu.Unlock()
	default:
		// Job not done yet; skip.
	}
}

// cancelPending aborts any in-flight background compression.
func (cs *compressionState) cancelPending() {
	cs.mu.Lock()
	defer cs.mu.Unlock()

	if cs.pendingJob != nil {
		cs.pendingJob.cancel()
		cs.pendingJob = nil
	}
}

// triggerAsync kicks off a background compression job.
func (cs *compressionState) triggerAsync(ctx context.Context, messages []llm.Message, cfg *CompressionConfig, result *LoopResult) {
	msgSnapshot := copyMessages(messages)
	asyncCtx, cancel := context.WithTimeout(context.WithoutCancel(ctx), 5*time.Minute)

	job := &compressionJob{done: make(chan struct{}), cancel: cancel}
	cs.mu.Lock()
	cs.pendingJob = job
	cs.mu.Unlock()

	go func() {
		defer cancel()
		rebuilt, _ := runCompression(asyncCtx, cs, msgSnapshot, cfg, result)

		cs.mu.Lock()
		defer cs.mu.Unlock()

		if cs.pendingJob != job {
			return // cancelled or superseded
		}
		job.rebuilt = rebuilt
		close(job.done)
	}()
}

// runCompression performs three-zone compression on messages.
// Zone layout: [frozen (0:N)] + [compress (N:C)] + [active (C:)].
// The compress zone is summarized by the LLM and replaced with a summary
// injected back into the user message.
func runCompression(ctx context.Context, cs *compressionState, msgs []llm.Message, cfg *CompressionConfig, result *LoopResult) ([]llm.Message, error) {
	if len(cfg.TemplateMessages) == 0 || len(msgs) <= cfg.FrozenZoneSize {
		return msgs[:min(len(msgs), cfg.FrozenZoneSize)], nil
	}

	part := partitionMessages(msgs, cfg.MaxTokens, cfg.FrozenZoneSize, 0)
	if part.compressEnd <= part.frozenEnd {
		return msgs, nil
	}

	contextXML := buildMessageXML(msgs[part.frozenEnd:part.compressEnd])

	compressionMsgs := make([]llm.Message, 0, len(cfg.TemplateMessages))
	for _, m := range cfg.TemplateMessages {
		content := strings.ReplaceAll(m.Content, cfg.ContextPlaceholder, contextXML)
		compressionMsgs = append(compressionMsgs, llm.NewTextMessage(m.Role, content))
	}

	startTime := time.Now()
	resp, err := cfg.Client.CompletionsWithCtx(ctx, llm.ChatRequest{
		Model:     cfg.Model,
		Messages:  compressionMsgs,
		MaxTokens: cfg.MaxTokens,
	})
	duration := time.Since(startTime)

	if err != nil {
		return msgs[:part.frozenEnd], fmt.Errorf("compression: %w", err)
	}

	if resp.Usage != nil {
		atomic.AddInt64(&result.TotalTokensUsed, int64(resp.Usage.TotalTokens))
		atomic.AddInt64(&result.TotalInputTokens, int64(resp.Usage.PromptTokens+resp.Usage.CacheReadTokens))
		atomic.AddInt64(&result.TotalOutputTokens, int64(resp.Usage.CompletionTokens+resp.Usage.CacheWriteTokens))
	}

	rawSummary := stripMarkdownFences(resp.Content())
	if rawSummary == "" {
		return msgs[:part.frozenEnd], nil
	}

	// Rebuild: [frozen] + summary-injected user message + [active].
	rebuilt := make([]llm.Message, cfg.FrozenZoneSize)
	copy(rebuilt, msgs[:cfg.FrozenZoneSize])

	userMsg := rebuilt[cfg.FrozenZoneSize-1]
	currentText := userMsg.ExtractText()
	summaryBlock := "\n\n" + cfg.SummaryTag + "\n" + rawSummary + "\n</" + cfg.SummaryTag[1:]
	// If the tag looks like <xxx>, create a proper closing tag.
	if strings.HasPrefix(cfg.SummaryTag, "<") && strings.Contains(cfg.SummaryTag, ">") {
		closeTag := "</" + strings.Trim(cfg.SummaryTag, "<>") + ">"
		summaryBlock = "\n\n" + cfg.SummaryTag + "\n" + rawSummary + closeTag
	}
	rebuilt[cfg.FrozenZoneSize-1] = llm.NewTextMessage(userMsg.Role, currentText+summaryBlock)

	for i := part.compressEnd; i < len(msgs); i++ {
		rebuilt = append(rebuilt, msgs[i])
	}

	return rebuilt, nil
}

// partitionMessages divides messages into frozen, compress, and active zones.
func partitionMessages(messages []llm.Message, maxTokens, frozenZoneSize, prevSummaryEstimate int) partitionResult {
	result := partitionResult{frozenEnd: frozenZoneSize}
	if len(messages) <= frozenZoneSize {
		result.compressEnd = len(messages)
		return result
	}

	result.rounds = groupIntoRounds(messages, frozenZoneSize)
	if len(result.rounds) == 0 {
		result.compressEnd = len(messages)
		return result
	}

	result.activeCount = computeActiveZoneSize(result.rounds, messages, maxTokens, prevSummaryEstimate)
	if result.activeCount >= len(result.rounds) {
		result.compressEnd = len(messages)
		result.activeCount = 0
		return result
	}

	activeStartIdx := len(result.rounds) - result.activeCount
	lastCompressRound := result.rounds[activeStartIdx-1]
	if len(lastCompressRound.toolIdxs) > 0 {
		result.compressEnd = lastCompressRound.toolIdxs[len(lastCompressRound.toolIdxs)-1] + 1
	} else {
		result.compressEnd = lastCompressRound.assistantIdx + 1
	}

	return result
}

// computeActiveZoneSize returns how many trailing rounds fit within the
// remaining token budget after accounting for frozen zone and compressed summary.
func computeActiveZoneSize(rounds []round, messages []llm.Message, maxTokens, reservedTokens int) int {
	budget := int(float64(maxTokens)*0.80) - reservedTokens
	if budget <= 0 {
		return 0
	}

	count := 0
	tokensUsed := 0
	for i := len(rounds) - 1; i >= 0; i-- {
		roundTokens := llm.CountTokens(messages[rounds[i].assistantIdx].ExtractText())
		for _, ti := range rounds[i].toolIdxs {
			roundTokens += llm.CountTokens(messages[ti].ExtractText())
		}
		if tokensUsed+roundTokens > budget {
			break
		}
		tokensUsed += roundTokens
		count++
	}
	return count
}

// groupIntoRounds parses messages[start:] into logical (assistant + tool_results) pairs.
func groupIntoRounds(messages []llm.Message, start int) []round {
	var rounds []round
	i := start
	for i < len(messages) {
		if messages[i].Role == "assistant" {
			r := round{assistantIdx: i}
			i++
			for i < len(messages) && messages[i].Role == "tool" {
				r.toolIdxs = append(r.toolIdxs, i)
				i++
			}
			rounds = append(rounds, r)
		} else {
			i++
		}
	}
	return rounds
}

// buildMessageXML formats messages as XML for the compression prompt.
func buildMessageXML(msgs []llm.Message) string {
	var sb strings.Builder
	for i, m := range msgs {
		sb.WriteString(fmt.Sprintf("<message id=\"%d\" role=\"%s\">\n", i, m.Role))
		sb.WriteString("    <content>\n")
		sb.WriteString(fmt.Sprintf("      %s\n", m.ExtractText()))
		sb.WriteString("    </content>\n")
		sb.WriteString("</message>")
		if i < len(msgs)-1 {
			sb.WriteString("\n")
		}
	}
	return sb.String()
}

// stripMarkdownFences removes ```json and ``` wrappers that some models
// add around structured outputs.
func stripMarkdownFences(s string) string {
	s = strings.TrimSpace(s)
	if strings.HasPrefix(s, "```") {
		if nl := strings.IndexByte(s, '\n'); nl >= 0 {
			s = s[nl+1:]
		} else {
			s = strings.TrimPrefix(s, "```json")
			s = strings.TrimPrefix(s, "```")
		}
	}
	s = strings.TrimSpace(s)
	if strings.HasSuffix(s, "```") {
		s = s[:len(s)-3]
		s = strings.TrimSpace(s)
	}
	return s
}

func copyMessages(msgs []llm.Message) []llm.Message {
	out := make([]llm.Message, len(msgs))
	copy(out, msgs)
	return out
}
```

### 4.4 `internal/agentloop/loop_test.go`

```go
package agentloop

import (
	"context"
	"testing"
)

// mockLLMClient is a test double that returns predetermined responses.
type mockLLMClient struct {
	responses []*llm.ChatResponse
	callIndex int
}

func (m *mockLLMClient) Completions(req llm.ChatRequest) (*llm.ChatResponse, error) {
	return m.CompletionsWithCtx(context.Background(), req)
}

func (m *mockLLMClient) CompletionsWithCtx(ctx context.Context, req llm.ChatRequest) (*llm.ChatResponse, error) {
	if m.callIndex >= len(m.responses) {
		// Default: no tool calls, return empty.
		content := "I'm done."
		return &llm.ChatResponse{
			Choices: []llm.Choice{
				{Message: llm.ResponseMessage{Content: &content}},
			},
		}, nil
	}
	resp := m.responses[m.callIndex]
	m.callIndex++
	return resp, nil
}

func (m *mockLLMClient) StreamCompletion(req llm.ChatRequest, cb func(chunk []byte) error) error {
	return nil
}

func (m *mockLLMClient) StreamCompletionWithCtx(ctx context.Context, req llm.ChatRequest, cb func(chunk []byte) error) error {
	return nil
}

// mockToolProvider is a simple tool that echoes its arguments.
type mockToolProvider struct {
	name string
	fn   func(args map[string]any) (string, error)
}

func (m *mockToolProvider) Name() string { return m.name }
func (m *mockToolProvider) Execute(args map[string]any) (string, error) {
	if m.fn != nil {
		return m.fn(args)
	}
	return "ok", nil
}

func TestRunLoop_CompletionTool(t *testing.T) {
	doneContent := "task done"
	toolCalls := []llm.ToolCall{
		{
			ID:   "call_1",
			Type: "function",
			Function: llm.FunctionCall{
				Name:      "task_done",
				Arguments: `{"state": "DONE"}`,
			},
		},
	}

	client := &mockLLMClient{
		responses: []*llm.ChatResponse{
			{
				Choices: []llm.Choice{
					{
						Message: llm.ResponseMessage{
							Content:   &doneContent,
							ToolCalls: toolCalls,
						},
					},
				},
			},
		},
	}

	messages := []llm.Message{
		llm.NewTextMessage("system", "You are a helpful assistant."),
		llm.NewTextMessage("user", "Finish the task."),
	}

	config := LoopConfig{
		MaxRounds:  10,
		MaxTokens:  32768,
		Model:      "test-model",
		ToolDefs:   []llm.ToolDef{},
	}

	result := RunLoop(context.Background(), client, messages, config, nil)

	if result.CompletionState != "DONE" {
		t.Errorf("expected DONE, got %s", result.CompletionState)
	}
	if result.CompletionSignal == nil {
		t.Fatal("expected completion signal")
	}
	if result.CompletionSignal.State != "DONE" {
		t.Errorf("expected signal state DONE, got %s", result.CompletionSignal.State)
	}
	if result.Rounds != 1 {
		t.Errorf("expected 1 round, got %d", result.Rounds)
	}
}

func TestRunLoop_ToolExecution(t *testing.T) {
	toolResult := "file contents here"
	assistantContent := "reading the file"
	toolCallContent := "calling file_read"

	// Round 1: LLM calls a tool.
	// Round 2: LLM calls task_done.
	call1 := []llm.ToolCall{
		{
			ID:       "call_1",
			Type:     "function",
			Function: llm.FunctionCall{Name: "file_read", Arguments: `{"file_path": "test.go"}`},
		},
	}
	call2 := []llm.ToolCall{
		{
			ID:       "call_2",
			Type:     "function",
			Function: llm.FunctionCall{Name: "task_done", Arguments: `{"state": "DONE"}`},
		},
	}

	client := &mockLLMClient{
		responses: []*llm.ChatResponse{
			{
				Choices: []llm.Choice{
					{
						Message: llm.ResponseMessage{
							Content:   &toolCallContent,
							ToolCalls: call1,
						},
					},
				},
			},
			{
				Choices: []llm.Choice{
					{
						Message: llm.ResponseMessage{
							Content:   &assistantContent,
							ToolCalls: call2,
						},
					},
				},
			},
		},
	}

	provider := &mockToolProvider{
		name: "file_read",
		fn: func(args map[string]any) (string, error) {
			return toolResult, nil
		},
	}

	messages := []llm.Message{
		llm.NewTextMessage("system", "You are a code reviewer."),
		llm.NewTextMessage("user", "Review the file."),
	}

	config := LoopConfig{
		MaxRounds: 10,
		MaxTokens: 32768,
		Model:     "test-model",
	}

	result := RunLoop(context.Background(), client, messages, config, []ToolProvider{provider})

	if result.CompletionState != "DONE" {
		t.Errorf("expected DONE, got %s", result.CompletionState)
	}
	if result.Rounds != 2 {
		t.Errorf("expected 2 rounds, got %d", result.Rounds)
	}
	if result.ToolCallCount != 2 {
		t.Errorf("expected 2 tool calls, got %d", result.ToolCallCount)
	}
}

func TestRunLoop_EmptyRounds(t *testing.T) {
	// LLM returns no tool calls for three consecutive rounds.
	emptyContent := "thinking..."

	client := &mockLLMClient{
		responses: []*llm.ChatResponse{
			{Choices: []llm.Choice{{Message: llm.ResponseMessage{Content: &emptyContent}}}},
			{Choices: []llm.Choice{{Message: llm.ResponseMessage{Content: &emptyContent}}}},
			{Choices: []llm.Choice{{Message: llm.ResponseMessage{Content: &emptyContent}}}},
		},
	}

	messages := []llm.Message{
		llm.NewTextMessage("system", "You are a helper."),
		llm.NewTextMessage("user", "Do something."),
	}

	config := LoopConfig{
		MaxRounds:      10,
		MaxEmptyRounds: 3,
		MaxTokens:      32768,
		Model:          "test-model",
	}

	result := RunLoop(context.Background(), client, messages, config, nil)

	if result.CompletionState != "EMPTY_ROUNDS" {
		t.Errorf("expected EMPTY_ROUNDS, got %s", result.CompletionState)
	}
}

func TestRunLoop_MaxRounds(t *testing.T) {
	// LLM keeps calling a tool that never triggers completion.
	roundContent := "tool call"
	var responses []*llm.ChatResponse
	for i := 0; i < 25; i++ {
		toolCall := []llm.ToolCall{
			{
				ID:       fmt.Sprintf("call_%d", i),
				Type:     "function",
				Function: llm.FunctionCall{Name: "my_tool", Arguments: `{}`},
			},
		}
		responses = append(responses, &llm.ChatResponse{
			Choices: []llm.Choice{
				{Message: llm.ResponseMessage{Content: &roundContent, ToolCalls: toolCall}},
			},
		})
	}

	client := &mockLLMClient{responses: responses}
	provider := &mockToolProvider{name: "my_tool"}

	messages := []llm.Message{
		llm.NewTextMessage("system", "You are a helper."),
		llm.NewTextMessage("user", "Do something."),
	}

	config := LoopConfig{
		MaxRounds:  5,
		MaxTokens:  32768,
		Model:      "test-model",
	}

	result := RunLoop(context.Background(), client, messages, config, []ToolProvider{provider})

	if result.CompletionState != "MAX_ROUNDS" {
		t.Errorf("expected MAX_ROUNDS, got %s", result.CompletionState)
	}
	if result.Rounds != 5 {
		t.Errorf("expected 5 rounds, got %d", result.Rounds)
	}
}

func TestLoopConfig_Defaults(t *testing.T) {
	config := LoopConfig{}
	config.Defaults()

	if config.MaxRounds != 20 {
		t.Errorf("expected MaxRounds=20, got %d", config.MaxRounds)
	}
	if config.MaxEmptyRounds != 3 {
		t.Errorf("expected MaxEmptyRounds=3, got %d", config.MaxEmptyRounds)
	}
	if config.CompletionTool != "task_done" {
		t.Errorf("expected CompletionTool='task_done', got %s", config.CompletionTool)
	}
	if config.CompletionStateParam != "state" {
		t.Errorf("expected CompletionStateParam='state', got %s", config.CompletionStateParam)
	}
	if config.MaxTokens != 32768 {
		t.Errorf("expected MaxTokens=32768, got %d", config.MaxTokens)
	}
}

func TestStripMarkdownFences(t *testing.T) {
	tests := []struct {
		input string
		want  string
	}{
		{"```json\n{\"a\":1}\n```", "{\"a\":1}"},
		{"```\n{\"a\":1}\n```", "{\"a\":1}"},
		{"{\"a\":1}", "{\"a\":1}"},
		{"  ```json\n{\"a\":1}\n```  ", "{\"a\":1}"},
	}

	for _, tt := range tests {
		got := stripMarkdownFences(tt.input)
		if got != tt.want {
			t.Errorf("stripMarkdownFences(%q) = %q, want %q", tt.input, got, tt.want)
		}
	}
}
```

> **注意**：`loop_test.go` 中用到了 `fmt`，需在 import 中添加。另外 `llm` 包的引用路径需要根据实际 module path 调整。

### 4.5 业务层适配示例：`agent.go` 改造后的 `performLlmCodeReview`

以下是改造后的 `agent.go` 中 `performLlmCodeReview` 的示意代码，展示业务层如何调用 `agentloop`：

```go
func (a *Agent) performLlmCodeReview(ctx context.Context, messages []llm.Message, newPath string) error {
	// 1. 构建工具提供者列表（业务层特有）
	providers := a.buildToolProviders(newPath)

	// 2. 配置循环参数
	config := agentloop.LoopConfig{
		MaxRounds:           a.args.Template.MaxToolRequestTimes,
		MaxEmptyRounds:      3,
		CompletionTool:      "task_done",
		CompletionStateParam: "state",
		EmptyRoundPrompt:    "You did not successfully call any tools. Please try again or use task_done if finished.",
		ToolDefs:            a.args.MainToolDefs,
		Model:               a.args.Model,
		MaxTokens:           a.args.Template.MaxTokens,
		// 压缩配置（如果模板有 MemoryCompressionTask）
		Compression: a.buildCompressionConfig(),
		// 回调
		OnRoundStart: func(ctx context.Context, round int, msgs []llm.Message) {
			fs := a.session.GetOrCreateFileSession(newPath)
			fs.AppendTaskRecord(session.MainTask, msgs)
		},
		OnLLMResponse: func(ctx context.Context, round int, resp *llm.ChatResponse, duration float64) {
			telemetry.RecordLLMRequest(ctx, a.args.Model, time.Duration(duration*float64(time.Second)), 0, "ok")
		},
	}

	// 3. 运行循环
	result := agentloop.RunLoop(ctx, a.args.LLMClient, messages, config, providers)

	// 4. 处理结果
	if result.Err != nil {
		return result.Err
	}

	// 更新 token 统计
	atomic.AddInt64(&a.totalTokensUsed, result.TotalTokensUsed)
	atomic.AddInt64(&a.totalInputTokens, result.TotalInputTokens)
	atomic.AddInt64(&a.totalOutputTokens, result.TotalOutputTokens)

	return nil
}

// buildToolProviders 将 tool.Registry 适配为 agentloop.ToolProvider 列表
func (a *Agent) buildToolProviders(newPath string) []agentloop.ToolProvider {
	var providers []agentloop.ToolProvider

	// task_done 特殊处理：用 BuiltinProvider 包装
	providers = append(providers, agentloop.NewBuiltinProvider("task_done", func(args map[string]any) (string, error) {
		return "Task completed successfully.", nil
	}))

	// 其他工具从 Registry 适配
	for name, p := range a.args.Tools {
		providers = append(providers, &toolProviderAdapter{provider: p, agent: a, newPath: newPath})
	}

	return providers
}

// toolProviderAdapter 将 tool.Provider 适配为 agentloop.ToolProvider
type toolProviderAdapter struct {
	provider tool.Provider
	agent    *Agent
	newPath  string
}

func (a *toolProviderAdapter) Name() string { return a.provider.Tool().Name() }
func (a *toolProviderAdapter) Execute(args map[string]any) (string, error) {
	// 业务层特有逻辑：为 code_comment 注入文件路径
	if a.provider.Tool() == tool.CodeComment && a.newPath != "" {
		if _, ok := args["path"]; !ok {
			args["path"] = a.newPath
		}
	}
	return a.provider.Execute(args)
}
```

## 5. 文件清单

```
internal/agentloop/
  types.go           # LoopConfig, LoopResult, ToolProvider, CompletionSignal, CompressionConfig
  loop.go            # RunLoop 主循环
  compression.go     # 上下文压缩（三区域压缩 + 异步压缩）
  loop_test.go        # 单元测试
```

## 6. 与现有代码的对应关系

| 现有代码 (agent.go) | 抽取后 (agentloop) |
|---|---|
| `performLlmCodeReview` 循环骨架 (L755-864) | `RunLoop()` |
| `executeToolCall` 通用路径 (L869-873, L879, L884-887, L963-978) | `RunLoop()` 内 tool 查找 + 执行 |
| `executeToolCall` 的 `task_done` 判断 (L875-876) | `config.CompletionTool` 匹配 |
| `executeToolCall` 的 `code_comment` 特殊路径 (L900-961) | **留在业务层**，通过 `toolProviderAdapter` |
| `addNextMessage` 消息追加 (L1025-1035) | `RunLoop()` 内消息追加 |
| `addNextMessage` 压缩逻辑 (L1003-1044) | `compression.go` |
| `runCompression` (L1158-1215) | `runCompression()` |
| `triggerAsyncCompression` (L1217-1241) | `compressionState.triggerAsync()` |
| `tryApplyPendingCompression` (L1245-1266) | `compressionState.tryApply()` |
| `cancelPendingCompression` (L1268-1277) | `compressionState.cancelPending()` |
| `groupIntoRounds` (L1056-1073) | `groupIntoRounds()` |
| `computeActiveZoneSize` (L1077-1097) | `computeActiveZoneSize()` |
| `partitionMessages` (L1102-1133) | `partitionMessages()` |
| `stripMarkdownFences` (L1137-1153) | `stripMarkdownFences()` |
| `countMessagesTokens` (L1047-1053) | `countMessagesTokens()` |
| `tool.TaskDone` 硬编码 | `config.CompletionTool` |
| `tool.Registry` + `tool.Provider` | `map[string]ToolProvider` + `ToolProvider` 接口 |
| `tool.TaskCheckpoint` | `CompletionSignal` |
| `tool.ToolCallResult` | 直接用 `llm.NewToolResultMessage()` |
| 空轮次提示 (L801) | `config.EmptyRoundPrompt` |
| `stdarg` / `telemetry` / `session` | `config.OnRoundStart` / `OnLLMResponse` / `OnToolCall` 回调 |