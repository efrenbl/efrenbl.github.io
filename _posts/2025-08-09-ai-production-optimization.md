---
layout: post
title: "AI in Production: Cost and Performance Optimization (Part 3/4)"
date: 2025-08-09 10:00:00 -0000
categories: [blog]
tags: [ai, production, optimization, performance, caching, cost-management, python]
---

This is the third part of our series on AI in production. If you haven't read the previous parts, you can find them here: [Part 1](/blog/2025/08/07/ai-production-fundamentals/) | [Part 2](/blog/2025/08/08/ai-production-patterns/)

In this installment, we focus on two critical aspects for AI success in production: **cost optimization** and **performance**. These optimizations can mean the difference between a viable project and one that consumes budget without control.

---

## Cost Optimization Strategies

### Dynamic Model Selection

The key to controlling costs is using the right model for each task. Not all queries require the most powerful model.

```python
from dataclasses import dataclass
from typing import List, Dict, Any, Optional
import tiktoken
import re

@dataclass
class ModelConfig:
    """Model configuration with cost metrics"""
    name: str
    cost_per_token_input: float
    cost_per_token_output: float
    max_tokens: int
    performance_tier: int  # 1=economy, 2=standard, 3=premium
    
    def calculate_cost(self, input_tokens: int, output_tokens: int) -> float:
        """Calculate estimated cost"""
        return (
            input_tokens * self.cost_per_token_input + 
            output_tokens * self.cost_per_token_output
        )

class CostOptimizer:
    """Intelligent model selection based on complexity and cost"""
    
    def __init__(self):
        self.models = {
            "economy": ModelConfig(
                name="gpt-3.5-turbo-2025-05-13",
                cost_per_token_input=0.0005 / 1000,
                cost_per_token_output=0.0015 / 1000,
                max_tokens=4096,
                performance_tier=1
            ),
            "standard": ModelConfig(
                name="gpt-4o-mini-2025-05-13", 
                cost_per_token_input=0.00015 / 1000,
                cost_per_token_output=0.0006 / 1000,
                max_tokens=8192,
                performance_tier=2
            ),
            "premium": ModelConfig(
                name="gpt-4o-2025-05-13",
                cost_per_token_input=0.0025 / 1000,
                cost_per_token_output=0.01 / 1000,
                max_tokens=8192,
                performance_tier=3
            )
        }
        
        # Patterns for complexity classification
        self.complexity_patterns = {
            "high": [
                r"analyze in depth",
                r"detailed explanation",
                r"complex reasoning",
                r"multiple perspectives",
                r"critical analysis"
            ],
            "medium": [
                r"explain briefly",
                r"summarize in", 
                r"compare.*with",
                r"pros and cons",
                r"advantages.*disadvantages"
            ],
            "low": [
                r"translate.*to",
                r"correct.*grammar",
                r"format.*as",
                r"extract.*information",
                r"convert.*to"
            ]
        }
    
    def select_optimal_model(
        self, 
        prompt: str, 
        user_context: Optional[Dict] = None,
        budget_constraint: Optional[float] = None
    ) -> str:
        """Select optimal model based on complexity and constraints"""
        
        # 1. Analyze prompt complexity
        complexity = self._analyze_prompt_complexity(prompt)
        
        # 2. Estimate tokens
        estimated_input_tokens = self._estimate_tokens(prompt)
        estimated_output_tokens = self._estimate_output_tokens(prompt, complexity)
        
        # 3. Consider user context
        user_tier = user_context.get("tier", "standard") if user_context else "standard"
        
        # 4. Select model based on criteria
        selected_model = self._select_model_by_criteria(
            complexity,
            estimated_input_tokens,
            estimated_output_tokens,
            user_tier,
            budget_constraint
        )
        
        return selected_model
    
    def _analyze_prompt_complexity(self, prompt: str) -> str:
        """Analyze prompt complexity using patterns"""
        prompt_lower = prompt.lower()
        
        # Check high complexity patterns
        for pattern in self.complexity_patterns["high"]:
            if re.search(pattern, prompt_lower):
                return "high"
        
        # Check medium complexity patterns
        for pattern in self.complexity_patterns["medium"]:
            if re.search(pattern, prompt_lower):
                return "medium"
        
        # Check low complexity patterns
        for pattern in self.complexity_patterns["low"]:
            if re.search(pattern, prompt_lower):
                return "low"
        
        # Additional heuristics
        if len(prompt) > 1000:
            return "high"
        elif len(prompt) > 300:
            return "medium"
        else:
            return "low"
    
    def _estimate_tokens(self, text: str) -> int:
        """Estimate number of tokens using tiktoken"""
        try:
            encoding = tiktoken.encoding_for_model("gpt-4")
            return len(encoding.encode(text))
        except Exception:
            # Fallback estimation: ~4 characters per token
            return len(text) // 4
    
    def _estimate_output_tokens(self, prompt: str, complexity: str) -> int:
        """Estimate output tokens based on prompt and complexity"""
        base_estimates = {
            "low": 50,
            "medium": 200,
            "premium": 500
        }
        
        base = base_estimates.get(complexity, 200)
        
        # Adjust based on indicators in the prompt
        if "detailed" in prompt.lower() or "complete explanation" in prompt.lower():
            base *= 2
        elif "brief" in prompt.lower() or "summarize" in prompt.lower():
            base = int(base * 0.5)
        
        return base
    
    def _select_model_by_criteria(
        self,
        complexity: str,
        input_tokens: int,
        output_tokens: int,
        user_tier: str,
        budget_constraint: Optional[float]
    ) -> str:
        """Select model based on multiple criteria"""
        
        # Complexity to preferred model mapping
        complexity_to_model = {
            "low": "economy",
            "medium": "standard", 
            "high": "premium"
        }
        
        preferred_model = complexity_to_model[complexity]
        
        # Check user restrictions
        if user_tier == "basic" and preferred_model == "premium":
            preferred_model = "standard"
        elif user_tier == "economy":
            preferred_model = "economy"
        
        # Check budget constraint
        if budget_constraint:
            model_config = self.models[preferred_model]
            estimated_cost = model_config.calculate_cost(input_tokens, output_tokens)
            
            if estimated_cost > budget_constraint:
                # Downgrade to more economical model
                if preferred_model == "premium":
                    preferred_model = "standard"
                elif preferred_model == "standard":
                    preferred_model = "economy"
        
        return self.models[preferred_model].name
    
    def estimate_cost(
        self, 
        model_tier: str, 
        input_tokens: int, 
        output_tokens: int
    ) -> float:
        """Estimate cost for a specific configuration"""
        if model_tier not in self.models:
            raise ValueError(f"Unknown model tier: {model_tier}")
        
        return self.models[model_tier].calculate_cost(input_tokens, output_tokens)

class CostTracker:
    """AI API cost tracker and alerts"""
    
    def __init__(self, monthly_budget: float = 1000.0):
        self.monthly_budget = monthly_budget
        self.daily_budget = monthly_budget / 30
        
        # Metrics storage (use database in production)
        self.daily_costs = {}
        self.monthly_costs = {}
        self.user_costs = {}
        self.endpoint_costs = {}
        
        # Alert configuration
        self.alert_thresholds = {
            "daily": 0.8,    # 80% of daily budget
            "monthly": 0.9,  # 90% of monthly budget
            "user": 50.0     # $50 per user per day
        }
    
    def track_api_call(
        self,
        user_id: str,
        endpoint: str,
        model: str,
        input_tokens: int,
        output_tokens: int,
        cost: float,
        timestamp: Optional[datetime] = None
    ):
        """Register API call for cost tracking"""
        if not timestamp:
            timestamp = datetime.now()
        
        date_key = timestamp.strftime("%Y-%m-%d")
        month_key = timestamp.strftime("%Y-%m")
        
        # Update daily costs
        if date_key not in self.daily_costs:
            self.daily_costs[date_key] = 0
        self.daily_costs[date_key] += cost
        
        # Update monthly costs
        if month_key not in self.monthly_costs:
            self.monthly_costs[month_key] = 0
        self.monthly_costs[month_key] += cost
        
        # Update costs per user
        user_key = f"{user_id}:{date_key}"
        if user_key not in self.user_costs:
            self.user_costs[user_key] = 0
        self.user_costs[user_key] += cost
        
        # Update costs per endpoint
        endpoint_key = f"{endpoint}:{date_key}"
        if endpoint_key not in self.endpoint_costs:
            self.endpoint_costs[endpoint_key] = 0
        self.endpoint_costs[endpoint_key] += cost
        
        # Check alerts
        self._check_budget_alerts(date_key, month_key, user_id, cost)
    
    def _check_budget_alerts(
        self, 
        date_key: str, 
        month_key: str, 
        user_id: str, 
        cost: float
    ):
        """Check and send budget alerts"""
        
        # Daily budget alert
        daily_spent = self.daily_costs[date_key]
        if daily_spent >= self.daily_budget * self.alert_thresholds["daily"]:
            self._send_alert(
                "daily_budget",
                f"Daily budget: {daily_spent:.2f}/${self.daily_budget:.2f}"
            )
        
        # Monthly budget alert
        monthly_spent = self.monthly_costs[month_key]
        if monthly_spent >= self.monthly_budget * self.alert_thresholds["monthly"]:
            self._send_alert(
                "monthly_budget",
                f"Monthly budget: {monthly_spent:.2f}/${self.monthly_budget:.2f}"
            )
        
        # User alert
        user_key = f"{user_id}:{date_key}"
        user_daily_spent = self.user_costs[user_key]
        if user_daily_spent >= self.alert_thresholds["user"]:
            self._send_alert(
                "user_budget",
                f"User {user_id}: ${user_daily_spent:.2f} spent today"
            )
    
    def _send_alert(self, alert_type: str, message: str):
        """Send alert (implement according to notification system)"""
        logger.warning(f"COST ALERT [{alert_type}]: {message}")
        
        # In production, integrate with:
        # - Slack/Teams notifications
        # - Email alerts  
        # - PagerDuty for critical alerts
        # - Dashboard updates
    
    def get_cost_summary(self, period: str = "daily") -> Dict[str, Any]:
        """Get cost summary"""
        if period == "daily":
            today = datetime.now().strftime("%Y-%m-%d")
            return {
                "period": "daily",
                "date": today,
                "total_cost": self.daily_costs.get(today, 0),
                "budget": self.daily_budget,
                "budget_used_pct": (self.daily_costs.get(today, 0) / self.daily_budget) * 100
            }
        elif period == "monthly":
            this_month = datetime.now().strftime("%Y-%m")
            return {
                "period": "monthly",
                "month": this_month,
                "total_cost": self.monthly_costs.get(this_month, 0),
                "budget": self.monthly_budget,
                "budget_used_pct": (self.monthly_costs.get(this_month, 0) / self.monthly_budget) * 100
            }
```

