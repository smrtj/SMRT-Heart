# SMRT CRM Platform - Development Standards

## Overview

This document establishes the coding conventions, testing requirements, documentation standards, and quality gates that ensure consistent, maintainable, and scalable development across all modules of the SMRT CRM Platform.

## Code Style and Conventions

### Python Code Standards

#### General Principles
- Follow PEP 8 with specific adaptations for our multi-tenant architecture
- Use type hints for all function parameters and return values
- Prefer composition over inheritance
- Write self-documenting code with clear variable and function names

#### Naming Conventions
```python
# Variables and functions: snake_case
phone_number_id = "abc-123-def"
def get_phone_number_by_id(phone_id: str) -> PhoneNumber:
    pass

# Classes: PascalCase
class PhoneNumberService:
    pass

# Constants: UPPER_SNAKE_CASE
DATABASE_CONNECTION_TIMEOUT = 30
DEFAULT_TENANT_PERMISSIONS = ["read", "write"]

# Private methods and attributes: leading underscore
class TenantManager:
    def __init__(self):
        self._cached_permissions = {}
    
    def _load_permissions_from_db(self) -> dict:
        pass

# Multi-tenant specific: include tenant context
def get_phone_numbers_for_tenant(tenant_id: UUID) -> List[PhoneNumber]:
    pass

def create_human_with_tenant_isolation(human_data: dict, tenant_context: TenantContext) -> Human:
    pass
```

#### File Organization
```
module_name/
├── __init__.py
├── models/          # Pydantic models and database schemas
│   ├── __init__.py
│   ├── phone_number.py
│   └── human.py
├── services/        # Business logic layer
│   ├── __init__.py
│   ├── phone_service.py
│   └── human_service.py
├── api/            # FastAPI route handlers
│   ├── __init__.py
│   ├── phone_routes.py
│   └── human_routes.py
├── database/       # Database access layer
│   ├── __init__.py
│   ├── repositories.py
│   └── migrations/
├── utils/          # Utility functions
│   ├── __init__.py
│   ├── validators.py
│   └── formatters.py
├── tests/          # Test files mirror source structure
│   ├── test_models/
│   ├── test_services/
│   └── test_api/
└── config.py       # Module configuration
```

#### Error Handling
```python
# Use custom exception hierarchy
class SMRTCRMException(Exception):
    """Base exception for SMRT CRM Platform"""
    pass

class TenantPermissionError(SMRTCRMException):
    """Raised when tenant lacks required permissions"""
    def __init__(self, tenant_id: str, required_permission: str):
        super().__init__(f"Tenant {tenant_id} lacks permission: {required_permission}")

class PhoneNumberNotFoundError(SMRTCRMException):
    """Raised when phone number doesn't exist or isn't accessible"""
    pass

# Always include tenant context in error messages
def get_phone_number(phone_id: UUID, tenant_context: TenantContext) -> PhoneNumber:
    try:
        phone = repository.get_by_id(phone_id)
        if not has_permission(tenant_context, phone, 'read'):
            raise TenantPermissionError(tenant_context.tenant_id, 'read_phone_number')
        return phone
    except DatabaseError as e:
        logger.error(f"Database error for tenant {tenant_context.tenant_id}: {e}")
        raise SMRTCRMException("Unable to retrieve phone number") from e
```

#### Logging Standards
```python
import structlog

# Use structured logging with tenant context
logger = structlog.get_logger()

def create_phone_number(data: dict, tenant_context: TenantContext) -> PhoneNumber:
    logger.info(
        "Creating phone number",
        tenant_id=tenant_context.tenant_id,
        user_id=tenant_context.user_id,
        phone_number=data.get('phone_number', 'REDACTED'),
        operation="create_phone_number"
    )
    
    try:
        # Implementation
        result = service.create(data)
        
        logger.info(
            "Phone number created successfully",
            tenant_id=tenant_context.tenant_id,
            phone_number_id=result.id,
            operation="create_phone_number"
        )
        
        return result
    except Exception as e:
        logger.error(
            "Failed to create phone number",
            tenant_id=tenant_context.tenant_id,
            error=str(e),
            operation="create_phone_number"
        )
        raise
```

