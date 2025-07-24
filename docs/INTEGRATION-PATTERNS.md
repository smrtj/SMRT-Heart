# SMRT CRM Platform - Integration Patterns and Protocols

## Overview

This document defines the standardized patterns, protocols, and contracts for integrating third-party systems with the SMRT CRM Platform. These patterns ensure consistent, secure, and scalable integrations while maintaining multi-tenant isolation.

## Core Integration Architecture

### Integration Hub Pattern

```python
# Central integration coordinator
class IntegrationHub:
    """
    Central hub for managing all third-party integrations
    - Route data between platform and external systems
    - Handle authentication and authorization
    - Manage rate limiting and error handling
    - Ensure tenant isolation in all external communications
    """
    
    def __init__(self):
        self.connectors = {}  # Registry of integration connectors
        self.webhook_router = WebhookRouter()
        self.auth_manager = IntegrationAuthManager()
        self.rate_limiter = RateLimiter()
    
    async def register_integration(self, integration_config: IntegrationConfig):
        """Register a new third-party integration"""
        connector = self._create_connector(integration_config)
        self.connectors[integration_config.system_id] = connector
        
    async def sync_data(self, 
                       system_id: str, 
                       operation: str, 
                       data: dict, 
                       tenant_context: TenantContext) -> SyncResult:
        """Synchronize data with external system"""
        connector = self.connectors.get(system_id)
        if not connector:
            raise IntegrationNotFoundError(system_id)
            
        # Apply tenant-specific rate limiting
        await self.rate_limiter.check_limit(tenant_context.tenant_id, system_id)
        
        # Execute sync with proper error handling
        return await connector.sync(operation, data, tenant_context)
```

### Standard Integration Connector Interface

```python
from abc import ABC, abstractmethod
from typing import Dict, Any, Optional
from enum import Enum

class SyncDirection(Enum):
    INBOUND = "inbound"    # External system -> SMRT CRM
    OUTBOUND = "outbound"  # SMRT CRM -> External system
    BIDIRECTIONAL = "bidirectional"

class IntegrationConnector(ABC):
    """Base class for all third-party system integrations"""
    
    def __init__(self, config: IntegrationConfig):
        self.config = config
        self.auth_handler = self._setup_authentication()
        self.data_mapper = self._setup_data_mapping()
    
    @abstractmethod
    async def authenticate(self, tenant_context: TenantContext) -> AuthResult:
        """Authenticate with the external system"""
        pass
    
    @abstractmethod
    async def sync_inbound(self, data: Dict[str, Any], tenant_context: TenantContext) -> SyncResult:
        """Receive data from external system"""
        pass
    
    @abstractmethod
    async def sync_outbound(self, data: Dict[str, Any], tenant_context: TenantContext) -> SyncResult:
        """Send data to external system"""
        pass
    
    @abstractmethod
    async def validate_connection(self, tenant_context: TenantContext) -> bool:
        """Test connection to external system"""
        pass
    
    async def handle_webhook(self, payload: dict, headers: dict, tenant_context: TenantContext) -> WebhookResult:
        """Process incoming webhook from external system"""
        # Verify webhook signature
        if not await self._verify_webhook_signature(payload, headers):
            raise WebhookSecurityError("Invalid webhook signature")
        
        # Transform data and sync to platform
        transformed_data = await self.data_mapper.transform_inbound(payload)
        return await self.sync_inbound(transformed_data, tenant_context)
```

## Webhook Framework

### Webhook Router and Processing

```python
class WebhookRouter:
    """Route incoming webhooks to appropriate handlers"""
    
    def __init__(self):
        self.routes = {}  # system_id -> WebhookHandler mapping
        self.security_validator = WebhookSecurityValidator()
    
    def register_webhook_handler(self, system_id: str, handler: WebhookHandler):
        """Register webhook handler for a specific integration"""
        self.routes[system_id] = handler
    
    async def process_webhook(self, 
                            system_id: str, 
                            payload: dict, 
                            headers: dict,
                            tenant_context: TenantContext) -> WebhookResult:
        """Process incoming webhook with security validation"""
        
        # Security checks
        await self.security_validator.validate_request(system_id, payload, headers)
        
        # Route to appropriate handler
        handler = self.routes.get(system_id)
        if not handler:
            raise WebhookHandlerNotFoundError(system_id)
        
        # Process with tenant isolation
        result = await handler.process(payload, headers, tenant_context)
        
        # Log for audit trail
        await self._log_webhook_processing(system_id, result, tenant_context)
        
        return result

class WebhookHandler:
    """Base webhook handler with common functionality"""
    
    def __init__(self, system_id: str):
        self.system_id = system_id
        self.signature_validator = SignatureValidator()
        self.data_transformer = DataTransformer()
    
    async def process(self, payload: dict, headers: dict, tenant_context: TenantContext) -> WebhookResult:
        """Process webhook payload"""
        try:
            # Validate signature
            if not await self.signature_validator.validate(payload, headers, self.system_id):
                return WebhookResult(success=False, error="Invalid signature")
            
            # Transform data
            transformed_data = await self.data_transformer.transform(payload, self.system_id)
            
            # Update platform data
            result = await self._update_platform_data(transformed_data, tenant_context)
            
            return WebhookResult(success=True, data=result)
            
        except Exception as e:
            logger.error(f"Webhook processing failed for {self.system_id}", 
                        error=str(e), tenant_id=tenant_context.tenant_id)
            return WebhookResult(success=False, error=str(e))
```