---

## Intelligent Caching Strategy

A well-designed cache system can reduce AI costs by 60-80% while improving latency.

```python
import redis
import hashlib
from typing import Optional, Any, List, Dict
import pickle
import json
import time
from datetime import datetime, timedelta
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity

class IntelligentCache:
    """Multi-level cache with semantic similarity matching"""
    
    def __init__(
        self,
        redis_client: redis.Redis,
        similarity_threshold: float = 0.85,
        max_cache_size: int = 10000
    ):
        self.redis = redis_client
        self.similarity_threshold = similarity_threshold
        self.max_cache_size = max_cache_size
        
        # TTL configuration by content type
        self.ttl_config = {
            "static": 86400 * 7,    # 7 days for static content
            "dynamic": 3600,        # 1 hour for dynamic content
            "user_specific": 1800,  # 30 minutes for user-specific content
            "realtime": 300         # 5 minutes for real-time content
        }
        
        # Vectorizer for semantic similarity
        self.vectorizer = TfidfVectorizer(
            max_features=1000,
            stop_words='english',
            lowercase=True
        )
        
        # Local cache for similarity
        self.similarity_cache = {}
        self.prompt_vectors = {}
        
    def _generate_cache_key(self, prompt: str, **kwargs) -> str:
        """Generate deterministic cache key"""
        # Include relevant parameters in the key
        key_data = {
            "prompt": prompt.strip().lower(),
            "model": kwargs.get("model", ""),
            "temperature": kwargs.get("temperature", 0.3),
            "max_tokens": kwargs.get("max_tokens", 100)
        }
        
        key_string = json.dumps(key_data, sort_keys=True)
        return hashlib.sha256(key_string.encode()).hexdigest()
    
    async def get(self, prompt: str, **kwargs) -> Optional[Dict[str, Any]]:
        """Search cache with exact and semantic matching"""
        
        # 1. Exact search
        exact_key = self._generate_cache_key(prompt, **kwargs)
        cached_result = self.redis.get(f"exact:{exact_key}")
        
        if cached_result:
            try:
                data = json.loads(cached_result)
                data["cache_type"] = "exact"
                data["cache_hit"] = True
                return data
            except json.JSONDecodeError:
                pass
        
        # 2. Semantic similarity search
        similar_result = await self._find_similar_cached_result(prompt, **kwargs)
        if similar_result:
            similar_result["cache_type"] = "semantic"
            similar_result["cache_hit"] = True
            return similar_result
        
        return None
    
    async def set(
        self, 
        prompt: str, 
        result: str, 
        content_type: str = "dynamic",
        metadata: Optional[Dict] = None,
        **kwargs
    ):
        """Store result in cache with metadata"""
        cache_key = self._generate_cache_key(prompt, **kwargs)
        
        cache_data = {
            "result": result,
            "prompt": prompt,
            "timestamp": time.time(),
            "content_type": content_type,
            "metadata": metadata or {},
            "model": kwargs.get("model", ""),
            "parameters": {
                "temperature": kwargs.get("temperature", 0.3),
                "max_tokens": kwargs.get("max_tokens", 100)
            }
        }
        
        # Store with appropriate TTL
        ttl = self.ttl_config.get(content_type, self.ttl_config["dynamic"])
        
        try:
            self.redis.setex(
                f"exact:{cache_key}",
                ttl,
                json.dumps(cache_data)
            )
            
            # Update similarity index
            await self._update_similarity_index(prompt, cache_key, result)
            
        except Exception as e:
            logger.error(f"Error storing in cache: {e}")
    
    async def _find_similar_cached_result(
        self, 
        prompt: str, 
        **kwargs
    ) -> Optional[Dict[str, Any]]:
        """Find similar result using semantic analysis"""
        
        try:
            # Get similar prompt keys
            similar_keys = await self._find_similar_prompts(prompt)
            
            for similar_key, similarity_score in similar_keys:
                if similarity_score >= self.similarity_threshold:
                    cached_result = self.redis.get(f"exact:{similar_key}")
                    
                    if cached_result:
                        try:
                            data = json.loads(cached_result)
                            data["similarity_score"] = similarity_score
                            return data
                        except json.JSONDecodeError:
                            continue
            
            return None
            
        except Exception as e:
            logger.error(f"Error in semantic search: {e}")
            return None
    
    async def _find_similar_prompts(self, prompt: str) -> List[tuple]:
        """Find similar prompts using TF-IDF and cosine similarity"""
        
        # Get all prompts in cache (in production, use a more efficient index)
        pattern = "exact:*"
        cached_keys = self.redis.keys(pattern)
        
        if not cached_keys:
            return []
        
        # Extract prompts from cached keys
        cached_prompts = []
        valid_keys = []
        
        for key in cached_keys[:100]:  # Limit for performance
            try:
                cached_data = self.redis.get(key)
                if cached_data:
                    data = json.loads(cached_data)
                    cached_prompts.append(data["prompt"])
                    valid_keys.append(key.decode().replace("exact:", ""))
            except Exception:
                continue
        
        if not cached_prompts:
            return []
        
        # Calculate similarity using TF-IDF
        try:
            all_prompts = cached_prompts + [prompt]
            tfidf_matrix = self.vectorizer.fit_transform(all_prompts)
            
            # Calculate cosine similarity
            query_vector = tfidf_matrix[-1]  # The last one is our prompt
            similarities = cosine_similarity(query_vector, tfidf_matrix[:-1]).flatten()
            
            # Sort by similarity and return top results
            similar_indices = similarities.argsort()[-10:][::-1]  # Top 10
            
            results = []
            for idx in similar_indices:
                if similarities[idx] > 0.1:  # Minimum threshold
                    results.append((valid_keys[idx], similarities[idx]))
            
            return results
            
        except Exception as e:
            logger.error(f"Error calculating similarity: {e}")
            return []
    
    async def _update_similarity_index(self, prompt: str, cache_key: str, result: str):
        """Update similarity index (simplified implementation)"""
        # In production, use a more sophisticated vector index like:
        # - Elasticsearch with dense vector fields
        # - Pinecone
        # - Weaviate
        # - FAISS
        pass
    
    def invalidate_pattern(self, pattern: str):
        """Invalidate cache entries matching pattern"""
        keys = self.redis.keys(f"exact:*{pattern}*")
        if keys:
            self.redis.delete(*keys)
    
    def get_cache_stats(self) -> Dict[str, Any]:
        """Get cache statistics"""
        info = self.redis.info()
        
        # Count keys by type
        exact_keys = len(self.redis.keys("exact:*"))
        
        return {
            "total_keys": exact_keys,
            "memory_usage": info.get("used_memory_human", "unknown"),
            "hit_rate": info.get("keyspace_hits", 0) / max(1, info.get("keyspace_hits", 0) + info.get("keyspace_misses", 0)),
            "connected_clients": info.get("connected_clients", 0)
        }

# Distributed cache with multiple levels
class MultiTierCache:
    """Multi-level cache system: local memory, Redis, and persistent"""
    
    def __init__(self, redis_client: redis.Redis):
        self.local_cache = {}  # Local memory cache
        self.intelligent_cache = IntelligentCache(redis_client)
        self.max_local_size = 100
        
    async def get(self, prompt: str, **kwargs) -> Optional[Dict[str, Any]]:
        """Multi-level cache search"""
        cache_key = self.intelligent_cache._generate_cache_key(prompt, **kwargs)
        
        # Level 1: Local memory cache
        if cache_key in self.local_cache:
            result = self.local_cache[cache_key]
            result["cache_level"] = "local"
            return result
        
        # Level 2: Intelligent Redis
        redis_result = await self.intelligent_cache.get(prompt, **kwargs)
        if redis_result:
            # Promote to local cache
            self._promote_to_local(cache_key, redis_result)
            redis_result["cache_level"] = "redis"
            return redis_result
        
        # Level 3: Persistent cache (database)
        # In production, implement database search
        
        return None
    
    async def set(self, prompt: str, result: str, **kwargs):
        """Store at multiple levels"""
        cache_key = self.intelligent_cache._generate_cache_key(prompt, **kwargs)
        
        cache_data = {
            "result": result,
            "timestamp": time.time(),
            "access_count": 1
        }
        
        # Store at all levels
        self._promote_to_local(cache_key, cache_data)
        await self.intelligent_cache.set(prompt, result, **kwargs)
    
    def _promote_to_local(self, key: str, data: Dict[str, Any]):
        """Promote entry to local cache with LRU eviction"""
        if len(self.local_cache) >= self.max_local_size:
            # Remove oldest entry (simple LRU)
            oldest_key = min(
                self.local_cache.keys(),
                key=lambda k: self.local_cache[k].get("timestamp", 0)
            )
            del self.local_cache[oldest_key]
        
        self.local_cache[key] = data.copy()
```

