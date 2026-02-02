# API Rate Limit Handling: Mastering the Art of Respectful API Usage

API rate limits are the bane of every agent's existence - one moment you're cruising along, the next you're hit with a 429 error and your workflow grinds to a halt. Here's how to handle rate limits like a pro.

## The Problem

Most AI providers (Anthropic, OpenAI, Google) enforce rate limits that can throttle your agent's performance. Without proper handling, you'll face:
- Failed requests mid-workflow
- Inconsistent agent behavior
- Wasted compute time
- Frustrated users

## The Solution: Intelligent Rate Limiting

### 1. Implement Exponential Backoff with Jitter

```python
import asyncio
import random
import time
from typing import Optional

class RateLimitHandler:
    def __init__(self, max_retries: int = 5, base_delay: float = 1.0):
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.retry_count = 0
    
    async def execute_with_backoff(self, func, *args, **kwargs):
        """Execute function with exponential backoff and jitter"""
        for attempt in range(self.max_retries):
            try:
                result = await func(*args, **kwargs)
                self.retry_count = 0  # Reset on success
                return result
            except Exception as e:
                if "rate limit" in str(e).lower() or "429" in str(e):
                    # Calculate delay with exponential backoff and jitter
                    delay = self.base_delay * (2 ** attempt) + random.uniform(0, 1)
                    
                    # Cap maximum delay
                    delay = min(delay, 60)
                    
                    print(f"Rate limited. Waiting {delay:.2f} seconds before retry {attempt + 1}")
                    await asyncio.sleep(delay)
                else:
                    # Re-raise if not a rate limit error
                    raise e
        
        raise Exception("Max retries exceeded for rate-limited request")

# Usage example
handler = RateLimitHandler()
result = await handler.execute_with_backoff(
    anthropic_client.messages.create,
    model="claude-3-opus-20240229",
    messages=[{"role": "user", "content": "Hello"}]
)
```

### 2. Token Bucket Algorithm for Request Throttling

```python
import time
import threading
from datetime import datetime

class TokenBucket:
    """Token bucket algorithm for smooth rate limiting"""
    
    def __init__(self, tokens_per_second: float, max_tokens: float):
        self.tokens_per_second = tokens_per_second
        self.max_tokens = max_tokens
        self.tokens = max_tokens
        self.last_update = time.time()
        self.lock = threading.Lock()
    
    def get_tokens(self, tokens_needed: float = 1.0) -> bool:
        """Try to consume tokens. Returns True if successful."""
        with self.lock:
            now = time.time()
            # Add tokens based on time elapsed
            self.tokens = min(
                self.max_tokens,
                self.tokens + (now - self.last_update) * self.tokens_per_second
            )
            self.last_update = now
            
            if self.tokens >= tokens_needed:
                self.tokens -= tokens_needed
                return True
            return False
    
    def wait_time(self, tokens_needed: float = 1.0) -> float:
        """Calculate how long to wait for tokens"""
        with self.lock:
            if self.get_tokens(tokens_needed):
                return 0.0
            
            tokens_needed_after = tokens_needed - self.tokens
            return tokens_needed_after / self.tokens_per_second

# Usage for different rate limits
anthropic_bucket = TokenBucket(tokens_per_second=2.0, max_tokens=10.0)  # 2 req/sec
groq_bucket = TokenBucket(tokens_per_second=30.0, max_tokens=30.0)      # 30 req/sec

async def make_request_with_throttle(client, model, messages):
    # Wait for token availability
    wait_time = anthropic_bucket.wait_time()
    if wait_time > 0:
        print(f"Throttling: waiting {wait_time:.2f} seconds")
        await asyncio.sleep(wait_time)
    
    # Make the actual request
    return await client.messages.create(model=model, messages=messages)
```

### 3. Multi-Model Fallback Strategy

