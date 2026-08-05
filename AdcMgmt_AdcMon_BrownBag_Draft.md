# ADC Management & Monitoring (AdcMgmt/AdcMon) Brown Bag Session

**Presenter:** Mike Biancaniello  
**Duration:** 30-60 minutes (including Q&A)  
**Target Audience:** Engineers across ADC, Snowsk8s, AI pillars  

---

## 1. What Is It?

**AdcMgmt/AdcMon** is the unified management and monitoring system for ADCv3 (Application Delivery Controller v3) containers across our datacenter infrastructure.

**Two Main Components:**

- **AdcMgt**: Management platform for creating, configuring, and operating ADCv3 instances. Provides a ServiceNow-based interface to define, manage, and track container deployments.
- **Adcmon**: Monitoring service that watches ADCv3 container health and manages BGP routes to provide similar functionality to RHI (Router Health Injection).

**Key Architecture Concept:**

- An **"Adcmon Service"** = a set of redundant adcmon hosts monitoring the exact same set of containers
- Adcmon services are **rack-based**: each adcmon service monitors only containers in its rack
- **Minimum requirement**: at least one adcmon service must exist in every ADCv3 rack
- **Automatic assignment**: when a new Deploy Host (container) is created, an adcmon service from the same rack is automatically assigned (can be manually overridden)

---

## 2. Why Do We Need It?

### Problems Solved

- **Container Health Monitoring**: Automated health checks for ADCv3 instances instead of manual status tracking
- **Route Failover Automation**: Automatic BGP route advertisement/withdrawal based on container health (similar to RHI)
- **Operational Simplicity**: Single pane of glass for managing thousands of ADC instances across datacenters
- **Geographic Scalability**: Enables consistent ADC management across multiple datacenters and geographic regions
- **Infrastructure Management**: Centralized deployment infrastructure (Deploy Hosts, Deploy Types, Pools, etc.)

### Value Proposition

- **High Availability**: Automatic failover via BGP route updates when containers go unhealthy - no manual intervention needed
- **Reduced Manual Intervention**: Eliminates manual route tweaking and status updates for individual container failures
- **Compliance & Standardization**: Single system of record for ADC configuration and state across all environments
- **Cross-Region Knowledge Sharing**: Enables follow-the-sun support model (Q3/Q4 goal) - any engineer can support ADC in any region
- **Scalability**: Can manage deployments from small single-container labs to large multi-datacenter deployments

---

## 3. How Does It Work?

### High-Level Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  AdcMgt (Management Plane) - ServiceNow Application            │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │  Deploy Hosts    │  │  Deploy Types    │  │   Pools      │  │
│  │  (Containers)    │  │  (Configuration) │  │  (Members)   │  │
│  └────────┬─────────┘  └──────────────────┘  └──────────────┘  │
│           │                                                     │
│           └─────────┬────────────────────────────────────────┘  │
└────────────────────┼──────────────────────────────────────────┘
                     │ Assigns & Associates
                     ↓
┌────────────────────────────────────────────────────────────────┐
│  Adcmon Services (Monitoring Plane)                            │
│  Per-Rack Redundant Monitoring Hosts                           │
│  ┌────────────────┐                                            │
│  │  Health Check  │  - Poll container status continuously    │
│  │  BGP Management│  - Update BGP route advertisements        │
│  │  Route Updates │  - React to state changes in real-time   │
│  └────────────────┘                                            │
└────────────┬───────────────────────────────────────────────────┘
             │ BGP Route Updates
             ↓
    ┌──────────────────┐
    │  Network Routers │ ← Traffic paths determined by BGP routes
    └──────────────────┘
