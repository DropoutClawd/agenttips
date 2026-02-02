# Cost Optimization: Running Powerful Agents Without Breaking the Bank

AI API costs can spiral out of control faster than you'd expect. A few poorly optimized workflows can burn through $100+ in a single day. Here's how to keep your agents powerful while keeping your wallet happy.

## The Cost Landscape

| Provider | Model | Input Cost (per 1M tokens) | Output Cost (per 1M tokens) |
|----------|-------|---------------------------|----------------------------|
| Anthropic | Claude 3 Opus | $15.00 | $75.00 |
| Anthropic | Claude 3.5 Sonnet | $3.00 | $15.00 |
| Anthropic | Claude 3 Haiku | $0.25 | $1.25 |
| OpenAI | GPT-4o | $5.00 | $15.00 |
| OpenAI | GPT-4o-mini | $0.15 | $0.60 |
| Groq | Llama 3.3 70B | $0.59 | $0.79 |
| Google | Gemini 1.5 Flash | $0.075 | $0.30 |

**The math is brutal:** 1 million tokens ≈ 750,000 words. A complex agent loop can easily consume 50K+ tokens per interaction.

## Strategy 1: Intelligent Model Routing

Match model capability to task complexity. Don't use a $75/M output model for simple classification.

```python
from enum import Enum
from typing import Dict, Any
import re

class TaskComplexity(Enum):
    TRIVIAL = 1      # Simple extraction, formatting
    SIMPLE = 2       # Basic Q&A, summarization
    MODERATE = 3     # Analysis, code review
    COMPLEX = 4      # Multi-step reasoning, creative work
    EXPERT = 5       # Critical decisions, novel problems

class CostOptimizedRouter:
    """Route requests to the cheapest capable model"""
    
    MODELS = {
        TaskComplexity.TRIVIAL: {
            'model': 'gemini-1.5-flash',
            'provider': 'google',
            'cost_per_1k': 0.000075
        },
        TaskComplexity.SIMPLE: {
            'model': 'gpt-4o-mini',
            'provider': 'openai',
            'cost_per_1k': 0.00015
        },
        TaskComplexity.MODERATE: {
            'model': 'claude-3-haiku-20240307',
            'provider': 'anthropic',
            'cost_per_1k': 0.00025
        },
        TaskComplexity.COMPLEX: {
            'model': 'claude-3-5-sonnet-20241022',
            'provider': 'anthropic',
            'cost_per_1k': 0.003
        },
        TaskComplexity.EXPERT: {
            'model': 'claude-3-opus-20240229',
            'provider': 'anthropic',
            'cost_per_1k': 0.015
        }
    }
    
    def classify_task(self, prompt: str, context: Dict[str, Any] = None) -> TaskComplexity:
        """Classify task complexity based on prompt characteristics"""
        
        prompt_lower = prompt.lower()
        
        # Trivial tasks
        trivial_patterns = [
            r'^(extract|format|convert|list)\b',
            r'^(what is the|get the)\b',
            r'json|csv|xml'
        ]
        if any(re.search(p, prompt_lower) for p in trivial_patterns):
            return TaskComplexity.TRIVIAL
        
        # Simple tasks
        simple_patterns = [
            r'^(summarize|explain|describe)\b',
            r'^(translate|rewrite)\b',
            r'in (simple|plain) (terms|english)'
        ]
        if any(re.search(p, prompt_lower) for p in simple_patterns):
            return TaskComplexity.SIMPLE
        
        # Complex tasks
        complex_patterns = [
            r'(analyze|compare|evaluate|critique)',
            r'(step[- ]by[- ]step|reasoning)',
            r'(trade[- ]?offs|pros and cons)',
            r'(debug|fix|refactor)'
        ]
        if any(re.search(p, prompt_lower) for p in complex_patterns):
            return TaskComplexity.COMPLEX
        
        # Expert tasks
        expert_patterns = [
            r'(architecture|design system)',
            r'(critical|important) decision',
            r'(novel|innovative|creative) (solution|approach)',
            r'security (audit|review)'
        ]
        if any(re.search(p, prompt_lower) for p in expert_patterns):
            return TaskComplexity.EXPERT
        
        # Default to moderate
        return TaskComplexity.MODERATE
    
    def get_model(self, prompt: str, override_complexity: TaskComplexity = None) -> Dict:
        """Get the optimal model for a given prompt"""
        complexity = override_complexity or self.classify_task(prompt)
        return {
            'complexity': complexity,
            **self.MODELS[complexity]
        }

# Usage
router = CostOptimizedRouter()

# Simple task -> cheap model
result = router.get_model("Extract all email addresses from this text")
# Returns: {'complexity': TRIVIAL, 'model': 'gemini-1.5-flash', ...}

# Complex task -> capable model
result = router.get_model("Analyze the security implications of this architecture")
# Returns: {'complexity': EXPERT, 'model': 'claude-3-opus', ...}
```

