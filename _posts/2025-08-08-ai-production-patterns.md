---
layout: post
title: "AI in Production: Integration Patterns and Resilience (Part 2/4"
date: 2025-08-08 10:00:00 -0000
categories: [blog]
tags: [ai, production, resilience, circuit-breaker, caching, fastapi, python]
---

This is the second part of our series on AI in production. If you haven't read [Part 1](/blog/2025/08/07/ai-production-fundamentals/), I recommend starting there.

In this installment, we'll focus on the critical patterns that distinguish a prototype from a robust, reliable AI system in production.

---

## Production API Integration Patterns

### The Circuit Breaker Pattern

Prevent cascading failures with intelligent circuit breaking. It's essential when you depend on external services like AI APIs.

```python
from typing import Optional, Callable, Any
import time
import asyncio
from collections import deque
from enum import Enum
import logging

logger = logging.getLogger(__name__)

class CircuitState(Enum):
    CLOSED = "closed"       # Normal operation
    OPEN = "open"          # Circuit is open
    HALF_OPEN = "half_open" # Testing recovery

class AICircuitBreaker:
    """Circuit breaker for AI API calls with adaptive behavior"""
    
    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: int = 60,
        success_threshold: int = 3,
        timeout_seconds: float = 30.0
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.success_threshold = success_threshold
        self.timeout_seconds = timeout_seconds
        
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
        
        # Performance metrics
        self.call_history = deque(maxlen=100)
        self.total_calls = 0
        self.total_failures = 0
        
    async def call(self, func: Callable, *args, **kwargs) -> Any:
        """Execute function with circuit breaker protection"""
        self.total_calls += 1
        
        # Check if we should open the circuit
        if self.state == CircuitState.OPEN:
            if self._should_attempt_reset():
                self.state = CircuitState.HALF_OPEN
                self.success_count = 0
                logger.info("Circuit breaker transitioning to HALF_OPEN")
            else:
                raise CircuitBreakerOpenError("Circuit breaker is OPEN")
        
        try:
            # Execute function with timeout
            result = await asyncio.wait_for(
                func(*args, **kwargs),
                timeout=self.timeout_seconds
            )
            
            # Record success
            self._record_success()
            return result
            
        except Exception as e:
            # Record failure
            self._record_failure()
            raise e
    
    def _should_attempt_reset(self) -> bool:
        """Determine whether it's time to attempt closing the circuit"""
        if self.last_failure_time is None:
            return True
        
        return (time.time() - self.last_failure_time) >= self.recovery_timeout
    
    def _record_success(self):
        """Record a successful call"""
        self.call_history.append({"timestamp": time.time(), "success": True})
        
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.success_threshold:
                self.state = CircuitState.CLOSED
                self.failure_count = 0
                logger.info("Circuit breaker CLOSED - service recovered")
        elif self.state == CircuitState.CLOSED:
            # Reset failure count in normal state
            self.failure_count = max(0, self.failure_count - 1)
    
    def _record_failure(self):
        """Record a failed call"""
        self.total_failures += 1
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        self.call_history.append({"timestamp": time.time(), "success": False})
        
        if self.failure_count >= self.failure_threshold:
            if self.state != CircuitState.OPEN:
                self.state = CircuitState.OPEN
                logger.warning(f"Circuit breaker OPENED after {self.failure_count} failures")
    
    def get_metrics(self) -> dict:
        """Get circuit breaker metrics"""
        recent_calls = [call for call in self.call_history
                       if time.time() - call["timestamp"] < 300]  # Last 5 minutes
        
        if recent_calls:
            success_rate = sum(1 for call in recent_calls if call["success"]) / len(recent_calls)
        else:
            success_rate = 1.0
        
        return {
            "state": self.state.value,
            "total_calls": self.total_calls,
            "total_failures": self.total_failures,
            "failure_rate": self.total_failures / max(1, self.total_calls),
            "recent_success_rate": success_rate,
            "current_failure_count": self.failure_count
        }

class CircuitBreakerOpenError(Exception):
    """Exception when the circuit breaker is open"""
    pass

# Example usage with OpenAI
class ResilientAIClient:
    """AI client with integrated circuit breaker"""
    
    def __init__(self, openai_client):
        self.client = openai_client
        self.circuit_breaker = AICircuitBreaker(
            failure_threshold=3,
            recovery_timeout=30
        )
    
    async def generate_completion(self, messages: list, **kwargs) -> str:
        """Generate completion with circuit breaker protection"""
        try:
            response = await self.circuit_breaker.call(
                self._call_openai,
                messages,
                **kwargs
            )
            return response.choices[0].message.content
        except CircuitBreakerOpenError:
            # Fallback when the circuit breaker is open
            return await self._fallback_response(messages)
    
    async def _call_openai(self, messages: list, **kwargs):
        """Direct call to OpenAI"""
        return await self.client.chat.completions.create(
            model=kwargs.get("model", "gpt-4o-mini"),
            messages=messages,
            **kwargs
        )
    
    async def _fallback_response(self, messages: list) -> str:
        """Fallback response when AI is not available"""
        return "I'm sorry, the AI service is temporarily unavailable. Please try again later."
```