### Webhook Security Framework

```python
import hmac
import hashlib
from typing import Dict, Optional

class WebhookSecurityValidator:
    """Validate webhook security across all integrations"""
    
    def __init__(self):
        self.signature_validators = {
            'github': self._validate_github_signature,
            'stripe': self._validate_stripe_signature,
            'custom': self._validate_custom_signature
        }
    
    async def validate_request(self, system_id: str, payload: dict, headers: dict) -> bool:
        """Validate webhook request security"""
        
        # Rate limiting per source IP
        client_ip = headers.get('x-forwarded-for', headers.get('remote-addr'))
        await self._check_rate_limit(client_ip, system_id)
        
        # Signature validation
        signature_type = await self._detect_signature_type(system_id, headers)
        validator = self.signature_validators.get(signature_type)
        
        if not validator:
            raise WebhookSecurityError(f"No signature validator for {signature_type}")
        
        return await validator(payload, headers, system_id)
    
    async def _validate_custom_signature(self, payload: dict, headers: dict, system_id: str) -> bool:
        """Validate custom HMAC-SHA256 signatures"""
        received_signature = headers.get('x-webhook-signature')
        if not received_signature:
            return False
        
        # Get system-specific secret from secure storage
        secret = await self._get_webhook_secret(system_id)
        
        # Calculate expected signature
        payload_str = json.dumps(payload, sort_keys=True, separators=(',', ':'))
        expected_signature = hmac.new(
            secret.encode(), 
            payload_str.encode(), 
            hashlib.sha256
        ).hexdigest()
        
        return hmac.compare_digest(f"sha256={expected_signature}", received_signature)
```

## Standard Data Mapping Framework

### Universal Data Transformer

```python
class DataTransformer:
    """Transform data between external systems and SMRT CRM format"""
    
    def __init__(self):
        self.mapping_rules = {}  # system_id -> MappingRule
        self.field_validators = FieldValidatorRegistry()
    
    async def transform_inbound(self, system_id: str, external_data: dict) -> dict:
        """Transform external system data to SMRT CRM format"""
        mapping_rule = self.mapping_rules.get(system_id)
        if not mapping_rule:
            raise MappingRuleNotFoundError(system_id)
        
        transformed = {}
        
        for smrt_field, external_field in mapping_rule.field_mappings.items():
            # Extract value using path notation (e.g., "contact.phone.number")
            value = self._extract_value(external_data, external_field)
            
            # Apply field-specific transformations
            if value is not None:
                transformer = mapping_rule.field_transformers.get(smrt_field)
                if transformer:
                    value = await transformer.transform(value)
                
                # Validate transformed value
                validator = self.field_validators.get(smrt_field)
                if validator and not await validator.validate(value):
                    logger.warning(f"Validation failed for field {smrt_field}: {value}")
                    continue
                
                transformed[smrt_field] = value
        
        return transformed
    
    async def transform_outbound(self, system_id: str, smrt_data: dict) -> dict:
        """Transform SMRT CRM data to external system format"""
        mapping_rule = self.mapping_rules.get(system_id)
        external_data = {}
        
        for smrt_field, value in smrt_data.items():
            external_field = mapping_rule.reverse_mappings.get(smrt_field)
            if external_field:
                # Apply reverse transformation
                transformer = mapping_rule.reverse_transformers.get(smrt_field)
                if transformer:
                    value = await transformer.transform(value)
                
                self._set_nested_value(external_data, external_field, value)
        
        return external_data

# Example mapping configuration
sarah_ai_mapping = MappingRule(
    system_id="sarah_ai",
    field_mappings={
        "phone_number": "call.destination_number",
        "call_duration": "call.duration_seconds", 
        "call_outcome": "call.disposition",
        "agent_name": "agent.display_name",
        "call_recording_url": "recording.download_url"
    },
    field_transformers={
        "phone_number": PhoneNumberTransformer(),  # Normalize to E.164
        "call_duration": DurationTransformer(),    # Convert to seconds
        "call_outcome": OutcomeTransformer()       # Map disposition codes
    }
)
```

