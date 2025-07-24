# SMRT CRM Platform - Module Checkout System

## Overview

The Module Checkout System ensures coordinated development across the SMRT CRM Platform by preventing overlapping work, managing dependencies, and maintaining clear ownership of development modules during active work periods.

## System Architecture

### Core Components
1. **Module Registry** - Central tracking of all platform modules
2. **Checkout Database** - Active reservations and assignments  
3. **Dependency Tracker** - Module interdependencies and blocking relationships
4. **Notification System** - Team coordination and status updates
5. **Integration Gates** - Quality controls before module release

## Module Registry

### Available Modules

```yaml
modules:
  # Core Platform Modules
  database-core:
    id: "db-001"
    name: "Database Core Schema"
    description: "Core phone_numbers and humans tables with relationships"
    repository: "git@github.com:smrt-crm/database-core.git"
    dependencies: []
    estimated_effort: "2-3 weeks"
    complexity: "high"
    
  api-core:
    id: "api-001" 
    name: "Core API Layer"
    description: "FastAPI backend with authentication and CRUD operations"
    repository: "git@github.com:smrt-crm/api-core.git"
    dependencies: ["db-001"]
    estimated_effort: "3-4 weeks"
    complexity: "high"
    
  # Frontend Modules
  frontend-web:
    id: "fe-001"
    name: "Web Frontend Application"
    description: "React/Vue/Svelte web interface"
    repository: "git@github.com:smrt-crm/frontend-web.git"
    dependencies: ["api-001"]
    estimated_effort: "4-6 weeks"
    complexity: "medium"
    
  frontend-mobile:
    id: "fe-002"
    name: "Mobile Frontend Application"
    description: "React Native mobile application"
    repository: "git@github.com:smrt-crm/frontend-mobile.git"
    dependencies: ["api-001"]
    estimated_effort: "6-8 weeks"
    complexity: "high"
    
  # Integration Modules
  integration-sarah-ai:
    id: "int-001"
    name: "Sarah AI Integration"
    description: "Real-time calling system integration"
    repository: "git@github.com:smrt-crm/integration-sarah-ai.git"
    dependencies: ["api-001", "db-001"]
    estimated_effort: "2-3 weeks"
    complexity: "medium"
    
  integration-payment:
    id: "int-002"
    name: "Payment Processing Integration"
    description: "Card Connect and payment processor integrations"
    repository: "git@github.com:smrt-crm/integration-payment.git"
    dependencies: ["api-001"]
    estimated_effort: "3-4 weeks"
    complexity: "medium"
    
  integration-hub:
    id: "int-003"
    name: "Integration Hub Framework"
    description: "Unified third-party integration management"
    repository: "git@github.com:smrt-crm/integration-hub.git"
    dependencies: ["api-001"]
    estimated_effort: "4-5 weeks"
    complexity: "high"
    
  # Analytics and Reporting
  analytics-core:
    id: "ana-001"
    name: "Analytics Engine"
    description: "Business intelligence and reporting system"
    repository: "git@github.com:smrt-crm/analytics-core.git"
    dependencies: ["db-001", "api-001"]
    estimated_effort: "5-6 weeks"
    complexity: "high"
    
  analytics-dashboard:
    id: "ana-002"
    name: "Analytics Dashboard"
    description: "Real-time metrics and visualization interface"
    repository: "git@github.com:smrt-crm/analytics-dashboard.git"
    dependencies: ["ana-001", "fe-001"]
    estimated_effort: "3-4 weeks"
    complexity: "medium"
    
  # Security and Compliance
  auth-system:
    id: "auth-001"
    name: "Multi-Tenant Authentication"
    description: "JWT authentication with role-based access control"
    repository: "git@github.com:smrt-crm/auth-system.git"
    dependencies: ["db-001"]
    estimated_effort: "3-4 weeks"
    complexity: "high"
    
  compliance-framework:
    id: "comp-001"
    name: "Compliance and Consent Management"
    description: "TCPA, GDPR compliance with consent tracking"
    repository: "git@github.com:smrt-crm/compliance-framework.git"
    dependencies: ["db-001", "api-001"]
    estimated_effort: "4-5 weeks"
    complexity: "high"
    
  # Data Management
  data-migration:
    id: "data-001"
    name: "Data Migration Tools"
    description: "ETL tools for legacy data import and transformation"
    repository: "git@github.com:smrt-crm/data-migration.git"  
    dependencies: ["db-001"]
    estimated_effort: "2-3 weeks"
    complexity: "medium"
    
  data-backup:
    id: "data-002"
    name: "Backup and Recovery System"
    description: "Automated backup, archival, and disaster recovery"
    repository: "git@github.com:smrt-crm/data-backup.git"
    dependencies: ["db-001"]
    estimated_effort: "2-3 weeks"
    complexity: "medium"
    
  # DevOps and Infrastructure
  devops-deployment:
    id: "ops-001"
    name: "Deployment Infrastructure"
    description: "Docker, Kubernetes, CI/CD pipelines"
    repository: "git@github.com:smrt-crm/devops-deployment.git"
    dependencies: []
    estimated_effort: "3-4 weeks"
    complexity: "medium"
    
  devops-monitoring:
    id: "ops-002"
    name: "Monitoring and Alerting"
    description: "Prometheus, Grafana, log aggregation"
    repository: "git@github.com:smrt-crm/devops-monitoring.git"
    dependencies: ["ops-001"]
    estimated_effort: "2-3 weeks"
    complexity: "medium"
```

