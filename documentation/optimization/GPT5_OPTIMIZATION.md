# GPT-5 Optimization Guide

## Overview

This document explains the GPT-5 optimization features implemented in the chatbot to improve response times and performance.

## Key Changes

### 1. Responses API vs Chat Completions API

**Previous Setup:**
- Used `openai.chat.completions.create()` for all models
- GPT-5 models weren't optimized for this legacy API

**New Setup:**
- Automatic detection of GPT-5 family models (gpt-5, gpt-5-mini, gpt-5-nano)
- Uses `openai.responses.create()` for GPT-5 models (optimized)
- Falls back to Chat Completions API for non-GPT-5 models (gpt-4o-mini, gpt-3.5-turbo)

### 2. Reasoning Effort Parameter

Controls how much the model "thinks" before responding:

- **`minimal`** (RECOMMENDED for chatbot): Fastest time-to-first-token, minimal reasoning tokens
- **`low`**: Reduced reasoning, good for simple tasks
- **`medium`**: Balanced (default if not specified)
- **`high`**: Deep reasoning for complex multi-step tasks

**Configuration:**
```env
GPT5_REASONING_EFFORT=minimal
```

### 3. Verbosity Parameter

Controls response length:

- **`low`** (RECOMMENDED for chatbot): Concise, to-the-point responses
- **`medium`**: Balanced (default if not specified)
- **`high`**: Comprehensive, detailed responses

**Configuration:**
```env
GPT5_VERBOSITY=low
```

## Performance Impact

Based on OpenAI documentation:

| Parameter | Value | Impact |
|-----------|-------|--------|
| reasoning_effort | minimal | Fastest time-to-first-token, 50-80% faster than medium |
| verbosity | low | 20-40% shorter responses, faster generation |

**Expected Results:**
- Reduced latency from ~16.9s to potentially 5-8s
- Lower token usage (cost savings)
- Maintained quality for chatbot use cases

## Configuration

### Environment Variables

```env
# Use GPT-5 model
GEN_MODEL=gpt-5-mini

# Optimize for chatbot performance
GPT5_REASONING_EFFORT=minimal
GPT5_VERBOSITY=low
```

### Supported Models

**GPT-5 Family (uses Responses API):**
- `gpt-5` - Most capable, highest intelligence
- `gpt-5-mini` - Best balance of speed, cost, and capability
- `gpt-5-nano` - Fastest, most cost-effective

**Non-GPT-5 Models (uses Chat Completions API):**
- `gpt-4o-mini` - Fast, cost-effective
- `gpt-3.5-turbo` - Very fast, legacy model

## Tuning Guide

### For Different Use Cases

**Fast Chatbot (Current Setup):**
```env
GPT5_REASONING_EFFORT=minimal
GPT5_VERBOSITY=low
```

**Detailed Support:**
```env
GPT5_REASONING_EFFORT=low
GPT5_VERBOSITY=medium
```

**Complex Problem Solving:**
```env
GPT5_REASONING_EFFORT=high
GPT5_VERBOSITY=high
```

## Monitoring

The system logs which API and parameters are being used:

```
🧠 Using Responses API with reasoning_effort: minimal, verbosity: low
⏱️ Retrieval Performance: { total: "1234ms" }
🤖 OpenAI Generation: 3456ms
📊 Performance Breakdown: { generation: "3456ms", total: "5000ms" }
```

## Migration Notes

### Breaking Changes

- None! The system automatically detects the model type
- Backward compatible with non-GPT-5 models
- No changes needed to API responses

### Unsupported Parameters

GPT-5 models **do NOT support**:
- `temperature` - Removed for GPT-5 models
- `top_p` - Not supported
- `logprobs` - Not supported

These are automatically handled in the code.

## Further Optimizations

### Future Considerations

1. **Responses API State Management:**
   - Use `previous_response_id` for multi-turn conversations
   - Reduces token usage by persisting reasoning context

2. **Structured Outputs:**
   - Use `text.format` for JSON responses
   - Better than `response_format` in Chat Completions

3. **Native Tools:**
   - Upgrade to Responses API built-in tools
   - Better performance than custom function calling

## References

- [GPT-5 Prompting Guide](https://cookbook.openai.com/examples/gpt-5/gpt-5_prompting_guide)
- [Migrate to Responses API](https://platform.openai.com/docs/guides/migrate-to-responses)
- [GPT-5 Model Documentation](https://platform.openai.com/docs/models/gpt-5-mini)