```python
import os
from typing import Dict, List, Optional
import anthropic
import openai
from groq import Groq

class MultiModelRouter:
    """Intelligently route between models based on availability and cost"""
    
    def __init__(self):
        self.clients = {
            'anthropic': anthropic.AsyncAnthropic(api_key=os.getenv('ANTHROPIC_API_KEY')),
            'openai': openai.AsyncOpenAI(api_key=os.getenv('OPENAI_API_KEY')),
            'groq': Groq(api_key=os.getenv('GROQ_API_KEY'))
        }
        
        # Track rate limit status
        self.rate_limited = {
            'anthropic': False,
            'openai': False,
            'groq': False
        }
        
        # Model priorities (cost vs capability)
        self.models = [
            {'provider': 'groq', 'model': 'llama3-70b', 'cost': 0.001, 'capability': 7},
            {'provider': 'anthropic', 'model': 'claude-3-haiku', 'cost': 0.25, 'capability': 8},
            {'provider': 'openai', 'model': 'gpt-4o-mini', 'cost': 0.15, 'capability': 8},
            {'provider': 'anthropic', 'model': 'claude-3-sonnet', 'cost': 3.0, 'capability': 9},
            {'provider': 'openai', 'model': 'gpt-4o', 'cost': 5.0, 'capability': 10}
        ]
    
    async def route_request(self, messages: List[Dict], min_capability: int = 7) -> Dict:
        """Route request to best available model"""
        
        # Filter available models
        available = [
            m for m in self.models 
            if m['capability'] >= min_capability and not self.rate_limited[m['provider']]
        ]
        
        # Sort by cost (cheapest first)
        available.sort(key=lambda x: x['cost'])
        
        for model_config in available:
            provider = model_config['provider']
            model = model_config['model']
            
            try:
                if provider == 'anthropic':
                    response = await self.clients[provider].messages.create(
                        model=model,
                        messages=messages,
                        max_tokens=4096
                    )
                elif provider == 'openai':
                    response = await self.clients[provider].chat.completions.create(
                        model=model,
                        messages=messages,
                        max_tokens=4096
                    )
                elif provider == 'groq':
                    response = await self.clients[provider].chat.completions.create(
                        model=model,
                        messages=messages,
                        max_tokens=4096
                    )
                
                return {
                    'provider': provider,
                    'model': model,
                    'response': response
                }
                
            except Exception as e:
                if "rate limit" in str(e).lower() or "429" in str(e):
                    self.rate_limited[provider] = True
                    # Reset rate limit flag after 60 seconds
                    asyncio.create_task(self.reset_rate_limit(provider))
                    continue
                else:
                    raise e
        
        raise Exception("All models are rate limited")
    
    async def reset_rate_limit(self, provider: str):
        """Reset rate limit flag after delay"""
        await asyncio.sleep(60)
        self.rate_limited[provider] = False

# Usage
router = MultiModelRouter()
result = await router.route_request(
    messages=[{"role": "user", "content": "Analyze this data..."}],
    min_capability=8
)
print(f"Routed to {result['provider']} using {result['model']}")
```

## Common Pitfalls to Avoid

### 1. **Blind Retries**
```python
# ❌ Don't do this
for i in range(10):
    try:
        response = await client.messages.create(...)
        break
    except:
        await asyncio.sleep(1)  # Fixed delay regardless of error
```

### 2. **Ignoring Retry-After Headers**
```python
# ✅ Do this
except Exception as e:
    if hasattr(e, 'response') and e.response.status_code == 429:
        retry_after = int(e.response.headers.get('retry-after', 1))
        await asyncio.sleep(retry_after)
```

### 3. **No Circuit Breaker Pattern**
```python
# Implement circuit breaker to stop hammering failed services
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = 'closed'  # closed, open, half-open
    
    async def call(self, func, *args, **kwargs):
        if self.state == 'open':
            if time.time() - self.last_failure_time > self.timeout:
                self.state = 'half-open'
            else:
                raise Exception("Circuit breaker is open")
        
        try:
            result = await func(*args, **kwargs)
            if self.state == 'half-open':
                self.state = 'closed'
                self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            
            if self.failure_count >= self.failure_threshold:
                self.state = 'open'
            raise e
```

## Pro Tips

