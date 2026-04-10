# Banking Process Migration - Quick Start Guide

Get up and running with the banking process migration demo in 30 minutes.

## 🎯 Goal

Migrate 3 banking process instances from BAMOE v8 to v9 using the ADM tool, demonstrating zero-downtime migration capability.

## ✅ Prerequisites Checklist

- [ ] V8 Business Central running (http://localhost:8180/business-central)
- [ ] V8 PostgreSQL running (port 5438)
- [ ] V9 Quarkus service running (http://localhost:8080 or 8083)
- [ ] V9 PostgreSQL running (port 5433)
- [ ] ADM tool v0.1.0 downloaded and configured
- [ ] 3 BPMN files ready: account-opening, wire-transfer, credit-card-application

## 📦 Step 1: Deploy to V8 (10 minutes)

### 1.1 Access Business Central
```powershell
start http://localhost:8180/business-central
# Login: admin / admin
```

### 1.2 Create Project Structure
1. Click "Design" → "Spaces"
2. Click "Add Space"
   - Name: `banking-demo`
   - Click "Save"
3. Click "Add Project"
   - Name: `banking-processes`
   - Click "Add"

### 1.3 Import BPMN Files
For each file (account-opening.bpmn, wire-transfer.bpmn, credit-card-application.bpmn):
1. Click "Import Asset"
2. Browse to file location
3. Click "Ok"
4. Verify it appears in asset list

### 1.4 Build and Deploy
1. Click "Build" → "Build & Deploy"
2. Wait for "Build Successful" message
3. Note the container ID: `banking-demo_banking-processes_1.0.0-SNAPSHOT`

### 1.5 Verify Deployment
```powershell
# Check KIE Server has the processes
curl -X GET "http://localhost:8080/kie-server/services/rest/server/containers/banking-demo_banking-processes_1.0.0-SNAPSHOT/processes" `
  -u admin:admin `
  -H "Accept: application/json"
```

Expected: JSON response with 3 process definitions

## 🚀 Step 2: Create Test Instances in V8 (5 minutes)

### 2.1 Account Opening (2 instances)

**Instance 1 - Happy Path:**
```powershell
curl -X POST "http://localhost:8080/kie-server/services/rest/server/containers/banking-demo_banking-processes_1.0.0-SNAPSHOT/processes/account-opening/instances" `
  -u admin:admin `
  -H "Content-Type: application/json" `
  -d '{
    "customerId": "CUST-001",
    "accountType": "CHECKING",
    "initialDeposit": 50000,
    "branchCode": "NYC-001",
    "employeeId": "EMP-123",
    "requiresManagerApproval": false
  }'
```

**Instance 2 - Requires Approval (In-Flight):**
```powershell
curl -X POST "http://localhost:8080/kie-server/services/rest/server/containers/banking-demo_banking-processes_1.0.0-SNAPSHOT/processes/account-opening/instances" `
  -u admin:admin `
  -H "Content-Type: application/json" `
  -d '{
    "customerId": "CUST-002",
    "accountType": "SAVINGS",
    "initialDeposit": 1500000,
    "branchCode": "NYC-001",
    "employeeId": "EMP-456",
    "requiresManagerApproval": true
  }'
```
*Note: This will wait at Manager Approval task - perfect for testing in-flight migration*

### 2.2 Wire Transfer (2 instances)

**Instance 1 - Auto-Approved:**
```powershell
curl -X POST "http://localhost:8080/kie-server/services/rest/server/containers/banking-demo_banking-processes_1.0.0-SNAPSHOT/processes/wire-transfer/instances" `
  -u admin:admin `
  -H "Content-Type: application/json" `
  -d '{
    "fromAccountId": "ACC-001",
    "toAccountId": "ACC-002",
    "amountInCents": 50000,
    "currency": "USD",
    "requestTimestamp": 1712764800000,
    "isFraudulent": false,
    "requiresReview": false
  }'
```

**Instance 2 - Fraud Review (In-Flight):**
```powershell
curl -X POST "http://localhost:8080/kie-server/services/rest/server/containers/banking-demo_banking-processes_1.0.0-SNAPSHOT/processes/wire-transfer/instances" `
  -u admin:admin `
  -H "Content-Type: application/json" `
  -d '{
    "fromAccountId": "ACC-003",
    "toAccountId": "ACC-004",
    "amountInCents": 10000000,
    "currency": "USD",
    "requestTimestamp": 1712764800000,
    "isFraudulent": true,
    "requiresReview": true
  }'
```

