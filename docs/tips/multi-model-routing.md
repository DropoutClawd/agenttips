# Multi-Model Routing: Orchestrating AI Models Like a Pro

Why settle for one model when you can have the best of all worlds? Multi-model routing lets you use the right tool for each job - cheap models for simple tasks, powerful models for complex reasoning, and fast models when speed matters.

## Why Multi-Model?

| Scenario | Best Model Choice | Reason |
|----------|------------------|--------|
| Simple classification | GPT-4o-mini, Haiku | Cheap, fast, accurate enough |
| Complex reasoning | Claude Opus, GPT-4 | Deep thinking required |
| Code generation | Claude Sonnet, Codestral | Specialized training |
| Real-time chat | Groq Llama, Gemini Flash | Ultra-low latency |
| Long documents | Claude (200K), Gemini (1M) | Large context windows |
| Cost-sensitive batch | Llama 3 (local), Mixtral | Free/cheap at scale |

## Strategy 1: Capability-Based Routing

```python
from enum import Enum, auto
from dataclasses import dataclass
from typing import Dict, List, Optional, Any
import asyncio

class Capability(Enum):
    REASONING = auto()      # Complex logic, math, analysis
    CODING = auto()         # Code generation and review
    CREATIVITY = auto()     # Writing, brainstorming
    SPEED = auto()          # Low latency required
    CONTEXT = auto()        # Long document processing
    COST = auto()           # Budget optimization
    VISION = auto()         # Image understanding
    TOOLS = auto()          # Function calling

@dataclass
class ModelSpec:
    name: str
    provider: str
    capabilities: Dict[Capability, int]  # 1-10 rating
    cost_per_1k_tokens: float
    max_context: int
    avg_latency_ms: int

class CapabilityRouter:
    """Route requests based on required capabilities"""
    
    MODELS = [
        ModelSpec(
            name="claude-3-opus-20240229",
            provider="anthropic",
            capabilities={
                Capability.REASONING: 10,
                Capability.CODING: 9,
                Capability.CREATIVITY: 10,
                Capability.TOOLS: 9,
                Capability.CONTEXT: 8,
            },
            cost_per_1k_tokens=0.045,
            max_context=200000,
            avg_latency_ms=3000
        ),
        ModelSpec(
            name="claude-3-5-sonnet-20241022",
            provider="anthropic",
            capabilities={
                Capability.REASONING: 9,
                Capability.CODING: 10,
                Capability.CREATIVITY: 9,
                Capability.TOOLS: 10,
                Capability.CONTEXT: 8,
            },
            cost_per_1k_tokens=0.009,
            max_context=200000,
            avg_latency_ms=1500
        ),
        ModelSpec(
            name="claude-3-haiku-20240307",
            provider="anthropic",
            capabilities={
                Capability.REASONING: 7,
                Capability.CODING: 7,
                Capability.CREATIVITY: 7,
                Capability.SPEED: 9,
                Capability.TOOLS: 8,
            },
            cost_per_1k_tokens=0.00075,
            max_context=200000,
            avg_latency_ms=500
        ),
        ModelSpec(
            name="gpt-4o",
            provider="openai",
            capabilities={
                Capability.REASONING: 9,
                Capability.CODING: 9,
                Capability.VISION: 10,
                Capability.TOOLS: 10,
            },
            cost_per_1k_tokens=0.01,
            max_context=128000,
            avg_latency_ms=2000
        ),
        ModelSpec(
            name="gpt-4o-mini",
            provider="openai",
            capabilities={
                Capability.REASONING: 7,
                Capability.CODING: 7,
                Capability.SPEED: 8,
                Capability.COST: 9,
                Capability.VISION: 8,
            },
            cost_per_1k_tokens=0.00045,
            max_context=128000,
            avg_latency_ms=800
        ),
        ModelSpec(
            name="llama-3.3-70b-versatile",
            provider="groq",
            capabilities={
                Capability.REASONING: 8,
                Capability.CODING: 8,
                Capability.SPEED: 10,
                Capability.COST: 10,
            },
            cost_per_1k_tokens=0.0007,
            max_context=128000,
            avg_latency_ms=200
        ),
        ModelSpec(
            name="gemini-1.5-flash",
            provider="google",
            capabilities={
                Capability.SPEED: 9,
                Capability.COST: 10,
                Capability.CONTEXT: 10,
                Capability.VISION: 9,
            },
            cost_per_1k_tokens=0.0002,
            max_context=1000000,
            avg_latency_ms=400
        ),
    ]
    
    def __init__(self):
        self.model_index = {m.name: m for m in self.MODELS}
    
    def select_model(
        self,
        required: List[Capability],
        preferred: List[Capability] = None,
        min_scores: Dict[Capability, int] = None,
        max_cost: float = None,
        max_latency: int = None,
        min_context: int = None
    ) -> ModelSpec:
        """Select best model based on requirements"""
        
        candidates = self.MODELS.copy()
        min_scores = min_scores or {}
        preferred = preferred or []
        
        # Filter by hard constraints
        if max_cost:
            candidates = [m for m in candidates if m.cost_per_1k_tokens <= max_cost]
        
        if max_latency:
            candidates = [m for m in candidates if m.avg_latency_ms <= max_latency]
        
        if min_context:
            candidates = [m for m in candidates if m.max_context >= min_context]
        
        # Filter by minimum capability scores
        for cap, min_score in min_scores.items():
            candidates = [
                m for m in candidates 
                if m.capabilities.get(cap, 0) >= min_score
            ]
        
        # Filter by required capabilities
        for cap in required:
            candidates = [
                m for m in candidates 
                if m.capabilities.get(cap, 0) >= 5  # Minimum threshold
            ]
        
        if not candidates:
            raise ValueError("No model matches all requirements")
        
        # Score remaining candidates
        def score_model(model: ModelSpec) -> float:
            score = 0
            
            # Required capabilities (weighted heavily)
            for cap in required:
                score += model.capabilities.get(cap, 0) * 3
            
            # Preferred capabilities
            for cap in preferred:
                score += model.capabilities.get(cap, 0)
            
            # Cost efficiency bonus (lower is better)
            score += (0.05 - model.cost_per_1k_tokens) * 100
            
            return score
        
        # Return highest scoring model
        return max(candidates, key=score_model)

# Usage examples
router = CapabilityRouter()

# For a coding task that needs to be fast
model = router.select_model(
    required=[Capability.CODING],
    preferred=[Capability.SPEED],
    max_latency=1000
)
print(f"Selected: {model.name}")  # Likely Groq Llama or Haiku

# For complex reasoning, cost doesn't matter
model = router.select_model(
    required=[Capability.REASONING],
    min_scores={Capability.REASONING: 9}
)
print(f"Selected: {model.name}")  # Likely Opus

# For processing a huge document cheaply
model = router.select_model(
    required=[Capability.CONTEXT],
    preferred=[Capability.COST],
    min_context=500000
)
print(f"Selected: {model.name}")  # Gemini 1.5 Flash
```

