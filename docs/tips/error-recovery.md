# Error Recovery: Building Agents That Never Give Up

Errors are inevitable. Network timeouts, malformed responses, API outages, invalid JSON - your agent will face them all. The difference between a fragile agent and a robust one isn't avoiding errors, it's recovering from them gracefully.

## The Error Taxonomy

Understanding error types is the first step to handling them properly:

| Error Type | Examples | Recovery Strategy |
|------------|----------|-------------------|
| Transient | Network timeout, 503 | Retry with backoff |
| Rate Limit | 429, quota exceeded | Backoff + fallback |
| Client Error | 400, invalid request | Fix and retry |
| Auth Error | 401, 403 | Refresh credentials |
| Server Error | 500, 502 | Wait and retry |
| Parse Error | Invalid JSON | Request regeneration |
| Logic Error | Wrong output format | Re-prompt with examples |

## Strategy 1: Structured Error Handling

```python
import asyncio
from enum import Enum
from dataclasses import dataclass
from typing import Optional, Callable, Any
import traceback

class ErrorCategory(Enum):
    TRANSIENT = "transient"      # Retry immediately
    RATE_LIMIT = "rate_limit"    # Retry with longer delay
    CLIENT = "client"            # Fix request and retry
    AUTH = "auth"                # Refresh credentials
    SERVER = "server"            # Wait and retry
    PARSE = "parse"              # Request regeneration
    FATAL = "fatal"              # Don't retry

@dataclass
class ClassifiedError:
    category: ErrorCategory
    original_error: Exception
    message: str
    retry_after: Optional[float] = None
    should_retry: bool = True

class ErrorClassifier:
    """Classify errors into actionable categories"""
    
    @staticmethod
    def classify(error: Exception) -> ClassifiedError:
        error_str = str(error).lower()
        
        # Rate limiting
        if "429" in error_str or "rate limit" in error_str:
            retry_after = ErrorClassifier._extract_retry_after(error)
            return ClassifiedError(
                category=ErrorCategory.RATE_LIMIT,
                original_error=error,
                message="Rate limited",
                retry_after=retry_after or 60.0,
                should_retry=True
            )
        
        # Authentication
        if "401" in error_str or "403" in error_str or "unauthorized" in error_str:
            return ClassifiedError(
                category=ErrorCategory.AUTH,
                original_error=error,
                message="Authentication failed",
                should_retry=True  # After credential refresh
            )
        
        # Server errors
        if any(code in error_str for code in ["500", "502", "503", "504"]):
            return ClassifiedError(
                category=ErrorCategory.SERVER,
                original_error=error,
                message="Server error",
                retry_after=5.0,
                should_retry=True
            )
        
        # Network/transient
        if any(term in error_str for term in ["timeout", "connection", "network"]):
            return ClassifiedError(
                category=ErrorCategory.TRANSIENT,
                original_error=error,
                message="Network error",
                retry_after=1.0,
                should_retry=True
            )
        
        # Parse errors
        if any(term in error_str for term in ["json", "parse", "decode", "invalid"]):
            return ClassifiedError(
                category=ErrorCategory.PARSE,
                original_error=error,
                message="Parse error",
                should_retry=True
            )
        
        # Client errors (likely our fault)
        if "400" in error_str or "invalid request" in error_str:
            return ClassifiedError(
                category=ErrorCategory.CLIENT,
                original_error=error,
                message="Invalid request",
                should_retry=False  # Need to fix the request
            )
        
        # Unknown - treat as fatal
        return ClassifiedError(
            category=ErrorCategory.FATAL,
            original_error=error,
            message=f"Unknown error: {error}",
            should_retry=False
        )
    
    @staticmethod
    def _extract_retry_after(error: Exception) -> Optional[float]:
        """Extract retry-after header if available"""
        if hasattr(error, 'response') and hasattr(error.response, 'headers'):
            retry_after = error.response.headers.get('retry-after')
            if retry_after:
                try:
                    return float(retry_after)
                except ValueError:
                    pass
        return None


class ResilientExecutor:
    """Execute operations with intelligent error recovery"""
    
    def __init__(
        self,
        max_retries: int = 5,
        base_delay: float = 1.0,
        max_delay: float = 60.0,
        on_error: Optional[Callable] = None
    ):
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.max_delay = max_delay
        self.on_error = on_error
        self.error_log = []
    
    async def execute(
        self,
        func: Callable,
        *args,
        fallback: Optional[Callable] = None,
        **kwargs
    ) -> Any:
        """Execute function with automatic error recovery"""
        
        last_error = None
        
        for attempt in range(self.max_retries):
            try:
                return await func(*args, **kwargs)
                
            except Exception as e:
                classified = ErrorClassifier.classify(e)
                self.error_log.append({
                    'attempt': attempt + 1,
                    'category': classified.category.value,
                    'message': classified.message,
                    'traceback': traceback.format_exc()
                })
                
                # Notify error handler
                if self.on_error:
                    self.on_error(classified, attempt + 1)
                
                # Don't retry fatal errors
                if not classified.should_retry:
                    last_error = classified.original_error
                    break
                
                # Calculate delay
                if classified.retry_after:
                    delay = classified.retry_after
                else:
                    delay = min(
                        self.base_delay * (2 ** attempt),
                        self.max_delay
                    )
                
                print(f"Attempt {attempt + 1} failed: {classified.message}. "
                      f"Retrying in {delay:.1f}s...")
                
                await asyncio.sleep(delay)
                last_error = classified.original_error
        
        # All retries exhausted - try fallback
        if fallback:
            print("All retries exhausted. Attempting fallback...")
            try:
                return await fallback(*args, **kwargs)
            except Exception as e:
                print(f"Fallback also failed: {e}")
        
        raise last_error or Exception("All retries exhausted")

# Usage
executor = ResilientExecutor(
    max_retries=5,
    on_error=lambda e, attempt: print(f"Error on attempt {attempt}: {e.message}")
)

result = await executor.execute(
    api_client.complete,
    prompt="Hello",
    fallback=backup_client.complete  # Use different provider as fallback
)
```