### Request Pooling and Batching

Reduce costs and latency with smart batching:

```python
from asyncio import Queue, gather, create_task
from typing import List, Tuple, Dict, Any
import asyncio
import time
import json

class AIRequestBatcher:
    """Groups multiple requests to reduce API calls and costs"""
    
    def __init__(
        self,
        batch_size: int = 10,
        batch_timeout: float = 0.1,
        max_queue_size: int = 1000
    ):
        self.batch_size = batch_size
        self.batch_timeout = batch_timeout
        self.max_queue_size = max_queue_size
        
        self.request_queue = Queue(maxsize=max_queue_size)
        self.processor_task = None
        self.is_running = False
        
        # Metrics
        self.total_requests = 0
        self.total_batches = 0
        self.batch_efficiency = deque(maxlen=100)
    
    async def start(self):
        """Start the batch processor"""
        if self.is_running:
            return
        
        self.is_running = True
        self.processor_task = create_task(self._process_batches())
    
    async def stop(self):
        """Stop the batch processor"""
        self.is_running = False
        if self.processor_task:
            self.processor_task.cancel()
            try:
                await self.processor_task
            except asyncio.CancelledError:
                pass
    
    async def submit(self, prompt: str, **kwargs) -> str:
        """Submit request for batch processing"""
        if not self.is_running:
            await self.start()
        
        future = asyncio.Future()
        request_item = {
            "prompt": prompt,
            "kwargs": kwargs,
            "future": future,
            "timestamp": time.time()
        }
        
        try:
            await self.request_queue.put(request_item)
            self.total_requests += 1
            return await future
        except asyncio.QueueFull:
            raise Exception("Request queue full. Try again later.")
    
    async def _process_batches(self):
        """Main batch processor"""
        while self.is_running:
            batch = []
            batch_start = time.time()
            
            try:
                # Collect requests for the batch
                while (len(batch) < self.batch_size and
                       (time.time() - batch_start) < self.batch_timeout):
                    
                    try:
                        item = await asyncio.wait_for(
                            self.request_queue.get(),
                            timeout=self.batch_timeout / 4
                        )
                        batch.append(item)
                    except asyncio.TimeoutError:
                        break
                
                if batch:
                    await self._execute_batch(batch)
                    
            except Exception as e:
                logger.error(f"Error processing batch: {e}")
                # Fail all futures in the batch
                for item in batch:
                    if not item["future"].done():
                        item["future"].set_exception(e)
    
    async def _execute_batch(self, batch: List[Dict]):
        """Execute a batch of requests"""
        self.total_batches += 1
        batch_size = len(batch)
        
        try:
            # Group similar prompts for optimization
            grouped_requests = self._group_similar_requests(batch)
            
            # Execute each group
            for group in grouped_requests:
                await self._execute_request_group(group)
                
            # Record batch efficiency
            self.batch_efficiency.append(batch_size)
            
        except Exception as e:
            logger.error(f"Error executing batch: {e}")
            # Fail all futures
            for item in batch:
                if not item["future"].done():
                    item["future"].set_exception(e)
    
    def _group_similar_requests(self, batch: List[Dict]) -> List[List[Dict]]:
        """Group similar requests for optimization"""
        # Group by similar parameters (model, temperature, etc.)
        groups = {}
        
        for item in batch:
            # Create grouping key based on parameters
            group_key = (
                item["kwargs"].get("model", "default"),
                item["kwargs"].get("temperature", 0.3),
                item["kwargs"].get("max_tokens", 100)
            )
            
            if group_key not in groups:
                groups[group_key] = []
            groups[group_key].append(item)
        
        return list(groups.values())
    
    async def _execute_request_group(self, group: List[Dict]):
        """Execute a group of requests with similar parameters"""
        if len(group) == 1:
            # Individual request
            item = group[0]
            try:
                result = await self._single_ai_call(item["prompt"], **item["kwargs"])
                item["future"].set_result(result)
            except Exception as e:
                item["future"].set_exception(e)
        else:
            # Batch of multiple requests
            await self._batch_ai_call(group)
    
    async def _single_ai_call(self, prompt: str, **kwargs) -> str:
        """Individual AI call"""
        # Implement the call to your AI provider
        # For example, with OpenAI:
        client = kwargs.pop("client", None)
        if not client:
            raise ValueError("AI client required")
        
        response = await client.chat.completions.create(
            model=kwargs.get("model", "gpt-4o-mini"),
            messages=[{"role": "user", "content": prompt}],
            max_tokens=kwargs.get("max_tokens", 100),
            temperature=kwargs.get("temperature", 0.3)
        )
        
        return response.choices[0].message.content
    
    async def _batch_ai_call(self, group: List[Dict]):
        """Execute multiple requests as an optimized batch"""
        # Implement provider-specific batching logic
        # Some providers support multiple requests in a single call
        
        # For OpenAI, we can use multiple concurrent calls
        tasks = []
        for item in group:
            task = create_task(self._single_ai_call(item["prompt"], **item["kwargs"]))
            tasks.append((task, item))
        
        # Execute all tasks concurrently
        for task, item in tasks:
            try:
                result = await task
                item["future"].set_result(result)
            except Exception as e:
                item["future"].set_exception(e)
    
    def get_metrics(self) -> Dict[str, Any]:
        """Get batcher metrics"""
        avg_batch_size = sum(self.batch_efficiency) / len(self.batch_efficiency) if self.batch_efficiency else 0
        
        return {
            "total_requests": self.total_requests,
            "total_batches": self.total_batches,
            "avg_batch_size": avg_batch_size,
            "queue_size": self.request_queue.qsize(),
            "is_running": self.is_running
        }

# Batcher usage example
async def example_batcher_usage():
    batcher = AIRequestBatcher(batch_size=5, batch_timeout=0.2)
    await batcher.start()
    
    # Send multiple requests
    tasks = []
    for i in range(20):
        task = batcher.submit(
            f"Generate a summary of topic #{i}",
            client=openai.AsyncOpenAI(),
            max_tokens=50
        )
        tasks.append(task)
    
    # Wait for results
    results = await gather(*tasks)
    
    print(f"Processed {len(results)} requests")
    print(f"Metrics: {batcher.get_metrics()}")
    
    await batcher.stop()
```