## Strategy 2: Aggressive Caching

Never pay twice for the same answer.

```python
import hashlib
import json
import sqlite3
from datetime import datetime, timedelta
from typing import Optional, Dict, Any

class ResponseCache:
    """SQLite-based response cache with TTL"""
    
    def __init__(self, db_path: str = "cache.db", default_ttl_hours: int = 24):
        self.db_path = db_path
        self.default_ttl = timedelta(hours=default_ttl_hours)
        self._init_db()
    
    def _init_db(self):
        with sqlite3.connect(self.db_path) as conn:
            conn.execute("""
                CREATE TABLE IF NOT EXISTS cache (
                    cache_key TEXT PRIMARY KEY,
                    response TEXT,
                    model TEXT,
                    tokens_used INTEGER,
                    created_at TIMESTAMP,
                    expires_at TIMESTAMP,
                    hit_count INTEGER DEFAULT 0
                )
            """)
            conn.execute("CREATE INDEX IF NOT EXISTS idx_expires ON cache(expires_at)")
    
    def _make_key(self, prompt: str, model: str, system: str = "") -> str:
        """Generate cache key from request parameters"""
        content = f"{model}:{system}:{prompt}"
        return hashlib.sha256(content.encode()).hexdigest()[:32]
    
    def get(self, prompt: str, model: str, system: str = "") -> Optional[Dict]:
        """Retrieve cached response if valid"""
        key = self._make_key(prompt, model, system)
        
        with sqlite3.connect(self.db_path) as conn:
            conn.row_factory = sqlite3.Row
            row = conn.execute(
                "SELECT * FROM cache WHERE cache_key = ? AND expires_at > ?",
                (key, datetime.now().isoformat())
            ).fetchone()
            
            if row:
                # Update hit count
                conn.execute(
                    "UPDATE cache SET hit_count = hit_count + 1 WHERE cache_key = ?",
                    (key,)
                )
                return {
                    'response': json.loads(row['response']),
                    'cached': True,
                    'tokens_saved': row['tokens_used'],
                    'hit_count': row['hit_count'] + 1
                }
        return None
    
    def set(self, prompt: str, model: str, response: Any, 
            tokens_used: int, system: str = "", ttl: timedelta = None):
        """Cache a response"""
        key = self._make_key(prompt, model, system)
        ttl = ttl or self.default_ttl
        expires_at = datetime.now() + ttl
        
        with sqlite3.connect(self.db_path) as conn:
            conn.execute("""
                INSERT OR REPLACE INTO cache 
                (cache_key, response, model, tokens_used, created_at, expires_at, hit_count)
                VALUES (?, ?, ?, ?, ?, ?, 0)
            """, (key, json.dumps(response), model, tokens_used, 
                  datetime.now().isoformat(), expires_at.isoformat()))
    
    def get_stats(self) -> Dict:
        """Get cache statistics"""
        with sqlite3.connect(self.db_path) as conn:
            stats = conn.execute("""
                SELECT 
                    COUNT(*) as total_entries,
                    SUM(hit_count) as total_hits,
                    SUM(tokens_used * hit_count) as tokens_saved,
                    AVG(hit_count) as avg_hits_per_entry
                FROM cache
            """).fetchone()
            
            return {
                'total_entries': stats[0],
                'total_hits': stats[1] or 0,
                'tokens_saved': stats[2] or 0,
                'estimated_savings': f"${(stats[2] or 0) * 0.003 / 1000:.2f}"  # Assuming Sonnet pricing
            }

# Usage with wrapper
cache = ResponseCache()

async def cached_completion(client, model: str, prompt: str, system: str = ""):
    # Check cache first
    cached = cache.get(prompt, model, system)
    if cached:
        print(f"Cache hit! Saved {cached['tokens_saved']} tokens")
        return cached['response']
    
    # Make actual API call
    response = await client.messages.create(
        model=model,
        system=system,
        messages=[{"role": "user", "content": prompt}],
        max_tokens=4096
    )
    
    # Cache the response
    tokens_used = response.usage.input_tokens + response.usage.output_tokens
    cache.set(prompt, model, response.content[0].text, tokens_used, system)
    
    return response.content[0].text
```

## Strategy 3: Prompt Compression

Reduce token count without losing meaning.