## Checkout Database Schema

```sql
CREATE TABLE module_checkouts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    module_id VARCHAR(20) NOT NULL, -- References modules.yaml
    checked_out_by VARCHAR(100) NOT NULL, -- Developer name or team identifier
    checked_out_by_email VARCHAR(255) NOT NULL,
    checkout_date TIMESTAMP DEFAULT NOW(),
    estimated_completion_date TIMESTAMP NOT NULL,
    actual_completion_date TIMESTAMP,
    checkout_reason TEXT, -- "Initial development", "Bug fix", "Enhancement"
    status VARCHAR(20) DEFAULT 'active', -- 'active', 'completed', 'abandoned', 'blocked'
    progress_percentage INTEGER DEFAULT 0 CHECK (progress_percentage >= 0 AND progress_percentage <= 100),
    last_progress_update TIMESTAMP DEFAULT NOW(),
    blocking_issues TEXT[],
    notes TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE checkout_notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),  
    checkout_id UUID NOT NULL REFERENCES module_checkouts(id),
    notification_type VARCHAR(50) NOT NULL, -- 'checkout', 'progress', 'completion', 'blocking', 'overdue'
    recipient_email VARCHAR(255) NOT NULL,
    message TEXT NOT NULL,
    sent_at TIMESTAMP DEFAULT NOW(),
    delivery_status VARCHAR(20) DEFAULT 'pending' -- 'pending', 'sent', 'failed'
);

CREATE TABLE module_dependencies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    module_id VARCHAR(20) NOT NULL,
    depends_on_module_id VARCHAR(20) NOT NULL,
    dependency_type VARCHAR(50) NOT NULL, -- 'blocks', 'integrates_with', 'extends'
    is_critical BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Indexes for performance
CREATE INDEX idx_module_checkouts_active ON module_checkouts(module_id, status) WHERE status = 'active';
CREATE INDEX idx_module_checkouts_developer ON module_checkouts(checked_out_by, status);
CREATE INDEX idx_checkout_notifications_pending ON checkout_notifications(delivery_status) WHERE delivery_status = 'pending';
```

## Checkout API Interface

### REST API Endpoints