### Phone Number Standardization

```python
class PhoneNumberTransformer:
    """Standardize phone numbers across all integrations"""
    
    def __init__(self):
        self.country_codes = CountryCodeRegistry()
        self.formatter = PhoneNumberFormatter()
    
    async def transform(self, raw_phone: str) -> str:
        """Transform any phone number format to E.164 standard"""
        # Remove all non-digit characters
        digits_only = re.sub(r'\D', '', raw_phone)
        
        # Handle different input formats
        if len(digits_only) == 10:
            # Assume US/Canada
            return f"+1{digits_only}"
        elif len(digits_only) == 11 and digits_only.startswith('1'):
            # US/Canada with country code
            return f"+{digits_only}"
        elif digits_only.startswith('0'):
            # Remove leading zero for international formatting
            return f"+{digits_only[1:]}"
        else:
            # Already international format
            return f"+{digits_only}"
    
    async def validate_e164(self, phone_number: str) -> bool:
        """Validate E.164 format"""
        pattern = r'^\+[1-9]\d{1,14}$'
        return bool(re.match(pattern, phone_number))
```

## Integration-Specific Patterns

### Sarah AI Integration

```python
class SarahAIConnector(IntegrationConnector):
    """Integration with Sarah AI calling system"""
    
    def __init__(self, config: IntegrationConfig):
        super().__init__(config)
        self.webhook_events = {
            'call.started': self._handle_call_started,
            'call.ended': self._handle_call_ended,
            'call.transferred': self._handle_call_transferred,
            'recording.available': self._handle_recording_available
        }
    
    async def sync_inbound(self, data: dict, tenant_context: TenantContext) -> SyncResult:
        """Process Sarah AI event data"""
        event_type = data.get('event_type')
        handler = self.webhook_events.get(event_type)
        
        if not handler:
            return SyncResult(success=False, error=f"Unknown event type: {event_type}")
        
        return await handler(data, tenant_context)
    
    async def _handle_call_ended(self, data: dict, tenant_context: TenantContext) -> SyncResult:
        """Process completed call and update CRM records"""
        phone_number = data['call']['destination_number']
        
        # Find or create phone number record
        phone_record = await self._find_or_create_phone_number(phone_number, tenant_context)
        
        # Create interaction record
        interaction = {
            'phone_number_id': phone_record.id,
            'interaction_type': 'outbound_call',
            'duration_seconds': data['call']['duration'],
            'outcome': self._map_disposition(data['call']['disposition']),
            'agent_name': data['agent']['name'],
            'tenant_id': tenant_context.tenant_id,
            'metadata': {
                'sarah_call_id': data['call']['id'],
                'call_quality_score': data['call'].get('quality_score'),
                'recording_url': data.get('recording', {}).get('url')
            }
        }
        
        interaction_id = await self._create_interaction(interaction)
        
        # Update phone number statistics
        await self._update_phone_statistics(phone_record.id, data['call'])
        
        return SyncResult(success=True, data={'interaction_id': interaction_id})
```

### Payment Processing Integration

```python
class PaymentProcessorConnector(IntegrationConnector):
    """Integration with payment processing systems"""
    
    async def sync_inbound(self, data: dict, tenant_context: TenantContext) -> SyncResult:
        """Process payment events and update merchant records"""
        event_type = data.get('type')
        
        if event_type == 'payment.succeeded':
            return await self._handle_payment_success(data, tenant_context)
        elif event_type == 'payment.failed':
            return await self._handle_payment_failure(data, tenant_context)
        elif event_type == 'merchant.updated':
            return await self._handle_merchant_update(data, tenant_context)
        
        return SyncResult(success=False, error=f"Unhandled event type: {event_type}")
    
    async def _handle_payment_success(self, data: dict, tenant_context: TenantContext) -> SyncResult:
        """Process successful payment and link to customer record"""
        payment_data = data['data']['object']
        
        # Extract customer phone from payment metadata
        customer_phone = payment_data.get('metadata', {}).get('customer_phone')
        if not customer_phone:
            return SyncResult(success=False, error="No customer phone in payment data")
        
        # Find existing phone number record
        phone_record = await self._find_phone_number(customer_phone, tenant_context)
        if not phone_record:
            return SyncResult(success=False, error="Customer phone not found in CRM")
        
        # Create payment interaction
        interaction = {
            'phone_number_id': phone_record.id,
            'interaction_type': 'payment_processed',
            'outcome': 'payment_successful',
            'tenant_id': tenant_context.tenant_id,
            'metadata': {
                'payment_id': payment_data['id'],
                'amount': payment_data['amount'],
                'currency': payment_data['currency'],
                'processor': 'card_connect'
            }
        }
        
        await self._create_interaction(interaction)
        
        # Update customer payment history
        await self._update_payment_history(phone_record.id, payment_data)
        
        return SyncResult(success=True)
```

