AWS Multi-CDN Control Lab

A Proof of Concept (PoC) demonstrating **CDN Vendor Redundancy** for High Availability. This lab shows how to achieve **automatic failover between multiple CDN providers** (Cloudflare and CloudFront) using a single origin server and intelligent DNS routing.

🎯 The Problem

**CDN Vendor Lock-in Risk**: Relying on a single CDN provider creates a single point of failure. When your primary CDN experiences outages, degraded performance, or regional issues, your entire service becomes unavailable until the CDN recovers.

💡 The Solution

This lab implements a **multi-CDN architecture** with:

**Single Origin**: One origin server that both CDN vendors pull content from, eliminating the need to maintain multiple origin infrastructures.

**Intelligent DNS Failover**: AWS Route 53 with health checks automatically routes users between Cloudflare (primary) and CloudFront (backup) based on CDN availability and geographic location.

**Manual Override Controls**: Route 53 Health Check manipulation provides immediate manual control for planned maintenance, emergency response, or operational requirements.

🏗 Architecture

## AWS Route 53 + Multi-CDN (Cloudflare & CloudFront) Hybrid Cloud High Availability Architecture

### 1. Architecture Overview

This solution utilizes AWS Route 53 as the traffic control plane, integrating Cloudflare (Primary CDN) and CloudFront (Backup/Regional CDN).


**Core Logic:**
- **Layered Routing**: Global and South America strategies are decoupled (Layer 1 & Layer 2) for CDN selection, but **share the same origin infrastructure**
- **Shared Origin**: Both Cloudflare and CloudFront CDNs (Global & SA) pull content from the same backend origin servers
- **Multi-source Fault Signals:**
  - Signal Source A: AWS CloudWatch Synthetics (external synthetic testing)
  - Signal Source B: User-built monitoring (internal business metrics, such as 5xx rate)
  - Aggregation Logic: HC-Auto = Signal Source A OR Signal Source B
- **Fine-grained Control**: Retain independent manual override capability for Global/South America regions through standard Route 53 health check manipulation

**Architecture Diagram:**

```mermaid
graph TD
    UserGlobal[Global Users]
    UserSA[South America Users]

    UserGlobal --> R53_Entry
    UserSA --> R53_Entry

    subgraph DNS["DNS Traffic Flow"]
        R53_Entry["api.cloudfront-ha.lab.zzhe.xyz"] -->|Default| R53_Global["Layer 2: Global"]
        R53_Entry -->|South America| R53_SA["Layer 2: South America"]

        R53_Global -->|Primary| CF_Global[Cloudflare CDN]
        R53_Global -->|Secondary| AWS_Global[CloudFront CDN]

        R53_SA -->|Primary| CF_SA[Cloudflare CDN]
        R53_SA -->|Secondary| AWS_SA[CloudFront CDN]
    end

    subgraph Origin["Single Origin Infrastructure"]
        ALB_Origin["ALB: Origin Server<br/>origin.cloudfront-ha.lab.zzhe.xyz"]
        EC2_Origin["EC2 Instances<br/>Application Server"]

        ALB_Origin --> EC2_Origin
    end

    subgraph Monitor["Monitoring & Control Plane"]
        CW_Canary["CloudWatch Synthetics<br/>External Testing"] -->|Fail| Alarm_AWS["Alarm: Canary Fail"]

        User_Sys["User Monitoring<br/>Prometheus/Zabbix"] -->|API Push| CW_Metric["CloudWatch Custom Metric<br/>App 5xx Rate"]
        CW_Metric -->|Threshold| Alarm_User["Alarm: App Metric High"]

        Alarm_AWS --> Composite_Alarm["Composite Alarm<br/>(OR Logic)"]
        Alarm_User --> Composite_Alarm

        Composite_Alarm --> HC_Auto["HC-Auto: Health Check"]
    end

    %% Both CDNs pull from single origin
    CF_Global -.->|Pulls from| ALB_Origin
    AWS_Global -.->|Pulls from| ALB_Origin
    CF_SA -.->|Pulls from| ALB_Origin
    AWS_SA -.->|Pulls from| ALB_Origin

    %% Health check directly controls CDN routing
    HC_Auto -.->|Controls Health| CF_Global
    HC_Auto -.->|Controls Health| CF_SA

    %% Origin monitoring (single origin)
    CW_Canary -.->|Monitors CDNs| CF_Global
    User_Sys -.->|Monitors Origin| ALB_Origin
```