```

### Container Monitoring & Failover Flow

1. **Container Creation** → Deploy Host created in AdcMgt
2. **Service Assignment** → Adcmon service in same rack is automatically assigned (or manually override)
3. **Continuous Monitoring** → Adcmon hosts poll container health at regular intervals
4. **State Change Reaction**:
   - **Container Healthy** → BGP routes advertised → traffic flows to it
   - **Container Unhealthy** → BGP routes withdrawn → traffic fails over to other instances
   - **Container Recovered** → BGP routes re-advertised → traffic restored

### Key Database Tables

| Table | Purpose | Key Fields | Dependencies |
|-------|---------|-----------|--------------|
| `adcmon` | Adcmon service definitions - defines monitoring services per rack | rack (auto), datacenter (derived) | Must have at least one per rack |
| `adcmonhosts` | Individual adcmon monitor hosts - the actual servers doing monitoring | adcmon_service, host, rack (derived), datacenter (derived) | Links to adcmon service |
| `deployhosts` | Deploy instances (the ADCv3 containers being monitored) | adcmon_service (auto-assigned), host | Assigned to adcmon_service in same rack |

### Automation & Integration

- **Provisioning system** → Creates Deploy Host in AdcMgt
- **AdcMgt** → Auto-selects adcmon service from same rack
- **Adcmon** → Begins health polling immediately
- **Adcmon** → Updates BGP routes based on health status
- **Network routers** → Make routing decisions based on BGP advertisements

---

## 4. How Do Customers and Automations Interact With It?

### Customer/External Interfaces

**Direct API Users:**
- CNS (Customer Networking System) users request new ADC deployments
- Provisioning systems query ADCv3 container status via REST APIs
- Monitoring/observability systems scrape health metrics from Adcmon
- Automation platforms check deployment readiness

**BGP Route Recipients:**
- Core network routers receive BGP route updates from Adcmon hosts
- Make routing decisions based on advertised/withdrawn routes
- Customers experience this as automatic failover between ADC instances - no customer code changes needed

**Impact to Customers:**
- If container goes unhealthy → BGP routes withdrawn → traffic automatically fails over to healthy instances
- No customer awareness needed - happens transparently
- Customers see consistent availability even during container issues

### Integration Points

**Inbound to AdcMgt:**
- Provisioning scripts create Deploy Hosts via REST API or UI wizard
- Configuration management systems push policy updates (TLS, pool members, etc.)
- Automation queries container inventory and current status
- External services check deployment readiness

**Outbound from Adcmon:**
- BGP neighbor sessions advertise/withdraw routes to network routers
- Metrics pushed to monitoring system (Prometheus, Grafana, etc.)
- Logs sent to centralized logging platform (Splunk, etc.)
- Webhooks to incident management systems on health events
- REST API exposes status for external query

---

## 5. How Do I As an Engineer Interact With It?

### Access & Permissions

**Getting Access:**
1. Submit request via **PAGA tool** (in ServiceNow at datacenter.service-now.com)
2. **Must request access for each environment separately** (Dev, Staging, Prod, etc.)
3. See **KB0538549** for access matrix or check the access_roles documentation

**Role-Based Access (choose what you need):**

| Role | PAGA Persona | Best For | Capabilities |
|------|--------------|----------|--------------|
| `x_snc_adcmgt.ro` | Read-Only | Troubleshooting, viewing status | View all data - sufficient for management or troubleshooting |
| `x_snc_adcmgt.operations` | Operator | Day-to-day ops, fixing issues | Manage ADC definitions, commit/deprovision containers, fix issues |
| `x_snc_adcmgt.infra_admin` | Infra Admin | Infrastructure setup | Create Deploy Hosts/Types, manage pools, setup IPAM, params |
| `x_snc_adcmgt.adcmon_admin` | Adcmon Admin | Adcmon-specific work | Administer adcmon services, override service assignments |
| `x_snc_adcmgt.expert` | Expert | Low-level debugging | For true experts only - direct access to internal structures |

**Limited/Specialized Roles:**
- `x_snc_adcmgt.pool_admin` - Manage pools and pool members only
- `x_snc_adcmgt.settings_admin` - Manage instance/pool/ADC settings
- `x_snc_adcmgt.mtls_admin` - Manage mTLS certificates

### Common Engineer Tasks

| Task | Role Needed | Tools | Process |
|------|-------------|-------|---------|
| Check container health | Read-Only or higher | AdcMgmt UI | Query `deployhosts` table, check status column |
| Verify BGP routes advertised | Adcmon Admin | AdcMgmt UI + SSH to router | Query adcmon service status; run `show ip bgp` |
| Troubleshoot container issues | Read-Only or higher | AdcMgmt UI, logs | View container status, check adcmon health logs |
| Update ADC config (pools, members, aliases) | Operations | AdcMgmt UI - Config section | Edit via web UI or REST API |
| Modify container deployment (size, type, etc.) | Operations | AdcMgmt UI - Deploy section | Update deployment_size, deployment_type fields |
| Create new Deploy Host/Deploy Type | Infrastructure Admin | AdcMgmt UI or wizard | Use "Generic ADC Builder" wizard for streamlined creation |
| Override adcmon service assignment | Infra Admin or Adcmon Admin | AdcMgmt UI - edit deployhosts | Change `deployhosts.adcmon_service` field (rare) |
| Commit/Deprovision ADCs | Operations + action role | AdcMgmt UI | Use commit workflow button (requires additional action role) |

### Example Workflow: Adding a New ADC Container

**Via the Generic ADC Builder Wizard (recommended):**
```
1. Navigate to AdcMgt → Wizards → Generic ADC Builder
2. Fill in parameters:
   - Customer/Service name
   - Deployment location/rack
   - Configuration template (pool members, TLS settings, etc.)
