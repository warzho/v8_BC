# Banking Process Migration Templates

A collection of 10 BPMN process definitions designed for BAMOE v8→v9 ADM migration testing, representing realistic banking workflows with varying complexity levels.

## 🎯 Purpose

These processes are specifically designed to:
- **Test ADM migration capabilities** from BAMOE v8 to v9
- **Use only primitive variable types** (String, Integer, Boolean, Long, Float, Double) - compatible with ADM v0.1.0
- **Represent realistic banking scenarios** that resonate with financial services clients
- **Provide a spectrum of complexity** from simple approvals to multi-stage orchestrations

## 📋 Process Inventory

| # | Process ID | Name | Complexity | Variables | Human Tasks | Description |
|---|---|---|---|---|---|---|
| 1 | `account-opening` | Account Opening | ⭐ Simple | 6 | 1 | Basic account creation with compliance check |
| 2 | `wire-transfer` | Wire Transfer | ⭐ Simple | 7 | 1 | Domestic wire transfer with fraud screening |
| 3 | `credit-card-application` | Credit Card Application | ⭐⭐ Medium | 9 | 2 | Credit card approval with credit check |
| 4 | `loan-origination` | Personal Loan Origination | ⭐⭐ Medium | 11 | 3 | Personal loan application and underwriting |
| 5 | `mortgage-application` | Mortgage Application | ⭐⭐⭐ Complex | 14 | 4 | Residential mortgage with appraisal |
| 6 | `trade-settlement` | Trade Settlement | ⭐⭐ Medium | 10 | 2 | Securities trade clearing and settlement |
| 7 | `kyc-refresh` | KYC Refresh | ⭐⭐ Medium | 8 | 2 | Customer due diligence update |
| 8 | `fraud-investigation` | Fraud Investigation | ⭐⭐⭐ Complex | 12 | 3 | Suspicious activity investigation |
| 9 | `commercial-lending` | Commercial Lending | ⭐⭐⭐⭐ Very Complex | 16 | 5 | Business loan with financial analysis |
| 10 | `regulatory-reporting` | Regulatory Reporting | ⭐⭐ Medium | 9 | 2 | Compliance report generation and review |

## 🏗️ Architecture

### Variable Type Compliance

All processes use **only primitive types** to ensure ADM compatibility:

```
✅ Supported:
- String (customer names, IDs, statuses)
- Integer (amounts in cents, counts)
- Boolean (flags, approvals)
- Long (timestamps, large amounts)
- Float (percentages, rates)
- Double (precise calculations)

❌ Not Used:
- Custom objects (Customer, Account, etc.)
- Collections (List, Map)
- Date objects (use Long timestamps instead)
```

### Process Patterns

Each process follows banking industry patterns:

1. **Intake** - Capture request details
2. **Validation** - Check data completeness
3. **Risk Assessment** - Automated screening (simulated via script tasks)
4. **Human Review** - Approval tasks for exceptions
5. **Execution** - Process the transaction
6. **Notification** - Inform stakeholders

## 📦 File Structure

```
banking-migration-processes/
├── README.md (this file)
├── DEPLOYMENT-GUIDE.md (how to deploy to v8 and v9)
├── MIGRATION-GUIDE.md (how to execute the migration)
├── simple/
│   ├── account-opening.bpmn
│   └── wire-transfer.bpmn
├── medium/
│   ├── credit-card-application.bpmn
│   ├── loan-origination.bpmn
│   ├── trade-settlement.bpmn
│   ├── kyc-refresh.bpmn
│   └── regulatory-reporting.bpmn
├── complex/
│   ├── mortgage-application.bpmn
│   ├── fraud-investigation.bpmn
│   └── commercial-lending.bpmn
└── test-data/
    ├── account-opening-test-data.json
    ├── wire-transfer-test-data.json
    └── ... (test data for each process)
```

## 🚀 Quick Start

### Step 1: Deploy to V8 Business Central

1. Access Business Central: http://localhost:8180/business-central
2. Login: `admin` / `admin`
3. Create Space: `banking-migration-demo`
4. Create Project: `banking-processes`
5. Import each BPMN file via "Import Asset"
6. Build & Deploy the project

### Step 2: Create Process Instances

Use the REST API or Business Central UI to start instances:

```bash
# Example: Start account opening process
curl -X POST http://localhost:8080/kie-server/services/rest/server/containers/banking-processes_1.0.0/processes/account-opening/instances \
  -H "Content-Type: application/json" \
  -u admin:admin \
  -d '{
    "customerId": "CUST-12345",
    "accountType": "CHECKING",
    "initialDeposit": 50000,
    "branchCode": "NYC-001",
    "employeeId": "EMP-789",
    "requiresManagerApproval": false
  }'
```

### Step 3: Migrate to V9

1. Ensure V9 service has the same BPMN files deployed
2. Configure ADM tool with v8 and v9 database connections
3. Run migration: `docker compose -f docker-compose.yaml up`
4. Verify in V9 database

### Step 4: Verify Migration

Query V9 PostgreSQL to confirm migrated instances:

