---
name: opencode-prompt-optimizer
description: "Integrate Opencode prompt optimization pipeline to compress and optimize prompts before LLM calls, reducing token costs by 40-60%."
---

# Opencode Prompt Optimizer

Connects to the Opencode Prompt Pipeline Dashboard to compress, optimize, and track token usage across AI agent interactions. Reduces costs while maintaining output quality.

## Quick quality checklist

- `name` matches folder name exactly (kebab-case)
- All examples are tested and runnable
- Includes both Bash and Node.js examples
- Uses local API (no external dependencies)
- No secrets, API keys, or personal data in examples

## When to use

- Use case 1: When preparing prompts for expensive LLM calls (Claude, GPT-4)
- Use case 2: When tracking token usage across multiple agent platforms
- Use case 3: When optimizing prompt pipelines for cost savings

## Required tools / APIs

- Opencode server running locally (default: http://localhost:5750)
- No additional API keys required
- Optional: Prometheus/Grafana for advanced monitoring

Install options:

```bash
# Navigate to Opencode directory
cd /Users/jerryb/Documents/Opencode

# Install dependencies (if not already done)
npm install

# Start the server
npm run start
# Server runs on http://localhost:5750
```

## Skills

### basic_usage

Query Opencode API for prompt statistics and optimization metrics.

```bash
# Get current statistics
curl -s http://localhost:5750/api/v1/stats | jq '.'

# Get recent job history
curl -s http://localhost:5750/api/v1/history | jq '.[0:5]'

# Get provider-specific stats
curl -s http://localhost:5750/api/v1/provider-stats | jq '.'
```

**Node.js:**

```javascript
async function getOpencodeStats() {
  const res = await fetch('http://localhost:5750/api/v1/stats');
  return await res.json();
}

async function getRecentJobs(limit = 10) {
  const res = await fetch(`http://localhost:5750/api/v1/history?limit=${limit}`);
  return await res.json();
}

// Usage
// getOpencodeStats().then(console.log);
```

### robust_usage

Production-ready integration with error handling and retry logic.

```bash
# Test connectivity with retry
for i in 1 2 3; do
  if curl -sf --max-time 5 http://localhost:5750/api/v1/health > /dev/null; then
    echo "Opencode server is healthy"
    break
  fi
  echo "Retry $i/3..."
  sleep 2
done
```

**Node.js:**

```javascript
import { setTimeout } from 'timers/promises';

class OpencodeClient {
  constructor(baseUrl = 'http://localhost:5750') {
    this.baseUrl = baseUrl;
    this.maxRetries = 3;
    this.timeout = 10000;
  }
  
  async request(endpoint, options = {}) {
    const { retries = 0, ...fetchOptions } = options;
    
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), this.timeout);
    
    try {
      const res = await fetch(`${this.baseUrl}${endpoint}`, {
        ...fetchOptions,
        signal: controller.signal,
        headers: {
          'Content-Type': 'application/json',
          ...fetchOptions.headers
        }
      });
      clearTimeout(timeoutId);
      
      if (!res.ok) {
        throw new Error(`HTTP ${res.status}: ${res.statusText}`);
      }
      
      return await res.json();
    } catch (err) {
      clearTimeout(timeoutId);
      
      if (retries < this.maxRetries) {
        await setTimeout(1000 * Math.pow(2, retries));
        return this.request(endpoint, { ...options, retries: retries + 1 });
      }
      throw err;
    }
  }
  
  async getStats() {
    return this.request('/api/v1/stats');
  }
  
  async getHistory(params = {}) {
    const query = new URLSearchParams(params).toString();
    return this.request(`/api/v1/history${query ? `?${query}` : ''}`);
  }
  
  async getProviderStats() {
    return this.request('/api/v1/provider-stats');
  }
  
  async optimizePrompt(prompt, context = '') {
    return this.request('/api/v1/optimize', {
      method: 'POST',
      body: JSON.stringify({ prompt, context })
    });
  }
  
  async getSavings(days = 7) {
    return this.request(`/api/v1/savings?days=${days}`);
  }
  
  async healthCheck() {
    try {
      const res = await fetch(`${this.baseUrl}/api/v1/health`, {
        signal: AbortSignal.timeout(5000)
      });
      return res.ok;
    } catch {
      return false;
    }
  }
}