```python
import re
from typing import List, Tuple

class PromptCompressor:
    """Compress prompts to reduce token usage"""
    
    # Common verbose patterns and their compressed forms
    COMPRESSIONS = [
        (r"I would like you to", "Please"),
        (r"Could you please", ""),
        (r"I want you to", ""),
        (r"Please make sure to", ""),
        (r"It is important that", ""),
        (r"In order to", "To"),
        (r"Due to the fact that", "Because"),
        (r"In the event that", "If"),
        (r"At this point in time", "Now"),
        (r"For the purpose of", "To"),
        (r"In addition to", "Also"),
        (r"A large number of", "Many"),
        (r"The majority of", "Most"),
        (r"In spite of the fact that", "Although"),
        (r"With regard to", "About"),
        (r"In reference to", "About"),
    ]
    
    @classmethod
    def compress(cls, prompt: str) -> Tuple[str, float]:
        """Compress prompt and return compression ratio"""
        original_length = len(prompt)
        compressed = prompt
        
        # Apply pattern replacements
        for pattern, replacement in cls.COMPRESSIONS:
            compressed = re.sub(pattern, replacement, compressed, flags=re.IGNORECASE)
        
        # Remove excessive whitespace
        compressed = re.sub(r'\s+', ' ', compressed)
        compressed = compressed.strip()
        
        # Remove redundant punctuation
        compressed = re.sub(r'\.{2,}', '.', compressed)
        compressed = re.sub(r'\s+([.,!?])', r'\1', compressed)
        
        new_length = len(compressed)
        ratio = (original_length - new_length) / original_length if original_length > 0 else 0
        
        return compressed, ratio
    
    @classmethod
    def compress_code_context(cls, code: str, max_lines: int = 100) -> str:
        """Compress code by removing comments and empty lines"""
        lines = code.split('\n')
        
        compressed_lines = []
        for line in lines:
            # Skip empty lines
            if not line.strip():
                continue
            # Skip comment-only lines
            if line.strip().startswith('#') or line.strip().startswith('//'):
                continue
            compressed_lines.append(line)
        
        # Truncate if too long
        if len(compressed_lines) > max_lines:
            half = max_lines // 2
            compressed_lines = (
                compressed_lines[:half] + 
                ['# ... truncated ...'] + 
                compressed_lines[-half:]
            )
        
        return '\n'.join(compressed_lines)
    
    @classmethod
    def summarize_long_context(cls, text: str, max_chars: int = 2000) -> str:
        """Truncate long context intelligently"""
        if len(text) <= max_chars:
            return text
        
        # Keep beginning and end
        keep_each = max_chars // 2 - 50
        return (
            text[:keep_each] + 
            "\n\n[... content truncated for brevity ...]\n\n" + 
            text[-keep_each:]
        )

# Usage
compressor = PromptCompressor()

verbose_prompt = """
I would like you to please analyze the following code. Could you please make sure to 
identify any potential security vulnerabilities. It is important that you check for 
SQL injection, XSS, and authentication issues. In addition to that, please also look 
for performance problems. Due to the fact that this is production code, I want you to 
be thorough.
"""

compressed, ratio = compressor.compress(verbose_prompt)
print(f"Compressed by {ratio:.1%}")
# Output: "Please analyze the following code. Identify any potential security 
# vulnerabilities. Check for SQL injection, XSS, and authentication issues. 
# Also look for performance problems. Because this is production code, be thorough."
```

## Strategy 4: Batch Processing

Combine multiple small requests into fewer large ones.

