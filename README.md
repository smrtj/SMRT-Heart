# SMRT CRM Platform

**Smart Real-time Relationship Management Platform**  
*Multi-Tenant SaaS Platform for Contact Intelligence and Business Operations*

## Quick Start

This is the main coordination repository for the SMRT CRM Platform. All development modules are managed as Git submodules to enable parallel team development while maintaining integration coordination.

### Repository Structure

```
smrt-crm-platform/
├── README.md                    # This file
├── docs/                        # Project documentation
│   ├── SMRT-CRM-PLATFORM-README.md            # Complete project overview
│   ├── SMRT-CRM-PLATFORM-ARCHITECTURE.md      # Technical architecture
│   ├── SMRT-CRM-DATABASE-SCHEMA.md           # Database design
│   ├── DEVELOPMENT-STANDARDS.md               # Coding standards
│   ├── INTEGRATION-PATTERNS.md                # Integration framework
│   └── MODULE-CHECKOUT-SYSTEM.md              # Team coordination
├── modules/                     # Git submodules for each development part
│   ├── database-core/          # Core database schema and migrations
│   ├── api-core/               # FastAPI backend (Part 2)
│   ├── frontend-web/           # Web interface (Part 1)
│   ├── auth-system/            # Authentication (Part 11)
│   ├── integrations/           # Third-party connectors (Parts 4,8,9)
│   ├── analytics/              # Business intelligence (Part 5)
│   └── ...                     # Additional modules
├── tests/                       # Integration tests
│   ├── integration/            # Cross-module integration tests
│   └── e2e/                    # End-to-end system tests
├── scripts/                     # Deployment and utility scripts
├── config/                      # Shared configuration files
└── .gitmodules                  # Submodule configuration
```

## Core Concept

The SMRT CRM Platform is built around a **two-core data model**:

- **Phone Numbers**: Universal identifiers with rich interaction history
- **Humans**: Individual entities with roles, relationships, and business context
- **Dynamic Linking**: Many-to-many relationships with contextual metadata

This enables unprecedented data intelligence sharing across multiple tenants while maintaining strict isolation and access controls.

## Development Workflow

### 1. Check Out a Module
```bash
# Check available modules
./scripts/checkout.sh list-available

# Reserve a module for development
./scripts/checkout.sh checkout api-001 "Your Name" "your@email.com" 21 "Core API development"
```

### 2. Clone and Work
```bash
# Clone the specific module repository
git clone git@github.com:smrt-crm/api-core.git modules/api-core
cd modules/api-core

# Create feature branch
git checkout -b feature/your-feature-name

# Develop following coding standards in docs/DEVELOPMENT-STANDARDS.md
```

### 3. Integration Testing
```bash
# Return to main repo and run integration tests
cd ../..
./scripts/test-integration.sh api-core

# Update progress
./scripts/checkout.sh progress your-checkout-id 75 "Feature complete, tests passing"
```

### 4. Complete Module
```bash
# Mark module as complete
./scripts/checkout.sh complete your-checkout-id "All features implemented"

# System automatically runs quality gates and creates integration PR
```

## Key Documentation

- **[Complete Project Overview](docs/SMRT-CRM-PLATFORM-README.md)** - Full business model and technical vision
- **[Technical Architecture](docs/SMRT-CRM-PLATFORM-ARCHITECTURE.md)** - System design and module interactions  
- **[Database Schema](docs/SMRT-CRM-DATABASE-SCHEMA.md)** - Two-core data model and multi-tenant design
- **[Development Standards](docs/DEVELOPMENT-STANDARDS.md)** - Coding conventions and quality requirements
- **[Integration Patterns](docs/INTEGRATION-PATTERNS.md)** - Third-party system integration framework
- **[Module Checkout System](docs/MODULE-CHECKOUT-SYSTEM.md)** - Team coordination and conflict prevention

## Getting Started

### For New Developers
1. Read the [Project Overview](docs/SMRT-CRM-PLATFORM-README.md) to understand the vision
2. Review [Development Standards](docs/DEVELOPMENT-STANDARDS.md) for coding requirements
3. Check [Module Checkout System](docs/MODULE-CHECKOUT-SYSTEM.md) for team workflow
4. Choose a module and check it out for development

### For Business Users
1. Review the [Architecture Documentation](docs/SMRT-CRM-PLATFORM-ARCHITECTURE.md) for capabilities
2. Understand the multi-tenant model and data sharing benefits
3. Contact the team for tenant onboarding and API access

## Module Status

Use the checkout system to see current module assignments:
```bash
./scripts/checkout.sh list-active
```

## Support

- **Technical Questions**: Review documentation in `docs/` directory
- **Module Conflicts**: Use the checkout system to coordinate with team
- **Integration Issues**: Run integration tests and check module interfaces
- **Architecture Decisions**: All documented in architecture files

---

**Built for Scale. Designed for Intelligence. Powered by Relationships.**

*The SMRT CRM Platform transforms every phone number into business intelligence and every interaction into competitive advantage.*