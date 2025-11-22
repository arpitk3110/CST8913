## Task 1 – Set Up Tooling for Discovery

### Do you use agent-based, agentless, or a mix for discovery?
For Tailwind Traders, a **mix of agentless and agent-based discovery** is the best choice.  
Agentless discovery helps collect information quickly from VMware VMs with minimal effort, while agent-based discovery is useful when deeper dependency details or longer data-collection windows are needed. Since the application includes web, app, and SQL servers, using both methods provides complete visibility.

---

### How many appliances do you need and why?
Tailwind needs **one Azure Migrate appliance** for the entire VMware environment.  
A single appliance can discover multiple servers inside the same vCenter environment, and there is no requirement for separate appliances per tier. This keeps the setup simple while still collecting software inventory, dependency data, and SQL details for **WEB01, WEB02, APP01, and SQL01**.

---

### Credentials required for discovery

#### **Software Inventory**
Windows or Linux **administrator credentials** are required so the appliance can log in remotely and read installed applications, OS features, and roles. These credentials are used only for inventory collection.

#### **SQL Discovery**
SQL Server authentication or Windows authentication credentials are needed to access SQL instances. This allows Azure Migrate to detect SQL versions, databases, configurations, and migration readiness for **SQL01**.

#### **Dependency Mapping**
- **Agentless:** Requires **vCenter read-only credentials** so the appliance can collect network flow and connection data.  
- **Agent-based:** Requires **admin-level credentials on each server** to install the monitoring agent and capture detailed process-level dependencies.

---

### Three best practices Tailwind should follow during discovery
1. **Run discovery for at least 30 days**  
   This captures full performance and dependency behavior and helps avoid incorrect sizing.

2. **Verify appliance health regularly**  
   The migration team should ensure data is flowing into Azure Migrate and resolve credential or connection issues quickly.

3. **Prepare all credentials in advance**  
   Coordinating with the infrastructure and SQL teams ensures that all inventory, SQL, and dependency scans run smoothly without delays.

---

## Task 2 – Perform Assessment Planning

### What assessment type should Tailwind choose (production vs non-production)?
Tailwind should use a **production assessment**.  
The workload includes critical web, API, and SQL servers, and the application has strict downtime requirements. A production assessment provides more accurate sizing, performance patterns, and cost estimates based on real usage.

### What target region and performance-history duration should be used?
- **Target Region:** Canada Central.  
- **Performance-History Duration:** **30 days**, which aligns with recommended best practices.  
  A full month of performance data helps capture peak, average, and seasonal patterns and prevents both under-sizing and over-sizing.

### Should the company use performance-based sizing or on-premises VM size? Why?
Tailwind should use **performance-based sizing**.  
This method uses real CPU, memory, disk, and network metrics collected during discovery to recommend the right Azure VM size. It avoids blindly copying the on-prem VMware configuration, which might be oversized or outdated. Performance-based sizing leads to better cost optimization while still ensuring enough capacity for production needs.

### What comfort factor, pricing model, and licensing model will you select?
- **Comfort Factor:** **30%**  
  This provides a small buffer for unexpected performance spikes without significantly increasing cost.
  
- **Pricing Model:** **Pay-As-You-Go**  
  This model gives flexibility during the migration period. Tailwind can switch to Reserved Instances later once usage stabilizes.

- **Licensing Model:** **Azure Hybrid Benefit** (if Tailwind owns Windows/SQL licenses).  
  This reduces VM and SQL licensing costs.  
  If license ownership is unclear, choose **license-included** as the safer option.


## Task 3 – Dependency Analysis

### 1. Application components involved

The main components of the Tailwind application are:

- **WEB01, WEB02** – Frontend web servers (VMware)
- **APP01** – API / application server (VMware)
- **SQL01** – SQL Server 2017 database server (physical)
- **LB01** – Internal load balancer for the web tier
- **Nightly backup system**
- **Firewall(s)** controlling inbound and outbound traffic

### 2. At least 8 key dependencies

Examples of important dependencies between these components:

1. **Client → LB01** – HTTP/HTTPS traffic from users to the load balancer (ports 80/443).  
2. **LB01 → WEB01/WEB02** – Internal HTTP/HTTPS traffic to distribute requests across the web servers.  
3. **WEB01/WEB02 → APP01** – API calls over HTTP/HTTPS or a custom port (for example, 8080) to the application server.  
4. **APP01 → SQL01** – Database connections over TCP 1433 (default SQL Server port).  
5. **APP01 → external payment or third-party APIs** – Outbound HTTPS (443) to external endpoints.  
6. **All servers → Active Directory / DNS** – Authentication and name resolution using Kerberos (88), LDAP (389/636), and DNS (53).  
7. **Backup system → SQL01 and other servers** – Backup traffic using SMB (445) or backup-agent ports to copy data during nightly jobs.  
8. **Monitoring / logging tools → all servers** – Agent or agentless connections collecting performance metrics and logs.