---

## Resilience and High Availability

### Multi-Region Failover

```python
from typing import List, Optional, Dict
import aiohttp
import random
import time

class MultiRegionAIClient:
    """Geo-distributed AI client with automatic failover"""
    
    def __init__(self, endpoints: List[Dict[str, Any]]):
        self.endpoints = endpoints
        self.endpoint_health = {ep["name"]: {"healthy": True, "last_check": 0}
                               for ep in endpoints}
        self.current_endpoint_index = 0
        self.health_check_interval = 30  # seconds
    
    async def call_with_failover(self, prompt: str, **kwargs) -> str:
        """Make a call with automatic failover"""
        attempts = 0
        last_exception = None
        
        while attempts < len(self.endpoints):
            endpoint = self._get_next_healthy_endpoint()
            
            if not endpoint:
                await self._check_all_endpoints_health()
                endpoint = self._get_next_healthy_endpoint()
                
                if not endpoint:
                    raise Exception("All endpoints are unavailable")
            
            try:
                result = await self._call_endpoint(endpoint, prompt, **kwargs)
                
                # Mark endpoint as healthy
                self.endpoint_health[endpoint["name"]]["healthy"] = True
                self.endpoint_health[endpoint["name"]]["last_check"] = time.time()
                
                return result
                
            except Exception as e:
                last_exception = e
                attempts += 1
                
                # Mark endpoint as unhealthy
                self.endpoint_health[endpoint["name"]]["healthy"] = False
                self.endpoint_health[endpoint["name"]]["last_check"] = time.time()
                
                logger.warning(f"Endpoint {endpoint['name']} failed: {e}")
        
        raise Exception(f"All endpoints failed. Last error: {last_exception}")
    
    def _get_next_healthy_endpoint(self) -> Optional[Dict]:
        """Get the next healthy endpoint"""
        healthy_endpoints = [
            ep for ep in self.endpoints
            if self.endpoint_health[ep["name"]]["healthy"]
        ]
        
        if not healthy_endpoints:
            return None
        
        # Simple load balancing - round robin
        endpoint = healthy_endpoints[self.current_endpoint_index % len(healthy_endpoints)]
        self.current_endpoint_index += 1
        
        return endpoint
    
    async def _call_endpoint(self, endpoint: Dict, prompt: str, **kwargs) -> str:
        """Make a call to a specific endpoint"""
        if endpoint["type"] == "openai":
            return await self._call_openai(endpoint, prompt, **kwargs)
        elif endpoint["type"] == "anthropic":
            return await self._call_anthropic(endpoint, prompt, **kwargs)
        elif endpoint["type"] == "custom":
            return await self._call_custom(endpoint, prompt, **kwargs)
        else:
            raise ValueError(f"Unsupported endpoint type: {endpoint['type']}")
    
    async def _call_openai(self, endpoint: Dict, prompt: str, **kwargs) -> str:
        """OpenAI-specific call"""
        import openai
        
        client = openai.AsyncOpenAI(
            api_key=endpoint["api_key"],
            base_url=endpoint.get("base_url")
        )
        
        response = await client.chat.completions.create(
            model=endpoint.get("model", "gpt-4o-mini"),
            messages=[{"role": "user", "content": prompt}],
            **kwargs
        )
        
        return response.choices[0].message.content
    
    async def _call_anthropic(self, endpoint: Dict, prompt: str, **kwargs) -> str:
        """Anthropic-specific call"""
        # Implement Anthropics call
        pass
    
    async def _call_custom(self, endpoint: Dict, prompt: str, **kwargs) -> str:
        """Call to a custom endpoint"""
        async with aiohttp.ClientSession() as session:
            payload = {
                "prompt": prompt,
                **kwargs
            }
            
            async with session.post(
                endpoint["url"],
                json=payload,
                headers=endpoint.get("headers", {}),
                timeout=aiohttp.ClientTimeout(total=30)
            ) as response:
                if response.status != 200:
                    raise Exception(f"HTTP {response.status}: {await response.text()}")
                
                data = await response.json()
                return data.get("content", "")
    
    async def _check_all_endpoints_health(self):
        """Check the health of all endpoints"""
        for endpoint in self.endpoints:
            if (time.time() - self.endpoint_health[endpoint["name"]]["last_check"]
                > self.health_check_interval):
                
                try:
                    # Simple health check
                    await self._call_endpoint(
                        endpoint,
                        "Health check",
                        max_tokens=1
                    )
                    self.endpoint_health[endpoint["name"]]["healthy"] = True
                except Exception:
                    self.endpoint_health[endpoint["name"]]["healthy"] = False
                
                self.endpoint_health[endpoint["name"]]["last_check"] = time.time()

# Example configuration
endpoints_config = [
    {
        "name": "openai-us-east",
        "type": "openai",
        "api_key": "sk-...",
        "model": "gpt-4o-mini",
        "region": "us-east"
    },
    {
        "name": "openai-eu-west",
        "type": "openai",
        "api_key": "sk-...",
        "model": "gpt-4o-mini",
        "region": "eu-west"
    },
    {
        "name": "custom-api",
        "type": "custom",
        "url": "https://api.custom.com/v1/chat",
        "headers": {"Authorization": "Bearer token"},
        "region": "us-west"
    }
]

# Usage
multi_region_client = MultiRegionAIClient(endpoints_config)
```