## Strategy 2: Output Validation and Self-Healing

```python
import json
import re
from typing import Dict, Any, Optional, List
from pydantic import BaseModel, ValidationError

class OutputValidator:
    """Validate and repair LLM outputs"""
    
    @staticmethod
    def extract_json(text: str) -> Optional[Dict]:
        """Extract JSON from potentially malformed response"""
        
        # Try direct parse first
        try:
            return json.loads(text)
        except json.JSONDecodeError:
            pass
        
        # Try to find JSON in markdown code blocks
        patterns = [
            r'```json\s*([\s\S]*?)\s*```',
            r'```\s*([\s\S]*?)\s*```',
            r'\{[\s\S]*\}',
            r'\[[\s\S]*\]'
        ]
        
        for pattern in patterns:
            matches = re.findall(pattern, text)
            for match in matches:
                try:
                    return json.loads(match)
                except json.JSONDecodeError:
                    continue
        
        return None
    
    @staticmethod
    def repair_json(malformed: str) -> Optional[Dict]:
        """Attempt to repair common JSON issues"""
        
        repaired = malformed
        
        # Fix trailing commas
        repaired = re.sub(r',\s*}', '}', repaired)
        repaired = re.sub(r',\s*]', ']', repaired)
        
        # Fix unquoted keys
        repaired = re.sub(r'(\{|,)\s*(\w+)\s*:', r'\1"\2":', repaired)
        
        # Fix single quotes
        repaired = repaired.replace("'", '"')
        
        # Fix missing quotes on string values
        repaired = re.sub(r':\s*([a-zA-Z_][a-zA-Z0-9_]*)\s*([,}])', r': "\1"\2', repaired)
        
        try:
            return json.loads(repaired)
        except json.JSONDecodeError:
            return None
    
    @staticmethod
    def validate_schema(data: Dict, schema: type[BaseModel]) -> tuple[bool, Optional[BaseModel], Optional[str]]:
        """Validate data against Pydantic schema"""
        try:
            validated = schema(**data)
            return True, validated, None
        except ValidationError as e:
            return False, None, str(e)


class SelfHealingAgent:
    """Agent that can detect and recover from output errors"""
    
    def __init__(self, client, model: str):
        self.client = client
        self.model = model
        self.validator = OutputValidator()
    
    async def get_structured_output(
        self,
        prompt: str,
        schema: type[BaseModel],
        max_repair_attempts: int = 3
    ) -> BaseModel:
        """Get validated structured output with self-healing"""
        
        system_prompt = f"""You must respond with valid JSON matching this schema:
{schema.model_json_schema()}

Respond with ONLY the JSON object, no other text."""
        
        for attempt in range(max_repair_attempts):
            response = await self.client.messages.create(
                model=self.model,
                system=system_prompt,
                messages=[{"role": "user", "content": prompt}],
                max_tokens=4096
            )
            
            text = response.content[0].text
            
            # Try to extract JSON
            data = self.validator.extract_json(text)
            
            if data is None:
                # Try repair
                data = self.validator.repair_json(text)
            
            if data is None:
                # Ask model to fix its output
                prompt = f"""Your previous response was not valid JSON:
---
{text}
---

Please respond with ONLY valid JSON matching the schema. No explanation."""
                continue
            
            # Validate against schema
            valid, result, error = self.validator.validate_schema(data, schema)
            
            if valid:
                return result
            
            # Ask model to fix validation errors
            prompt = f"""Your JSON had validation errors:
{error}

Original response:
{text}

Please fix the errors and respond with valid JSON only."""
        
        raise ValueError(f"Could not get valid output after {max_repair_attempts} attempts")

# Usage with Pydantic schema
class TaskAnalysis(BaseModel):
    task_name: str
    complexity: int  # 1-10
    estimated_hours: float
    required_skills: List[str]
    risks: List[str]

agent = SelfHealingAgent(client, "claude-3-sonnet-20240229")
result = await agent.get_structured_output(
    prompt="Analyze this task: Build a REST API for user authentication",
    schema=TaskAnalysis
)
print(result.model_dump())
```