---

## Performance Optimization

### Response Time Optimization

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor
from typing import List, Dict, Any, Optional
import numpy as np
import time

class PerformanceOptimizer:
    """Optimize AI response times for production workloads"""
    
    def __init__(self):
        self.thread_pool = ThreadPoolExecutor(max_workers=10)
        self.request_metrics = []
        self.warmup_cache = {}
        
    async def optimize_single_request(
        self, 
        prompt: str, 
        client,
        **kwargs
    ) -> Dict[str, Any]:
        """Optimize a single request with multiple techniques"""
        
        start_time = time.time()
        
        # 1. Optimized prompt preparation
        optimized_prompt = self._optimize_prompt_structure(prompt)
        
        # 2. Parameters optimized for speed
        optimized_params = self._optimize_parameters_for_speed(kwargs)
        
        # 3. Execute with optimized timeout
        try:
            response = await asyncio.wait_for(
                client.chat.completions.create(
                    model=optimized_params["model"],
                    messages=[{"role": "user", "content": optimized_prompt}],
                    **optimized_params
                ),
                timeout=optimized_params.get("timeout", 15.0)
            )
            
            duration = time.time() - start_time
            
            # Record metrics
            self.request_metrics.append({
                "duration": duration,
                "tokens": response.usage.total_tokens,
                "model": optimized_params["model"],
                "timestamp": time.time()
            })
            
            return {
                "content": response.choices[0].message.content,
                "duration": duration,
                "tokens_used": response.usage.total_tokens,
                "optimization_applied": True
            }
            
        except asyncio.TimeoutError:
            # Fallback to faster model
            return await self._fast_fallback(optimized_prompt, client)
    
    def _optimize_prompt_structure(self, prompt: str) -> str:
        """Optimize prompt structure for faster response"""
        
        # Optimization techniques:
        # 1. Remove redundant text
        # 2. Use more direct instructions
        # 3. Limit response scope
        
        if len(prompt) > 1000:
            # For long prompts, add brevity instruction
            return f"{prompt}\n\nRespond concisely and directly."
        
        return prompt
    
    def _optimize_parameters_for_speed(self, kwargs: Dict[str, Any]) -> Dict[str, Any]:
        """Optimize parameters for maximum speed"""
        
        optimized = kwargs.copy()
        
        # Parameters for speed
        optimized.update({
            "max_tokens": min(kwargs.get("max_tokens", 150), 150),  # Limit response
            "temperature": 0.1,  # Lower temperature = faster response
            "top_p": 0.8,       # Focus on most probable tokens
            "timeout": 15.0     # Aggressive timeout
        })
        
        # Select faster model if not specified
        if "model" not in optimized:
            optimized["model"] = "gpt-3.5-turbo"  # Faster model
        
        return optimized
    
    async def _fast_fallback(self, prompt: str, client) -> Dict[str, Any]:
        """Ultra-fast fallback when timeout occurs"""
        
        try:
            # Ultra-simplified prompt
            simple_prompt = prompt[:200] + "... (brief response)"
            
            response = await asyncio.wait_for(
                client.chat.completions.create(
                    model="gpt-3.5-turbo",
                    messages=[{"role": "user", "content": simple_prompt}],
                    max_tokens=50,
                    temperature=0
                ),
                timeout=5.0
            )
            
            return {
                "content": response.choices[0].message.content + " [response shortened due to timeout]",
                "duration": 5.0,
                "tokens_used": response.usage.total_tokens,
                "fallback_used": True
            }
            
        except Exception:
            return {
                "content": "Response unavailable due to service timeout",
                "duration": 5.0,
                "tokens_used": 0,
                "error_fallback": True
            }
    
    def get_performance_stats(self) -> Dict[str, Any]:
        """Get performance statistics"""
        if not self.request_metrics:
            return {"error": "No metrics available"}
        
        durations = [m["duration"] for m in self.request_metrics[-100:]]  # Last 100
        
        return {
            "avg_duration": np.mean(durations),
            "p95_duration": np.percentile(durations, 95),
            "p99_duration": np.percentile(durations, 99),
            "total_requests": len(self.request_metrics),
            "requests_per_minute": len([m for m in self.request_metrics 
                                      if time.time() - m["timestamp"] < 60])
        }

