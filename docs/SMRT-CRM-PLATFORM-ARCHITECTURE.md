# SMRT CRM Platform - Technical Architecture

## Executive Summary

The SMRT CRM Platform is a multi-tenant SaaS system built around a revolutionary two-core data model: **Phone Numbers** and **Humans** as independent but linkable entities. This architecture enables unprecedented data intelligence sharing while maintaining strict tenant isolation and dynamic access controls.

## Core Data Architecture

### The Two-Core Model

#### Phone Numbers Entity
```sql
phone_numbers (
    id UUID PRIMARY KEY,
    phone_number VARCHAR(20) UNIQUE NOT NULL, -- E.164 format
    country_code VARCHAR(5),
    area_code VARCHAR(10),
    is_mobile BOOLEAN,
    carrier_info JSONB,
    dnc_status VARCHAR(20) DEFAULT 'unknown',
    dnc_date TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    created_by_tenant_id UUID,
    metadata JSONB
);
```

#### Humans Entity  
```sql
humans (
    id UUID PRIMARY KEY,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email VARCHAR(255),
    company_name VARCHAR(255),
    job_title VARCHAR(100),
    employee_id VARCHAR(50),
    contractor_id VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    created_by_tenant_id UUID,
    metadata JSONB
);
```

#### Phone-Human Relationships
```sql
phone_human_links (
    id UUID PRIMARY KEY,
    phone_number_id UUID REFERENCES phone_numbers(id),
    human_id UUID REFERENCES humans(id),
    relationship_type VARCHAR(50), -- 'personal', 'work', 'customer_service', etc.
    is_primary BOOLEAN DEFAULT FALSE,
    verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    created_by_tenant_id UUID,
    metadata JSONB
);
```

### Multi-Tenant Data Isolation

#### Tenant Management
```sql
tenants (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    subscription_tier VARCHAR(50),
    enabled_modules TEXT[], -- ['crm', 'dialer', 'analytics', 'integrations']
    created_at TIMESTAMP DEFAULT NOW(),
    settings JSONB
);
```

#### Dynamic Access Control
```sql
data_access_permissions (
    id UUID PRIMARY KEY,
    tenant_id UUID REFERENCES tenants(id),
    entity_type VARCHAR(50), -- 'phone_number', 'human', 'interaction'
    entity_id UUID,
    field_name VARCHAR(100), -- null means entire record
    access_level VARCHAR(20), -- 'none', 'read', 'write', 'admin'
    granted_by_tenant_id UUID,
    expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);
```

#### Consent and Compliance Tracking
```sql
consent_records (
    id UUID PRIMARY KEY,
    phone_number_id UUID REFERENCES phone_numbers(id),
    human_id UUID REFERENCES humans(id),
    consent_type VARCHAR(50), -- 'tcpa_call', 'marketing_sms', 'data_sharing'
    consent_status VARCHAR(20), -- 'granted', 'revoked', 'expired'
    consent_source VARCHAR(100), -- 'web_form', 'phone_call', 'email'
    consent_date TIMESTAMP NOT NULL,
    expiration_date TIMESTAMP,
    tenant_id UUID REFERENCES tenants(id),
    legal_basis TEXT,
    metadata JSONB
);
```

## Platform Architecture

### API Layer Architecture

#### Core API Structure
```
api/
├── auth/           # Authentication and authorization
├── phone_numbers/  # Phone number CRUD and search
├── humans/         # Human entity management  
├── relationships/  # Phone-human linking
├── tenants/        # Multi-tenant management
├── permissions/    # Dynamic access control
├── integrations/   # Third-party connectors
├── webhooks/       # Real-time event notifications
├── analytics/      # Business intelligence queries
└── compliance/     # Consent and regulatory tools
```

#### Authentication Flow
```python
# JWT-based multi-tenant authentication
class TenantAuthentication:
    def authenticate(self, token: str) -> TenantUser:
        # Verify JWT signature
        # Extract tenant context
        # Load user permissions
        # Return authenticated tenant user
        
    def authorize(self, user: TenantUser, resource: str, action: str) -> bool:
        # Check tenant subscription includes required module
        # Verify user role allows action
        # Check dynamic data access permissions
        # Return authorization decision
```

