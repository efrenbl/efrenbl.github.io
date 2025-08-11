---
layout: post
title: "AI in Production: Monitoring, Security and Advanced Deployment (Part 4/4)"
date: 2025-08-10 10:00:00 -0000
categories: [blog]
tags: [ai, production, monitoring, security, deployment, observability, compliance]
---

This is the final part of our series on AI in production. Check the previous parts: [Part 1](/blog/2025/08/07/ai-production-fundamentals/) | [Part 2](/blog/2025/08/08/ai-production-patterns/) | [Part 3](/blog/2025/08/09/ai-production-optimization/)

In this final installment, we cover the critical aspects for enterprise-scale AI operations: **comprehensive observability**, **security and compliance**, and **advanced deployment strategies**.

---

## Comprehensive Monitoring and Observability

### Integral Metrics System

```python
from prometheus_client import Counter, Histogram, Gauge, CollectorRegistry
import structlog
from typing import Dict, Any, Optional, List
from datetime import datetime, timedelta
import json
import numpy as np
from collections import deque, defaultdict

# Prometheus metrics for AI
ai_requests_total = Counter(
    'ai_requests_total', 
    'Total AI API requests', 
    ['model', 'status', 'endpoint', 'user_tier']
)

ai_request_duration = Histogram(
    'ai_request_duration_seconds', 
    'AI request duration', 
    ['model', 'complexity'],
    buckets=[0.1, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0, 60.0]
)

ai_tokens_used = Counter(
    'ai_tokens_used_total', 
    'Total tokens used', 
    ['model', 'type', 'endpoint']
)

ai_cost_total = Counter(
    'ai_cost_dollars_total', 
    'Total AI API cost in dollars', 
    ['model', 'endpoint', 'user_tier']
)

ai_cache_hit_rate = Gauge(
    'ai_cache_hit_rate', 
    'AI cache hit rate'
)

ai_model_performance = Histogram(
    'ai_model_performance_score',
    'AI model performance score',
    ['model', 'task_type'],
    buckets=[0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0]
)

# Structured logging
logger = structlog.get_logger()

class ObservabilityMiddleware:
    """Middleware for comprehensive tracking of AI interactions"""
    
    def __init__(self):
        self.request_history = deque(maxlen=10000)
        self.performance_baseline = {}
        self.anomaly_threshold = 2.0  # Standard deviations
        
    async def track_ai_interaction(
        self,
        request_id: str,
        user_id: str,
        endpoint: str,
        model: str,
        prompt: str,
        response: str,
        duration: float,
        tokens_used: int,
        cost: float,
        cached: bool = False,
        error: Optional[str] = None
    ):
        """Record complete AI interaction"""
        
        timestamp = datetime.now()
        
        # Determine complexity and category
        complexity = self._analyze_complexity(prompt)
        task_type = self._classify_task_type(prompt)
        user_tier = self._get_user_tier(user_id)
        
        # Prometheus metrics
        status = "error" if error else "success"
        
        ai_requests_total.labels(
            model=model,
            status=status, 
            endpoint=endpoint,
            user_tier=user_tier
        ).inc()
        
        if not error:
            ai_request_duration.labels(
                model=model,
                complexity=complexity
            ).observe(duration)
            
            ai_tokens_used.labels(
                model=model,
                type="total",
                endpoint=endpoint
            ).inc(tokens_used)
            
            ai_cost_total.labels(
                model=model,
                endpoint=endpoint,
                user_tier=user_tier
            ).inc(cost)
        
        # Logging estructurado
        log_data = {
            "request_id": request_id,
            "user_id": user_id,
            "endpoint": endpoint,
            "model": model,
            "complexity": complexity,
            "task_type": task_type,
            "duration": duration,
            "tokens_used": tokens_used,
            "cost": cost,
            "cached": cached,
            "user_tier": user_tier,
            "timestamp": timestamp.isoformat(),
            "prompt_length": len(prompt),
            "response_length": len(response),
            "error": error
        }
        
        if error:
            logger.error("AI request failed", **log_data)
        else:
            logger.info("AI request completed", **log_data)
        
        # Store for trend analysis
        self.request_history.append(log_data)
        
        # Detect anomalies
        await self._check_for_anomalies(log_data)
    
    def _analyze_complexity(self, prompt: str) -> str:
        """Analizar complejidad del prompt"""
        if len(prompt) > 1000:
            return "high"
        elif len(prompt) > 300:
            return "medium"
        else:
            return "low"
    
    def _classify_task_type(self, prompt: str) -> str:
        """Clasificar tipo de tarea basado en el prompt"""
        prompt_lower = prompt.lower()
        
        task_patterns = {
            "translation": ["traduce", "translate", "traducir"],
            "summarization": ["resume", "resumen", "summarize"],
            "analysis": ["analyze", "analiza", "analysis"],
            "generation": ["genera", "create", "escribe"],
            "qa": ["what is", "qué es", "explain", "explica"],
            "extraction": ["extrae", "extract", "encuentra", "find"]
        }
        
        for task_type, patterns in task_patterns.items():
            if any(pattern in prompt_lower for pattern in patterns):
                return task_type
        
        return "general"
    
    def _get_user_tier(self, user_id: str) -> str:
        """Obtener tier del usuario (integrar con sistema de usuarios)"""
        # In production, query user database
        return "standard"  # Default
    
    async def _check_for_anomalies(self, current_request: Dict[str, Any]):
        """Detect anomalies in performance and usage"""
        
        # Obtener requests recientes similares
        recent_requests = [
            req for req in self.request_history
            if (req["model"] == current_request["model"] and
                req["complexity"] == current_request["complexity"] and
                not req.get("error"))
        ]
        
        if len(recent_requests) < 10:  # Necesitamos suficiente historia
            return
        
        # Analyze duration
        durations = [req["duration"] for req in recent_requests[-50:]]
        mean_duration = np.mean(durations)
        std_duration = np.std(durations)
        
        if (current_request["duration"] > mean_duration + 
            self.anomaly_threshold * std_duration):
            
            logger.warning(
                "Performance anomaly detected",
                request_id=current_request["request_id"],
                current_duration=current_request["duration"],
                expected_duration=mean_duration,
                deviation=current_request["duration"] - mean_duration
            )
        
        # Analizar costo
        costs = [req["cost"] for req in recent_requests[-50:]]
        mean_cost = np.mean(costs)
        std_cost = np.std(costs)
        
        if (current_request["cost"] > mean_cost + 
            self.anomaly_threshold * std_cost):
            
            logger.warning(
                "Cost anomaly detected",
                request_id=current_request["request_id"],
                current_cost=current_request["cost"],
                expected_cost=mean_cost
            )

class DriftDetector:
    """Detect changes in AI model behavior over time"""
    
    def __init__(self, window_size: int = 1000):
        self.window_size = window_size
        self.response_patterns = defaultdict(deque)
        self.quality_metrics = defaultdict(deque)
        self.baseline_established = False
        
    def analyze_response_drift(
        self,
        model: str,
        task_type: str,
        prompt: str,
        response: str,
        quality_score: Optional[float] = None
    ):
        """Analyze drift in model responses"""
        
        # Extract response features
        features = self._extract_response_features(response)
        
        # Almacenar en ventana deslizante
        key = f"{model}:{task_type}"
        self.response_patterns[key].append(features)
        
        if len(self.response_patterns[key]) > self.window_size:
            self.response_patterns[key].popleft()
        
        # Store quality metrics if available
        if quality_score is not None:
            self.quality_metrics[key].append(quality_score)
            if len(self.quality_metrics[key]) > self.window_size:
                self.quality_metrics[key].popleft()
        
        # Detectar drift si tenemos suficientes datos
        if len(self.response_patterns[key]) >= 100:
            drift_detected = self._detect_drift(key)
            if drift_detected:
                self._alert_drift(model, task_type, drift_detected)
    
    def _extract_response_features(self, response: str) -> Dict[str, float]:
        """Extract numerical features from response"""
        return {
            "length": len(response),
            "sentence_count": response.count('.') + response.count('!') + response.count('?'),
            "word_count": len(response.split()),
            "avg_word_length": np.mean([len(word) for word in response.split()]) if response.split() else 0,
            "punctuation_ratio": sum(1 for char in response if char in '.,!?;:') / max(1, len(response)),
            "uppercase_ratio": sum(1 for char in response if char.isupper()) / max(1, len(response))
        }
    
    def _detect_drift(self, key: str) -> Optional[Dict[str, Any]]:
        """Detect statistical drift in features"""
        patterns = list(self.response_patterns[key])
        
        # Divide into windows for comparison
        window_1 = patterns[:len(patterns)//2]
        window_2 = patterns[len(patterns)//2:]
        
        if len(window_1) < 50 or len(window_2) < 50:
            return None
        
        drift_results = {}
        
        # Compare each feature
        for feature in patterns[0].keys():
            values_1 = [p[feature] for p in window_1]
            values_2 = [p[feature] for p in window_2]
            
            mean_1, std_1 = np.mean(values_1), np.std(values_1)
            mean_2, std_2 = np.mean(values_2), np.std(values_2)
            
            # Test simple de drift (diferencia significativa en medias)
            if std_1 > 0 and std_2 > 0:
                z_score = abs(mean_2 - mean_1) / np.sqrt(std_1**2 + std_2**2)
                if z_score > 2.0:  # Umbral de significancia
                    drift_results[feature] = {
                        "mean_change": mean_2 - mean_1,
                        "relative_change": (mean_2 - mean_1) / mean_1 if mean_1 != 0 else 0,
                        "z_score": z_score
                    }
        
        return drift_results if drift_results else None
    
    def _alert_drift(self, model: str, task_type: str, drift_data: Dict[str, Any]):
        """Alertar sobre drift detectado"""
        logger.warning(
            "Model drift detected",
            model=model,
            task_type=task_type,
            drift_features=list(drift_data.keys()),
            drift_details=drift_data
        )

# Sistema de alertas configurables
class AlertManager:
    """Gestor de alertas para sistemas de AI"""
    
    def __init__(self):
        self.alert_rules = {
            "high_cost": {
                "threshold": 100.0,  # $100/hora
                "window": 3600,      # 1 hora
                "severity": "warning"
            },
            "high_latency": {
                "threshold": 5.0,    # 5 segundos
                "window": 300,       # 5 minutos
                "severity": "critical"
            },
            "high_error_rate": {
                "threshold": 0.05,   # 5% error rate
                "window": 600,       # 10 minutos
                "severity": "critical"
            },
            "cache_miss_rate": {
                "threshold": 0.8,    # 80% miss rate
                "window": 1800,      # 30 minutos
                "severity": "warning"
            }
        }
        
        self.alert_history = deque(maxlen=1000)
        
    def check_alerts(self, metrics: Dict[str, Any]):
        """Verificar condiciones de alerta"""
        current_time = datetime.now()
        
        for rule_name, rule_config in self.alert_rules.items():
            if self._evaluate_alert_condition(rule_name, rule_config, metrics):
                self._trigger_alert(rule_name, rule_config, metrics, current_time)
    
    def _evaluate_alert_condition(
        self, 
        rule_name: str, 
        rule_config: Dict[str, Any], 
        metrics: Dict[str, Any]
    ) -> bool:
        """Evaluate if alert condition is met"""
        
        if rule_name == "high_cost":
            return metrics.get("hourly_cost", 0) > rule_config["threshold"]
        elif rule_name == "high_latency":
            return metrics.get("avg_latency", 0) > rule_config["threshold"]
        elif rule_name == "high_error_rate":
            return metrics.get("error_rate", 0) > rule_config["threshold"]
        elif rule_name == "cache_miss_rate":
            return metrics.get("cache_miss_rate", 0) > rule_config["threshold"]
        
        return False
    
    def _trigger_alert(
        self, 
        rule_name: str, 
        rule_config: Dict[str, Any], 
        metrics: Dict[str, Any],
        timestamp: datetime
    ):
        """Disparar alerta"""
        
        alert = {
            "rule": rule_name,
            "severity": rule_config["severity"],
            "timestamp": timestamp.isoformat(),
            "metrics": metrics,
            "threshold": rule_config["threshold"]
        }
        
        self.alert_history.append(alert)
        
        # Critical log
        logger.critical(
            f"Alert triggered: {rule_name}",
            **alert
        )
        
        # In production, integrate with notification systems:
        # - Slack/Teams webhooks
        # - PagerDuty for critical alerts
        # - Email notifications
        # - SMS for critical alerts
```