// Usage
// const client = new OpencodeClient();
// const stats = await client.getStats();
// console.log('Total tokens saved:', stats.totalSaved);
```

### advanced_usage

Integrate prompt optimization into agent workflows with automatic compression.

```bash
# Example: Compress a prompt before sending to LLM
PROMPT="Write a detailed technical analysis of the recent market trends in AI agent frameworks, comparing LangChain, DSPy, and custom solutions..."

# Call Opencode optimization endpoint
OPTIMIZED=$(curl -s -X POST http://localhost:5750/api/v1/optimize \
  -H "Content-Type: application/json" \
  -d "{\"prompt\": \"$PROMPT\", \"context\": \"Agent framework comparison\"}")

echo "$OPTIMIZED" | jq '.'
```

**Node.js:**

```javascript
import { OpencodeClient } from './opencode-client.js';

class PromptOptimizer {
  constructor() {
    this.client = new OpencodeClient();
    this.cache = new Map();
    this.cacheTTL = 3600000; // 1 hour
  }
  
  async optimize(prompt, options = {}) {
    const { context = '', maxTokens = null } = options;
    
    // Check cache
    const cacheKey = `${prompt}:${context}:${maxTokens}`;
    const cached = this.cache.get(cacheKey);
    if (cached && Date.now() - cached.timestamp < this.cacheTTL) {
      return cached.result;
    }
    
    // Call Opencode API
    const result = await this.client.optimizePrompt(prompt, context);
    
    // Cache result
    this.cache.set(cacheKey, {
      result,
      timestamp: Date.now()
    });
    
    return result;
  }
  
  async optimizeWithFallback(prompt, options = {}) {
    try {
      return await this.optimize(prompt, options);
    } catch (err) {
      console.warn('Opencode optimization failed, using original prompt:', err.message);
      return {
        original: prompt,
        compressed: prompt,
        savedTokens: 0,
        savingsPercent: 0,
        note: 'Optimization unavailable'
      };
    }
  }
  
  async trackUsage(platform, provider, inputTokens, outputTokens) {
    return this.client.request('/api/v1/record', {
      method: 'POST',
      body: JSON.stringify({
        platform,
        provider,
        inputRaw: inputTokens,
        outputRaw: outputTokens,
        timestamp: new Date().toISOString()
      })
    });
  }
  
  async getCostReport(days = 7) {
    return this.client.getSavings(days);
  }
}

// Usage in agent workflow
// const optimizer = new PromptOptimizer();
// const optimized = await optimizer.optimizeWithFallback(userPrompt);
// console.log('Original tokens:', optimized.original.length);
// console.log('Optimized tokens:', optimized.compressed.length);
// console.log('Savings:', optimized.savingsPercent + '%');
```

### integration_with_hermes

Connect Opencode to Hermes agent pipeline for automatic prompt compression.

```javascript
// hermes-plugin-opencode.js
import { OpencodeClient } from './opencode-client.js';

class HermesOpencodePlugin {
  constructor(config = {}) {
    this.client = new OpencodeClient(config.baseUrl);
    this.enabled = config.enabled !== false;
    this.platform = config.platform || 'hermes';
  }
  
  async beforeLLMCall(messages) {
    if (!this.enabled) return messages;
    
    try {
      const lastUserMessage = messages.filter(m => m.role === 'user').pop();
      if (!lastUserMessage) return messages;
      
      const optimization = await this.client.optimizePrompt(
        lastUserMessage.content
      );
      
      // Replace original message with optimized version
      const optimizedMessages = [...messages];
      const lastIdx = optimizedMessages.length - 1;
      optimizedMessages[lastIdx] = {
        ...lastUserMessage,
        content: optimization.compressed || optimization.original
      };
      
      return optimizedMessages;
    } catch (err) {
      console.warn('Opencode plugin error:', err.message);
      return messages;
    }
  }
  