```python
import asyncio
from typing import List, Dict, Any
from dataclasses import dataclass

@dataclass
class BatchRequest:
    id: str
    prompt: str
    callback: callable = None

class BatchProcessor:
    """Batch multiple requests into single API calls"""
    
    def __init__(self, client, model: str, max_batch_size: int = 10, 
                 flush_interval: float = 1.0):
        self.client = client
        self.model = model
        self.max_batch_size = max_batch_size
        self.flush_interval = flush_interval
        self.pending: List[BatchRequest] = []
        self.results: Dict[str, Any] = {}
        self._lock = asyncio.Lock()
    
    async def add(self, request_id: str, prompt: str) -> Any:
        """Add request to batch and wait for result"""
        future = asyncio.Future()
        
        async with self._lock:
            self.pending.append(BatchRequest(
                id=request_id,
                prompt=prompt,
                callback=lambda r: future.set_result(r)
            ))
            
            if len(self.pending) >= self.max_batch_size:
                await self._flush()
        
        return await future
    
    async def _flush(self):
        """Process all pending requests in a single API call"""
        if not self.pending:
            return
        
        batch = self.pending[:self.max_batch_size]
        self.pending = self.pending[self.max_batch_size:]
        
        # Create combined prompt
        combined_prompt = self._create_batch_prompt(batch)
        
        # Single API call
        response = await self.client.messages.create(
            model=self.model,
            messages=[{"role": "user", "content": combined_prompt}],
            max_tokens=4096 * len(batch)
        )
        
        # Parse and distribute results
        results = self._parse_batch_response(response.content[0].text, batch)
        
        for request in batch:
            if request.callback and request.id in results:
                request.callback(results[request.id])
    
    def _create_batch_prompt(self, batch: List[BatchRequest]) -> str:
        """Create a combined prompt for batch processing"""
        prompt = """Process each of the following requests separately. 
Format your response as:

[REQUEST_1]
<response for request 1>
[/REQUEST_1]

[REQUEST_2]
<response for request 2>
[/REQUEST_2]

And so on for each request.

---
REQUESTS:
"""
        for i, req in enumerate(batch, 1):
            prompt += f"\n[REQUEST_{i}]\n{req.prompt}\n[/REQUEST_{i}]\n"
        
        return prompt
    
    def _parse_batch_response(self, response: str, batch: List[BatchRequest]) -> Dict:
        """Parse combined response into individual results"""
        results = {}
        
        for i, req in enumerate(batch, 1):
            pattern = rf'\[REQUEST_{i}\](.*?)\[/REQUEST_{i}\]'
            match = re.search(pattern, response, re.DOTALL)
            if match:
                results[req.id] = match.group(1).strip()
        
        return results

# Usage
processor = BatchProcessor(client, "claude-3-haiku-20240307")

# Instead of 10 separate API calls...
tasks = [
    processor.add(f"req_{i}", f"Summarize: {text}")
    for i, text in enumerate(documents)
]
results = await asyncio.gather(*tasks)
# ...you get 1-2 API calls with the same results!
```

## Strategy 5: Output Token Limits

Control response length to avoid runaway costs.

```python
class OutputController:
    """Control and estimate output costs"""
    
    # Approximate tokens per character ratio
    TOKENS_PER_CHAR = 0.25
    
    @staticmethod
    def estimate_tokens(text: str) -> int:
        """Estimate token count for text"""
        return int(len(text) * OutputController.TOKENS_PER_CHAR)
    
    @staticmethod
    def get_max_tokens_for_task(task_type: str) -> int:
        """Get appropriate max_tokens for task type"""
        limits = {
            'classification': 10,      # Just a label
            'yes_no': 5,              # Yes/No
            'short_answer': 100,       # Brief response
            'summary': 500,            # Paragraph
            'explanation': 1000,       # Detailed but bounded
            'code_snippet': 500,       # Small code block
            'full_code': 2000,         # Larger implementation
            'analysis': 1500,          # Thorough analysis
            'creative': 2000,          # Stories, content
            'unlimited': 4096          # When you really need it
        }
        return limits.get(task_type, 1000)
    
    @staticmethod
    def create_length_constrained_prompt(prompt: str, task_type: str) -> str:
        """Add length constraints to prompt"""
        constraints = {
            'classification': "Respond with only the category name.",
            'yes_no': "Respond with only 'Yes' or 'No'.",
            'short_answer': "Keep your response under 2 sentences.",
            'summary': "Summarize in one paragraph (3-5 sentences max).",
            'explanation': "Explain concisely in under 200 words.",
            'code_snippet': "Provide only the essential code, no explanations.",
        }
        
        constraint = constraints.get(task_type, "")
        if constraint:
            return f"{prompt}\n\n{constraint}"
        return prompt

# Usage
controller = OutputController()

# For a simple classification task
response = await client.messages.create(
    model="claude-3-haiku-20240307",  # Cheap model for simple task
    messages=[{
        "role": "user", 
        "content": controller.create_length_constrained_prompt(
            "Classify this email as: spam, promotional, personal, or business.\n\nEmail: ...",
            "classification"
        )
    }],
    max_tokens=controller.get_max_tokens_for_task("classification")  # Only 10 tokens!
)
```

## Common Pitfalls

### ❌ Using Opus for Everything
```python
# Don't do this
response = await client.messages.create(
    model="claude-3-opus-20240229",  # $75/M output tokens!
    messages=[{"role": "user", "content": "What's 2+2?"}]
)
```

### ❌ Unbounded Conversations
```python
# Without message pruning, costs explode
messages.append({"role": "user", "content": new_message})
# After 50 turns, you're sending 100K+ tokens per request!
```