class StreamingOptimizer:
    """Optimize streaming responses for better UX"""
    
    def __init__(self):
        self.chunk_buffer = []
        self.buffer_size = 50  # Characters before sending chunk
        
    async def optimized_streaming_response(
        self, 
        prompt: str, 
        client,
        **kwargs
    ):
        """Generate optimized streaming response"""
        
        try:
            response = await client.chat.completions.create(
                model=kwargs.get("model", "gpt-4o-mini"),
                messages=[{"role": "user", "content": prompt}],
                stream=True,
                **kwargs
            )
            
            buffer = ""
            async for chunk in response:
                if chunk.choices and chunk.choices[0].delta.content:
                    content = chunk.choices[0].delta.content
                    buffer += content
                    
                    # Send chunks when buffer reaches target size
                    if len(buffer) >= self.buffer_size or content.endswith(('.', '!', '?', '\n')):
                        yield {
                            "type": "content",
                            "data": buffer,
                            "timestamp": time.time()
                        }
                        buffer = ""
            
            # Send last chunk if content remains
            if buffer:
                yield {
                    "type": "content", 
                    "data": buffer,
                    "timestamp": time.time()
                }
            
            # Completion signal
            yield {
                "type": "done",
                "data": "",
                "timestamp": time.time()
            }
            
        except Exception as e:
            yield {
                "type": "error",
                "data": str(e),
                "timestamp": time.time()
            }