## Strategy 2: Task-Based Auto-Router

```python
import re
from typing import Tuple

class TaskAutoRouter:
    """Automatically detect task type and route to best model"""
    
    TASK_PATTERNS = {
        'code_generation': [
            r'write (a |the )?(code|function|class|script)',
            r'implement',
            r'create (a |the )?(program|api|endpoint)',
            r'```\w+',  # Code blocks in input
        ],
        'code_review': [
            r'review (this |the )?(code|pr|pull request)',
            r'find (bugs|issues|problems)',
            r'what\'s wrong with',
            r'debug',
        ],
        'analysis': [
            r'analyze',
            r'compare',
            r'evaluate',
            r'assess',
            r'pros and cons',
        ],
        'summarization': [
            r'summarize',
            r'tldr',
            r'key points',
            r'brief overview',
        ],
        'creative': [
            r'write (a |the )?(story|poem|essay|article)',
            r'brainstorm',
            r'creative',
            r'imagine',
        ],
        'simple_qa': [
            r'^what (is|are|was|were)',
            r'^who (is|are|was|were)',
            r'^when (did|was|were)',
            r'^where (is|are|was|were)',
            r'^how (do|does|did)',
        ],
        'classification': [
            r'classify',
            r'categorize',
            r'which (category|type|kind)',
            r'is this (a |an )?',
        ],
        'extraction': [
            r'extract',
            r'find all',
            r'list (the |all )?',
            r'get (the |all )?',
        ],
        'translation': [
            r'translate',
            r'in (spanish|french|german|chinese|japanese)',
            r'to (spanish|french|german|chinese|japanese)',
        ],
    }
    
    TASK_TO_MODEL = {
        'code_generation': ('claude-3-5-sonnet-20241022', 'anthropic'),
        'code_review': ('claude-3-5-sonnet-20241022', 'anthropic'),
        'analysis': ('claude-3-opus-20240229', 'anthropic'),
        'summarization': ('claude-3-haiku-20240307', 'anthropic'),
        'creative': ('claude-3-opus-20240229', 'anthropic'),
        'simple_qa': ('llama-3.3-70b-versatile', 'groq'),
        'classification': ('gpt-4o-mini', 'openai'),
        'extraction': ('gemini-1.5-flash', 'google'),
        'translation': ('gpt-4o-mini', 'openai'),
        'default': ('claude-3-5-sonnet-20241022', 'anthropic'),
    }
    
    def detect_task(self, prompt: str) -> str:
        """Detect task type from prompt"""
        prompt_lower = prompt.lower()
        
        for task_type, patterns in self.TASK_PATTERNS.items():
            for pattern in patterns:
                if re.search(pattern, prompt_lower):
                    return task_type
        
        return 'default'
    
    def route(self, prompt: str) -> Tuple[str, str, str]:
        """Route prompt to appropriate model"""
        task_type = self.detect_task(prompt)
        model, provider = self.TASK_TO_MODEL[task_type]
        return model, provider, task_type