### 2.3 Credit Card Application (2 instances)

**Instance 1 - Auto-Approved:**
```powershell
curl -X POST "http://localhost:8080/kie-server/services/rest/server/containers/banking-demo_banking-processes_1.0.0-SNAPSHOT/processes/credit-card-application/instances" `
  -u admin:admin `
  -H "Content-Type: application/json" `
  -d '{
    "customerId": "CUST-003",
    "cardType": "PLATINUM",
    "requestedLimit": 500000,
    "annualIncome": 10000000,
    "creditScore": 780,
    "requiresUnderwriterReview": false
  }'
```

**Instance 2 - Underwriter Review (In-Flight):**
```powershell
curl -X POST "http://localhost:8080/kie-server/services/rest/server/containers/banking-demo_banking-processes_1.0.0-SNAPSHOT/processes/credit-card-application/instances" `
  -u admin:admin `
  -H "Content-Type: application/json" `
  -d '{
    "customerId": "CUST-004",
    "cardType": "GOLD",
    "requestedLimit": 3000000,
    "annualIncome": 7500000,
    "creditScore": 640,
    "requiresUnderwriterReview": true
  }'
```

### 2.4 Verify Instances in V8 Database
```powershell
docker exec -it bamoe-v8-postgres psql -U bamoe -d bamoe_v8
```

```sql
-- Check process instances
SELECT 
    processinstanceid,
    processid,
    status,
    start_date
FROM processinstancelog
WHERE processid IN ('account-opening', 'wire-transfer', 'credit-card-application')
ORDER BY start_date DESC;

-- Expected: 6 instances (2 per process)
-- Status: 1 = Active, 2 = Completed
```

## 📋 Step 3: Deploy to V9 (5 minutes)

### 3.1 Copy BPMN Files to V9 Service
```powershell
# Navigate to V9 resources directory
cd business-services\quarkus\hiring-service\src\main\resources

# Create directory for banking processes
New-Item -ItemType Directory -Path "banking-processes" -Force

# Copy BPMN files (adjust path to your location)
Copy-Item "C:\Users\WarrenZhou\bamoe-demos\banking-migration-processes\simple\*.bpmn" -Destination "banking-processes\"
Copy-Item "C:\Users\WarrenZhou\bamoe-demos\banking-migration-processes\medium\credit-card-application.bpmn" -Destination "banking-processes\"
```

### 3.2 Verify V9 Service Loaded Processes
```powershell
# Check health endpoint
curl http://localhost:8080/q/health
```

Expected response should include:
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
        "credit-card-application": "1.0"
      }
    }
  ]
}
```

### 3.3 Verify V9 Database is Empty
```powershell
docker exec -it bamoe-v9-postgres psql -U bamoe -d bamoe_v9
```

```sql
-- Should return 0
SELECT COUNT(*) FROM process_instances;
```

## 🔄 Step 4: Configure ADM Tool (5 minutes)

### 4.1 Locate ADM Configuration
```powershell
cd "$env:USERPROFILE\OneDrive - IBM\Documents\bamoe-adm-tool-0.1.0\container-images"
```

### 4.2 Update .env File
Edit `.env` file with these values:

```properties
# V8 Source Configuration
V8_TIMER_MODE=SKIP
V8_ABORT_INSTANCES=false
V8_DB_URL=jdbc:postgresql://host.docker.internal:5438/bamoe_v8
V8_DB_USERNAME=bamoe
V8_DB_PASSWORD=bamoe123
V8_DEPLOYMENT_ID=banking-demo_banking-processes_1.0.0-SNAPSHOT
V8_PROCESS_ID=account-opening,wire-transfer,credit-card-application

# V9 Target Configuration
V9_DB_URL=jdbc:postgresql://host.docker.internal:5433/bamoe_v9
V9_DB_USERNAME=bamoe
V9_DB_PASSWORD=bamoe123
```

### 4.3 Copy V8 KJAR to ADM Tool
```powershell
# Find the KJAR in your local Maven repository
$kjarPath = "$env:USERPROFILE\.m2\repository\banking-demo\banking-processes\1.0.0-SNAPSHOT"