## Strategy 3: Checkpoint and Resume

```python
import json
import os
from datetime import datetime
from typing import Dict, Any, List, Optional
from dataclasses import dataclass, asdict
import hashlib

@dataclass
class Checkpoint:
    workflow_id: str
    step_index: int
    step_name: str
    state: Dict[str, Any]
    timestamp: str
    completed_steps: List[str]

class CheckpointManager:
    """Save and restore workflow state for crash recovery"""
    
    def __init__(self, checkpoint_dir: str = ".checkpoints"):
        self.checkpoint_dir = checkpoint_dir
        os.makedirs(checkpoint_dir, exist_ok=True)
    
    def _get_path(self, workflow_id: str) -> str:
        return os.path.join(self.checkpoint_dir, f"{workflow_id}.json")
    
    def save(self, checkpoint: Checkpoint):
        """Save checkpoint to disk"""
        path = self._get_path(checkpoint.workflow_id)
        with open(path, 'w') as f:
            json.dump(asdict(checkpoint), f, indent=2)
        print(f"Checkpoint saved: {checkpoint.step_name}")
    
    def load(self, workflow_id: str) -> Optional[Checkpoint]:
        """Load checkpoint from disk"""
        path = self._get_path(workflow_id)
        if os.path.exists(path):
            with open(path, 'r') as f:
                data = json.load(f)
                return Checkpoint(**data)
        return None
    
    def clear(self, workflow_id: str):
        """Clear checkpoint after successful completion"""
        path = self._get_path(workflow_id)
        if os.path.exists(path):
            os.remove(path)


class ResumableWorkflow:
    """Workflow that can resume from failures"""
    
    def __init__(self, workflow_id: str = None):
        self.workflow_id = workflow_id or self._generate_id()
        self.checkpoint_manager = CheckpointManager()
        self.steps: List[tuple[str, callable]] = []
        self.state: Dict[str, Any] = {}
    
    def _generate_id(self) -> str:
        return hashlib.md5(str(datetime.now()).encode()).hexdigest()[:8]
    
    def add_step(self, name: str, func: callable):
        """Add a step to the workflow"""
        self.steps.append((name, func))
    
    async def run(self, initial_state: Dict[str, Any] = None) -> Dict[str, Any]:
        """Run workflow with automatic checkpointing"""
        
        # Check for existing checkpoint
        checkpoint = self.checkpoint_manager.load(self.workflow_id)
        
        if checkpoint:
            print(f"Resuming from checkpoint: {checkpoint.step_name}")
            self.state = checkpoint.state
            start_index = checkpoint.step_index + 1
            completed = set(checkpoint.completed_steps)
        else:
            self.state = initial_state or {}
            start_index = 0
            completed = set()
        
        # Execute steps
        for i, (name, func) in enumerate(self.steps[start_index:], start=start_index):
            if name in completed:
                print(f"Skipping already completed: {name}")
                continue
            
            print(f"Executing step {i + 1}/{len(self.steps)}: {name}")
            
            try:
                # Execute step
                result = await func(self.state)
                
                # Update state with result
                if result:
                    self.state.update(result)
                
                # Save checkpoint
                completed.add(name)
                self.checkpoint_manager.save(Checkpoint(
                    workflow_id=self.workflow_id,
                    step_index=i,
                    step_name=name,
                    state=self.state,
                    timestamp=datetime.now().isoformat(),
                    completed_steps=list(completed)
                ))
                
            except Exception as e:
                print(f"Step '{name}' failed: {e}")
                print("Workflow paused. Resume with same workflow_id to continue.")
                raise
        
        # Clear checkpoint on success
        self.checkpoint_manager.clear(self.workflow_id)
        print("Workflow completed successfully!")
        
        return self.state

# Usage
async def step_fetch_data(state):
    # Simulate fetching data
    data = await fetch_from_api()
    return {"raw_data": data}

async def step_process(state):
    # Process the data
    processed = await process_data(state["raw_data"])
    return {"processed_data": processed}

async def step_analyze(state):
    # Analyze (might fail!)
    analysis = await llm_analyze(state["processed_data"])
    return {"analysis": analysis}

async def step_report(state):
    # Generate report
    report = await generate_report(state["analysis"])
    return {"report": report}

# Create workflow
workflow = ResumableWorkflow(workflow_id="analysis-2024-01-15")
workflow.add_step("fetch_data", step_fetch_data)
workflow.add_step("process", step_process)
workflow.add_step("analyze", step_analyze)
workflow.add_step("report", step_report)

# Run (will resume from last checkpoint if crashed)
result = await workflow.run()
```