---

## Security and Compliance

### PII and Sensitive Data Protection

```python
import re
from typing import Dict, List, Tuple, Optional, Set
from dataclasses import dataclass
import hashlib
import logging
from enum import Enum

class PIIType(Enum):
    EMAIL = "email"
    PHONE = "phone" 
    SSN = "ssn"
    CREDIT_CARD = "credit_card"
    NAME = "name"
    ADDRESS = "address"
    IP_ADDRESS = "ip_address"
    DATE_OF_BIRTH = "date_of_birth"

@dataclass
class PIIMatch:
    """Represents a found PII match"""
    type: PIIType
    value: str
    start: int
    end: int
    confidence: float
    replacement: str

class PIIProtector:
    """Remove or mask PII before sending to AI APIs"""
    
    def __init__(self):
        # Patrones regex para diferentes tipos de PII
        self.patterns = {
            PIIType.EMAIL: re.compile(
                r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'
            ),
            PIIType.PHONE: re.compile(
                r'(\+?1[-.\s]?)?\(?([0-9]{3})\)?[-.\s]?([0-9]{3})[-.\s]?([0-9]{4})'
            ),
            PIIType.SSN: re.compile(
                r'\b\d{3}-?\d{2}-?\d{4}\b'
            ),
            PIIType.CREDIT_CARD: re.compile(
                r'\b(?:\d{4}[-.\s]?){3}\d{4}\b'
            ),
            PIIType.IP_ADDRESS: re.compile(
                r'\b(?:[0-9]{1,3}\.){3}[0-9]{1,3}\b'
            ),
            PIIType.DATE_OF_BIRTH: re.compile(
                r'\b\d{1,2}/\d{1,2}/\d{4}\b|\b\d{4}-\d{2}-\d{2}\b'
            )
        }
        
        # Nombres comunes para detectar (lista simplificada)
        self.common_names = {
            "juan", "maría", "carlos", "ana", "luis", "carmen", "josé", "francisco",
            "antonio", "dolores", "manuel", "pilar", "david", "teresa", "miguel"
        }
        
        # Replacement configuration
        self.replacement_templates = {
            PIIType.EMAIL: "[EMAIL_{}]",
            PIIType.PHONE: "[PHONE_{}]", 
            PIIType.SSN: "[SSN_{}]",
            PIIType.CREDIT_CARD: "[CARD_{}]",
            PIIType.NAME: "[NAME_{}]",
            PIIType.ADDRESS: "[ADDRESS_{}]",
            PIIType.IP_ADDRESS: "[IP_{}]",
            PIIType.DATE_OF_BIRTH: "[DOB_{}]"
        }
    
    def protect_text(self, text: str) -> Tuple[str, Dict[str, str]]:
        """
        Proteger texto removiendo/enmascarando PII
        
        Returns:
            Tuple[str, Dict[str, str]]: (protected_text, reversal_mapping)
        """
        
        protected_text = text
        reversal_mapping = {}
        pii_matches = []
        
        # Detectar todos los tipos de PII
        for pii_type, pattern in self.patterns.items():
            matches = list(pattern.finditer(text))
            for match in matches:
                pii_matches.append(PIIMatch(
                    type=pii_type,
                    value=match.group(),
                    start=match.start(),
                    end=match.end(),
                    confidence=1.0,
                    replacement=""
                ))
        
        # Detect names (more complex logic)
        name_matches = self._detect_names(text)
        pii_matches.extend(name_matches)
        
        # Ordenar por posición para reemplazo consistente
        pii_matches.sort(key=lambda x: x.start, reverse=True)
        
        # Realizar reemplazos
        for i, match in enumerate(pii_matches):
            # Generar ID único para el reemplazo
            replacement_id = hashlib.md5(f"{match.value}{i}".encode()).hexdigest()[:8]
            replacement = self.replacement_templates[match.type].format(replacement_id)
            
            # Almacenar para reversión
            reversal_mapping[replacement] = match.value
            
            # Realizar reemplazo
            protected_text = (
                protected_text[:match.start] + 
                replacement + 
                protected_text[match.end:]
            )
        
        return protected_text, reversal_mapping
    
    def restore_text(self, protected_text: str, reversal_mapping: Dict[str, str]) -> str:
        """Restore original text using reversal mapping"""
        restored_text = protected_text
        
        for replacement, original in reversal_mapping.items():
            restored_text = restored_text.replace(replacement, original)
        
        return restored_text
    
    def _detect_names(self, text: str) -> List[PIIMatch]:
        """Detect names in text (simplified heuristic)"""
        matches = []
        words = text.split()
        
        for i, word in enumerate(words):
            # Buscar palabras capitalizadas que podrían ser nombres
            if (word.lower() in self.common_names and 
                word[0].isupper() and 
                len(word) > 2):
                
                # Calcular posición en texto original
                start_pos = text.find(word)
                end_pos = start_pos + len(word)
                
                matches.append(PIIMatch(
                    type=PIIType.NAME,
                    value=word,
                    start=start_pos,
                    end=end_pos,
                    confidence=0.8,
                    replacement=""
                ))
        
        return matches
    
    def validate_response(self, response: str, reversal_mapping: Dict[str, str]) -> bool:
        """Validar que la respuesta no contenga PII no autorizada"""
        
        # Verificar que no se hayan filtrado datos originales
        for original_value in reversal_mapping.values():
            if original_value.lower() in response.lower():
                logger.warning(f"PII leak detected in response: {original_value}")
                return False
        
        return True

class SecureAIGateway:
    """Secure gateway for AI calls with audit logging"""
    
    def __init__(self, pii_protector: PIIProtector):
        self.pii_protector = pii_protector
        self.audit_log = []
        
    async def secure_ai_call(
        self,
        user_id: str,
        original_prompt: str,
        ai_client,
        **kwargs
    ) -> Dict[str, Any]:
        """Realizar llamada a AI con protecciones de seguridad"""
        
        request_id = hashlib.md5(f"{user_id}{original_prompt}{time.time()}".encode()).hexdigest()[:12]
        
        # 1. Proteger PII en el prompt
        protected_prompt, reversal_mapping = self.pii_protector.protect_text(original_prompt)
        
        # 2. Auditar la request
        audit_entry = {
            "request_id": request_id,
            "user_id": user_id,
            "timestamp": datetime.now().isoformat(),
            "original_prompt_hash": hashlib.sha256(original_prompt.encode()).hexdigest(),
            "protected_prompt_hash": hashlib.sha256(protected_prompt.encode()).hexdigest(),
            "pii_detected": len(reversal_mapping) > 0,
            "pii_types": list(set(match.split('_')[0] for match in reversal_mapping.keys()))
        }
        
        try:
            # 3. Realizar llamada a AI con prompt protegido
            response = await ai_client.chat.completions.create(
                messages=[{"role": "user", "content": protected_prompt}],
                **kwargs
            )
            
            ai_response = response.choices[0].message.content
            
            # 4. Validar que la respuesta no contenga PII filtrada
            is_safe = self.pii_protector.validate_response(ai_response, reversal_mapping)
            
            if not is_safe:
                audit_entry.update({
                    "status": "blocked",
                    "reason": "pii_leak_detected"
                })
                self.audit_log.append(audit_entry)
                raise SecurityException("Respuesta bloqueada por filtración de PII")
            
            # 5. Restaurar PII autorizada en respuesta (si es necesario)
            # En la mayoría de casos, NO queremos restaurar PII en la respuesta
            final_response = ai_response
            
            audit_entry.update({
                "status": "success",
                "response_length": len(ai_response),
                "tokens_used": response.usage.total_tokens
            })
            
            return {
                "response": final_response,
                "request_id": request_id,
                "pii_protected": len(reversal_mapping) > 0,
                "audit_entry": audit_entry
            }
            
        except Exception as e:
            audit_entry.update({
                "status": "error",
                "error": str(e)
            })
            raise e
        
        finally:
            self.audit_log.append(audit_entry)
    
    def get_audit_report(self, user_id: Optional[str] = None) -> Dict[str, Any]:
        """Generate audit report"""
        
        filtered_logs = self.audit_log
        if user_id:
            filtered_logs = [log for log in self.audit_log if log["user_id"] == user_id]
        
        total_requests = len(filtered_logs)
        pii_requests = len([log for log in filtered_logs if log["pii_detected"]])
        blocked_requests = len([log for log in filtered_logs if log["status"] == "blocked"])
        
        return {
            "total_requests": total_requests,
            "pii_requests": pii_requests,
            "pii_rate": pii_requests / max(1, total_requests),
            "blocked_requests": blocked_requests,
            "block_rate": blocked_requests / max(1, total_requests),
            "user_id": user_id
        }

class SecurityException(Exception):
    """Exception for security violations"""
    pass

# Compliance y Auditoría
class ComplianceManager:
    """Asegurar que el uso de AI cumple con requisitos regulatorios"""
    
    def __init__(self):
        self.compliance_rules = {
            "gdpr": {
                "data_retention_days": 30,
                "consent_required": True,
                "right_to_delete": True
            },
            "ccpa": {
                "data_retention_days": 365,
                "opt_out_required": True
            },
            "hipaa": {
                "data_retention_days": 2555,  # 7 años
                "encryption_required": True,
                "audit_trail_required": True
            }
        }
        
        self.audit_trail = []
        
    def validate_request(
        self,
        user_id: str,
        data_classification: str,
        applicable_regulations: List[str]
    ) -> bool:
        """Validar que la request cumple con compliance"""
        
        for regulation in applicable_regulations:
            if regulation not in self.compliance_rules:
                logger.warning(f"Unknown regulation: {regulation}")
                continue
            
            rules = self.compliance_rules[regulation]
            
            # Verificar consentimiento (simplificado)
            if rules.get("consent_required") and not self._check_consent(user_id):
                return False
            
            # Verificar clasificación de datos para HIPAA
            if regulation == "hipaa" and data_classification == "phi":
                if not self._validate_hipaa_requirements():
                    return False
        
        return True
    
    def _check_consent(self, user_id: str) -> bool:
        """Verificar que el usuario ha dado consentimiento"""
        # In production, query consent database
        return True  # Simplificado
    
    def _validate_hipaa_requirements(self) -> bool:
        """Validate HIPAA-specific requirements"""
        # Verificar que el entorno cumple con HIPAA
        # - Encriptación en tránsito y reposo
        # - Logging de auditoría
        # - Controles de acceso
        return True  # Simplificado
    
    def log_data_processing(
        self,
        user_id: str,
        data_type: str,
        purpose: str,
        regulations: List[str]
    ):
        """Record data processing for audit"""
        
        self.audit_trail.append({
            "timestamp": datetime.now().isoformat(),
            "user_id": user_id,
            "data_type": data_type,
            "purpose": purpose,
            "regulations": regulations,
            "system": "ai_service"
        })
```