# Usage
router = TaskAutoRouter()

prompts = [
    "Write a Python function to merge two sorted arrays",
    "What is the capital of France?",
    "Summarize this article about climate change...",
    "Analyze the pros and cons of microservices architecture",
    "Extract all email addresses from this text",
]

for prompt in prompts:
    model, provider, task = router.route(prompt)
    print(f"Task: {task:20} -> {provider}/{model}")
```

## Strategy 3: Ensemble Routing (Multiple Models)

```python
import asyncio
from typing import List, Dict, Any
from collections import Counter

class EnsembleRouter:
    """Use multiple models and aggregate results"""
    
    def __init__(self, clients: Dict[str, Any]):
        self.clients = clients
    
    async def query_model(
        self, 
        provider: str, 
        model: str, 
        prompt: str
    ) -> Dict[str, Any]:
        """Query a single model"""
        client = self.clients[provider]
        
        try:
            if provider == 'anthropic':
                response = await client.messages.create(
                    model=model,
                    messages=[{"role": "user", "content": prompt}],
                    max_tokens=1024
                )
                return {
                    'provider': provider,
                    'model': model,
                    'response': response.content[0].text,
                    'success': True
                }
            elif provider == 'openai':
                response = await client.chat.completions.create(
                    model=model,
                    messages=[{"role": "user", "content": prompt}],
                    max_tokens=1024
                )
                return {
                    'provider': provider,
                    'model': model,
                    'response': response.choices[0].message.content,
                    'success': True
                }
        except Exception as e:
            return {
                'provider': provider,
                'model': model,
                'error': str(e),
                'success': False
            }
    
    async def consensus_query(
        self,
        prompt: str,
        models: List[tuple[str, str]],  # (provider, model) pairs
        min_agreement: int = 2
    ) -> Dict[str, Any]:
        """Query multiple models and return consensus answer"""
        
        # Query all models in parallel
        tasks = [
            self.query_model(provider, model, prompt)
            for provider, model in models
        ]
        results = await asyncio.gather(*tasks)
        
        # Filter successful responses
        successful = [r for r in results if r['success']]
        
        if len(successful) < min_agreement:
            raise ValueError("Not enough successful responses for consensus")
        
        # For classification tasks, use voting
        responses = [r['response'].strip().lower() for r in successful]
        counter = Counter(responses)
        
        most_common, count = counter.most_common(1)[0]
        
        return {
            'consensus': most_common,
            'agreement': count,
            'total': len(successful),
            'confidence': count / len(successful),
            'all_responses': successful
        }
    
    async def best_of_n(
        self,
        prompt: str,
        models: List[tuple[str, str]],
        judge_model: tuple[str, str] = ('anthropic', 'claude-3-opus-20240229')
    ) -> Dict[str, Any]:
        """Generate multiple responses and have a judge pick the best"""
        
        # Get responses from all models
        tasks = [
            self.query_model(provider, model, prompt)
            for provider, model in models
        ]
        results = await asyncio.gather(*tasks)
        successful = [r for r in results if r['success']]
        
        if len(successful) == 1:
            return successful[0]
        
        # Have judge model pick the best
        judge_prompt = f"""Original question: {prompt}

Here are {len(successful)} responses from different AI models. Pick the best one.

"""
        for i, r in enumerate(successful, 1):
            judge_prompt += f"Response {i} ({r['model']}):\n{r['response']}\n\n"
        
        judge_prompt += """Which response is best? Respond with just the number (1, 2, etc.) 
and a brief explanation why."""
        
        judge_result = await self.query_model(
            judge_model[0], 
            judge_model[1], 
            judge_prompt
        )
        
        # Parse judge's choice
        import re
        match = re.search(r'\b([1-9])\b', judge_result['response'])
        if match:
            choice = int(match.group(1)) - 1
            if 0 <= choice < len(successful):
                return {
                    'best_response': successful[choice],
                    'judge_reasoning': judge_result['response'],
                    'all_responses': successful
                }
        
        # Fallback to first response if parsing fails
        return {
            'best_response': successful[0],
            'judge_reasoning': judge_result['response'],
            'all_responses': successful
        }