# Copy to ADM kjars directory
Copy-Item "$kjarPath\*.jar" -Destination "kjars\" -Force
```

## 🎬 Step 5: Execute Migration (5 minutes)

### 5.1 Run ADM Tool
```powershell
cd "$env:USERPROFILE\OneDrive - IBM\Documents\bamoe-adm-tool-0.1.0\container-images"
docker compose -f docker-compose.yaml up
```

### 5.2 Monitor Migration Output
Watch for these key messages:
```
✅ Connected to V8 database
✅ Connected to V9 database
✅ Found X process instances to migrate
✅ Migrating instance: <instance-id>
✅ Migration completed successfully
```

### 5.3 Check Migration Logs
```powershell
# View detailed logs
docker logs <adm-container-name>
```

## ✅ Step 6: Verify Migration (5 minutes)

### 6.1 Check V9 Database
```powershell
docker exec -it bamoe-v9-postgres psql -U bamoe -d bamoe_v9
```

```sql
-- Count migrated instances
SELECT 
    process_id,
    COUNT(*) as instance_count,
    SUM(CASE WHEN state = 1 THEN 1 ELSE 0 END) as active,
    SUM(CASE WHEN state = 2 THEN 1 ELSE 0 END) as completed
FROM process_instances
WHERE process_id IN ('account-opening', 'wire-transfer', 'credit-card-application')
GROUP BY process_id
ORDER BY process_id;

-- Expected: 6 instances total (2 per process)
```

### 6.2 Verify Process Variables
```sql
-- Check variables for account-opening
SELECT 
    pi.process_instance_id,
    pi.process_id,
    pv.variable_name,
    pv.variable_value
FROM process_instances pi
JOIN process_instance_variables pv ON pi.process_instance_id = pv.process_instance_id
WHERE pi.process_id = 'account-opening'
ORDER BY pi.process_instance_id, pv.variable_name;

-- Verify: customerId, accountType, initialDeposit, etc. are present
```

### 6.3 Check V8 Source Database
```sql
-- V8 instances should still exist (ADM doesn't delete source)
SELECT COUNT(*) FROM processinstancelog
WHERE processid IN ('account-opening', 'wire-transfer', 'credit-card-application');

-- Expected: 6 (same as before migration)
```

### 6.4 Test V9 Service with Migrated Instance
```powershell
# Get a migrated instance ID from the database query above
$instanceId = "<instance-id-from-query>"

# Query the instance via V9 REST API
curl "http://localhost:8080/account-opening/instances/$instanceId" `
  -H "Accept: application/json"
```

Expected: JSON response with instance details

## 🎉 Success Criteria

- [ ] 6 process instances migrated (2 per process)
- [ ] All process variables preserved
- [ ] In-flight instances at correct wait states
- [ ] Completed instances marked as completed
- [ ] V9 service can query migrated instances
- [ ] No errors in ADM logs
- [ ] V8 source data unchanged

## 📊 Demo Talking Points

### Business Value
- **Zero Downtime:** Processes migrated while running
- **Data Integrity:** All variables and state preserved
- **Selective Migration:** Choose which instances to migrate
- **Rollback Capability:** Source data remains intact

### Technical Highlights
- **Primitive Types Only:** ADM v0.1.0 limitation demonstrated
- **1:1 Process Definitions:** Same BPMN in v8 and v9
- **Database-Level Migration:** Direct PostgreSQL to PostgreSQL
- **Container-Based Tool:** Easy deployment and execution

### Client Scenarios
- **Modernization:** Moving from v8 to cloud-native v9
- **Consolidation:** Merging multiple v8 environments
- **Upgrade Path:** Phased migration approach
- **Risk Mitigation:** Test migration before production cutover

## 🐛 Troubleshooting

### Issue: No Instances Found
**Check:**
- V8 database connection in .env
- Deployment ID matches exactly
- Process IDs are comma-separated, no spaces

### Issue: Migration Fails
**Check:**
- V9 database has same process definitions
- KJAR is in kjars/ directory
- Both databases are accessible from Docker container

### Issue: Variables Missing
**Check:**
- Variable types are primitive (String, Integer, Boolean, Long, Float, Double)
- Variable names match exactly between v8 and v9
- No custom objects or collections used

## 📚 Next Steps

1. **Expand Testing:** Add more process instances with different states
2. **Test Complex Scenarios:** Try processes with parallel gateways, subprocesses
3. **Performance Testing:** Migrate larger batches (50-100 instances)
4. **Production Planning:** Document migration runbook for actual cutover

---

**Estimated Time:** 30 minutes  
**Difficulty:** Intermediate  
**Prerequisites:** V8 and V9 environments running  
**Outcome:** 6 process instances successfully migrated from v8 to v9