#### Data Access Layer
```python
class DataAccessController:
    def filter_for_tenant(self, query: SQLQuery, tenant: Tenant) -> SQLQuery:
        # Add tenant isolation filters
        # Apply dynamic permission constraints
        # Respect data sharing agreements
        # Return filtered query
        
    def audit_access(self, tenant: Tenant, resource: dict, action: str):
        # Log all data access for compliance
        # Track usage for billing
        # Monitor for suspicious activity
```

### Integration Architecture

#### Webhook System
```python
# Standardized webhook events
class WebhookEvents:
    PHONE_NUMBER_CREATED = "phone_number.created"
    HUMAN_UPDATED = "human.updated"
    RELATIONSHIP_LINKED = "relationship.linked"
    CONSENT_GRANTED = "consent.granted"
    CALL_COMPLETED = "call.completed"
    
    def publish(self, event: str, data: dict, tenant_filter: List[UUID]):
        # Send to all subscribed tenants
        # Respect tenant access permissions
        # Handle delivery failures with retry
```

#### Third-Party Integration Framework
```python
class IntegrationConnector:
    def sync_inbound(self, system: str, data: dict) -> bool:
        # Validate incoming data
        # Map to platform schema
        # Apply tenant attribution
        # Update core entities
        
    def sync_outbound(self, entity: dict, target_systems: List[str]) -> bool:
        # Check tenant permissions for each system
        # Transform data for target format
        # Send with proper authentication
        # Track sync status
```

## Module Architecture

### Module 1: Frontend Applications
**Repository**: `frontend/`
**Technologies**: React/Vue/Svelte + TypeScript + Tailwind CSS
**Responsibilities**:
- Tenant-specific user interfaces
- Real-time dashboard updates
- Mobile-responsive design
- Integration with core API

### Module 2: Core API (Primary Focus)
**Repository**: `api-core/`
**Technologies**: FastAPI + PostgreSQL + psycopg2
**Responsibilities**:
- Central business logic
- Data access layer
- Authentication and authorization
- Multi-tenant isolation

### Module 3: Database Management
**Repository**: `database/`
**Technologies**: PostgreSQL + migration scripts
**Responsibilities**:
- Schema design and evolution
- Performance optimization
- Backup and recovery procedures
- Data archiving strategies

### Module 4: Sarah AI Integration
**Repository**: `integrations/sarah-ai/`
**Technologies**: Python + WebSocket + REST APIs
**Responsibilities**:
- Real-time call event processing
- Automatic activity logging
- Lead qualification integration
- Call outcome tracking

### Module 5: Analytics Engine
**Repository**: `analytics/`
**Technologies**: Python + pandas + PostgreSQL + Redis
**Responsibilities**:
- Business intelligence queries
- Performance metrics calculation
- Trend analysis and reporting
- Data visualization APIs

### Module 6: DevOps Infrastructure
**Repository**: `devops/`
**Technologies**: Docker + Kubernetes + CI/CD pipelines
**Responsibilities**:
- Automated deployment
- Monitoring and alerting
- Load balancing and scaling
- Security scanning

### Module 7: Data Migration Tools
**Repository**: `data-migration/`
**Technologies**: Python + ETL frameworks
**Responsibilities**:
- Legacy data import
- Data cleansing and validation
- Format transformation
- Migration verification

### Module 8: Third-Party Integrations
**Repository**: `integrations/third-party/`
**Technologies**: Python + REST APIs + OAuth
**Responsibilities**:
- Payment processor connections
- Document management integration
- CRM system synchronization
- Communication platform links

### Module 9: Integration Hub
**Repository**: `integration-hub/`
**Technologies**: Python + message queues + API gateway
**Responsibilities**:
- Unified integration management
- Webhook routing and processing
- Data transformation services
- Integration monitoring

### Module 10: Merchant Scoring
**Repository**: `merchant-scoring/`
**Technologies**: Python + machine learning libraries
**Responsibilities**:
- Risk assessment algorithms
- Performance prediction models
- Scoring dashboard interfaces
- Historical analysis tools