# Usage
ensemble = EnsembleRouter(clients={
    'anthropic': anthropic_client,
    'openai': openai_client,
    'groq': groq_client
})

# Consensus for classification
result = await ensemble.consensus_query(
    prompt="Is this email spam or not spam?\n\nSubject: You've won $1M!!!",
    models=[
        ('anthropic', 'claude-3-haiku-20240307'),
        ('openai', 'gpt-4o-mini'),
        ('groq', 'llama-3.3-70b-versatile')
    ]
)
print(f"Consensus: {result['consensus']} ({result['confidence']:.0%} agreement)")

# Best-of-N for quality
result = await ensemble.best_of_n(
    prompt="Explain quantum entanglement to a 10-year-old",
    models=[
        ('anthropic', 'claude-3-sonnet-20240229'),
        ('openai', 'gpt-4o'),
    ]
)
print(f"Best response from: {result['best_response']['model']}")
```

## Strategy 4: Dynamic Load Balancing

```python
import asyncio
import time
from dataclasses import dataclass, field
from typing import Dict, List, Optional
import random

@dataclass
class ProviderHealth:
    provider: str
    success_count: int = 0
    failure_count: int = 0
    total_latency_ms: float = 0
    last_failure: Optional[float] = None
    circuit_open: bool = False
    
    @property
    def success_rate(self) -> float:
        total = self.success_count + self.failure_count
        return self.success_count / total if total > 0 else 1.0
    
    @property
    def avg_latency(self) -> float:
        return self.total_latency_ms / self.success_count if self.success_count > 0 else 0
    
    @property
    def score(self) -> float:
        """Health score for load balancing (higher is better)"""
        if self.circuit_open:
            return 0
        # Weighted combination of success rate and latency
        latency_score = max(0, 1 - self.avg_latency / 5000)  # Normalize to 5s
        return self.success_rate * 0.7 + latency_score * 0.3

class LoadBalancer:
    """Intelligent load balancing across providers"""
    
    def __init__(self):
        self.health: Dict[str, ProviderHealth] = {}
        self.circuit_break_threshold = 5  # Failures before opening circuit
        self.circuit_reset_time = 60  # Seconds before trying again
    
    def get_health(self, provider: str) -> ProviderHealth:
        if provider not in self.health:
            self.health[provider] = ProviderHealth(provider=provider)
        return self.health[provider]
    
    def record_success(self, provider: str, latency_ms: float):
        """Record successful request"""
        health = self.get_health(provider)
        health.success_count += 1
        health.total_latency_ms += latency_ms
        health.circuit_open = False
    
    def record_failure(self, provider: str):
        """Record failed request"""
        health = self.get_health(provider)
        health.failure_count += 1
        health.last_failure = time.time()
        
        # Check circuit breaker
        recent_failures = health.failure_count
        if recent_failures >= self.circuit_break_threshold:
            health.circuit_open = True
            print(f"Circuit breaker OPEN for {provider}")
    
    def check_circuit(self, provider: str) -> bool:
        """Check if circuit should be closed (allow retry)"""
        health = self.get_health(provider)
        if not health.circuit_open:
            return True
        
        if health.last_failure and (time.time() - health.last_failure > self.circuit_reset_time):
            health.circuit_open = False
            health.failure_count = 0  # Reset for fresh start
            print(f"Circuit breaker CLOSED for {provider}")
            return True
        
        return False
    
    def select_provider(
        self, 
        providers: List[str],
        strategy: str = "weighted"
    ) -> str:
        """Select best provider based on health"""
        
        # Filter out circuit-broken providers
        available = [p for p in providers if self.check_circuit(p)]
        
        if not available:
            # All circuits open - try the one that failed longest ago
            available = providers
            print("Warning: All circuits open, selecting least-recently-failed")
        
        if strategy == "weighted":
            # Weighted random selection based on health scores
            scores = [max(self.get_health(p).score, 0.1) for p in available]
            total = sum(scores)
            probs = [s / total for s in scores]
            return random.choices(available, weights=probs)[0]
        
        elif strategy == "best":
            # Always pick healthiest
            return max(available, key=lambda p: self.get_health(p).score)
        
        elif strategy == "round_robin":
            # Simple round robin
            return available[int(time.time()) % len(available)]
        
        else:
            return random.choice(available)
    
    def get_stats(self) -> Dict:
        """Get health statistics for all providers"""
        return {
            provider: {
                'success_rate': f"{health.success_rate:.1%}",
                'avg_latency': f"{health.avg_latency:.0f}ms",
                'circuit_open': health.circuit_open,
                'score': f"{health.score:.2f}"
            }
            for provider, health in self.health.items()
        }