### Graceful Degradation

```python
class DegradationStrategy:
    """Fallback strategies when AI is unavailable"""
    
    def __init__(self):
        self.fallback_responses = {
            "summary": "Summary temporarily unavailable",
            "translation": "Translation temporarily unavailable",
            "analysis": "Analysis temporarily unavailable"
        }
        
        self.cached_responses = {}  # Cache of previous responses
        
    async def get_fallback_response(
        self,
        request_type: str,
        context: Dict[str, Any]
    ) -> str:
        """Get a fallback response based on request type"""
        
        # 1. Try a response from cache of similar requests
        cached = await self._get_similar_cached_response(request_type, context)
        if cached:
            return f"[CACHED] {cached}"
        
        # 2. Rule-based response
        rule_based = await self._get_rule_based_response(request_type, context)
        if rule_based:
            return f"[RULE-BASED] {rule_based}"
        
        # 3. Generic fallback response
        return self.fallback_responses.get(
            request_type,
            "Service temporarily unavailable"
        )
    
    async def _get_similar_cached_response(
        self,
        request_type: str,
        context: Dict[str, Any]
    ) -> Optional[str]:
        """Search for a response in cache for similar requests"""
        # Implement semantic similarity search
        # For example, using embeddings to find similar contexts
        pass
    
    async def _get_rule_based_response(
        self,
        request_type: str,
        context: Dict[str, Any]
    ) -> Optional[str]:
        """Generate response based on predefined rules"""
        if request_type == "summary":
            text = context.get("text", "")
            if text:
                # Simple extractive summary
                sentences = text.split(". ")
                if len(sentences) > 3:
                    return ". ".join(sentences[:3]) + "..."
        
        elif request_type == "sentiment":
            text = context.get("text", "").lower()
            positive_words = ["excellent", "good", "positive", "great"]
            negative_words = ["bad", "terrible", "negative", "horrible"]
            
            pos_count = sum(1 for word in positive_words if word in text)
            neg_count = sum(1 for word in negative_words if word in text)
            
            if pos_count > neg_count:
                return "Sentiment: Positive"
            elif neg_count > pos_count:
                return "Sentiment: Negative"
            else:
                return "Sentiment: Neutral"
        
        return None
```