### Database Standards

#### Schema Design Principles
```sql
-- Use UUIDs for all primary keys
CREATE TABLE phone_numbers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- ... other fields
);

-- Always include tenant tracking
CREATE TABLE humans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- ... business fields
    created_by_tenant_id UUID NOT NULL,
    updated_by_tenant_id UUID NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Use consistent naming
-- Tables: plural nouns (phone_numbers, humans, interactions)
-- Columns: snake_case (phone_number, first_name, created_at)
-- Foreign keys: singular_table_id (tenant_id, phone_number_id)
-- Indexes: idx_table_column (idx_phone_numbers_tenant_id)
```

#### Migration Standards
```python
# migrations/001_create_phone_numbers.py
from alembic import op
import sqlalchemy as sa

def upgrade():
    """Create phone_numbers table with proper indexes and constraints"""
    op.create_table(
        'phone_numbers',
        sa.Column('id', sa.UUID(), nullable=False, default=sa.text('gen_random_uuid()')),
        sa.Column('phone_number', sa.String(20), nullable=False),
        sa.Column('created_by_tenant_id', sa.UUID(), nullable=False),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('phone_number', name='uq_phone_numbers_number')
    )
    
    # Create performance indexes
    op.create_index('idx_phone_numbers_tenant', 'phone_numbers', ['created_by_tenant_id'])
    
    # Enable row-level security
    op.execute("ALTER TABLE phone_numbers ENABLE ROW LEVEL SECURITY")

def downgrade():
    """Drop phone_numbers table and all related objects"""
    op.drop_table('phone_numbers')
```

### API Design Standards

#### FastAPI Route Structure
```python
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel
from typing import List, Optional
from uuid import UUID

router = APIRouter(prefix="/api/v1/phone-numbers", tags=["phone-numbers"])

# Request/Response models
class PhoneNumberCreate(BaseModel):
    phone_number: str
    number_type: Optional[str] = None
    notes: Optional[str] = None

class PhoneNumberResponse(BaseModel):
    id: UUID
    phone_number: str
    number_type: str
    created_at: datetime
    # Exclude sensitive fields from response

# Route handlers with proper error handling
@router.post("/", response_model=PhoneNumberResponse, status_code=201)
async def create_phone_number(
    data: PhoneNumberCreate,
    tenant_context: TenantContext = Depends(get_tenant_context)
) -> PhoneNumberResponse:
    """
    Create a new phone number record.
    
    Requires: create_phone_number permission
    """
    try:
        phone_number = await phone_service.create(data.dict(), tenant_context)
        return PhoneNumberResponse.from_orm(phone_number)
    except TenantPermissionError as e:
        raise HTTPException(status_code=403, detail=str(e))
    except ValidationError as e:
        raise HTTPException(status_code=400, detail=str(e))

# Consistent endpoint patterns
# GET /api/v1/phone-numbers - List with pagination
# GET /api/v1/phone-numbers/{id} - Get specific record
# POST /api/v1/phone-numbers - Create new record
# PUT /api/v1/phone-numbers/{id} - Update entire record
# PATCH /api/v1/phone-numbers/{id} - Partial update
# DELETE /api/v1/phone-numbers/{id} - Soft delete
```

#### Response Format Standards
```python
# Standard success response
{
    "success": true,
    "data": { ... },
    "metadata": {
        "total_count": 150,
        "page": 1,
        "page_size": 20,
        "tenant_id": "abc-123-def"
    }
}

# Standard error response
{
    "success": false,
    "error": {
        "code": "TENANT_PERMISSION_ERROR",
        "message": "Insufficient permissions to access phone number",
        "details": {
            "required_permission": "read_phone_number",
            "tenant_id": "abc-123-def"
        }
    },
    "metadata": {
        "request_id": "req-789-xyz",
        "timestamp": "2024-01-15T10:30:00Z"
    }
}
```

## Testing Standards