---

## Advanced Deployment and Operations

### Blue-Green Deployment for AI

```python
from typing import Dict, Any, List, Optional
import asyncio
import json
import time
from enum import Enum

class DeploymentState(Enum):
    BLUE = "blue"
    GREEN = "green"
    TRANSITION = "transition"

class BlueGreenDeployment:
    """Zero-downtime deployment for AI services"""
    
    def __init__(self):
        self.environments = {
            "blue": {
                "active": True,
                "traffic_percentage": 100,
                "model_version": "v1.0",
                "health_status": "healthy"
            },
            "green": {
                "active": False,
                "traffic_percentage": 0,
                "model_version": None,
                "health_status": "not_deployed"
            }
        }
        
        self.deployment_state = DeploymentState.BLUE
        self.validation_tests = []
        
    async def deploy_new_version(
        self,
        new_model_version: str,
        validation_criteria: Dict[str, Any]
    ) -> bool:
        """Deploy new model version with validation"""
        
        # 1. Determinar ambiente target (el inactivo)
        target_env = "green" if self.deployment_state == DeploymentState.BLUE else "blue"
        
        logger.info(f"Starting deployment of {new_model_version} to {target_env}")
        
        try:
            # 2. Desplegar en ambiente target
            await self._deploy_to_environment(target_env, new_model_version)
            
            # 3. Ejecutar health checks
            if not await self._health_check(target_env):
                raise DeploymentError(f"Health check failed for {target_env}")
            
            # 4. Ejecutar validación funcional
            validation_passed = await self._run_validation_tests(
                target_env, 
                validation_criteria
            )
            
            if not validation_passed:
                raise DeploymentError("Validation tests failed")
            
            # 5. Canary deployment - dirigir pequeño porcentaje de tráfico
            await self._start_canary_deployment(target_env, canary_percentage=5)
            
            # 6. Monitorear métricas durante canary
            canary_success = await self._monitor_canary_metrics(target_env, duration=300)
            
            if not canary_success:
                await self._rollback_deployment(target_env)
                raise DeploymentError("Canary deployment failed metrics validation")
            
            # 7. Gradualmente incrementar tráfico
            await self._gradual_traffic_shift(target_env)
            
            # 8. Completar switch
            await self._complete_deployment_switch(target_env)
            
            logger.info(f"Successfully deployed {new_model_version}")
            return True
            
        except Exception as e:
            logger.error(f"Deployment failed: {e}")
            await self._rollback_deployment(target_env)
            return False
    
    async def _deploy_to_environment(self, env: str, model_version: str):
        """Deploy model to specific environment"""
        
        # In production, this would include:
        # - Actualizar configuración de Kubernetes
        # - Descargar nuevos modelos
        # - Actualizar variables de entorno
        # - Restart servicios
        
        self.environments[env].update({
            "model_version": model_version,
            "health_status": "deploying"
        })
        
        # Simulate deployment time
        await asyncio.sleep(2)
        
        self.environments[env]["health_status"] = "healthy"
    
    async def _health_check(self, env: str) -> bool:
        """Ejecutar health checks comprehensivos"""
        
        checks = [
            self._check_service_availability(env),
            self._check_model_loading(env),
            self._check_dependencies(env),
            self._check_resource_usage(env)
        ]
        
        results = await asyncio.gather(*checks, return_exceptions=True)
        
        for i, result in enumerate(results):
            if isinstance(result, Exception) or not result:
                logger.error(f"Health check {i} failed for {env}: {result}")
                return False
        
        return True
    
    async def _check_service_availability(self, env: str) -> bool:
        """Verificar disponibilidad del servicio"""
        # Implementar ping a endpoint de health
        return True
    
    async def _check_model_loading(self, env: str) -> bool:
        """Verificar que el modelo se carga correctamente"""
        # Hacer request de prueba al modelo
        return True
    
    async def _check_dependencies(self, env: str) -> bool:
        """Verificar dependencias (Redis, DB, etc.)"""
        return True
    
    async def _check_resource_usage(self, env: str) -> bool:
        """Verificar uso de recursos (CPU, memoria)"""
        return True
    
    async def _run_validation_tests(
        self,
        env: str,
        criteria: Dict[str, Any]
    ) -> bool:
        """Execute functional validation tests"""
        
        test_suite = ValidationTestSuite(env, criteria)
        results = await test_suite.run_all_tests()
        
        passed_tests = sum(1 for result in results if result["passed"])
        total_tests = len(results)
        
        success_rate = passed_tests / total_tests
        required_success_rate = criteria.get("min_success_rate", 0.95)
        
        logger.info(f"Validation tests: {passed_tests}/{total_tests} passed ({success_rate:.2%})")
        
        return success_rate >= required_success_rate
    
    async def _start_canary_deployment(self, target_env: str, canary_percentage: int):
        """Start canary deployment directing small percentage of traffic"""
        
        current_env = "blue" if target_env == "green" else "green"
        
        self.environments[current_env]["traffic_percentage"] = 100 - canary_percentage
        self.environments[target_env]["traffic_percentage"] = canary_percentage
        self.environments[target_env]["active"] = True
        
        self.deployment_state = DeploymentState.TRANSITION
        
        logger.info(f"Started canary deployment: {canary_percentage}% to {target_env}")
    
    async def _monitor_canary_metrics(self, target_env: str, duration: int) -> bool:
        """Monitor metrics during canary deployment"""
        
        start_time = time.time()
        
        while time.time() - start_time < duration:
            # Recolectar métricas del ambiente canary
            metrics = await self._collect_environment_metrics(target_env)
            
            # Evaluar contra baseline
            if not self._evaluate_canary_metrics(metrics):
                return False
            
            await asyncio.sleep(30)  # Check cada 30 segundos
        
        return True
    
    async def _collect_environment_metrics(self, env: str) -> Dict[str, float]:
        """Collect metrics from specific environment"""
        # In production, query Prometheus, DataDog, etc.
        return {
            "error_rate": 0.01,
            "avg_latency": 0.5,
            "p95_latency": 1.2,
            "throughput": 100.0
        }
    
    def _evaluate_canary_metrics(self, metrics: Dict[str, float]) -> bool:
        """Evaluate if canary metrics are acceptable"""
        
        thresholds = {
            "error_rate": 0.05,      # Max 5% error rate
            "avg_latency": 2.0,      # Max 2s average latency
            "p95_latency": 5.0       # Max 5s p95 latency
        }
        
        for metric, value in metrics.items():
            if metric in thresholds and value > thresholds[metric]:
                logger.warning(f"Canary metric {metric} exceeded threshold: {value} > {thresholds[metric]}")
                return False
        
        return True
    
    async def _gradual_traffic_shift(self, target_env: str):
        """Gradually increase traffic to new environment"""
        
        traffic_steps = [10, 25, 50, 75, 100]
        current_env = "blue" if target_env == "green" else "green"
        
        for target_percentage in traffic_steps:
            # Actualizar distribución de tráfico
            self.environments[target_env]["traffic_percentage"] = target_percentage
            self.environments[current_env]["traffic_percentage"] = 100 - target_percentage
            
            logger.info(f"Traffic shift: {target_percentage}% to {target_env}")
            
            # Monitorear por un período
            await asyncio.sleep(120)  # Esperar 2 minutos entre incrementos
            
            # Verificar métricas
            metrics = await self._collect_environment_metrics(target_env)
            if not self._evaluate_canary_metrics(metrics):
                raise DeploymentError(f"Metrics degraded at {target_percentage}% traffic")
    
    async def _complete_deployment_switch(self, target_env: str):
        """Completar el switch al nuevo ambiente"""
        
        current_env = "blue" if target_env == "green" else "green"
        
        # Desactivar ambiente anterior
        self.environments[current_env].update({
            "active": False,
            "traffic_percentage": 0
        })
        
        # Activar completamente nuevo ambiente
        self.environments[target_env].update({
            "active": True,
            "traffic_percentage": 100
        })
        
        # Actualizar estado de deployment
        self.deployment_state = DeploymentState.GREEN if target_env == "green" else DeploymentState.BLUE
        
        logger.info(f"Deployment switch completed to {target_env}")
    
    async def _rollback_deployment(self, failed_env: str):
        """Rollback en caso de falla"""
        
        logger.warning(f"Rolling back deployment from {failed_env}")
        
        # Restaurar estado anterior
        current_env = "blue" if failed_env == "green" else "green"
        
        self.environments[current_env].update({
            "active": True,
            "traffic_percentage": 100
        })
        
        self.environments[failed_env].update({
            "active": False,
            "traffic_percentage": 0,
            "health_status": "rollback"
        })
        
        self.deployment_state = DeploymentState.BLUE if current_env == "blue" else DeploymentState.GREEN

class ValidationTestSuite:
    """Suite de tests para validar deployments de AI"""
    
    def __init__(self, environment: str, criteria: Dict[str, Any]):
        self.environment = environment
        self.criteria = criteria
        self.test_cases = self._load_test_cases()
    
    def _load_test_cases(self) -> List[Dict[str, Any]]:
        """Load test cases for validation"""
        return [
            {
                "name": "basic_functionality",
                "prompt": "¿Qué es la inteligencia artificial?",
                "expected_keywords": ["inteligencia", "artificial", "tecnología"],
                "max_response_time": 5.0
            },
            {
                "name": "complex_reasoning",
                "prompt": "Explica las ventajas y desventajas de usar AI en medicina",
                "expected_keywords": ["ventajas", "desventajas", "medicina"],
                "min_response_length": 100,
                "max_response_time": 10.0
            },
            {
                "name": "error_handling", 
                "prompt": "",  # Prompt vacío para probar error handling
                "expect_error": True
            }
        ]
    
    async def run_all_tests(self) -> List[Dict[str, Any]]:
        """Ejecutar todos los tests de validación"""
        
        results = []
        
        for test_case in self.test_cases:
            try:
                result = await self._run_single_test(test_case)
                results.append(result)
            except Exception as e:
                results.append({
                    "test_name": test_case["name"],
                    "passed": False,
                    "error": str(e)
                })
        
        return results
    
    async def _run_single_test(self, test_case: Dict[str, Any]) -> Dict[str, Any]:
        """Ejecutar un test individual"""
        
        start_time = time.time()
        
        # Simulate call to test environment
        # In production, make real request to environment endpoint
        response = await self._make_test_request(test_case["prompt"])
        
        duration = time.time() - start_time
        
        # Evaluar resultado
        passed = self._evaluate_test_result(test_case, response, duration)
        
        return {
            "test_name": test_case["name"],
            "passed": passed,
            "duration": duration,
            "response_length": len(response) if response else 0
        }
    
    async def _make_test_request(self, prompt: str) -> str:
        """Hacer request de prueba al ambiente"""
        # In production, make HTTP request to environment-specific endpoint
        if not prompt:
            raise ValueError("Empty prompt")
        
        # Simulate response
        await asyncio.sleep(0.1)
        return f"Respuesta simulada para: {prompt}"
    
    def _evaluate_test_result(
        self,
        test_case: Dict[str, Any],
        response: str,
        duration: float
    ) -> bool:
        """Evaluar si el resultado del test es válido"""
        
        # Verificar tiempo de respuesta
        max_time = test_case.get("max_response_time")
        if max_time and duration > max_time:
            return False
        
        # Verificar longitud mínima de respuesta
        min_length = test_case.get("min_response_length")
        if min_length and len(response) < min_length:
            return False
        
        # Verificar palabras clave esperadas
        expected_keywords = test_case.get("expected_keywords", [])
        for keyword in expected_keywords:
            if keyword.lower() not in response.lower():
                return False
        
        return True

class DeploymentError(Exception):
    """Excepción para errores de deployment"""
    pass
```