3. Submit
4. System automatically:
   - Creates Deploy Host record
   - Selects adcmon service from same rack
   - Assigns deployment infrastructure
5. Adcmon begins monitoring immediately
6. Once healthy, BGP routes advertised automatically
7. Traffic flows to new container
```

**Manual Creation (if needed):**
```
1. In AdcMgt: Create new Deploy Host record
   - Specify rack location
   - Specify configuration parameters
   - Or leave adcmon_service blank for auto-assignment
2. System automatically:
   - Derives datacenter from rack
   - Auto-selects adcmon service from same rack
   - Sets defaults for related fields
3. Adcmon begins health checks
4. Once healthy, routes advertised
```

### Key Commands & Locations

**Where to Find It:**
- **AdcMgt URL**: Access via ServiceNow at your datacenter instance
- **Wizards**: Use "Generic ADC Builder" for new deployments
- **API Docs**: REST API available for automation

**Debugging on Command Line:**
```bash
# SSH to adcmon host to verify it's running
ssh <adcmon_host>
ps aux | grep adcmon

# Check BGP session to router
show ip bgp summary
show ip bgp neighbors

# Test container reachability
ping <container_ip>
curl -I http://<container_ip>:<health_check_port>/health

# Check adcmon logs
tail -f /var/log/adcmon/*.log
```

---

## 6. How Do We Troubleshoot and Support It?

### Common Issues & Resolution

| Issue | Root Cause | Resolution Path |
|-------|-----------|-----------------|
| Container not receiving traffic | BGP routes not advertised (likely unhealthy) | 1. Check adcmon health logs<br>2. Verify container is accessible from adcmon<br>3. Check BGP session to router<br>4. If routes stuck: manually re-issue withdrawal/advertisement |
| Adcmon service down | Host/network failure or no redundancy | 1. Verify which adcmon hosts are down<br>2. If only one left: activate standby/failover<br>3. Reroute deployhosts if needed<br>4. Contact Adcmon team for failed hosts |
| Routes stuck advertised (should withdraw) | Stale health state in adcmon cache | 1. Force health check re-evaluation<br>2. Check adcmon logs for polling errors<br>3. Verify network connectivity from adcmon to container<br>4. Manual BGP withdrawal if urgent |
| New container won't auto-assign adcmon | Rack/datacenter mismatch or no adcmon available | 1. Verify deployhosts.rack value is correct<br>2. Verify adcmon service exists in that rack<br>3. If not, create new adcmon service<br>4. Manually assign adcmon_service field if needed |
| Health checks timing out | Network latency or container overload | 1. Check container CPU/memory/network metrics<br>2. Verify network path to container (MTU, firewalls)<br>3. Increase health check timeout if needed<br>4. Check container logs for slow responses |

### Debugging Techniques (Step-by-Step)

**Step 1: Verify AdcMgt State**
```
In AdcMgmt UI:
- Query deployhosts table
- Look for the container in question
- Check status field (should be "healthy" if working)
- Check last_status_update timestamp (should be recent, within 1-2 minutes)
- Note the assigned adcmon_service
```

**Step 2: Check Adcmon Service Health**
```
In AdcMgmt UI:
- Navigate to Adcmon Services table
- Find the service assigned to your container
- Check which adcmon_hosts are in the service
- Verify at least one host is "online"
```

**Step 3: Check Adcmon Logs**
```
SSH to one of the adcmon hosts:
- Review health check results for your container
- Look for timeout/error messages
- Check BGP session logs
- Look for network connectivity issues
```

**Step 4: Verify BGP**
```
SSH to network router (from adcmon host):
- Confirm BGP session with adcmon host is "Established"
- Verify routes for container are advertised/withdrawn as expected
- Check BGP community values match expected policy
```

**Step 5: Escalation if Needed**
- If adcmon service is down → contact Adcmon/Platform team
- If BGP routes not updating → contact Network Engineering
- If container misconfigured → check AdcMgt records, fix config
- If automation issue → contact Platform/Provisioning team

### Support Escalation Path

**Refer to KB1120228** - CNS escalation process for network issues

**Team-specific escalation:**
1. **Adcmon service health/failures** → Adcmon team (see contacts in Section 9)
2. **ADCv3 container issues** → ADC Management team
3. **BGP/Network issues** → Network Engineering team
4. **Provisioning/Automation failures** → Platform Engineering team
5. **General AdcMgt questions** → Use Slack channels (see Section 9)

---

## 7. How Do We Know It's Not Working Properly?

### Symptoms of Health Issues

**Container Level**
- ❌ Container marked "unhealthy" in AdcMgmt UI (status field)
- ❌ Health check timeout/failure in adcmon logs
- ❌ Container responds to ping/curl but routes not advertised
- ❌ Customers report connection timeouts to ADCv3 instances
- ❌ Status hasn't updated in > 5 minutes (stale data)

**Service Level (Adcmon)**
- ❌ Adcmon service status shows "down" or "degraded"
- ❌ Only one adcmon host responding (lost redundancy)
- ❌ No BGP route updates for 5+ minutes (should be near-real-time)
- ❌ Adcmon hosts unable to reach container IP addresses
- ❌ Adcmon logs show repeated health check failures

**Network Level**
- ❌ BGP session to router down/flapping
- ❌ Routes stuck (not updating when container health changes)
- ❌ BGP communities not matching expected values
- ❌ Router can't reach adcmon host addresses
- ❌ Customers unable to connect to services despite containers appearing "healthy"

### Key Monitoring Alerts to Watch For

**Critical (page on-call):**
- Adcmon service with ALL hosts down → no monitoring, no BGP updates
- Container health check failing for >2-3 consecutive polls → likely unresponsive
- BGP session down to primary network neighbor → routes not being updated
- Datacenter-wide container health anomaly → systematic issue

**Warning (create ticket):**
- Single adcmon host down (but service has redundancy)
- Last_status_update older than 5 minutes
- Rapid health state oscillations (flapping - toggling healthy/unhealthy rapidly)
- BGP community value mismatches

### Health Check Verification (Manual)

You can manually verify health from any adcmon host:

```bash
# From adcmon host, test container reachability
ping <container_ip>

# Test health check endpoint (if exposed)
curl -I http://<container_ip>:<health_check_port>/health

# Check BGP session status to router
show ip bgp summary
show ip bgp neighbors <router_ip>

# View BGP routes for this container
show ip bgp <container_prefix>
```

**Healthy Indicators:**
- ✅ Container responds to ping
- ✅ Health check endpoint returns 200 OK
- ✅ BGP session to router is "Established"
- ✅ BGP routes advertised for container prefix
- ✅ Status updated within last 1-2 minutes

---

## 8. Where Are the Documents?

### Official Documentation (in Git)

**Location:** `/git/cns_doc/content/NSD/2_ADC_Landing_Page/`

**Core Resources:**
- **Adcmon Services Overview**: `AdcMgt_Datacenter/AdcMgt_Adcmon/_index.md` ← **Start here for Adcmon**
- **Adcmon Setup & Installation**: `AdcMgt_Datacenter/AdcMgt_Adcmon/Setup/_index.md`
- **Testing Guide** ("Faking It"): `AdcMgt_Datacenter/AdcMgt_Adcmon/Setup/Faking It/_index.md`
- **AdcMgt Main Hub**: `AdcMgt_Datacenter/_index.md`

**Access & Permissions:**
- **Access Roles Reference**: `AdcMgt_Datacenter/access_roles/_index.md` - Detailed descriptions of all roles
- **Access Role Naming Convention**: `AdcMgt_Datacenter/access_roles/naming_convention/_index.md`

**Deployment & Operations:**
- **Deployment FAQ**: `AdcMgt_Datacenter/Deployment/f.a.q./_index.md`
- **Deployment Sizes**: `AdcMgt_Datacenter/Deployment/deployment_size/_index.md`
- **Deprovision Deploy Hosts**: `AdcMgt_Datacenter/Deployment/deprovision_deployhosts/_index.md`
- **Pooled Deployment Groups**: `AdcMgt_Datacenter/Deployment/pooled_deployment_groups/_index.md`

**Troubleshooting & Management:**
- **ADCv2 Error Management**: `AdcMgt_Datacenter/ADCv2 Error Management/` - Error classification and handling
- **Instance Data Cleanup**: `AdcMgt_Datacenter/Instance Data Cleanup/` - Data remediation procedures
- **Snowsk8s (ADCv2 Kubernetes)**: `AdcMgt_Datacenter/snowsk8s/` - Troubleshooting and FAQ

**Developer Resources:**
- **Dev Onboarding**: `AdcMgt_Datacenter/dev-n00b/Overview/_index.md` - Getting started for developers
- **Table Structure**: `AdcMgt_Datacenter/dev-n00b/table_structure/_index.md`
- **API Docs**: `AdcMgt_Datacenter/dev-n00b/codes/api/`

**Wizards & Tools:**
- **Generic ADC Builder**: `AdcMgt_Datacenter/wizards/generic_adc_builder/user_guide/_index.md`
- **Generic F5 Migrater**: `AdcMgt_Datacenter/wizards/generic_f5_migration/_index.md`

### Knowledge Base Articles (in ServiceNow)

- **KB0538549**: "Reference | DATACENTER Instance Access" - Access matrix and request procedures
- **KB0819143**: Access request procedures - detailed walkthrough of PAGA access process
- **KB1120228**: [Escalation process into CNS - Network Engineering](https://support.servicenow.com/kb?id=kb_article_view&sysparm_article=KB1120228) - Escalation paths for networking issues

### Tools & Portals

- **PAGA Tool**: Access request portal at `datacenter.service-now.com` - Use to request AdcMgt access
- **AdcMgmt UI**: Primary interface - access via ServiceNow datacenter instance
- **ServiceNow**: Additional Access Request form if PAGA option not available

---

## 9. Where Can I Go to Get More Help?

### Direct Contacts

**Subject Matter Experts (SMEs):**
- **AdcMgmt Questions** → [Platform/ADC team - see team chat or directory]
- **Adcmon Issues** → [Adcmon team lead - contact via Slack]
- **BGP/Network Integration** → Network Engineering team (see escalation KB)
- **Provisioning/Automation** → Platform Engineering team

**Finding People:**
- Check the AdcMgt documentation authors (listed in doc headers)
- Look for "Author [Name]" tags in documentation pages
- Ask in Slack channels (see below)

### Slack Channels

**For AdcMgmt/Adcmon Questions:**
- `#adc-management` - General AdcMgmt questions and discussions
- `#adcmon-support` - Adcmon-specific issues and troubleshooting
- `#adc-dev` - For developers working on ADC systems
- `#platform-engineering` - Deployment and automation questions

**For Related Topics:**
- `#network-engineering` - BGP, routing, network issues
- `#incident-management` - If escalating a page or critical issue
- `#documentation` - Help with finding or improving docs

### Mailing Lists/Distribution Groups

- `adc-team@[company]` - General ADC inquiries and announcements
- `adcmon-team@[company]` - Adcmon team direct (if available)
- `network-eng@[company]` - Network team issues

### Escalation & Support

1. **Immediate Help** → Ask in Slack (usually 30-min response in business hours)
2. **Complex Issues** → File ServiceNow ticket in CNS queue (reference KB1120228)
3. **Emergency/Outage** → Page on-call via PagerDuty (use incident severity guide)
4. **Documentation Issues** → Submit pull request or comment in wiki/docs

### Learning Resources for Deeper Dives

**For Understanding ADCv3 Architecture:**
- ADCv3 Design Document: [See ADCv3 directory in main ADC docs]
- ADCv3 Specification: `ADCv3/Specification/` in ADC Landing Page

**For BGP & Route Management:**
- Company BGP Standards & Policy: [Ask Network Engineering team]
- BGP Community Tags Reference: [Check Network Engineering wiki]

**For Ops/SRE Engineers Specifically:**
- **On-Call Runbook for Adcmon Issues**: [Link TBD - ask Adcmon team]
- **Adcmon Alerting Guide**: [See Adcmon documentation]
- **Common Deployment Patterns**: `AdcMgt_Datacenter/Deployment/` docs

**For Writing/Maintaining Automation:**
- **REST API Documentation**: `dev-n00b/codes/api/` in AdcMgt docs
- **REST Executer Roles**: See access_roles docs for service account setup
- **Snowsk8s Integration** (for Kubernetes): `AdcMgt_Datacenter/snowsk8s/`

### Next Steps for Getting Deeper

1. **Get AdcMgmt Access** → Submit PAGA request (start with Read-Only)
2. **Explore the UI** → Tour AdcMgt interface, query some containers
3. **Run Lab Test** → Use "Faking It" guide to create test environment
4. **Shadow On-Call** → Observe issue triage if possible
5. **Join the Team** → Contribute back - share your learnings in Slack or wiki

### Feedback & Future Sessions

This is just the foundation! Areas for deeper follow-up sessions:
- **Advanced BGP configuration** - Policy setup, community tags, fail-over scenarios
- **Adcmon high availability** - Redundancy mechanics, failover processes
- **Integration with Snowsk8s** - Hybrid deployments (Kubernetes + ADCv3)
- **Performance tuning** - Health check optimization, route convergence timing
- **AI/ML-based health prediction** - Future enhancement possibilities

**Have a topic you'd like to teach?** Reach out on #adc-management Slack channel!

---

## Q&A Notes

*Capture questions and answers during/after the session here.*

---

## Session Feedback

**What was most useful about this session?**

**What should we cover more deeply?**

**Any follow-up questions?**

**Suggestions for future sessions?**

---

## Presenter Notes

### Timing Guidance (50-minute session)

- **Sections 1-3** (What/Why/How): ~15 minutes
- **Sections 4-5** (Interactions/Engineering): ~15 minutes
- **Sections 6-7** (Troubleshooting/Monitoring): ~12 minutes
- **Sections 8-9** (Resources/Help): ~5 minutes
- **Q&A**: ~3 minutes

### Suggested Demos/Live Examples

**Best demos to include:**
1. **Live AdcMgmt UI tour** (5 min)
   - Show deployhosts table and status
   - Query a real container
   - Show adcmon service assignment

2. **BGP route advertisement demo** (5 min) - if possible
   - SSH to adcmon host
   - Show `show ip bgp summary` and active routes
   - Optionally: gracefully stop a container and show route withdrawal

3. **Health check test** (3 min)
   - From adcmon host, `curl -I` to container health endpoint
   - Show logs when health changes

4. **Troubleshooting scenario walkthrough** (5 min)
   - Present: "Container not receiving traffic" scenario
   - Walk through debugging steps from Section 6
   - Show what each step reveals

### For Q&A

- Be honest about unknowns - get back to folks later
- Encourage follow-on conversations for complex questions
- Use this as feedback to improve documentation
- Mention that cross-training is ongoing - come back for follow-up sessions!

### Format Options

- **Slides**: Convert to PowerPoint if preferred (this template works as speaker notes)
- **Live Demo Focus**: Use this as outline, spend more time in UI and live systems
- **Hybrid**: Slides for architecture (1-3), demo for (4-5), screen-share for (6-7)

### Handout/References

Consider providing:
- Link to this documentation: `/git/cns_doc/content/NSD/2_ADC_Landing_Page/`
- PAGA access request form link
- Your contact info for follow-ups
- Suggested Slack channels to join