### Test Organization
```python
# tests/test_services/test_phone_service.py
import pytest
from unittest.mock import Mock, patch
from uuid import uuid4
from smrt_crm.services.phone_service import PhoneService
from smrt_crm.models.tenant import TenantContext

class TestPhoneService:
    """Test suite for PhoneService with multi-tenant scenarios"""
    
    @pytest.fixture
    def tenant_context(self):
        """Standard tenant context for testing"""
        return TenantContext(
            tenant_id=uuid4(),
            user_id=uuid4(),
            permissions=['read_phone_number', 'create_phone_number']
        )
    
    @pytest.fixture
    def phone_service(self):
        """Phone service with mocked dependencies"""
        return PhoneService(repository=Mock(), permission_service=Mock())
    
    def test_create_phone_number_success(self, phone_service, tenant_context):
        """Test successful phone number creation"""
        # Arrange
        phone_data = {"phone_number": "+1234567890", "number_type": "mobile"}
        expected_result = Mock(id=uuid4(), phone_number="+1234567890")
        phone_service.repository.create.return_value = expected_result
        
        # Act
        result = phone_service.create(phone_data, tenant_context)
        
        # Assert
        assert result == expected_result
        phone_service.repository.create.assert_called_once()
    
    def test_create_phone_number_permission_denied(self, phone_service, tenant_context):
        """Test phone number creation with insufficient permissions"""
        # Arrange
        tenant_context.permissions = ['read_phone_number']  # No create permission
        phone_data = {"phone_number": "+1234567890"}
        
        # Act & Assert
        with pytest.raises(TenantPermissionError):
            phone_service.create(phone_data, tenant_context)
    
    @pytest.mark.asyncio
    async def test_integration_with_database(self, test_database):
        """Integration test with real database"""
        # Test with actual database connection
        pass
```

### Test Coverage Requirements
- **Unit Tests**: 90% line coverage minimum
- **Integration Tests**: All API endpoints and database operations
- **E2E Tests**: Critical user workflows across module boundaries
- **Performance Tests**: Load testing for multi-tenant scenarios

### Testing Tools
```python
# conftest.py - Shared test configuration
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from fastapi.testclient import TestClient

@pytest.fixture(scope="session")
def test_database():
    """Create test database with proper isolation"""
    engine = create_engine("postgresql://test:test@localhost/test_smrt_crm")
    # Run migrations
    # Return configured session

@pytest.fixture
def api_client():
    """FastAPI test client with authentication"""
    from smrt_crm.main import app
    return TestClient(app)

@pytest.fixture
def authenticated_headers():
    """Generate JWT token for API testing"""
    return {"Authorization": "Bearer test-jwt-token"}
```

## Documentation Standards

### Code Documentation
```python
def get_phone_numbers_for_tenant(
    tenant_id: UUID, 
    filters: Optional[Dict] = None,
    page: int = 1,
    page_size: int = 20
) -> PaginatedResponse[PhoneNumber]:
    """
    Retrieve phone numbers accessible to a specific tenant.
    
    This function applies tenant isolation rules and dynamic permissions
    to return only phone numbers the tenant is authorized to access.
    
    Args:
        tenant_id: UUID of the requesting tenant
        filters: Optional dict of filter conditions:
            - area_code: Filter by area code (str)
            - number_type: Filter by type ('mobile', 'landline', etc.)
            - dnc_status: Filter by DNC status
        page: Page number for pagination (1-based)
        page_size: Number of records per page (max 100)
    
    Returns:
        PaginatedResponse containing:
            - data: List of PhoneNumber objects
            - total_count: Total matching records
            - page: Current page number
            - page_size: Records per page
    
    Raises:
        TenantPermissionError: If tenant lacks read_phone_number permission
        ValidationError: If filters contain invalid values
        
    Example:
        >>> tenant_id = UUID("abc-123-def")
        >>> filters = {"area_code": "555", "number_type": "mobile"}
        >>> result = get_phone_numbers_for_tenant(tenant_id, filters, page=1)
        >>> print(f"Found {result.total_count} phone numbers")
    """
```