### 2. Single Origin + Multi-CDN Architecture

**Key Concept**: This lab demonstrates **one origin server** with **two CDN vendors** for high availability. Route 53 handles failover between CDN vendors, not between origins.

**How it works:**
- **Single Origin**: One origin server (ALB + EC2) at `origin.cloudfront-ha.lab.zzhe.xyz`
- **Dual CDN Setup**:
  - **Cloudflare CDN** pulls content from the origin (Primary CDN)
  - **CloudFront CDN** pulls content from the same origin (Backup CDN)
- **Route 53 Failover**: Routes end users between Cloudflare and CloudFront based on CDN health
- **Geographic Routing**: Route 53 also provides geographic optimization (Global vs South America)

**Benefits:**
- **CDN Vendor Redundancy**: If one CDN provider has issues, automatic failover to the other
- **Single Origin Maintenance**: Only one application server to manage and update
- **Cost Effective**: No origin duplication, pay for CDN bandwidth only when needed
- **Vendor Independence**: Not locked into a single CDN provider

### 3. Detailed Configuration Steps

**Phase 1: Configure Multi-source Monitoring Signals (New)**

We need to integrate user monitoring data into AWS and merge it with AWS probe data.

**1. Integrate User-built Monitoring Data**

User systems (such as Prometheus, Zabbix, or log analysis systems) need to call AWS API to report key metrics.

Operation: Use AWS CLI or SDK (PutMetricData)

Example command:
```bash
aws cloudwatch put-metric-data \
  --namespace "Custom/AppMetrics" \
  --metric-name "ApiErrorRate" \
  --value 5.2 \
  --unit Percent \
  --dimensions Region=Global
```

**Create Alarm (Alarm B):**
- Create alarm in CloudWatch based on Custom/AppMetrics -> ApiErrorRate
- Threshold example: Trigger ALARM when error rate > 5% within 5 minutes

**2. Create Composite Alarm**

Merge "external synthetic testing alarm" and "internal business alarm" to avoid single-point false positives or missed alerts.

Prerequisites: Have Canary-based alarm (Alarm A) and user metric-based alarm (Alarm B)

Operation: Create Composite Alarm
- Logic expression: ALARM(Alarm_A) OR ALARM(Alarm_B)
- Effect: Composite alarm enters ALARM state when either external testing fails OR internal business metrics deteriorate

**Phase 2: Health Checks Setup**

Update basic signal definition, using composite alarm as automatic signal source.

**HC-Auto (Updated):**
- Type: CloudWatch Alarm
- Target: Select the Composite Alarm created in previous step
- Logic: When composite alarm triggers, this HC becomes Unhealthy

**Manual Control Health Checks:**
- **HC-Manual-Global**: Standard Route 53 health check for manual global control
- **HC-Manual-SA**: Standard Route 53 health check for manual South America control

**Calculated Health Checks (Logic Combination):**
- HC-Logic-Global (AND): HC-Auto + HC-Manual-Global
- HC-Logic-SA (AND): HC-Auto + HC-Manual-Global + HC-Manual-SA

**Phase 3: Configure Layered DNS Records - Manual Step-by-Step Guide**

This phase sets up the two-layer DNS routing structure that enables geographic-based CDN selection with automatic failover.

**Overview of DNS Layers:**
- **Layer 1 (Entry Point)**: Geographic routing that directs users to appropriate regional rules
- **Layer 2 (Regional Rules)**: Failover logic within each geographic region

### Step-by-Step DNS Configuration

#### Prerequisites
- Route 53 hosted zone for `cloudfront-ha.lab.zzhe.xyz`
- Health checks created from Phase 2: HC-Logic-Global and HC-Logic-SA
- CDN endpoints ready: Cloudflare and CloudFront configurations

---

#### **Step 1: Create Layer 2 Records (Regional Failover Rules)**