---

## Complete Integration Example

```python
from fastapi import FastAPI, Depends, HTTPException, BackgroundTasks
from fastapi.responses import StreamingResponse
import uvicorn
from typing import Optional
import asyncio

# Initialize application with all production capabilities
app = FastAPI(
    title="Production AI Service - Complete",
    version="2.0.0",
    description="Complete AI service with all production capabilities"
)

# Initialize components
pii_protector = PIIProtector()
secure_gateway = SecureAIGateway(pii_protector)
compliance_manager = ComplianceManager()
observability = ObservabilityMiddleware()
drift_detector = DriftDetector()
alert_manager = AlertManager()
blue_green = BlueGreenDeployment()

@app.post("/api/v2/secure-generate")
async def secure_generate(
    request: AIRequest,
    user_id: str,
    background_tasks: BackgroundTasks
):
    """Complete endpoint with all production protections"""
    
    request_id = f"req_{int(time.time() * 1000)}"
    start_time = time.time()
    
    try:
        # 1. Compliance validation
        if not compliance_manager.validate_request(
            user_id=user_id,
            data_classification="general",
            applicable_regulations=["gdpr"]
        ):
            raise HTTPException(status_code=403, detail="Request violates compliance rules")
        
        # 2. Secure AI call
        ai_result = await secure_gateway.secure_ai_call(
            user_id=user_id,
            original_prompt=request.prompt,
            ai_client=openai.AsyncOpenAI(),
            model="gpt-4o-mini",
            max_tokens=request.max_tokens,
            temperature=request.temperature
        )
        
        duration = time.time() - start_time
        
        # 3. Observability tracking (in background)
        background_tasks.add_task(
            observability.track_ai_interaction,
            request_id=request_id,
            user_id=user_id,
            endpoint="/api/v2/secure-generate",
            model="gpt-4o-mini",
            prompt=request.prompt,
            response=ai_result["response"],
            duration=duration,
            tokens_used=ai_result.get("tokens_used", 0),
            cost=0.01,  # Calculate real cost
            cached=False
        )
        
        # 4. Drift analysis (in background)
        background_tasks.add_task(
            drift_detector.analyze_response_drift,
            model="gpt-4o-mini",
            task_type="general",
            prompt=request.prompt,
            response=ai_result["response"]
        )
        
        # 5. Compliance logging
        compliance_manager.log_data_processing(
            user_id=user_id,
            data_type="text",
            purpose="ai_generation",
            regulations=["gdpr"]
        )
        
        return {
            "content": ai_result["response"],
            "request_id": ai_result["request_id"],
            "security": {
                "pii_protected": ai_result["pii_protected"],
                "audit_logged": True
            },
            "performance": {
                "duration": duration,
                "model": "gpt-4o-mini"
            }
        }
        
    except Exception as e:
        # Error logging with complete context
        logger.error(
            "Secure AI request failed",
            request_id=request_id,
            user_id=user_id,
            error=str(e),
            duration=time.time() - start_time
        )
        
        raise HTTPException(status_code=500, detail="AI service temporarily unavailable")

@app.get("/api/v2/health")
async def comprehensive_health_check():
    """Comprehensive system health check"""
    
    health_status = {
        "status": "healthy",
        "timestamp": datetime.now().isoformat(),
        "components": {}
    }
    
    # Verify critical components
    components = {
        "ai_service": lambda: True,  # Implementar ping a AI API
        "redis_cache": lambda: True,  # Ping a Redis
        "database": lambda: True,    # Ping a base de datos
        "security": lambda: True     # Verificar sistemas de seguridad
    }
    
    overall_healthy = True
    
    for component, check_func in components.items():
        try:
            is_healthy = check_func()
            health_status["components"][component] = {
                "status": "healthy" if is_healthy else "unhealthy",
                "checked_at": datetime.now().isoformat()
            }
            if not is_healthy:
                overall_healthy = False
        except Exception as e:
            health_status["components"][component] = {
                "status": "error",
                "error": str(e),
                "checked_at": datetime.now().isoformat()
            }
            overall_healthy = False
    
    health_status["status"] = "healthy" if overall_healthy else "degraded"
    
    return health_status

@app.get("/api/v2/metrics/comprehensive")
async def get_comprehensive_metrics():
    """Comprehensive system metrics"""
    
    return {
        "observability": {
            "request_count": len(observability.request_history),
            "performance_stats": observability.get_performance_stats() if hasattr(observability, 'get_performance_stats') else {}
        },
        "security": {
            "audit_summary": secure_gateway.get_audit_report(),
            "pii_detection_rate": 0.15  # Ejemplo
        },
        "compliance": {
            "audit_entries": len(compliance_manager.audit_trail),
            "regulations_covered": ["gdpr", "ccpa"]
        },
        "deployment": {
            "current_environment": blue_green.deployment_state.value,
            "environments": blue_green.environments
        }
    }

@app.post("/api/v2/admin/deploy")
async def trigger_deployment(
    new_version: str,
    validation_criteria: Optional[Dict[str, Any]] = None
):
    """Administrative endpoint to trigger deployment"""
    
    if not validation_criteria:
        validation_criteria = {
            "min_success_rate": 0.95,
            "max_error_rate": 0.05
        }
    
    try:
        success = await blue_green.deploy_new_version(
            new_model_version=new_version,
            validation_criteria=validation_criteria
        )
        
        return {
            "success": success,
            "message": f"Deployment of {new_version} {'completed' if success else 'failed'}",
            "current_state": blue_green.deployment_state.value
        }
        
    except Exception as e:
        raise HTTPException(
            status_code=500, 
            detail=f"Deployment failed: {str(e)}"
        )

if __name__ == "__main__":
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=8000,
        reload=False,  # Disabled in production
        workers=4,
        log_level="info"
    )
```