# Usage
balancer = LoadBalancer()

async def make_balanced_request(prompt: str) -> str:
    providers = ['anthropic', 'openai', 'groq']
    
    # Select provider
    provider = balancer.select_provider(providers, strategy="weighted")
    
    start = time.time()
    try:
        result = await make_request(provider, prompt)
        latency = (time.time() - start) * 1000
        balancer.record_success(provider, latency)
        return result
    except Exception as e:
        balancer.record_failure(provider)
        # Retry with different provider
        remaining = [p for p in providers if p != provider]
        if remaining:
            provider = balancer.select_provider(remaining)
            return await make_request(provider, prompt)
        raise

# Check stats periodically
print(balancer.get_stats())
```

## Common Pitfalls

### ❌ Hardcoding Model Names
```python
# Don't do this
response = await client.messages.create(
    model="claude-3-opus-20240229",  # Will break when deprecated
    ...
)

# Do this
MODEL_CONFIG = {
    'reasoning': os.getenv('REASONING_MODEL', 'claude-3-opus-20240229'),
    'fast': os.getenv('FAST_MODEL', 'claude-3-haiku-20240307'),
}
```

### ❌ Ignoring Context Window Limits
```python
# Check before sending
def safe_route(prompt: str, context: str) -> str:
    total_tokens = estimate_tokens(prompt + context)
    
    if total_tokens > 100000:
        return "gemini-1.5-flash"  # 1M context
    elif total_tokens > 30000:
        return "claude-3-5-sonnet"  # 200K context
    else:
        return "gpt-4o-mini"  # 128K context
```

### ❌ Not Handling Provider Differences
```python
# Different providers have different APIs
async def unified_complete(provider: str, model: str, prompt: str) -> str:
    if provider == 'anthropic':
        response = await anthropic_client.messages.create(
            model=model,
            messages=[{"role": "user", "content": prompt}]
        )
        return response.content[0].text
    elif provider == 'openai':
        response = await openai_client.chat.completions.create(
            model=model,
            messages=[{"role": "user", "content": prompt}]
        )
        return response.choices[0].message.content
    # ... etc
```

## Pro Tips

### 1. Model Warm-up
```python
# Pre-warm models to reduce first-request latency
async def warmup_models():
    tasks = [
        make_request(provider, "Hello")
        for provider in ['anthropic', 'openai', 'groq']
    ]
    await asyncio.gather(*tasks, return_exceptions=True)
```

### 2. Semantic Caching Across Models
```python
# Cache by semantic meaning, not exact prompt
from sentence_transformers import SentenceTransformer

class SemanticCache:
    def __init__(self, similarity_threshold: float = 0.95):
        self.encoder = SentenceTransformer('all-MiniLM-L6-v2')
        self.cache = []  # (embedding, prompt, response)
        self.threshold = similarity_threshold
    
    def get(self, prompt: str) -> Optional[str]:
        embedding = self.encoder.encode(prompt)
        
        for cached_emb, cached_prompt, response in self.cache:
            similarity = cosine_similarity(embedding, cached_emb)
            if similarity > self.threshold:
                return response
        return None
```

### 3. A/B Testing Models
```python
import random

class ABTester:
    def __init__(self):
        self.results = {}
    
    def select_variant(self, test_name: str, variants: List[str]) -> str:
        return random.choice(variants)
    
    def record_outcome(self, test_name: str, variant: str, 
                       success: bool, latency: float, cost: float):
        key = (test_name, variant)
        if key not in self.results:
            self.results[key] = {'successes': 0, 'total': 0, 
                                  'latency': [], 'cost': []}
        
        self.results[key]['total'] += 1
        if success:
            self.results[key]['successes'] += 1
        self.results[key]['latency'].append(latency)
        self.results[key]['cost'].append(cost)
```

## Quick Checklist

- [ ] Map task types to optimal models
- [ ] Implement automatic task detection
- [ ] Handle different provider APIs uniformly
- [ ] Track model health and latency
- [ ] Implement circuit breakers
- [ ] Use weighted load balancing
- [ ] Cache responses across models
- [ ] Monitor costs by model
- [ ] Plan for model deprecations
- [ ] Test fallback chains regularly

Remember: **The best model for a task isn't always the most powerful one - it's the one that delivers the right quality at the right speed and cost.** Build your routing logic with this principle in mind.