**1.1 Create Global Region Failover Records**

Navigate to Route 53 Console → Hosted Zones → `cloudfront-ha.lab.zzhe.xyz`

**Record 1: Global Primary (Cloudflare CDN)**
- **Record Name**: `global-rule.cloudfront-ha.lab.zzhe.xyz`
- **Record Type**: `CNAME`
- **Alias**: No
- **Value**: `your-site.cloudflare.com` (Cloudflare CDN endpoint that pulls from origin.cloudfront-ha.lab.zzhe.xyz)
- **Routing Policy**: `Failover`
- **Failover Type**: `Primary`
- **Health Check**: Select `HC-Logic-Global` (from Phase 2)
- **Record ID**: `global-primary`

**Record 2: Global Secondary (CloudFront CDN)**
- **Record Name**: `global-rule.cloudfront-ha.lab.zzhe.xyz` (same as primary)
- **Record Type**: `CNAME`
- **Alias**: Yes (for CloudFront distribution)
- **Value**: `d123abc456.cloudfront.net` (CloudFront distribution that pulls from origin.cloudfront-ha.lab.zzhe.xyz)
- **Routing Policy**: `Failover`
- **Failover Type**: `Secondary`
- **Health Check**: None (secondary doesn't need health check)
- **Record ID**: `global-secondary`

**1.2 Create South America Region Failover Records**

**Record 3: South America Primary (Cloudflare CDN)**
- **Record Name**: `sa-rule.cloudfront-ha.lab.zzhe.xyz`
- **Record Type**: `CNAME`
- **Alias**: No
- **Value**: `your-site.cloudflare.com` (same Cloudflare CDN endpoint)
- **Routing Policy**: `Failover`
- **Failover Type**: `Primary`
- **Health Check**: Select `HC-Logic-SA` (from Phase 2)
- **Record ID**: `sa-primary`

**Record 4: South America Secondary (CloudFront CDN)**
- **Record Name**: `sa-rule.cloudfront-ha.lab.zzhe.xyz` (same as SA primary)
- **Record Type**: `CNAME`
- **Alias**: Yes (for CloudFront distribution)
- **Value**: `d123abc456.cloudfront.net` (same CloudFront distribution)
- **Routing Policy**: `Failover`
- **Failover Type**: `Secondary`
- **Health Check**: None
- **Record ID**: `sa-secondary`

---

#### **Step 2: Create Layer 1 Record (Geographic Entry Point)**

**2.1 Create the Main API Entry Point**

**Record 5: Default/Global Geographic Rule**
- **Record Name**: `api.cloudfront-ha.lab.zzhe.xyz`
- **Record Type**: `CNAME`
- **Alias**: No
- **Value**: `global-rule.cloudfront-ha.lab.zzhe.xyz`
- **Routing Policy**: `Geolocation`
- **Location**: `Default` (catches all traffic not matching other rules)
- **Record ID**: `api-default-global`

**Record 6: South America Geographic Rule**
- **Record Name**: `api.cloudfront-ha.lab.zzhe.xyz` (same as default)
- **Record Type**: `CNAME`
- **Alias**: No
- **Value**: `sa-rule.cloudfront-ha.lab.zzhe.xyz`
- **Routing Policy**: `Geolocation`
- **Location**: `South America` (select continent)
- **Record ID**: `api-sa-specific`

---

#### **Step 3: Verification and Testing**

**3.1 DNS Propagation Check**
```bash
# Test from different geographic locations
dig api.cloudfront-ha.lab.zzhe.xyz
nslookup api.cloudfront-ha.lab.zzhe.xyz

# Test specific regional rules directly
dig global-rule.cloudfront-ha.lab.zzhe.xyz
dig sa-rule.cloudfront-ha.lab.zzhe.xyz
```

**3.2 Health Check Verification**
- Go to Route 53 Console → Health Checks
- Verify `HC-Logic-Global` and `HC-Logic-SA` show "Success" status
- Check CloudWatch alarms are in "OK" state

**3.3 End-to-End Flow Test**
```bash
# Test from Global location (should hit global-rule → Cloudflare)
curl -I https://api.cloudfront-ha.lab.zzhe.xyz

# Test failover by triggering health check failure
# (Use simulation scripts from Phase 4)
```

---

#### **Step 4: Advanced Configuration (Optional)**

**4.1 Add Additional Geographic Regions**
If you want more granular control, add specific countries/regions:

```
- Europe: europe-rule.cloudfront-ha.lab.zzhe.xyz
- Asia: asia-rule.cloudfront-ha.lab.zzhe.xyz
```

**4.2 Configure TTL Values**
- Set appropriate TTL values for faster failover:
  - Layer 1 records: 60 seconds
  - Layer 2 primary records: 60 seconds
  - Layer 2 secondary records: 300 seconds

**4.3 Add Monitoring**
```bash
# Set up CloudWatch monitoring for DNS queries
aws logs create-log-group --log-group-name "/aws/route53/api.cloudfront-ha.lab.zzhe.xyz"
```

---

#### **Step 5: Configuration Validation Checklist**

- [ ] Layer 2 Global failover: `global-rule.cloudfront-ha.lab.zzhe.xyz` created
- [ ] Layer 2 SA failover: `sa-rule.cloudfront-ha.lab.zzhe.xyz` created
- [ ] Layer 1 default route: `api.cloudfront-ha.lab.zzhe.xyz` → `global-rule`
- [ ] Layer 1 SA route: `api.cloudfront-ha.lab.zzhe.xyz` → `sa-rule`
- [ ] Health checks attached to primary records only
- [ ] DNS propagation completed (24-48 hours max)
- [ ] End-to-end testing successful from multiple geographic locations

**Final DNS Structure:**
```
api.cloudfront-ha.lab.zzhe.xyz (Entry Point)
├── Default Location → global-rule.cloudfront-ha.lab.zzhe.xyz
│   ├── Primary → your-site.cloudflare.com (with HC-Logic-Global)
│   └── Secondary → d123abc456.cloudfront.net
└── South America → sa-rule.cloudfront-ha.lab.zzhe.xyz
    ├── Primary → your-site.cloudflare.com (with HC-Logic-SA)
    └── Secondary → d123abc456.cloudfront.net

Single Origin: origin.cloudfront-ha.lab.zzhe.xyz
├── Cloudflare CDN pulls from origin
└── CloudFront CDN pulls from origin
```

### 4. Operations Scenario Testing

Scenario changes after adding user monitoring:

| Scenario | Trigger Condition | Result |
|----------|------------------|--------|
| External Network Outage | AWS Canary detects Cloudflare failure → Alarm A triggers → Composite Alarm triggers | Automatic switch to CloudFront |
| Internal Business Exception | User system detects API error rate spike → Push metrics → Alarm B triggers → Composite Alarm triggers | Automatic switch to CloudFront |
| False Positive Filtering (Optional) | If changed to AND logic, requires both parties to alarm simultaneously for switching (usually not recommended, OR logic recommended for high availability) | (Depends on composite alarm logic) |
| Manual Force Switchover | Administrator manually disables health check | Force switch to CloudFront |

### 5. Manual Failover Control

The architecture provides granular manual control through **standard Route 53 health check manipulation** for planned maintenance or emergency situations.

#### **How Manual Control Works**

The system uses **standard Route 53 health checks** for CDN routing:
- **HC-Logic-Global**: Calculated health check controlling Cloudflare routing for global traffic
- **HC-Logic-SA**: Calculated health check controlling Cloudflare routing for South America traffic

**Manual Override**: Simply **disable** the manual control health check (HC-Manual-Global or HC-Manual-SA) to force traffic to CloudFront (secondary CDN). When disabled, the calculated health check becomes unhealthy and Route 53 automatically routes to the secondary record (CloudFront).

#### **Manual Override Methods**

**1. Global Manual Failover (Affects All Regions)**
```bash
# Force ALL traffic to secondary CDN (CloudFront) by disabling manual control health check
aws route53 change-health-check \
  --health-check-id HC-MANUAL-GLOBAL \
  --disabled

# Restore to automatic behavior by re-enabling manual control health check
aws route53 change-health-check \
  --health-check-id HC-MANUAL-GLOBAL \
  --enable
```

**2. South America Specific Manual Failover**
```bash
# Force ONLY South America traffic to secondary CDN
aws route53 change-health-check \
  --health-check-id HC-MANUAL-SA \
  --disabled

# Restore South America to automatic behavior
aws route53 change-health-check \
  --health-check-id HC-MANUAL-SA \
  --enable
```

**3. Via AWS Console**
1. Navigate to **Route 53 Console → Health Checks**
2. Find the manual control health checks:
   - `HC-Manual-Global`: Controls global manual override
   - `HC-Manual-SA`: Controls South America manual override
3. **Disable** health check to force traffic to CloudFront
4. **Enable** health check to restore automatic routing

**4. Via Simulation Scripts**
```bash
# Use the provided simulation scripts
python3 simulation/disable_health_check.py --region global
python3 simulation/disable_health_check.py --region sa

# Restore automatic behavior
python3 simulation/enable_health_check.py --region global
python3 simulation/enable_health_check.py --region sa
```

#### **Manual Failover Scenarios**

| Scenario | HC-Auto | HC-Manual-Global | HC-Manual-SA | Global Traffic | SA Traffic |
|----------|---------|------------------|--------------|----------------|------------|
| **Normal Operation** | ✅ Healthy | ✅ Enabled | ✅ Enabled | Primary CDN (Cloudflare) | Primary CDN (Cloudflare) |
| **Automatic Failover** | ❌ Unhealthy | ✅ Enabled | ✅ Enabled | Secondary CDN (CloudFront) | Secondary CDN (CloudFront) |
| **Global Manual Failover** | ✅ Healthy | ❌ **Disabled** | ✅ Enabled | **Secondary CDN (CloudFront)** | **Secondary CDN (CloudFront)** |
| **SA Manual Failover** | ✅ Healthy | ✅ Enabled | ❌ **Disabled** | Primary CDN (Cloudflare) | **Secondary CDN (CloudFront)** |
| **Both Manual Override** | ✅ Healthy | ❌ Disabled | ❌ Disabled | **Secondary CDN (CloudFront)** | **Secondary CDN (CloudFront)** |
| **Emergency Override** | ❌ Unhealthy | ❌ Disabled | ❌ Disabled | **Secondary CDN (CloudFront)** | **Secondary CDN (CloudFront)** |

#### **Use Cases for Manual Control**

- **Planned Maintenance**: Switch traffic before performing CDN maintenance
- **Performance Testing**: Route specific regions to test CDN performance
- **Emergency Response**: Immediate traffic control without waiting for health checks
- **Cost Optimization**: Route traffic to preferred CDN based on pricing
- **Compliance**: Meet regional data sovereignty requirements

#### **Best Practices**

- **Document Changes**: Always log manual failover actions and reasons
- **Coordinate Teams**: Notify relevant teams before manual switches
- **Monitor Impact**: Watch metrics after manual failover to ensure proper operation
- **Plan Restoration**: Have clear procedures for returning to automatic mode
- **Test Regularly**: Practice manual failover procedures during maintenance windows

### 6. Advanced Failover Control: Disable Automatic Failback

In production environments, you may want **automatic failover** (fast response to outages) but **manual failback** (controlled recovery). This prevents "flapping" between CDNs when the primary CDN has intermittent issues.

#### **The Challenge: Automatic Failback**

By default, when HC-Auto recovers to healthy, traffic automatically returns to Cloudflare:
```
Time 10:00 - Cloudflare healthy → Traffic on Cloudflare
Time 10:05 - Cloudflare unhealthy → Traffic switches to CloudFront (Auto)
Time 10:07 - Cloudflare healthy again → Traffic switches back to Cloudflare (Auto)
Time 10:09 - Cloudflare unhealthy → Traffic switches to CloudFront again (Auto)
```

This "flapping" can cause service instability.

#### **Solution 1: Sticky Composite Alarm (Recommended)**

Create a composite alarm that requires **manual reset** after triggering:

**Step 1: Create Sticky Composite Alarm**
```bash
# Create composite alarm that stays in ALARM state until manually reset
aws cloudwatch put-composite-alarm \
  --alarm-name "Composite-Alarm-CDN-Health-Sticky" \
  --alarm-rule "ALARM(Alarm_AWS) OR ALARM(Alarm_User)" \
  --actions-enabled true \
  --treat-missing-data ignore \
  --alarm-actions "arn:aws:sns:region:account:manual-intervention-topic"
```

**Step 2: Update HC-Auto to Use Sticky Alarm**
```bash
# Point HC-Auto to the sticky composite alarm
aws route53 change-health-check \
  --health-check-id "HC-AUTO-ID" \
  --alarm-name "Composite-Alarm-CDN-Health-Sticky"
```

**Step 3: Manual Failback Process**
```bash
# When ready to failback, manually reset the alarm
aws cloudwatch set-alarm-state \
  --alarm-name "Composite-Alarm-CDN-Health-Sticky" \
  --state-value "OK" \
  --state-reason "Manual approval for failback to primary CDN"
```

**Behavior with Sticky Alarm:**
- ✅ **Automatic Failover**: When CDN issues occur → Traffic goes to CloudFront
- ❌ **No Automatic Failback**: Alarm stays in ALARM state
- 🔧 **Manual Failback**: Operations team resets alarm when ready

#### **Solution 2: Asymmetric Health Checks**

Use different health checks for failover (sensitive) and failback (manual control):

**Step 1: Create Failover Health Check (Sensitive)**
```bash
# Sensitive health check - fails quickly for fast failover
aws route53 create-health-check \
  --caller-reference "failover-check-$(date +%s)" \
  --health-check-config \
    Type=CLOUDWATCH_METRIC,AlarmRegion=us-east-1,AlarmName=Composite-Alarm-CDN-Health,InsufficientDataHealthStatus=Failure,RequestInterval=30,FailureThreshold=2
```

**Step 2: Create Failback Health Check (Manual Control)**
```bash
# Create health check that monitors manual approval endpoint
aws route53 create-health-check \
  --caller-reference "failback-check-$(date +%s)" \
  --health-check-config \
    Type=HTTPS,ResourcePath=/approve-failback,FullyQualifiedDomainName=control.cloudfront-ha.lab.zzhe.xyz,Port=443,RequestInterval=30,FailureThreshold=3
```

**Step 3: Create Asymmetric Logic Structure**
```bash
# Create separate calculated health checks for failover and failback
aws route53 create-health-check \
  --caller-reference "hc-logic-global-failover-$(date +%s)" \
  --health-check-config \
    Type=CALCULATED,ChildHealthChecks=["HC-Failover-ID","HC-Switch-Global-ID"],HealthThreshold=2,CloudWatchAlarmRegion=us-east-1

aws route53 create-health-check \
  --caller-reference "hc-logic-global-failback-$(date +%s)" \
  --health-check-config \
    Type=CALCULATED,ChildHealthChecks=["HC-Failback-ID","HC-Switch-Global-ID"],HealthThreshold=2,CloudWatchAlarmRegion=us-east-1
```

**Step 4: Configure DNS Records with Asymmetric Logic**
- **Primary Record (Cloudflare)**: Uses `HC-Logic-Global-Failback` (requires manual approval)
- **Secondary Record (CloudFront)**: Uses `HC-Logic-Global-Failover` (automatic)

**Step 5: Manual Failback Approval**
```bash
# Create endpoint that approves failback
curl -X POST https://control.cloudfront-ha.lab.zzhe.xyz/approve-failback \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"region": "global", "approved_by": "operations-team"}'
```

#### **Failover/Failback Behavior Comparison**

| Scenario | Default Behavior | Sticky Alarm (Solution 1) | Asymmetric Checks (Solution 2) |
|----------|------------------|----------------------------|--------------------------------|
| **CDN Issue Occurs** | Auto failover to CloudFront | Auto failover to CloudFront | Auto failover to CloudFront |
| **CDN Issue Resolves** | **Auto failback to Cloudflare** | Stays on CloudFront | Stays on CloudFront |
| **Manual Intervention** | N/A | Reset alarm → Failback | Approve endpoint → Failback |
| **Complexity** | Low | Medium | High |
| **Operational Control** | None | High | Very High |

#### **Implementation Recommendations**

**For Most Environments (Solution 1 - Sticky Alarm):**
- Simple to implement and understand
- Clear manual approval process
- Good balance of automation and control

**For Complex Environments (Solution 2 - Asymmetric Checks):**
- Maximum operational flexibility
- Separate approval workflows possible
- Integration with change management systems

### 7. Additional Notes

- **Access Authentication**: Servers running user monitoring systems need IAM Role or AK/SK configured with `cloudwatch:PutMetricData` permissions
- **Cost Optimization**: Custom Metrics and Alarms incur minimal fees, negligible compared to business high availability value
- **Latency**: PutMetricData to Alarm trigger typically has 1-3 minute delay (depends on Standard Resolution vs High Resolution). For second-level response, recommend enabling CloudWatch High Resolution Metrics (1-second granularity)

Directory Structure

aws-multi-cdn-control-lab/
├── infrastructure/    # Pulumi IaC for Route 53, Health Checks, single origin ALB, and CDN configurations
├── control-plane/     # EC2 instances for the Config API behind ALB
├── client-app/        # React Web App demonstrating the "Smart Client"
└── simulation/        # Python scripts to break CDN endpoints and manipulate health checks


Traffic Flow

**Single Origin Infrastructure:**
- **Origin Server**: Single ALB + EC2 instances at `origin.cloudfront-ha.lab.zzhe.xyz`

**CDN Vendor Setup:**
- **Cloudflare CDN**: Configured to pull content from `origin.cloudfront-ha.lab.zzhe.xyz` (Primary CDN)
- **CloudFront CDN**: Configured to pull content from `origin.cloudfront-ha.lab.zzhe.xyz` (Backup CDN)

**Important**: This lab demonstrates **CDN vendor failover**, not origin failover. Both Cloudflare and CloudFront cache content from the same single origin server. Route 53 routes end users to the healthier CDN vendor.

**Control Plane**: A highly available API running on EC2 instances behind ALB that manages the health check states for manual override control.

Client:

Fetches configuration on startup.

Attempts Primary.

If Primary fails (Chaos induced), automatically retries Secondary without waiting for DNS updates.

🚀 Getting Started

Prerequisites

AWS CLI (configured with Administrator access for the lab)

Pulumi CLI

Go 1.19+ (for infrastructure code)

Node.js v16+ (for the client)

Python 3.9+ (for simulation scripts)

Domain: Access to zzhe.xyz domain for creating lab subdomains (*.cloudfront-ha.lab.zzhe.xyz)

1. Deploy Infrastructure

Provision the ALB endpoints, EC2 instances, Route 53 zones, and Health Checks with cloudfront-ha.lab.zzhe.xyz subdomains.

cd infrastructure
pulumi stack init
pulumi up


Note the outputs:
- config_api_url: https://api.cloudfront-ha.lab.zzhe.xyz
- origin_server_url: https://origin.cloudfront-ha.lab.zzhe.xyz
- cloudflare_cname: (provided by Cloudflare setup)
- cloudfront_distribution: (AWS CloudFront distribution domain)

2. Configure Client

Update the client configuration with the ALB URLs from the Pulumi output.

cd ../client-app
# Edit .env or src/config.ts with the infrastructure URLs
npm install
npm run dev


3. Run Simulation (Chaos Mode)

Open the web app. You should see a Green status (Connected to Primary).

Scenario A: Simulate Technical Failure
Run the script to make the "Mock Primary" return 503 errors.

python3 simulation/break_primary.py


Observe the Client App automatically switch to Secondary (Yellow/Orange status).

Scenario B: Manual Kill Switch
Force traffic over by disabling Cloudflare health checks.

python3 simulation/disable_health_check.py --region global


💰 Cost Warning

This lab creates real AWS resources.

**Route 53 Health Checks**: This lab uses standard Route 53 Health Checks for manual override control, which is cost-effective at approximately $1-2/month for the health checks and alarms used in this architecture.

ALB + EC2: Costs for Application Load Balancers and EC2 instances running the mock services.

CloudWatch: Small costs for alarms and metrics.

Cleanup: Always run pulumi destroy when finished!

📜 License

MIT