## Strategy 4: Graceful Degradation

```python
from typing import Dict, Any, Optional, List
from dataclasses import dataclass
from enum import Enum

class ServiceLevel(Enum):
    FULL = "full"           # All features available
    DEGRADED = "degraded"   # Some features unavailable
    MINIMAL = "minimal"     # Basic functionality only
    OFFLINE = "offline"     # No external services

@dataclass
class ServiceStatus:
    llm_available: bool = True
    search_available: bool = True
    database_available: bool = True
    cache_available: bool = True

class GracefulAgent:
    """Agent that degrades gracefully when services fail"""
    
    def __init__(self):
        self.status = ServiceStatus()
        self.fallback_responses = self._load_fallback_responses()
    
    def _load_fallback_responses(self) -> Dict[str, str]:
        """Pre-computed responses for common queries"""
        return {
            "greeting": "Hello! I'm currently operating in offline mode with limited capabilities.",
            "help": "I can help with basic tasks. For complex queries, please try again later.",
            "error": "I encountered an issue. Please try a simpler request or try again later."
        }
    
    def get_service_level(self) -> ServiceLevel:
        """Determine current service level"""
        if all([self.status.llm_available, self.status.search_available, 
                self.status.database_available]):
            return ServiceLevel.FULL
        elif self.status.llm_available:
            return ServiceLevel.DEGRADED
        elif self.status.cache_available:
            return ServiceLevel.MINIMAL
        else:
            return ServiceLevel.OFFLINE
    
    async def process_request(self, request: str) -> str:
        """Process request with graceful degradation"""
        
        level = self.get_service_level()
        
        if level == ServiceLevel.FULL:
            return await self._full_process(request)
        elif level == ServiceLevel.DEGRADED:
            return await self._degraded_process(request)
        elif level == ServiceLevel.MINIMAL:
            return self._minimal_process(request)
        else:
            return self._offline_process(request)
    
    async def _full_process(self, request: str) -> str:
        """Full processing with all features"""
        # Search for context
        context = await self.search(request)
        # LLM processing
        response = await self.llm_complete(request, context)
        # Save to database
        await self.save_interaction(request, response)
        return response
    
    async def _degraded_process(self, request: str) -> str:
        """Processing without search/database"""
        try:
            response = await self.llm_complete(request, context=None)
            return f"{response}\n\n_(Some features are temporarily unavailable)_"
        except:
            return self._minimal_process(request)
    
    def _minimal_process(self, request: str) -> str:
        """Cached/pre-computed responses only"""
        # Try to match against cached responses
        request_lower = request.lower()
        
        if any(word in request_lower for word in ["hello", "hi", "hey"]):
            return self.fallback_responses["greeting"]
        elif any(word in request_lower for word in ["help", "what can"]):
            return self.fallback_responses["help"]
        else:
            return self.fallback_responses["error"]
    
    def _offline_process(self, request: str) -> str:
        """Completely offline response"""
        return ("I'm currently offline and cannot process requests. "
                "Please check your connection and try again.")
    
    async def health_check(self):
        """Update service status"""
        # Check LLM
        try:
            await self.llm_complete("test", timeout=5)
            self.status.llm_available = True
        except:
            self.status.llm_available = False
        
        # Check other services similarly...
```