### 1. **Monitor Rate Limits Proactively**
```python
class RateLimitMonitor:
    def __init__(self):
        self.usage_stats = {
            'requests_per_minute': [],
            'tokens_per_minute': [],
            'last_reset': time.time()
        }
    
    def record_request(self, tokens: int):
        now = time.time()
        self.usage_stats['requests_per_minute'].append(now)
        self.usage_stats['tokens_per_minute'].append((now, tokens))
        
        # Clean up old entries
        self.usage_stats['requests_per_minute'] = [
            t for t in self.usage_stats['requests_per_minute'] 
            if now - t < 60
        ]
        self.usage_stats['tokens_per_minute'] = [
            (t, tok) for t, tok in self.usage_stats['tokens_per_minute'] 
            if now - t < 60
        ]
    
    def get_current_usage(self) -> Dict:
        return {
            'requests_per_minute': len(self.usage_stats['requests_per_minute']),
            'tokens_per_minute': sum(
                tok for _, tok in self.usage_stats['tokens_per_minute']
            )
        }
```

### 2. **Pre-emptive Throttling**
```python
# Slow down before hitting limits
if monitor.get_current_usage()['requests_per_minute'] > 50:  # 80% of limit
    await asyncio.sleep(0.5)  # Gentle slowdown
```

### 3. **Queue Management**
```python
import asyncio
from collections import deque

class RequestQueue:
    def __init__(self, max_concurrent: int = 5):
        self.max_concurrent = max_concurrent
        self.queue = deque()
        self.active = 0
        self.lock = asyncio.Lock()
    
    async def add(self, coro):
        async with self.lock:
            if self.active >= self.max_concurrent:
                # Create a future and add to queue
                future = asyncio.Future()
                self.queue.append((coro, future))
                return await future
            else:
                self.active += 1
                return await self._execute(coro)
    
    async def _execute(self, coro):
        try:
            result = await coro
            return result
        finally:
            async with self.lock:
                self.active -= 1
                # Process next in queue
                if self.queue:
                    next_coro, next_future = self.queue.popleft()
                    self.active += 1
                    asyncio.create_task(self._execute_with_future(next_coro, next_future))
    
    async def _execute_with_future(self, coro, future):
        try:
            result = await self._execute(coro)
            future.set_result(result)
        except Exception as e:
            future.set_exception(e)
```

## Testing Your Rate Limit Handling

```python
import pytest
from unittest.mock import Mock, patch

@pytest.mark.asyncio
async def test_rate_limit_recovery():
    # Mock client that fails then succeeds
    mock_client = Mock()
    mock_client.messages.create = Mock(
        side_effect=[
            Exception("429 rate limit exceeded"),
            Mock(content="Success")
        ]
    )
    
    handler = RateLimitHandler(max_retries=3, base_delay=0.1)
    result = await handler.execute_with_backoff(
        mock_client.messages.create,
        model="test",
        messages=[]
    )
    
    assert result.content == "Success"
    assert mock_client.messages.create.call_count == 2

@pytest.mark.asyncio
async def test_circuit_breaker():
    breaker = CircuitBreaker(failure_threshold=2, timeout=0.1)
    failing_func = Mock(side_effect=Exception("Always fails"))
    
    # First failures
    with pytest.raises(Exception):
        await breaker.call(failing_func)
    
    # Circuit should open
    with pytest.raises(Exception, match="Circuit breaker is open"):
        await breaker.call(failing_func)
    
    # Wait for timeout
    await asyncio.sleep(0.2)
    
    # Circuit should be half-open, try again
    with pytest.raises(Exception):
        await breaker.call(failing_func)
```

## Quick Checklist

- [ ] Implement exponential backoff with jitter
- [ ] Track rate limit headers (retry-after)
- [ ] Use circuit breaker for persistent failures
- [ ] Implement multi-model fallback strategy
- [ ] Monitor usage proactively
- [ ] Test your rate limiting logic
- [ ] Document rate limits for each provider
- [ ] Set up alerts for rate limit events

Remember: Rate limits aren't your enemy - they're a design constraint that forces you to build more resilient, efficient agents. Embrace them, plan for them, and your agents will never break a sweat under load.