class PrecomputeCache:
    """Precompute common queries during low-traffic hours"""
    
    def __init__(self, cache: IntelligentCache):
        self.cache = cache
        self.common_patterns = [
            "What is {topic}?",
            "Explain {concept} in simple terms",
            "Advantages and disadvantages of {item}",
            "How does {technology} work",
            "Step-by-step guide for {task}"
        ]
        
    async def precompute_common_queries(self, topics: List[str], client):
        """Precompute responses for common queries"""
        
        for topic in topics:
            for pattern in self.common_patterns:
                prompt = pattern.format(topic=topic, concept=topic, item=topic, 
                                      technology=topic, task=topic)
                
                # Check if already cached
                cached = await self.cache.get(prompt, model="gpt-4o-mini")
                if not cached:
                    try:
                        # Generate and cache response
                        response = await client.chat.completions.create(
                            model="gpt-4o-mini",
                            messages=[{"role": "user", "content": prompt}],
                            max_tokens=200
                        )
                        
                        await self.cache.set(
                            prompt,
                            response.choices[0].message.content,
                            content_type="static",  # Long TTL for precomputed content
                            model="gpt-4o-mini"
                        )
                        
                        # Small pause to not overload the API
                        await asyncio.sleep(0.5)
                        
                    except Exception as e:
                        logger.error(f"Error precomputing {prompt}: {e}")
                        continue