---

## Series Conclusions

We have covered the complete journey from AI prototypes to enterprise-scale production systems:

### 🏗️ **Part 1 - Fundamentals**: 
- Solid architecture with engineering principles
- Quick start that works in production
- Prompt engineering as software discipline

### ⚡ **Part 2 - Resilience Patterns**:
- Circuit breakers and failover patterns
- Request batching for optimization
- Comprehensive error handling

### 💰 **Part 3 - Optimization**:
- Dynamic model selection for cost control
- Intelligent multi-level cache systems
- Performance optimization for sub-second responses

### 🔒 **Part 4 - Operations**:
- Comprehensive observability and monitoring
- Enterprise-grade security and compliance
- Zero-downtime deployment strategies

---

## Final Key Takeaways

✅ **Architecture**: AI in production is 20% model, 80% system  
✅ **Observability**: If you can't measure it, you can't optimize it  
✅ **Security**: PII protection and compliance are not optional  
✅ **Resilience**: Design assuming everything will fail  
✅ **Costs**: Cost optimization from day 1, not as afterthought  
✅ **Deployment**: Blue-green with automated validation  

### Next Steps

To bring these concepts to your organization:

1. **Start with the Quick Start** from Part 1
2. **Implement observability** before scaling
3. **Add security and compliance** from the beginning
4. **Optimize costs** continuously
5. **Automate deployment** for reliable releases

### Resources to Continue

- [OpenAI Production Best Practices](https://platform.openai.com/docs/guides/production-best-practices)
- [Prometheus Monitoring](https://prometheus.io/docs/)
- [FastAPI Production Deployment](https://fastapi.tiangolo.com/deployment/)

---

**Did you like this series?** Share it with your team and let me know on [LinkedIn](https://linkedin.com/in/efrenbl) what other topics you'd like me to cover in future series.

> **Remember**: True innovation in AI is not in the models—it's in how you integrate them reliably, securely and cost-effectively into systems that solve real business problems.

**Thank you for joining me on this journey from prototype to production at scale!** 🚀
