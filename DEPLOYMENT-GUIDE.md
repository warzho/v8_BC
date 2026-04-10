# Banking Process Deployment Guide

Step-by-step instructions for deploying the banking migration processes to both BAMOE v8 and v9 environments.

## 📋 Prerequisites

- ✅ V8 Business Central running on http://localhost:8180/business-central
- ✅ V8 PostgreSQL running on port 5438
- ✅ V9 Quarkus service running on http://localhost:8080 (or 8083)
- ✅ V9 PostgreSQL running on port 5433
- ✅ All 10 BPMN files from this repository

## 🎯 Deployment Strategy

We'll deploy the **same BPMN files** to both v8 and v9 to ensure process definition compatibility for ADM migration.

### Why Same Files?

ADM requires **identical process definitions** in source and target:
- Same process ID
- Same version
- Same node structure
- Same variable names and types

## 📦 Part 1: Deploy to V8 Business Central

### Step 1: Access Business Central

```powershell
# Open in browser
start http://localhost:8180/business-central
```

**Login:** `admin` / `admin`

### Step 2: Create Organizational Structure

1. **Create Space:**
   - Click "Design" → "Spaces"
   - Click "Add Space"
   - Name: `banking-migration-demo`
   - Description: `Banking processes for v8→v9 migration testing`
   - Click "Save"

2. **Create Project:**
   - Inside the space, click "Add Project"
   - Name: `banking-processes`
   - Description: `10 banking BPMN processes with primitive variables`
   - Click "Add"

### Step 3: Import BPMN Files

For each of the 10 BPMN files:

1. Click "Import Asset"
2. Browse to the BPMN file location
3. Click "Ok"
4. Verify the process appears in the asset list

**Import Order (recommended):**

```
Simple (2 files):
1. simple/account-opening.bpmn
2. simple/wire-transfer.bpmn

Medium (5 files):
3. medium/credit-card-application.bpmn
4. medium/loan-origination.bpmn
5. medium/trade-settlement.bpmn
6. medium/kyc-refresh.bpmn
7. medium/regulatory-reporting.bpmn

Complex (3 files):
8. complex/mortgage-application.bpmn
9. complex/fraud-investigation.bpmn
10. complex/commercial-lending.bpmn
```

### Step 4: Build and Deploy

1. Click "Build" → "Build & Deploy"
2. Wait for success message
3. Note the deployment ID (e.g., `banking-migration-demo_banking-processes_1.0.0-SNAPSHOT`)

### Step 5: Verify Deployment in KIE Server

```powershell
# List deployed containers
curl -X GET http://localhost:8080/kie-server/services/rest/server/containers `
  -u admin:admin `
  -H "Accept: application/json"

# List process definitions
curl -X GET "http://localhost:8080/kie-server/services/rest/server/containers/banking-migration-demo_banking-processes_1.0.0-SNAPSHOT/processes" `
  -u admin:admin `
  -H "Accept: application/json"
```

Expected: 10 process definitions listed

### Step 6: Create Test Process Instances

Use the provided test data files to create instances:

```powershell
# Example: Create account opening instance
curl -X POST "http://localhost:8080/kie-server/services/rest/server/containers/banking-migration-demo_banking-processes_1.0.0-SNAPSHOT/processes/account-opening/instances" `
  -u admin:admin `
  -H "Content-Type: application/json" `
  -d @test-data/account-opening-test-data.json
```

**Recommended Test Matrix:**

| Process | Happy Path | Exception | Rejection | In-Flight | Total |
|---|---|---|---|---|---|
| account-opening | 2 | 1 | 1 | 1 | 5 |
| wire-transfer | 2 | 1 | 1 | 1 | 5 |
| credit-card-application | 2 | 1 | 1 | 1 | 5 |
| loan-origination | 2 | 1 | 1 | 1 | 5 |
| mortgage-application | 2 | 1 | 1 | 1 | 5 |
| trade-settlement | 2 | 1 | 1 | 1 | 5 |
| kyc-refresh | 2 | 1 | 1 | 1 | 5 |
| fraud-investigation | 2 | 1 | 1 | 1 | 5 |
| commercial-lending | 2 | 1 | 1 | 1 | 5 |
| regulatory-reporting | 2 | 1 | 1 | 1 | 5 |
| **TOTAL** | | | | | **50** |

### Step 7: Verify Instances in V8 Database

```powershell
# Connect to V8 PostgreSQL
docker exec -it bamoe-v8-postgres psql -U bamoe -d bamoe_v8
```

```sql
-- Count process instances by process ID
SELECT 
    processid,
    COUNT(*) as instance_count,
    SUM(CASE WHEN status = 1 THEN 1 ELSE 0 END) as active,
    SUM(CASE WHEN status = 2 THEN 1 ELSE 0 END) as completed
FROM processinstancelog
WHERE processid IN (
    'account-opening',
    'wire-transfer',
    'credit-card-application',
    'loan-origination',
    'mortgage-application',
    'trade-settlement',
    'kyc-refresh',
    'fraud-investigation',
    'commercial-lending',
    'regulatory-reporting'
)
GROUP BY processid
ORDER BY processid;
```

Expected: 5 instances per process, mix of active and completed

---

## 🚀 Part 2: Deploy to V9 Quarkus Service

### Option A: Copy BPMN Files Directly (Recommended)

The V9 Quarkus service auto-deploys BPMN files from `src/main/resources/`.

```powershell
# Navigate to V9 service resources
cd business-services\quarkus\hiring-service\src\main\resources