```python
# FastAPI endpoints for module checkout system
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from typing import List, Optional
from datetime import datetime, timedelta

app = FastAPI(title="Module Checkout System")

class CheckoutRequest(BaseModel):
    module_id: str
    developer_name: str
    developer_email: str
    estimated_days: int
    reason: str
    notes: Optional[str] = None

class CheckoutResponse(BaseModel):
    checkout_id: str
    module_id: str
    status: str
    checkout_date: datetime
    estimated_completion_date: datetime

@app.post("/checkouts", response_model=CheckoutResponse)
async def checkout_module(request: CheckoutRequest):
    """Reserve a module for development"""
    # Check if module is already checked out
    existing_checkout = await get_active_checkout(request.module_id)
    if existing_checkout:
        raise HTTPException(
            status_code=409,
            detail=f"Module {request.module_id} is already checked out by {existing_checkout.developer_name}"
        )
    
    # Verify dependencies are available
    blocked_dependencies = await check_blocking_dependencies(request.module_id)
    if blocked_dependencies:
        raise HTTPException(
            status_code=400,
            detail=f"Cannot checkout module. Blocked by: {', '.join(blocked_dependencies)}"
        )
    
    # Create checkout record
    checkout = await create_checkout(request)
    
    # Send notifications
    await notify_team_checkout(checkout)
    
    return checkout

@app.get("/checkouts/active", response_model=List[CheckoutResponse])
async def get_active_checkouts():
    """Get all currently active module checkouts"""
    return await get_all_active_checkouts()

@app.put("/checkouts/{checkout_id}/progress")
async def update_progress(checkout_id: str, progress: int, notes: Optional[str] = None):
    """Update progress on a checked-out module"""
    checkout = await get_checkout_by_id(checkout_id)
    if not checkout:
        raise HTTPException(status_code=404, detail="Checkout not found")
    
    await update_checkout_progress(checkout_id, progress, notes)
    
    # Send progress notifications if significant milestone
    if progress in [25, 50, 75, 100]:
        await notify_progress_milestone(checkout, progress)
    
    return {"status": "updated", "progress": progress}

@app.post("/checkouts/{checkout_id}/complete")
async def complete_checkout(checkout_id: str, notes: Optional[str] = None):
    """Mark a module checkout as completed"""
    checkout = await get_checkout_by_id(checkout_id)
    if not checkout:
        raise HTTPException(status_code=404, detail="Checkout not found")
    
    # Run integration tests
    test_results = await run_integration_tests(checkout.module_id)
    if not test_results.passed:
        raise HTTPException(
            status_code=400,
            detail=f"Integration tests failed: {test_results.errors}"
        )
    
    await complete_checkout_record(checkout_id, notes)
    await notify_completion(checkout)
    
    # Auto-checkout dependent modules if requested
    await check_dependent_modules_ready(checkout.module_id)
    
    return {"status": "completed"}

@app.delete("/checkouts/{checkout_id}")
async def abandon_checkout(checkout_id: str, reason: str):
    """Abandon/cancel a module checkout"""
    checkout = await get_checkout_by_id(checkout_id)
    if not checkout:
        raise HTTPException(status_code=404, detail="Checkout not found")
    
    await abandon_checkout_record(checkout_id, reason)
    await notify_abandonment(checkout, reason)
    
    return {"status": "abandoned"}
```

## Command Line Interface

```bash
#!/bin/bash
# Module checkout CLI tool

CHECKOUT_API_BASE="https://platform.smrt-crm.com/api/checkout"

function checkout_module() {
    local module_id=$1
    local developer_name=$2
    local developer_email=$3
    local estimated_days=$4
    local reason=$5
    
    curl -X POST "$CHECKOUT_API_BASE/checkouts" \
        -H "Content-Type: application/json" \
        -d "{
            \"module_id\": \"$module_id\",
            \"developer_name\": \"$developer_name\", 
            \"developer_email\": \"$developer_email\",
            \"estimated_days\": $estimated_days,
            \"reason\": \"$reason\"
        }"
}

function list_available_modules() {
    echo "Available modules for checkout:"
    curl -s "$CHECKOUT_API_BASE/modules/available" | jq -r '.[] | "\(.id): \(.name) - \(.description)"'
}

function list_active_checkouts() {
    echo "Currently active checkouts:"
    curl -s "$CHECKOUT_API_BASE/checkouts/active" | jq -r '.[] | "\(.module_id): \(.checked_out_by) (due: \(.estimated_completion_date))"'
}

function update_progress() {
    local checkout_id=$1
    local progress=$2
    local notes=$3
    
    curl -X PUT "$CHECKOUT_API_BASE/checkouts/$checkout_id/progress" \
        -H "Content-Type: application/json" \
        -d "{\"progress\": $progress, \"notes\": \"$notes\"}"
}

# Usage examples:
# ./checkout.sh list-available
# ./checkout.sh checkout api-001 "John Smith" "john@smrt-crm.com" 21 "Initial API development"
# ./checkout.sh list-active
# ./checkout.sh progress abc-123-def 75 "Completed authentication module"
# ./checkout.sh complete abc-123-def "All tests passing, ready for integration"
```