### API Documentation
```python
# Use FastAPI's automatic OpenAPI generation with detailed descriptions
@router.get(
    "/phone-numbers/{phone_id}/relationships",
    response_model=List[PhoneHumanLinkResponse],
    summary="Get phone number relationships",
    description="""
    Retrieve all human relationships linked to a specific phone number.
    
    This endpoint returns relationships the requesting tenant has permission
    to view, which may be a subset of all relationships for the phone number.
    
    **Required Permissions**: `read_phone_number`, `read_phone_human_links`
    
    **Multi-Tenant Behavior**:
    - Returns only relationships visible to the requesting tenant
    - Respects data sharing agreements between tenants
    - Includes relationship metadata (confidence scores, verification status)
    
    **Rate Limiting**: 100 requests per minute per tenant
    """,
    responses={
        200: {"description": "List of phone-human relationships"},
        403: {"description": "Insufficient permissions"},
        404: {"description": "Phone number not found or not accessible"},
        429: {"description": "Rate limit exceeded"}
    }
)
```

### Module Documentation Structure
```markdown
# Module Name

## Overview
Brief description of module purpose and responsibilities

## Architecture
- Dependencies on other modules
- Integration points
- Data flow diagrams

## API Reference
- Endpoint documentation
- Request/response examples
- Error codes and handling

## Database Schema
- Tables and relationships
- Indexes and constraints
- Migration procedures

## Configuration
- Environment variables
- Configuration files
- Default values

## Development
- Setup instructions
- Testing procedures
- Deployment requirements

## Examples
- Common use cases
- Code samples
- Integration examples
```

## Security Standards

### Authentication and Authorization
```python
# Multi-tenant JWT token structure
{
    "sub": "user@example.com",
    "tenant_id": "abc-123-def",
    "permissions": ["read_phone_number", "create_human"],
    "role": "admin",
    "exp": 1640995200,
    "iat": 1640908800
}

# Permission checking decorator
def require_permission(permission: str):
    def decorator(func):
        async def wrapper(*args, **kwargs):
            tenant_context = kwargs.get('tenant_context')
            if not tenant_context or permission not in tenant_context.permissions:
                raise TenantPermissionError(tenant_context.tenant_id, permission)
            return await func(*args, **kwargs)
        return wrapper
    return decorator

@require_permission('create_phone_number')
async def create_phone_number(data: dict, tenant_context: TenantContext):
    pass
```

### Data Protection
```python
# Encrypt sensitive fields
from cryptography.fernet import Fernet

class EncryptedField:
    def __init__(self, encryption_key: bytes):
        self.cipher = Fernet(encryption_key)
    
    def encrypt(self, value: str) -> str:
        return self.cipher.encrypt(value.encode()).decode()
    
    def decrypt(self, encrypted_value: str) -> str:
        return self.cipher.decrypt(encrypted_value.encode()).decode()

# Always hash PII for matching, never store plain text
import hashlib

def hash_ssn_for_matching(ssn: str) -> str:
    """Create consistent hash for SSN matching without storing plain text"""
    return hashlib.sha256(f"ssn:{ssn}:salt".encode()).hexdigest()
```

### Input Validation
```python
from pydantic import BaseModel, validator
import re

class PhoneNumberCreate(BaseModel):
    phone_number: str
    number_type: Optional[str] = None
    
    @validator('phone_number')
    def validate_phone_number(cls, v):
        # Remove all non-digit characters
        cleaned = re.sub(r'\D', '', v)
        
        # Must be 10-15 digits (international format)
        if not 10 <= len(cleaned) <= 15:
            raise ValueError('Phone number must be 10-15 digits')
        
        # Convert to E.164 format
        if len(cleaned) == 10:
            cleaned = f"+1{cleaned}"  # Assume US if 10 digits
        elif not cleaned.startswith('+'):
            cleaned = f"+{cleaned}"
            
        return cleaned
    
    @validator('number_type')
    def validate_number_type(cls, v):
        if v and v not in ['mobile', 'landline', 'voip', 'toll_free']:
            raise ValueError('Invalid number type')
        return v
```

## Performance Standards