### CRM System Integration

```python
class CRMSystemConnector(IntegrationConnector):
    """Bidirectional integration with external CRM systems"""
    
    async def sync_outbound(self, data: dict, tenant_context: TenantContext) -> SyncResult:
        """Export SMRT CRM data to external CRM"""
        entity_type = data.get('entity_type')
        
        if entity_type == 'contact':
            return await self._sync_contact_outbound(data, tenant_context)
        elif entity_type == 'company':
            return await self._sync_company_outbound(data, tenant_context)
        elif entity_type == 'interaction':
            return await self._sync_interaction_outbound(data, tenant_context)
        
        return SyncResult(success=False, error=f"Unsupported entity type: {entity_type}")
    
    async def _sync_contact_outbound(self, data: dict, tenant_context: TenantContext) -> SyncResult:
        """Export contact data to external CRM"""
        contact_data = {
            'first_name': data.get('first_name'),
            'last_name': data.get('last_name'),
            'phone': data.get('phone_number'),
            'email': data.get('email'),
            'company': data.get('company_name'),
            'custom_fields': {
                'smrt_crm_id': data['id'],
                'last_call_date': data.get('last_contact_date'),
                'call_success_rate': data.get('success_rate', 0)
            }
        }
        
        # Send to external CRM API
        response = await self._make_api_request(
            method='POST',
            endpoint='/contacts',
            payload=contact_data,
            tenant_context=tenant_context
        )
        
        if response.success:
            # Store external ID for future sync
            await self._store_external_mapping(
                smrt_id=data['id'],
                external_id=response.data['id'],
                entity_type='contact',
                tenant_context=tenant_context
            )
        
        return response
```

## Webhook Event Standards

### Standard Webhook Payload Format

```json
{
    "event_id": "evt_abc123def456",
    "event_type": "phone_number.interaction.created",
    "timestamp": "2024-01-15T10:30:00Z",
    "source_system": "smrt_crm",
    "tenant_id": "tenant_abc123",
    "data": {
        "phone_number": "+1234567890",
        "interaction": {
            "id": "int_789xyz012",
            "type": "outbound_call",
            "outcome": "interested",
            "duration_seconds": 180,
            "agent_name": "John Smith",
            "created_at": "2024-01-15T10:27:00Z"
        }
    },
    "metadata": {
        "retry_count": 0,
        "signature": "sha256=abc123def456...",
        "version": "1.0"
    }
}
```

### Webhook Subscription Management

```python
class WebhookSubscriptionManager:
    """Manage webhook subscriptions for tenant integrations"""
    
    async def create_subscription(self, 
                                subscription: WebhookSubscription, 
                                tenant_context: TenantContext) -> str:
        """Create new webhook subscription"""
        subscription_id = str(uuid4())
        
        subscription_record = {
            'id': subscription_id,
            'tenant_id': tenant_context.tenant_id,
            'endpoint_url': subscription.endpoint_url,
            'event_types': subscription.event_types,
            'secret': await self._generate_webhook_secret(),
            'is_active': True,
            'created_at': datetime.utcnow(),
            'retry_policy': subscription.retry_policy
        }
        
        await self._store_subscription(subscription_record)
        
        # Test webhook endpoint
        test_result = await self._test_webhook_endpoint(subscription.endpoint_url)
        if not test_result.success:
            logger.warning(f"Webhook endpoint test failed: {test_result.error}")
        
        return subscription_id
    
    async def deliver_webhook(self, event: WebhookEvent, tenant_id: str):
        """Deliver webhook event to all subscribed endpoints"""
        subscriptions = await self._get_active_subscriptions(tenant_id, event.event_type)
        
        for subscription in subscriptions:
            try:
                await self._deliver_to_endpoint(event, subscription)
            except Exception as e:
                await self._handle_delivery_failure(event, subscription, e)
    
    async def _deliver_to_endpoint(self, event: WebhookEvent, subscription: dict):
        """Deliver single webhook with retry logic"""
        payload = {
            'event_id': event.id,
            'event_type': event.event_type,
            'timestamp': event.timestamp.isoformat(),
            'data': event.data,
            'metadata': {
                'retry_count': event.retry_count,
                'signature': self._generate_signature(event.data, subscription['secret'])
            }
        }
        
        headers = {
            'Content-Type': 'application/json',
            'X-Webhook-Signature': payload['metadata']['signature'],
            'X-Webhook-Event': event.event_type,
            'User-Agent': 'SMRT-CRM-Webhooks/1.0'
        }
        
        async with httpx.AsyncClient(timeout=30.0) as client:
            response = await client.post(
                subscription['endpoint_url'],
                json=payload,
                headers=headers
            )
            
            if response.status_code not in [200, 201, 204]:
                raise WebhookDeliveryError(f"HTTP {response.status_code}: {response.text}")
```