## Integration with Git Workflow

### Automated Branch Creation

```python
import git
import subprocess
from datetime import datetime

class GitIntegration:
    def __init__(self, main_repo_path: str):
        self.main_repo = git.Repo(main_repo_path)
        
    def create_module_branch(self, module_id: str, developer_name: str) -> str:
        """Create development branch for checked-out module"""
        branch_name = f"module/{module_id}/{developer_name.lower().replace(' ', '-')}"
        
        # Create branch from main
        new_branch = self.main_repo.create_head(branch_name, self.main_repo.heads.main)
        
        # Push to origin
        origin = self.main_repo.remote('origin')
        origin.push(new_branch)
        
        return branch_name
    
    def setup_submodule(self, module_id: str, repository_url: str) -> bool:
        """Add submodule for new module development"""
        try:
            subprocess.run([
                'git', 'submodule', 'add', 
                repository_url, 
                f'modules/{module_id}'
            ], cwd=self.main_repo.working_dir, check=True)
            
            subprocess.run([
                'git', 'commit', '-m', 
                f'Add submodule for {module_id}'
            ], cwd=self.main_repo.working_dir, check=True)
            
            return True
        except subprocess.CalledProcessError:
            return False
    
    def create_integration_pr(self, module_id: str, branch_name: str) -> str:
        """Create pull request for module integration"""
        # Using GitHub CLI or GitLab CLI
        subprocess.run([
            'gh', 'pr', 'create',
            '--title', f'Integrate {module_id} module',
            '--body', f'Integration of completed {module_id} module from branch {branch_name}',
            '--base', 'main',
            '--head', branch_name
        ], cwd=self.main_repo.working_dir)
```

## Quality Gates and Integration

### Pre-Integration Checklist

```yaml
integration_requirements:
  documentation:
    - module_readme_complete: true
    - api_documentation_updated: true  
    - database_schema_documented: true
    - integration_guide_written: true
    
  testing:
    - unit_tests_passing: true
    - integration_tests_passing: true
    - security_scan_clean: true
    - performance_benchmarks_met: true
    
  code_quality:
    - code_review_approved: true
    - linting_rules_followed: true
    - type_checking_passed: true
    - dependency_audit_clean: true
    
  coordination:
    - dependent_modules_notified: true
    - breaking_changes_documented: true
    - migration_scripts_provided: true
    - deployment_plan_reviewed: true
```

### Automated Quality Checks

```python
class QualityGateChecker:
    def __init__(self, module_id: str):
        self.module_id = module_id
        
    async def run_all_checks(self) -> bool:
        """Run comprehensive quality checks before integration"""
        checks = [
            self.check_documentation(),
            self.check_tests(),
            self.check_code_quality(),
            self.check_security(),
            self.check_performance(),
            self.check_dependencies()
        ]
        
        results = await asyncio.gather(*checks)
        return all(results)
    
    async def check_documentation(self) -> bool:
        """Verify all required documentation exists"""
        required_docs = [
            f"modules/{self.module_id}/README.md",
            f"modules/{self.module_id}/API.md", 
            f"modules/{self.module_id}/INTEGRATION.md"
        ]
        
        return all(os.path.exists(doc) for doc in required_docs)
    
    async def check_tests(self) -> bool:
        """Run test suite and verify coverage"""
        result = subprocess.run([
            'pytest', f'modules/{self.module_id}/tests/',
            '--cov', f'modules/{self.module_id}',
            '--cov-report', 'term-missing',
            '--cov-fail-under', '80'
        ], capture_output=True)
        
        return result.returncode == 0
```