  async afterLLMCall(inputTokens, outputTokens, provider) {
    if (!this.enabled) return;
    
    try {
      await this.client.trackUsage(this.platform, provider, inputTokens, outputTokens);
    } catch (err) {
      console.warn('Failed to track usage:', err.message);
    }
  }
}

// Register with Hermes
// export const opencodePlugin = new HermesOpencodePlugin();
```

**Python equivalent:**

```python
# hermes_plugin_opencode.py
import requests
from typing import List, Dict, Optional

class HermesOpencodePlugin:
    def __init__(self, base_url: str = "http://localhost:5750", platform: str = "hermes"):
        self.base_url = base_url
        self.platform = platform
        self.enabled = True
    
    def before_llm_call(self, messages: List[Dict]) -> List[Dict]:
        """Optimize prompts before sending to LLM."""
        if not self.enabled:
            return messages
        
        try:
            # Find last user message
            user_msgs = [m for m in messages if m.get('role') == 'user']
            if not user_msgs:
                return messages
            
            last_user = user_msgs[-1]
            optimization = requests.post(
                f"{self.base_url}/api/v1/optimize",
                json={"prompt": last_user["content"]}
            ).json()
            
            # Replace with optimized content
            messages = messages.copy()
            messages[-1] = {
                **last_user,
                "content": optimization.get("compressed", last_user["content"])
            }
            
            return messages
        
        except Exception as e:
            print(f"Opencode plugin error: {e}")
            return messages
    
    def after_llm_call(self, input_tokens: int, output_tokens: int, provider: str):
        """Track token usage."""
        if not self.enabled:
            return
        
        try:
            requests.post(
                f"{self.base_url}/api/v1/record",
                json={
                    "platform": self.platform,
                    "provider": provider,
                    "inputRaw": input_tokens,
                    "outputRaw": output_tokens
                }
            )
        except Exception as e:
            print(f"Failed to track usage: {e}")
```

## Output format

Define exactly what the agent should return.

- **Stats response:** `{ totalTokens, savedTokens, savingsPercent, providers: {...} }`
- **History response:** Array of `{ id, timestamp, promptTitle, inputRaw, inputCompressed, savedPct, platform, provider }`
- **Optimization response:** `{ original, compressed, savedTokens, savingsPercent }`
- **Error shape:** `{ error: string, solution: string, retry: boolean }`

## Rate limits / Best practices

- Cache optimization results for 1 hour to avoid redundant API calls
- Implement exponential backoff on Opencode server errors
- Monitor health endpoint before attempting optimization
- Fall back to original prompt if optimization fails
- Track usage even when optimization is disabled

## Agent prompt

```text
You have opencode-prompt-optimizer capability. When processing user prompts:

1. Check if Opencode server is available (health endpoint)
2. Before sending expensive prompts to LLM, optimize through Opencode
3. Track token usage after each LLM call
4. Report savings in your response summary
5. If Opencode is unavailable, use original prompt without error

Always prefer optimized prompts to reduce costs, but never block user requests.
```

## Troubleshooting

**Error scenario 1:**
- Symptom: "Connection refused to localhost:5750"
- Solution: Start Opencode server with `npm run start` from `/Users/jerryb/Documents/Opencode`

**Error scenario 2:**
- Symptom: "Optimization endpoint not found"
- Solution: Verify Opencode version supports /api/v1/optimize endpoint

**Error scenario 3:**
- Symptom: "Rate limit exceeded"
- Solution: Implement request queuing with delay between optimization calls

## See also

- [../obsidian-vault-integration/SKILL.md](../obsidian-vault-integration/SKILL.md) — Ingest Obsidian notes for context
- [../vooodoo-os-setup/SKILL.md](../vooodoo-os-setup/SKILL.md) — Set up VOODOO OS architecture

---

## Notes

- Skill file path should be `skills/opencode-prompt-optimizer/SKILL.md`
- Quote `description` when it includes `:` to avoid YAML parsing issues
- Keep examples copy-paste friendly and verify they run before submitting
- See [CONTRIBUTING.md](CONTRIBUTING.md) for full contribution standards