### Database Query Optimization
```python
# Use appropriate indexes for multi-tenant queries
async def get_phone_numbers_with_humans(tenant_id: UUID) -> List[dict]:
    """Optimized query for phone numbers with human relationships"""
    query = """
    SELECT 
        pn.id, pn.phone_number, pn.number_type,
        h.id as human_id, h.full_name, h.company_name
    FROM phone_numbers pn
    LEFT JOIN phone_human_links phl ON pn.id = phl.phone_number_id 
        AND phl.is_active = true
    LEFT JOIN humans h ON phl.human_id = h.id 
        AND h.is_deleted = false
    WHERE pn.created_by_tenant_id = %s 
        AND pn.is_deleted = false
    ORDER BY pn.created_at DESC
    LIMIT 100
    """
    # This query uses indexes: idx_phone_numbers_tenant, idx_phone_human_links_phone
    return await database.fetch_all(query, [tenant_id])
```

### Caching Strategy
```python
import redis
from functools import wraps

redis_client = redis.Redis(host='localhost', port=6379, db=0)

def cache_with_tenant_isolation(expiry_seconds: int = 300):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            tenant_context = kwargs.get('tenant_context')
            cache_key = f"tenant:{tenant_context.tenant_id}:func:{func.__name__}:{hash(str(args))}"
            
            # Try cache first
            cached_result = redis_client.get(cache_key)
            if cached_result:
                return json.loads(cached_result)
            
            # Execute function and cache result
            result = await func(*args, **kwargs)
            redis_client.setex(cache_key, expiry_seconds, json.dumps(result, default=str))
            
            return result
        return wrapper
    return decorator

@cache_with_tenant_isolation(expiry_seconds=600)
async def get_tenant_phone_count(tenant_context: TenantContext) -> int:
    """Cached phone number count per tenant"""
    pass
```

## Quality Gates

### Pre-Commit Hooks
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/psf/black
    rev: 22.3.0
    hooks:
      - id: black
        args: [--line-length=100]
  
  - repo: https://github.com/pycqa/isort
    rev: 5.10.1
    hooks:
      - id: isort
        args: [--profile=black, --line-length=100]
  
  - repo: https://github.com/pycqa/flake8
    rev: 4.0.1
    hooks:
      - id: flake8
        args: [--max-line-length=100, --extend-ignore=E203]
  
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v0.950
    hooks:
      - id: mypy
        args: [--strict]
  
  - repo: local
    hooks:
      - id: security-scan
        name: Security scan
        entry: bandit
        language: system
        args: [-r, ., -f, json]
        types: [python]
```

### Continuous Integration
```yaml
# .github/workflows/quality-check.yml
name: Quality Check
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v2
      
      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: 3.9
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt
      
      - name: Run tests with coverage
        run: |
          pytest --cov=smrt_crm --cov-report=xml --cov-fail-under=90
      
      - name: Type checking
        run: mypy smrt_crm/
      
      - name: Security scan
        run: bandit -r smrt_crm/
      
      - name: Upload coverage
        uses: codecov/codecov-action@v2
```

### Performance Benchmarks
```python
# tests/benchmarks/test_performance.py
import pytest
import time
from concurrent.futures import ThreadPoolExecutor

class TestPerformanceBenchmarks:
    
    def test_api_response_time(self, api_client):
        """API endpoints must respond within 500ms"""
        start_time = time.time()
        response = api_client.get("/api/v1/phone-numbers?limit=20")
        response_time = time.time() - start_time
        
        assert response.status_code == 200
        assert response_time < 0.5  # 500ms max
    
    def test_concurrent_requests(self, api_client):
        """System must handle 100 concurrent requests"""
        def make_request():
            return api_client.get("/api/v1/health")
        
        with ThreadPoolExecutor(max_workers=100) as executor:
            futures = [executor.submit(make_request) for _ in range(100)]
            results = [f.result() for f in futures]
        
        # All requests should succeed
        assert all(r.status_code == 200 for r in results)
```

These standards ensure consistent, secure, and maintainable code across all modules of the SMRT CRM Platform while supporting the unique requirements of multi-tenant architecture.