### 3. Items to filter out as noise

When cleaning up the dependency map, Tailwind should filter out common “noise” traffic, such as:

- Windows Update and antivirus cloud services  
- Generic internet endpoints not related to the application (for example, telemetry services)  
- Standard NTP time-sync traffic  
- Background management or monitoring traffic that is not specific to this workload  

This helps focus on the real application-level dependencies that matter for migration.

### 4. Business Requirements from the Application Owner

The migration team documented the following requirements based on discussions with the application owner:

#### **Business Criticality**
This is a high-criticality line-of-business application that supports daily operations. Any extended outage would impact internal users and customers, so stability and continuity are essential.

#### **Uptime & Downtime Requirements**
The application targets high uptime (around 99.9% in production).  
For the migration, the owner allows a maximum downtime window of **1 hour**, which means the migration steps must be planned and validated carefully to stay within that limit.

#### **Data Classification**
The SQL database stores business-sensitive information such as customer or order data.  
Data must be treated as **confidential**, and protected with encryption, restricted access, and proper backup policies during and after migration.

#### **Licensing Dependencies**
- SQL Server 2017 is licensed on-premises. Tailwind may apply **Azure Hybrid Benefit** if they have Software Assurance.  
- Some application components may depend on MAC address or IP-based licensing. These must be reviewed before moving the servers to Azure, because Azure VM IP/MAC values will change.

#### **Patching Requirements**
Server patching is performed during approved maintenance windows (usually off-hours). After migration, the same process should continue, potentially using **Azure Update Management** to automate OS and SQL patching.

#### **Firewall & IP Considerations**
- Only the load balancer (LB01) is publicly exposed; all backend servers (WEB, APP, SQL) are internal only.
- SQL01 accepts connections only from APP01 on defined ports (e.g., TCP 1433).
- Any partners or third-party services using **IP whitelisting** will need updated Azure IP ranges during cutover.
- Firewall rules must be updated so Azure VMs maintain the same communication patterns as the on-premises servers.


### Task 4 – Validate Assessment Results with Application Owner

#### Recommended VM Sizes for WEB, APP, and SQL Servers

- **WEB01 & WEB02 (Frontend Servers)**  
  **D2s v5** – 2 vCPUs, 8 GB RAM  
  Suitable for lightweight web workloads and aligns with common lift-and-shift recommendations.

- **APP01 (API / Application Server)**  
  **D4s v5** – 4 vCPUs, 16 GB RAM  
  Provides enough CPU and memory for API processing and application logic.

- **SQL01 (Database Server)**  
  **E8ds v5** – 8 vCPUs, 64 GB RAM  
  E-Series is memory-optimized and ideal for SQL Server workloads.

---

#### Components That May Need Optimization or Replacement in Azure

- **Load Balancer (LB01)**  
  Replace with **Azure Load Balancer** or **Azure Application Gateway** depending on L4 vs L7 needs.

- **SQL Server**  
  Evaluate migration to:  
  - **Azure SQL Managed Instance**  
  - **Azure SQL Database**  
  - **SQL Server on Azure VM** 

- **Backups**  
  Replace on-prem backup jobs with **Azure Backup** or **Recovery Services Vault**.

- **Firewall Rules**  
  Move to **Network Security Groups (NSGs)**, **Application Security Groups (ASGs)**, or **Azure Firewall** depending on complexity.

---

#### Dependency Validation

All dependencies identified from agentless and agent-based scans must be validated with the application owner, including:

- Web ↔ App communication  
- App ↔ SQL communication  
- Load balancer health probes  
- Backup jobs and schedules  
- External endpoints (e.g., SMTP, authentication APIs)

The application owner confirms that all critical ports, services, and data flows are captured, and any system-level noise has been filtered out.

---

#### SLA and Downtime Requirements

- Azure VM SLAs (typically **99.9%–99.95%**) meet the application’s availability needs.
- Tailwind’s **1-hour downtime window** is achievable because:  
  - VM replication reduces cutover time  
  - SQL migration steps are pre-validated  
  - Web and App tiers are stateless and quick to transition  

The application owner agrees that the proposed cutover plan fits the business expectations.

---

#### SQL Migration Options

Tailwind has three options for migrating SQL:

- **Rehost (SQL Server on Azure VM)**  
  Fastest option, keeps full SQL Server compatibility, and minimizes downtime.

- **Azure SQL Managed Instance**  
  High compatibility with automated patching, backups, and easier long-term management.

- **Azure SQL Database**  
  Fully managed PaaS service with strong automation and scaling, but may require schema or code changes depending on SQL features used.