## Team Coordination Features

### Daily Standup Integration

```python
async def generate_daily_standup_report():
    """Generate daily report for team standup meetings"""
    active_checkouts = await get_all_active_checkouts()
    
    report = {
        "date": datetime.now().isoformat(),
        "active_modules": len(active_checkouts),
        "modules_by_status": {},
        "upcoming_deadlines": [],
        "blocked_modules": []
    }
    
    for checkout in active_checkouts:
        # Group by progress status
        status = get_progress_status(checkout.progress_percentage)
        if status not in report["modules_by_status"]:
            report["modules_by_status"][status] = []
        report["modules_by_status"][status].append({
            "module_id": checkout.module_id,
            "developer": checkout.checked_out_by,
            "progress": checkout.progress_percentage
        })
        
        # Check for upcoming deadlines
        days_until_due = (checkout.estimated_completion_date - datetime.now()).days
        if days_until_due <= 3:
            report["upcoming_deadlines"].append(checkout)
            
        # Check for blocked modules
        if checkout.blocking_issues:
            report["blocked_modules"].append(checkout)
    
    return report
```

### Slack/Teams Integration

```python
async def send_slack_notification(webhook_url: str, message: dict):
    """Send notifications to team Slack channel"""
    import httpx
    
    async with httpx.AsyncClient() as client:
        await client.post(webhook_url, json={
            "text": message["title"],
            "blocks": [
                {
                    "type": "section",
                    "text": {
                        "type": "mrkdwn",
                        "text": message["description"]
                    }
                },
                {
                    "type": "actions",
                    "elements": message.get("actions", [])
                }
            ]
        })

# Example notifications
async def notify_checkout_conflict(module_id: str, requester: str, current_owner: str):
    await send_slack_notification(TEAM_WEBHOOK, {
        "title": f"Module Checkout Conflict: {module_id}",
        "description": f"{requester} tried to checkout {module_id} but it's currently owned by {current_owner}",
        "actions": [
            {
                "type": "button",
                "text": {"type": "plain_text", "text": "View Details"},
                "url": f"https://platform.smrt-crm.com/checkouts/{module_id}"
            }
        ]
    })
```

## Usage Examples

### Typical Workflow

```bash
# 1. Developer checks what modules are available
./checkout.sh list-available

# 2. Developer reserves a module for 3 weeks
./checkout.sh checkout api-001 "Claude AI" "claude@anthropic.com" 21 "Core API development"

# 3. System creates branch and sets up development environment
# 4. Developer works on module, updating progress regularly
./checkout.sh progress abc-123-def 25 "Authentication system completed"
./checkout.sh progress abc-123-def 50 "CRUD operations for phone numbers done"
./checkout.sh progress abc-123-def 75 "Multi-tenant isolation implemented"

# 5. Developer completes module
./checkout.sh complete abc-123-def "All features implemented, tests passing"

# 6. System runs quality gates, creates PR, notifies dependent module owners
```

### Team Coordination

```bash
# Daily standup report
curl -s "$CHECKOUT_API_BASE/reports/daily" | jq

# Check what's blocking a specific module
curl -s "$CHECKOUT_API_BASE/modules/db-002/dependencies"

# See who's working on what
curl -s "$CHECKOUT_API_BASE/checkouts/active" | jq -r '.[] | "\(.checked_out_by): \(.module_id) (\(.progress_percentage)%)"'
```

This checkout system ensures coordinated development while maintaining flexibility for the team to work efficiently on the SMRT CRM Platform.