# Create banking-processes directory
New-Item -ItemType Directory -Path "banking-processes" -Force

# Copy all BPMN files
Copy-Item -Path "C:\path\to\banking-migration-processes\simple\*.bpmn" -Destination "banking-processes\"
Copy-Item -Path "C:\path\to\banking-migration-processes\medium\*.bpmn" -Destination "banking-processes\"
Copy-Item -Path "C:\path\to\banking-migration-processes\complex\*.bpmn" -Destination "banking-processes\"
```

**Quarkus will hot-reload automatically** if the service is running in dev mode.

### Option B: Build KJAR and Deploy (Alternative)

If you prefer the traditional KJAR approach:

1. Create a Maven project with the BPMN files
2. Build the KJAR: `mvn clean install`
3. Copy KJAR to V9 service dependencies
4. Restart V9 service

### Step 8: Verify V9 Deployment

```powershell
# Check health endpoint
curl http://localhost:8080/q/health
```

Look for process definitions in the health check response:

```json
{
  "status": "UP",
  "checks": [
    {
      "name": "Processes",
      "status": "UP",
      "data": {
        "account-opening": "1.0",
        "wire-transfer": "1.0",
        "credit-card-application": "1.0",
        ...
      }
    }
  ]
}
```

### Step 9: Verify V9 Swagger UI

```powershell
# Open Swagger UI
start http://localhost:8080/q/swagger-ui
```

You should see REST endpoints for all 10 processes:
- `POST /{processId}` - Start new instance
- `GET /{processId}/instances` - List instances
- `GET /{processId}/instances/{instanceId}` - Get instance details

---

## 🔍 Verification Checklist

### V8 Environment

- [ ] Business Central accessible
- [ ] Project `banking-processes` created
- [ ] All 10 BPMN files imported
- [ ] Project built and deployed successfully
- [ ] KIE Server shows 10 process definitions
- [ ] 50 test instances created (5 per process)
- [ ] Database shows instances in `processinstancelog` table
- [ ] Some instances in "Active" state (for in-flight migration testing)

### V9 Environment

- [ ] Quarkus service running and healthy
- [ ] Health check shows all 10 processes loaded
- [ ] Swagger UI shows REST endpoints for all processes
- [ ] Database schema created (Flyway migrations ran)
- [ ] No process instances yet (will be created by ADM migration)

### Process Definition Compatibility

- [ ] Process IDs match between v8 and v9
- [ ] Process versions match (1.0)
- [ ] Variable names and types identical
- [ ] Node IDs consistent (important for ADM state mapping)

---

## 🐛 Troubleshooting

### Issue: BPMN File Won't Import to Business Central

**Symptoms:** Import fails with validation error

**Solutions:**
1. Check XML syntax (must be valid BPMN 2.0)
2. Ensure `processType="Public"` is set
3. Verify all `itemDefinition` elements are defined
4. Check for special characters in process ID

### Issue: Process Not Showing in KIE Server

**Symptoms:** Build succeeds but process not listed

**Solutions:**
1. Verify deployment ID is correct
2. Check KIE Server logs: `docker logs <kie-server-container>`
3. Restart KIE Server container
4. Rebuild and redeploy project

### Issue: V9 Service Not Loading Processes

**Symptoms:** Health check doesn't show processes

**Solutions:**
1. Check BPMN files are in `src/main/resources/`
2. Verify file extension is `.bpmn` (not `.bpmn2` or `.xml`)
3. Check Quarkus logs for parsing errors
4. Ensure `kmodule.xml` exists in `META-INF/`

### Issue: Variable Type Mismatch

**Symptoms:** ADM migration fails with type error

**Solutions:**
1. Verify all variables use primitive types only
2. Check `itemDefinition` matches variable type
3. Ensure no custom objects or collections
4. Use `Long` for timestamps, not `Date`

---

## 📊 Post-Deployment Validation

### V8 Instance Distribution

```sql
-- Check instance states
SELECT 
    status,
    CASE 
        WHEN status = 0 THEN 'Pending'
        WHEN status = 1 THEN 'Active'
        WHEN status = 2 THEN 'Completed'
        WHEN status = 3 THEN 'Aborted'
        WHEN status = 4 THEN 'Suspended'
        ELSE 'Unknown'
    END as status_name,
    COUNT(*) as count
FROM processinstancelog
GROUP BY status
ORDER BY status;
```

### V8 Variable Verification

```sql
-- Sample process variables
SELECT 
    pil.processid,
    pil.processinstanceid,
    vil.variableid,
    vil.value
FROM processinstancelog pil
JOIN variableinstancelog vil ON pil.processinstanceid = vil.processinstanceid
WHERE pil.processid = 'account-opening'
LIMIT 10;
```

### V9 Pre-Migration State

```sql
-- Should be empty before migration
SELECT COUNT(*) as instance_count
FROM process_instances;
```

Expected: 0 (V9 database is clean, ready for migration)

---

## ✅ Ready for Migration

Once both environments are deployed and verified:

1. **V8 has 50 active/completed process instances** across 10 process definitions
2. **V9 has the same 10 process definitions** loaded and ready
3. **Both databases are accessible** (ports 5438 and 5433)
4. **ADM tool is configured** with connection details

Proceed to **MIGRATION-GUIDE.md** for migration execution steps.

---

**Next Steps:**
- Review test data files in `test-data/` directory
- Customize instance counts based on your testing needs
- Proceed to migration when ready