## Common Pitfalls

### ❌ Catching All Exceptions Blindly
```python
# Don't do this
try:
    result = await api_call()
except Exception:
    pass  # Silent failure!
```

### ❌ Infinite Retry Loops
```python
# This can run forever
while True:
    try:
        result = await api_call()
        break
    except:
        await asyncio.sleep(1)
```

### ❌ Not Logging Errors
```python
# Always log for debugging
try:
    result = await api_call()
except Exception as e:
    logger.error(f"API call failed: {e}", exc_info=True)
    raise
```

### ❌ Retrying Non-Idempotent Operations
```python
# Dangerous! Could send multiple emails
async def send_and_retry():
    for _ in range(3):
        try:
            await send_email()  # Might have succeeded before timeout!
            break
        except TimeoutError:
            continue
```

## Pro Tips

### 1. Implement Idempotency Keys
```python
import uuid

class IdempotentExecutor:
    def __init__(self):
        self.completed_operations = set()
    
    async def execute_once(self, operation_id: str, func, *args, **kwargs):
        """Ensure operation only executes once"""
        if operation_id in self.completed_operations:
            print(f"Operation {operation_id} already completed, skipping")
            return None
        
        result = await func(*args, **kwargs)
        self.completed_operations.add(operation_id)
        return result

# Usage
executor = IdempotentExecutor()
op_id = str(uuid.uuid4())

# Safe to retry - will only execute once
for _ in range(3):
    try:
        await executor.execute_once(op_id, send_email, to="user@example.com")
        break
    except TimeoutError:
        continue
```

### 2. Dead Letter Queue for Failed Operations
```python
import json
from datetime import datetime
from collections import deque

class DeadLetterQueue:
    def __init__(self, max_size: int = 1000):
        self.queue = deque(maxlen=max_size)
    
    def add(self, operation: Dict, error: Exception):
        self.queue.append({
            "operation": operation,
            "error": str(error),
            "traceback": traceback.format_exc(),
            "timestamp": datetime.now().isoformat(),
            "retry_count": operation.get("retry_count", 0) + 1
        })
    
    def get_retriable(self, max_retries: int = 3) -> List[Dict]:
        return [
            item for item in self.queue 
            if item["retry_count"] < max_retries
        ]
    
    def save(self, path: str):
        with open(path, 'w') as f:
            json.dump(list(self.queue), f, indent=2)

dlq = DeadLetterQueue()

# When operation fails after all retries
try:
    await execute_with_retries(operation)
except Exception as e:
    dlq.add(operation, e)
    # Continue with other operations
```

### 3. Error Aggregation for Batch Operations
```python
from dataclasses import dataclass, field
from typing import List, Any

@dataclass
class BatchResult:
    successful: List[Any] = field(default_factory=list)
    failed: List[tuple[Any, Exception]] = field(default_factory=list)
    
    @property
    def success_rate(self) -> float:
        total = len(self.successful) + len(self.failed)
        return len(self.successful) / total if total > 0 else 0
    
    @property
    def all_succeeded(self) -> bool:
        return len(self.failed) == 0

async def batch_process(items: List[Any], process_func) -> BatchResult:
    result = BatchResult()
    
    for item in items:
        try:
            output = await process_func(item)
            result.successful.append((item, output))
        except Exception as e:
            result.failed.append((item, e))
    
    print(f"Batch complete: {result.success_rate:.1%} success rate")
    return result
```

## Quick Checklist

- [ ] Classify errors by type (transient, rate limit, client, etc.)
- [ ] Implement exponential backoff with jitter
- [ ] Set maximum retry limits
- [ ] Log all errors with context
- [ ] Implement fallback mechanisms
- [ ] Validate and repair LLM outputs
- [ ] Checkpoint long-running workflows
- [ ] Use idempotency keys for non-idempotent operations
- [ ] Implement graceful degradation
- [ ] Create dead letter queues for failed operations
- [ ] Test error recovery paths

Remember: **A robust agent isn't one that never fails - it's one that recovers gracefully when it does.** Plan for failure, and your agents will handle whatever the world throws at them.