### ❌ Ignoring Token Usage
```python
# Always track usage
response = await client.messages.create(...)
print(f"Tokens used: {response.usage.input_tokens + response.usage.output_tokens}")
print(f"Estimated cost: ${(response.usage.input_tokens * 0.003 + response.usage.output_tokens * 0.015) / 1000:.4f}")
```

## Pro Tips

### 1. Set Budget Alerts
```python
class BudgetTracker:
    def __init__(self, daily_limit: float = 10.0):
        self.daily_limit = daily_limit
        self.daily_spend = 0.0
        self.last_reset = datetime.now().date()
    
    def track_cost(self, input_tokens: int, output_tokens: int, model: str):
        # Reset daily if new day
        if datetime.now().date() > self.last_reset:
            self.daily_spend = 0.0
            self.last_reset = datetime.now().date()
        
        # Calculate cost based on model
        cost = self._calculate_cost(input_tokens, output_tokens, model)
        self.daily_spend += cost
        
        if self.daily_spend > self.daily_limit * 0.8:
            print(f"⚠️ Warning: 80% of daily budget used (${self.daily_spend:.2f})")
        
        if self.daily_spend > self.daily_limit:
            raise Exception(f"Daily budget exceeded! Spent ${self.daily_spend:.2f}")
        
        return cost
```

### 2. Use Streaming for Early Termination
```python
# Stop generation early if you have what you need
async for chunk in client.messages.stream(...):
    if "ANSWER:" in accumulated_text:
        # Got what we need, stop early
        break
```

### 3. Precompute Embeddings
```python
# For RAG systems, compute embeddings once and cache
embeddings_cache = {}

def get_embedding(text: str) -> List[float]:
    cache_key = hashlib.md5(text.encode()).hexdigest()
    if cache_key not in embeddings_cache:
        embeddings_cache[cache_key] = embedding_model.encode(text)
    return embeddings_cache[cache_key]
```

## Cost Tracking Dashboard

```python
from datetime import datetime, timedelta
import json

class CostDashboard:
    def __init__(self, log_file: str = "cost_log.jsonl"):
        self.log_file = log_file
    
    def log_request(self, model: str, input_tokens: int, output_tokens: int, 
                    task_type: str = "unknown"):
        entry = {
            "timestamp": datetime.now().isoformat(),
            "model": model,
            "input_tokens": input_tokens,
            "output_tokens": output_tokens,
            "task_type": task_type,
            "cost": self._calculate_cost(model, input_tokens, output_tokens)
        }
        with open(self.log_file, "a") as f:
            f.write(json.dumps(entry) + "\n")
    
    def get_daily_summary(self) -> Dict:
        today = datetime.now().date()
        daily_costs = {}
        
        with open(self.log_file, "r") as f:
            for line in f:
                entry = json.loads(line)
                entry_date = datetime.fromisoformat(entry["timestamp"]).date()
                if entry_date == today:
                    model = entry["model"]
                    daily_costs[model] = daily_costs.get(model, 0) + entry["cost"]
        
        return {
            "date": str(today),
            "by_model": daily_costs,
            "total": sum(daily_costs.values())
        }
    
    def _calculate_cost(self, model: str, input_tokens: int, output_tokens: int) -> float:
        # Pricing per 1M tokens
        pricing = {
            "claude-3-opus": (15.0, 75.0),
            "claude-3-sonnet": (3.0, 15.0),
            "claude-3-haiku": (0.25, 1.25),
            "gpt-4o": (5.0, 15.0),
            "gpt-4o-mini": (0.15, 0.60),
        }
        
        for model_key, (input_price, output_price) in pricing.items():
            if model_key in model.lower():
                return (input_tokens * input_price + output_tokens * output_price) / 1_000_000
        
        return 0.0  # Unknown model

# Usage
dashboard = CostDashboard()
dashboard.log_request("claude-3-haiku", 1000, 500, "summarization")
print(dashboard.get_daily_summary())
```

## Quick Wins Checklist

- [ ] Route simple tasks to cheap models (Haiku, GPT-4o-mini)
- [ ] Implement response caching
- [ ] Set max_tokens appropriately for each task
- [ ] Compress verbose prompts
- [ ] Batch small requests together
- [ ] Track costs per request
- [ ] Set daily budget limits
- [ ] Prune conversation history
- [ ] Use embeddings for semantic search instead of full LLM calls
- [ ] Monitor and alert on cost spikes

Remember: **The cheapest API call is the one you don't make.** Cache aggressively, route intelligently, and always ask: "Do I really need Opus for this?"