### Module 11: User Management
**Repository**: `auth-system/`
**Technologies**: FastAPI + JWT + OAuth providers
**Responsibilities**:
- Multi-role authentication
- Permission management
- SSO integration
- Audit trail logging

## Data Flow Architecture

### Typical Data Flow Example
1. **Sarah AI** makes outbound call to 555-123-4567
2. **Call outcome** posted to webhook endpoint
3. **Integration Hub** processes webhook, identifies phone number
4. **Core API** updates phone_numbers table with call result
5. **Permission system** determines which tenants can see this data
6. **Analytics Engine** updates real-time metrics
7. **Frontend Applications** receive real-time updates via WebSocket
8. **Third-party systems** notified via outbound webhooks

### Data Sharing Flow
1. **Tenant A** creates human record for "John Smith"
2. **Tenant B** calls phone number linked to John Smith
3. **Permission system** checks if Tenant B can see Tenant A's human data
4. **Filtered data** returned to Tenant B based on sharing agreements
5. **Consent records** verified before sharing sensitive information
6. **Audit logs** record all cross-tenant data access

## Security Architecture

### Multi-Tenant Isolation
- **Database Level**: Row-level security policies
- **API Level**: Tenant context in all queries
- **Application Level**: Permission verification for every operation
- **Network Level**: Tenant-specific rate limiting and access controls

### Data Protection
- **Encryption at Rest**: All sensitive data encrypted in database
- **Encryption in Transit**: TLS 1.3 for all API communications
- **Field-Level Encryption**: PII encrypted with tenant-specific keys
- **Audit Logging**: Complete trail of all data access and modifications

### Compliance Framework
- **TCPA Compliance**: Automated DNC checking and consent verification
- **GDPR Compliance**: Data deletion and portability tools
- **SOC 2 Compliance**: Security controls and audit procedures
- **Industry Standards**: Payment processing and data handling regulations

## Scalability Architecture

### Horizontal Scaling
- **API Layer**: Stateless services behind load balancers
- **Database Layer**: Read replicas and connection pooling
- **Integration Layer**: Message queues for asynchronous processing
- **Cache Layer**: Redis for frequently accessed data

### Performance Optimization
- **Database Indexes**: Optimized for multi-tenant queries
- **API Caching**: Response caching with tenant-aware invalidation
- **Connection Pooling**: Efficient database connection management
- **Async Processing**: Background jobs for heavy operations

## Deployment Architecture

### Environment Strategy
- **Development**: Individual developer environments
- **Staging**: Integration testing environment
- **Production**: High-availability production deployment
- **DR**: Disaster recovery environment for business continuity

### Infrastructure Components
- **Application Servers**: Kubernetes cluster with auto-scaling
- **Database Cluster**: PostgreSQL with streaming replication
- **Load Balancers**: HAProxy for traffic distribution
- **Monitoring Stack**: Prometheus + Grafana + alerting
- **Logging System**: ELK stack for centralized log analysis

## Development Workflow

### Git Strategy
- **Main Repository**: Integration and coordination
- **Module Repositories**: Independent development streams
- **Feature Branches**: Isolated development for new features
- **Release Branches**: Coordinated releases across modules

### Quality Gates
- **Code Review**: All changes reviewed by team members
- **Automated Testing**: Unit, integration, and end-to-end tests
- **Security Scanning**: Automated vulnerability detection
- **Performance Testing**: Load testing for scalability verification

## Future Considerations

### Planned Enhancements
- **Machine Learning**: Advanced analytics and prediction models
- **Global Deployment**: Multi-region deployment for performance
- **Advanced Integrations**: AI-powered data enrichment services
- **Mobile Applications**: Native mobile apps for field users

### Scalability Roadmap
- **Microservices Migration**: Break down monolithic components
- **Event-Driven Architecture**: Real-time data processing improvements
- **GraphQL API**: More flexible data querying capabilities
- **Blockchain Integration**: Immutable audit trails and consent records

---

*This architecture is designed to scale from startup to enterprise while maintaining data integrity, security, and performance across all tenant workloads.*