```

---

## Real-time Cost Monitoring

```python
from fastapi import FastAPI, Depends, HTTPException
from fastapi.responses import JSONResponse
import time

app = FastAPI()

# Global instances (use dependency injection in production)
cost_optimizer = CostOptimizer()
cost_tracker = CostTracker(monthly_budget=2000.0)
intelligent_cache = IntelligentCache(redis.Redis())
performance_optimizer = PerformanceOptimizer()

@app.post("/api/v1/generate")
async def generate_with_optimization(
    request: AIRequest,
    user_id: str = "default"
):
    """Optimized endpoint with complete cost and performance tracking"""
    
    start_time = time.time()
    
    # 1. Check cache first
    cached_result = await intelligent_cache.get(
        request.prompt,
        model=request.model,
        max_tokens=request.max_tokens,
        temperature=request.temperature
    )
    
    if cached_result:
        return JSONResponse({
            "content": cached_result["result"],
            "cached": True,
            "cache_type": cached_result.get("cache_type", "exact"),
            "cost": 0.0,
            "duration": time.time() - start_time
        })
    
    # 2. Select optimal model
    optimal_model = cost_optimizer.select_optimal_model(
        request.prompt,
        user_context={"tier": "standard"},
        budget_constraint=None
    )
    
    # 3. Estimate cost before call
    estimated_input_tokens = cost_optimizer._estimate_tokens(request.prompt)
    estimated_output_tokens = cost_optimizer._estimate_output_tokens(
        request.prompt, 
        cost_optimizer._analyze_prompt_complexity(request.prompt)
    )
    
    # 4. Perform optimized call
    try:
        result = await performance_optimizer.optimize_single_request(
            request.prompt,
            openai.AsyncOpenAI(),
            model=optimal_model,
            max_tokens=request.max_tokens,
            temperature=request.temperature
        )
        
        # 5. Calculate actual cost and register
        actual_cost = cost_optimizer.estimate_cost(
            "standard",  # Map model to tier
            estimated_input_tokens,
            result["tokens_used"]
        )
        
        cost_tracker.track_api_call(
            user_id=user_id,
            endpoint="/api/v1/generate",
            model=optimal_model,
            input_tokens=estimated_input_tokens,
            output_tokens=result["tokens_used"],
            cost=actual_cost
        )
        
        # 6. Cache result for future queries
        await intelligent_cache.set(
            request.prompt,
            result["content"],
            content_type="dynamic",
            metadata={"cost": actual_cost, "model": optimal_model},
            model=optimal_model,
            max_tokens=request.max_tokens,
            temperature=request.temperature
        )
        
        return JSONResponse({
            "content": result["content"],
            "cached": False,
            "model_used": optimal_model,
            "cost": actual_cost,
            "duration": result["duration"],
            "tokens_used": result["tokens_used"],
            "optimization_applied": result.get("optimization_applied", False)
        })
        
    except Exception as e:
        logger.error(f"Error in optimized generation: {e}")
        raise HTTPException(status_code=503, detail=str(e))