## Rate Limiting and Error Handling

### Integration Rate Limiting

```python
class IntegrationRateLimiter:
    """Rate limiting for third-party integrations per tenant"""
    
    def __init__(self):
        self.redis_client = redis.Redis()
        self.rate_limits = {
            'sarah_ai': {'calls_per_minute': 60, 'daily_limit': 10000},
            'payment_processor': {'calls_per_minute': 30, 'daily_limit': 5000},
            'external_crm': {'calls_per_minute': 100, 'daily_limit': 50000}
        }
    
    async def check_limit(self, tenant_id: str, system_id: str) -> bool:
        """Check if tenant has exceeded rate limits for system"""
        limits = self.rate_limits.get(system_id, {})
        
        # Check per-minute limit
        minute_key = f"rate_limit:{tenant_id}:{system_id}:minute:{int(time.time() / 60)}"
        minute_count = await self.redis_client.incr(minute_key)
        await self.redis_client.expire(minute_key, 60)
        
        if minute_count > limits.get('calls_per_minute', 1000):
            raise RateLimitExceededError(f"Per-minute limit exceeded for {system_id}")
        
        # Check daily limit
        day_key = f"rate_limit:{tenant_id}:{system_id}:day:{date.today().isoformat()}"
        day_count = await self.redis_client.incr(day_key)
        await self.redis_client.expire(day_key, 86400)
        
        if day_count > limits.get('daily_limit', 100000):
            raise RateLimitExceededError(f"Daily limit exceeded for {system_id}")
        
        return True
```

### Error Handling and Retry Logic

```python
class IntegrationErrorHandler:
    """Handle errors and implement retry logic for integrations"""
    
    def __init__(self):
        self.retry_policies = {
            'sarah_ai': RetryPolicy(max_attempts=3, backoff_factor=2),
            'payment_processor': RetryPolicy(max_attempts=5, backoff_factor=1.5),
            'external_crm': RetryPolicy(max_attempts=2, backoff_factor=3)
        }
    
    async def handle_integration_error(self, 
                                     error: Exception, 
                                     system_id: str, 
                                     operation_data: dict,
                                     tenant_context: TenantContext) -> ErrorResult:
        """Handle integration errors with appropriate retry logic"""
        
        retry_policy = self.retry_policies.get(system_id)
        if not retry_policy:
            return ErrorResult(should_retry=False, error=str(error))
        
        # Determine if error is retryable
        if isinstance(error, (TimeoutError, ConnectionError, httpx.RequestError)):
            should_retry = True
        elif isinstance(error, (AuthenticationError, PermissionError)):
            should_retry = False
        elif isinstance(error, httpx.HTTPStatusError):
            # Retry on 5xx errors, not on 4xx
            should_retry = 500 <= error.response.status_code < 600
        else:
            should_retry = False
        
        if should_retry and operation_data.get('retry_count', 0) < retry_policy.max_attempts:
            # Schedule retry with exponential backoff
            delay = retry_policy.calculate_delay(operation_data.get('retry_count', 0))
            await self._schedule_retry(operation_data, delay, tenant_context)
            
            return ErrorResult(should_retry=True, retry_delay=delay)
        
        # Log permanent failure
        await self._log_permanent_failure(error, system_id, operation_data, tenant_context)
        
        return ErrorResult(should_retry=False, error=str(error))
```

This integration framework provides a robust, secure, and scalable foundation for connecting the SMRT CRM Platform with any third-party system while maintaining strict multi-tenant isolation and data governance.