```sql
-- Check migrated process instances
SELECT 
    process_instance_id,
    process_id,
    process_version,
    state,
    start_date
FROM process_instances
WHERE process_id IN (
    'account-opening',
    'wire-transfer',
    'credit-card-application'
    -- ... add others
)
ORDER BY start_date DESC;

-- Check process variables
SELECT 
    pi.process_instance_id,
    pi.process_id,
    pv.variable_name,
    pv.variable_value
FROM process_instances pi
JOIN process_instance_variables pv ON pi.process_instance_id = pv.process_instance_id
WHERE pi.process_id = 'account-opening'
ORDER BY pi.process_instance_id, pv.variable_name;
```

## 💼 Business Context for Client Conversations

### Why These Processes Matter

**For a major bank**, these processes represent:

1. **Customer Onboarding** (account-opening, credit-card-application)
   - *Business Impact:* Faster time-to-revenue, improved customer experience
   - *Regulatory:* KYC/AML compliance, audit trail

2. **Transaction Processing** (wire-transfer, trade-settlement)
   - *Business Impact:* Operational efficiency, fraud prevention
   - *Regulatory:* OFAC screening, transaction monitoring

3. **Lending Operations** (loan-origination, mortgage-application, commercial-lending)
   - *Business Impact:* Risk-adjusted returns, portfolio management
   - *Regulatory:* Fair lending, credit reporting

4. **Risk & Compliance** (kyc-refresh, fraud-investigation, regulatory-reporting)
   - *Business Impact:* Reduced losses, regulatory fines avoidance
   - *Regulatory:* BSA/AML, FCPA, Dodd-Frank

### Migration Value Proposition

**Why migrate from v8 to v9?**

- **Modern Architecture:** Quarkus-based, cloud-native, faster startup
- **Lower TCO:** Reduced memory footprint, better resource utilization
- **Developer Experience:** Hot reload, better tooling, OpenAPI support
- **Future-Proof:** Active development, IBM support, community ecosystem

**ADM Tool Value:**

- **Zero Downtime:** Migrate running instances without stopping business
- **Data Integrity:** Preserves process state, variables, history
- **Selective Migration:** Choose which instances to migrate
- **Rollback Capability:** Can revert if issues arise

## 🔍 Variable Naming Conventions

All processes follow consistent naming:

- **IDs:** `customerId`, `accountId`, `transactionId` (String)
- **Amounts:** `amountInCents` (Integer for precision), `amountInDollars` (Double for display)
- **Timestamps:** `requestTimestamp`, `approvalTimestamp` (Long - milliseconds since epoch)
- **Flags:** `isApproved`, `requiresReview`, `isFraudulent` (Boolean)
- **Codes:** `branchCode`, `productCode`, `statusCode` (String)
- **Scores:** `creditScore`, `riskScore` (Integer 0-1000)
- **Rates:** `interestRate`, `feePercentage` (Float)

## 📊 Testing Strategy

### Recommended Test Scenarios

1. **Happy Path** - All approvals, no exceptions (2 instances per process)
2. **Exception Path** - Requires manual review (1 instance per process)
3. **Rejection Path** - Fails validation or approval (1 instance per process)
4. **In-Flight** - Leave some instances at human task wait states (1 instance per process)

**Total:** ~50 process instances across 10 process definitions

### Migration Verification Checklist

- [ ] All process instances migrated (count matches)
- [ ] Process variables preserved (spot check 5 instances)
- [ ] Process state correct (Active, Completed, etc.)
- [ ] Human tasks migrated (if applicable)
- [ ] Process history preserved
- [ ] No data corruption (validate key business variables)
- [ ] V9 service can continue execution (complete a migrated task)

## 🎓 Learning Objectives

By working through these processes, you'll understand:

1. **BPMN Fundamentals:** Start events, tasks, gateways, end events
2. **Process Variables:** How data flows through a process
3. **Human Tasks:** Work assignment, completion, escalation
4. **Service Tasks:** Automated steps (simulated with scripts)
5. **Gateways:** Conditional routing, parallel execution
6. **Migration Mechanics:** What ADM preserves, what it doesn't
7. **Database Schema:** How BAMOE stores process instances

## 📚 Additional Resources

- **BAMOE Documentation:** https://docs.redhat.com/en/documentation/red_hat_build_of_kogito
- **BPMN 2.0 Spec:** https://www.omg.org/spec/BPMN/2.0/
- **ADM Tool Guide:** (included in ADM v0.1.0 package)
- **Migration Best Practices:** See MIGRATION-GUIDE.md

## 🤝 Support

For questions or issues:
1. Check DEPLOYMENT-GUIDE.md for setup instructions
2. Check MIGRATION-GUIDE.md for migration steps
3. Review V8-BUSINESS-CENTRAL-SETUP-GUIDE.md for v8 environment
4. Review BAMOE-V9-MIGRATION-TARGET-SETUP-GUIDE.md for v9 environment

---

**Version:** 1.0.0  
**Last Updated:** 2026-04-10  
**Compatible with:** BAMOE 8.x (source), BAMOE 9.4.0 (target), ADM v0.1.0