@app.get("/api/v1/metrics/costs")
async def get_cost_metrics():
    """Get real-time cost metrics"""
    return {
        "daily_summary": cost_tracker.get_cost_summary("daily"),
        "monthly_summary": cost_tracker.get_cost_summary("monthly"),
        "cache_stats": intelligent_cache.get_cache_stats(),
        "performance_stats": performance_optimizer.get_performance_stats()
    }
```

---

## Next in the Series

In **Part 4 (final)**, we'll cover:

- **Advanced Monitoring and Observability**: Metrics, alerts, and drift detection
- **Security and Compliance**: PII protection, auditing, and compliance
- **Advanced Deployment**: Blue-green deployment, canary releases, and A/B testing

### Key Takeaways from this Part

✅ **Cost Optimization**: Intelligent model selection can reduce costs by 40-60%  
✅ **Intelligent Caching**: A multi-level cache system improves performance and reduces costs  
✅ **Real-time Monitoring**: Cost tracking prevents billing surprises  
✅ **Performance Optimization**: Specific techniques can achieve sub-second responses  

---

**Did you find this optimization useful?** [Part 4](/blog/2025/08/10/ai-production-deployment/) will close the series with advanced deployment and operations.

> **Remember**: In production, every millisecond and every cent counts. Optimization is not optional—it's the difference between a viable product and one that doesn't scale.