The application owner reviews all options and selects the one that best fits compatibility, performance, and downtime needs.


---

## Task 5 – Migration Plan (Runbook Style)

### 1. Pre-Migration Tasks
- Verify that all discovery and assessment data is complete and validated with the application owner.
- Confirm the approved 1-hour downtime window and notify all stakeholders.
- Take fresh on-prem backups of WEB01, WEB02, APP01, and SQL01.
- Ensure the Azure landing zone is ready (VNet, subnets, NSGs, resource groups).
- Pre-create target VM configurations and disk types based on assessment recommendations.
- Document all connection strings, IP addresses, firewall rules, and DNS records used by the application.
- Confirm the selected SQL migration approach (Rehost / SQL MI / SQL DB).

---

### 2. Migration Steps (Per Server Group)

#### **Wave 1 – WEB Tier (WEB01, WEB02)**
1. Replicate VMs to Azure using Azure Migrate Server Migration.
2. Perform a test migration to confirm that the application UI loads correctly.
3. Perform final cutover during the approved downtime window.
4. Validate web functionality and ensure load balancer rules work correctly.

#### **Wave 2 – APP Tier (APP01)**
1. Start replication and confirm dependencies match the assessment results.
2. Run a test migration and verify communication with the web tier.
3. Perform final cutover during the same or next maintenance window.
4. Validate all application API endpoints after migration.

#### **Wave 3 – SQL Tier (SQL01)**
1. Perform database migration using the selected option (Rehost / SQL MI / SQL DB).
2. Sync final data and cut over during the maintenance window.
3. Update application components to point to the new SQL endpoint.
4. Validate database integrity and performance metrics.

---

### 3. DNS Updates
- Update DNS records to map the web application hostname to the new Azure public or private IP.
- Update internal DNS entries for APP01 and SQL endpoints as required.
- Reduce DNS TTL before migration to enable faster DNS propagation.

---

### 4. Connection String Changes
- Update application configuration to reflect:
  - New SQL connection string
  - New private IP addresses for APP01 and SQL01
  - Any changes in ports or protocols
- Validate successful application connectivity after updates.

---

### 5. Load Balancer Considerations
- Replace the on-prem LB01 with an Azure Load Balancer.
- Configure backend pools for WEB01 and WEB02.
- Define health probes and load-balancing rules (ports 80/443).
- Validate that the APP tier communicates correctly with the WEB tier after migration.

---

### 6. SQL Migration Considerations
- Confirm the correct SQL migration option (Rehost VM, SQL MI, or SQL DB).
- Validate database compatibility, SQL feature requirements, and version support.
- Ensure the chosen storage tier meets performance needs.
- Run post-migration SQL integrity checks and verify scheduled jobs (backups, maintenance, etc.).

---

### 7. Post-Migration Validation Checklist
- Application loads successfully from Azure.
- WEB, APP, and SQL tiers communicate without errors.
- All dependencies (firewall rules, ports, third-party services) function correctly.
- Performance matches or exceeds on-premise values.
- Backup policies and monitoring alerts are configured in Azure.
- Decommission on-prem resources only after full validation and owner approval.

---

### 8. Back-Out Plan (If Migration Fails)
- Immediately revert DNS records back to the on-prem IP addresses.
- Redirect traffic back to on-prem WEB01, WEB02, APP01, and SQL01 servers.
- Restore the SQL database from the latest successful on-prem backup if needed.
- Notify stakeholders and reschedule the migration for a future maintenance window.
- Analyze the failure, document root cause, and update the runbook before retrying.

---

## Task 6 – Plan the Migration Waves

To reduce risk and follow a structured migration approach, the servers are grouped into migration waves based on dependencies, business impact, downtime limits, and testing needs.

### **Justification for Grouping**

- **Dependencies:**  
  The web tier is migrated first because it is stateless and has the fewest internal dependencies. The application server (APP01) depends on the web tier, so it must follow next. SQL is the final layer since all tiers rely on it.

- **Business Criticality:**  
  SQL01 is the most critical component in the stack, so it is placed in the final wave after all other application components are tested and validated.

- **Downtime Restrictions:**  
  SQL has the strictest downtime requirements and must be coordinated carefully within the approved 1-hour window. Migrating it last helps ensure proper preparation.

- **Testing Requirements:**  
  Web and application functionality can be tested earlier in the process. This reduces risk before moving the core database layer.

---

### **Final Migration Waves Table**

| Wave   | Servers        | Reason |
|--------|----------------|--------|
| **Wave 1** | WEB01, WEB02 | Stateless front-end servers, minimal risk, easy rollback. |
| **Wave 2** | APP01        | Depends on WEB tier, requires functional testing after Wave 1. |
| **Wave 3** | SQL01        | Highest risk, core data layer, strict downtime window, requires complete validation. |

---