---

## Comprehensive Error Handling

```python
from enum import Enum
from typing import Dict, Any, Optional, Callable
import logging
from datetime import datetime, timedelta
import json

class ErrorType(Enum):
    RATE_LIMIT = "rate_limit"
    AUTHENTICATION = "authentication"
    QUOTA_EXCEEDED = "quota_exceeded"
    MODEL_OVERLOADED = "model_overloaded"
    NETWORK_ERROR = "network_error"
    INVALID_REQUEST = "invalid_request"
    INTERNAL_ERROR = "internal_error"

class AIErrorHandler:
    """Comprehensive error handling with recovery strategies"""
    
    def __init__(self):
        self.error_counts = {}
        self.retry_strategies = {
            ErrorType.RATE_LIMIT: self._handle_rate_limit,
            ErrorType.MODEL_OVERLOADED: self._handle_overload,
            ErrorType.NETWORK_ERROR: self._handle_network_error,
            ErrorType.QUOTA_EXCEEDED: self._handle_quota_exceeded,
        }
        
        # Retry configuration by error type
        self.retry_config = {
            ErrorType.RATE_LIMIT: {"max_retries": 5, "base_delay": 1.0, "max_delay": 60.0},
            ErrorType.NETWORK_ERROR: {"max_retries": 3, "base_delay": 0.5, "max_delay": 10.0},
            ErrorType.MODEL_OVERLOADED: {"max_retries": 4, "base_delay": 2.0, "max_delay": 30.0},
        }
    
    async def handle_ai_call(
        self,
        func: Callable,
        *args,
        **kwargs
    ) -> Any:
        """Execute AI call with robust error handling"""
        last_error = None
        
        for attempt in range(self._get_max_retries()):
            try:
                result = await func(*args, **kwargs)
                
                # Reset error count on success
                func_name = func.__name__
                if func_name in self.error_counts:
                    self.error_counts[func_name] = 0
                
                return result
                
            except Exception as e:
                last_error = e
                error_type = self._classify_error(e)
                
                # Increment error counter
                func_name = func.__name__
                self.error_counts[func_name] = self.error_counts.get(func_name, 0) + 1
                
                # Structured error log
                logger.error(
                    "AI API call failed",
                    extra={
                        "function": func_name,
                        "attempt": attempt + 1,
                        "error_type": error_type.value,
                        "error_message": str(e),
                        "args_hash": hash(str(args))
                    }
                )
                
                # Apply recovery strategy
                if error_type in self.retry_strategies:
                    should_retry, delay = await self.retry_strategies[error_type](
                        e, attempt, func_name
                    )
                    
                    if should_retry and attempt < self._get_max_retries() - 1:
                        await asyncio.sleep(delay)
                        continue
                
                # If we get here, it can't be recovered
                break
        
        # All strategies failed
        raise AIServiceError(
            f"AI service failed after {self._get_max_retries()} attempts",
            error_type=self._classify_error(last_error),
            original_error=last_error
        )
    
    def _classify_error(self, error: Exception) -> ErrorType:
        """Classify error type based on the exception"""
        error_str = str(error).lower()
        
        if "rate limit" in error_str or "429" in error_str:
            return ErrorType.RATE_LIMIT
        elif "authentication" in error_str or "401" in error_str:
            return ErrorType.AUTHENTICATION
        elif "quota" in error_str or "billing" in error_str:
            return ErrorType.QUOTA_EXCEEDED
        elif "overloaded" in error_str or "503" in error_str:
            return ErrorType.MODEL_OVERLOADED
        elif "network" in error_str or "connection" in error_str:
            return ErrorType.NETWORK_ERROR
        elif "400" in error_str or "invalid" in error_str:
            return ErrorType.INVALID_REQUEST
        else:
            return ErrorType.INTERNAL_ERROR
    
    async def _handle_rate_limit(
        self,
        error: Exception,
        attempt: int,
        func_name: str
    ) -> tuple[bool, float]:
        """Handle rate limit errors"""
        config = self.retry_config[ErrorType.RATE_LIMIT]
        
        # Exponential backoff with jitter
        delay = min(
            config["base_delay"] * (2 ** attempt) + random.uniform(0, 1),
            config["max_delay"]
        )
        
        return attempt < config["max_retries"], delay
    
    async def _handle_overload(
        self,
        error: Exception,
        attempt: int,
        func_name: str
    ) -> tuple[bool, float]:
        """Handle model overload errors"""
        config = self.retry_config[ErrorType.MODEL_OVERLOADED]
        
        # Longer delay for overload
        delay = min(
            config["base_delay"] * (1.5 ** attempt),
            config["max_delay"]
        )
        
        return attempt < config["max_retries"], delay
    
    async def _handle_network_error(
        self,
        error: Exception,
        attempt: int,
        func_name: str
    ) -> tuple[bool, float]:
        """Handle network errors"""
        config = self.retry_config[ErrorType.NETWORK_ERROR]
        
        # Fast retry for network errors
        delay = config["base_delay"] * attempt
        
        return attempt < config["max_retries"], delay
    
    async def _handle_quota_exceeded(
        self,
        error: Exception,
        attempt: int,
        func_name: str
    ) -> tuple[bool, float]:
        """Handle quota exceeded errors"""
        # Do not retry - requires manual intervention
        logger.critical(f"API quota exceeded for {func_name}")
        return False, 0
    
    def _get_max_retries(self) -> int:
        """Get maximum number of retries"""
        return 5  # Default
    
    def get_error_stats(self) -> Dict[str, Any]:
        """Get error statistics"""
        return {
            "error_counts": self.error_counts.copy(),
            "total_errors": sum(self.error_counts.values()),
            "functions_with_errors": len(self.error_counts)
        }

class AIServiceError(Exception):
    """Custom exception for AI service errors"""
    
    def __init__(self, message: str, error_type: ErrorType, original_error: Exception):
        super().__init__(message)
        self.error_type = error_type
        self.original_error = original_error
        self.timestamp = datetime.now()

# Example usage of the error handler
error_handler = AIErrorHandler()

async def robust_ai_call(prompt: str, **kwargs):
    """AI call with robust error handling"""
    
    async def _make_ai_call():
        client = openai.AsyncOpenAI()
        response = await client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            **kwargs
        )
        return response.choices[0].message.content
    
    return await error_handler.handle_ai_call(_make_ai_call)
```

---

## Next in the Series

In **Part 3**, we'll cover:

- **Cost Optimization**: Strategies that can reduce expenses by 40–60%
- **Performance Optimization**: Techniques for sub-second response times
- **Intelligent Caching**: Multi-level cache systems with semantic similarity

### Resources to Dive Deeper

- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html) - Martin Fowler
- [Resilience Patterns](https://docs.microsoft.com/en-us/azure/architecture/patterns/category/resiliency) - Microsoft Azure Architecture
- [The Twelve-Factor App](https://12factor.net/) - Methodology for resilient apps

---

**Did you find this content useful?** [Part 3](/blog/2025/08/09/ai-production-optimization/) will be available soon, where we'll focus on cost and performance optimization.

> **Key:** Resilience is not optional in production. Systems fail, APIs have limits, and networks are unpredictable. Design assuming everything will fail at some point.
