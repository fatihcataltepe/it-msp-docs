# MSP Client Setup Example: 30-Person Accountancy Firm

## Overview
This document provides a detailed real-world example of what an MSP (Managed Service Provider) maintains for a mid-sized professional services firm: a 30-person accountancy practice based in London.

---

## Identity & Access Management

### Entra ID (Azure AD) Configuration
- **Total User Accounts**: 30 active users
- **Account Types**:
  - 27 staff accounts (partners, managers, junior accountants)
  - 2 service accounts (for integrations with accounting software)
  - 1 shared account for reception/front desk

### Authentication & Security
- **Multi-Factor Authentication (MFA)**: Enabled across all accounts
  - Primary method: Microsoft Authenticator app
  - Fallback: Email verification
  - Enforcement: Mandatory for all users

### Conditional Access Rules
These rules automatically enforce security policies:

| Scenario | Rule | Action |
|----------|------|--------|
| Login from outside UK | Verify with phone | Require additional verification |
| Login at unusual time (3 AM) | Geographic anomaly | Block or require MFA challenge |
| Device not compliant | Endpoint check | Require device update/reboot |
| Unregistered device | New device | Require re-authentication |

### Guest Access
- **External Contractors/Auditors**: 3-5 temporary guest accounts at any time
- **Permissions**: Limited to specific SharePoint sites (e.g., audit documents folder)
- **Expiration**: Auto-expires after 90 days; requires renewal request
- **Audit Trail**: All guest access logged and reviewed quarterly

---

## Device & Endpoint Management

### Device Inventory
| Device Type | Count | Management | Primary Users |
|------------|-------|-----------|----------------|
| Windows Laptops (Dell/HP) | 25 | Intune | Staff, Partners |
| iPads | 3 | Intune Mobile Device Management | Senior Partners |
| Shared Printers | 2 | Network managed | All staff |

### Device Policies (Enforced via Intune)

**Windows Laptops**:
- Operating System: Windows 11 Enterprise
- Encryption: BitLocker (full disk encryption)
- Updates: Automatic, installed every second Tuesday at 10 PM
- Antivirus: Microsoft Defender (always running)
- Firewall: Windows Defender Firewall (enabled)
- VPN: Auto-connect to firm VPN when off-premises
- Screen Lock: Require lock after 10 minutes inactivity
- Password Policy: Minimum 12 characters, change every 90 days

**iPads**:
- Mobile Device Management (MDM): Full enrollment
- App Management: Only approved apps can be installed
- Data Encryption: Enabled
- Conditional Access: Require MFA for accessing firm apps
- Remote Wipe: Enabled (if lost, MSP can wipe remotely)

### Device Lifecycle Management
- **New Starter**: Imaged laptop, added to Intune, accounts created, security policies pushed
- **Leaver**: Device wiped, account disabled, mailbox converted to shared mailbox (for handover)
- **Warranty & Repairs**: Tracked in IT Glue; MSP coordinates with vendors
- **Compliance Check**: Weekly scan for non-compliant devices (e.g., outdated OS)

---

## Collaboration & Productivity Services

### Microsoft 365 Stack

#### Teams
- **Workspace Structure**:
  - General channel (company-wide announcements)
  - Engagement teams (one team per client engagement, auto-archived after project close)
  - Department channels (Audit, Tax, Payroll, Finance)
  - Partners channel (senior leadership discussion)

- **Governance**:
  - New teams auto-create with governance policies applied
  - Retention: Channels archived after 12 months inactivity
  - Guest access: Enabled, but requires approval from engagement lead

- **Integration**: Connected to:
  - Accounting software (Xero/Sage) for reporting notifications
  - Shared SharePoint document library for auto-notification on file updates

#### SharePoint
- **Primary Repository**: Client document library
  - Structure: One folder per client, sub-folders by year/engagement type
  - Permissions: Access granted based on role and engagement (Finance team can't see tax client files)
  - Retention: Client files retained for 7 years post-engagement (regulatory requirement)

- **Collaboration Sites**:
  - Internal wiki (policies, procedures, templates)
  - Resource center (tax updates, compliance alerts)
  - Training library (video training for new joiners)

- **Backup**: Automatic (Microsoft maintains 3 copies globally)

#### OneDrive
- **Personal Storage**: 1 TB per user
- **Backup**: Automatic (always on)
- **Retention**: If user leaves, firm has 30 days to migrate files to shared mailbox
- **Sync**: Enabled for offline access (synced to local device, encrypted)

#### Exchange Online
- **Mailbox Type**: 50 GB per user
- **Shared Mailbox**: One shared inbox for "enquiries@firm.com" (monitored by 3 staff)
- **Archive**: Automatic archiving after 3 years
- **Retention Policy**: Compliance rule - emails in certain folders (client files) retained for 7 years

#### Email Security Add-On
- **Phishing Protection**: AI-powered, blocks 99% of phishing emails
- **Malware Scanning**: Real-time attachment scanning
- **External Email Tagging**: All emails from outside firm tagged "[EXTERNAL]" for user awareness
- **Encryption**: Ability to send encrypted emails to external parties (auto-detected for recipients with encryption enabled)

---

## Security & Monitoring

### Endpoint Security
- **Microsoft Defender**: Active on all 25 laptops and 3 iPads
  - Real-time threat protection
  - Regular definitions updates (daily)
  - Cloud-based threat intelligence

### Backup & Disaster Recovery

**Primary Backup** (Microsoft):
- OneDrive: Automatic, continuous backup to 3 geo-distributed data centers
- SharePoint: Automatic, 93-day retention (versioning available)
- Exchange: 30-day deleted item retention, recoverable items folder (3-year retention for compliance)

**Secondary Backup** (Third-Party Tool)**:
- All OneDrive/SharePoint files backed up weekly to offsite encrypted storage
- Retention: 12 months (for ransomware recovery scenario)
- Test Restore: Quarterly - MSP tests recovery of random 5-10 files to confirm integrity

### Ransomware Defense
- **Attack Detection**: Automatic detection of suspicious mass file deletion
- **Isolation Protocol**: Affected user account immediately locked, device isolated from network
- **Recovery Window**: Firm can recover from secondary backup within 4 hours

### Monitoring & Alerting

**MSP Monitoring Dashboard** (via RMM tool):
- Disk space: Alert if any device <20% free space
- Updates: Alert if Windows updates pending >7 days
- Malware: Real-time alert if Defender detects threat
- VPN: Alert if VPN connection unavailable
- Licensing: Alert 30 days before M365 licenses expire

**Security Alerts**:
- Failed login attempts: Alert if >5 in 10 minutes (possible brute force)
- Unusual access: Alert on after-hours file access to sensitive folders
- Permission changes: Log all SharePoint/OneDrive permission changes (audit trail)

---

## Support & Maintenance

### Support Structure
- **Primary Contact**: Dedicated account manager (shared with 2-3 similar firms)
- **Help Desk Team**: 2-3 L1 technicians who handle tickets (business hours: 8 AM - 6 PM, Monday-Friday)
- **Escalation**: Complex issues escalated to L2 engineer (typically resolves within 24 hours)

### Typical Ticket Categories & Resolution Time

| Issue | Frequency | Time to Resolve |
|-------|-----------|-----------------|
| Password reset | 2-3/week | 5 minutes |
| New user setup | 1-2/month | 2 hours |
| Device issue (printer, Wi-Fi) | 2-3/week | 30 minutes |
| Software licensing | 1-2/month | 1 hour |
| Permission access request | 3-5/week | 15 minutes |
| Security alert investigation | 1-2/month | 2-4 hours |
| Email recovery request | 1-2/month | 1 hour |

### Proactive Maintenance Schedule

**Weekly**:
- Compliance scan (devices, user permissions)
- Security log review
- Backup integrity check

**Monthly**:
- Account review (new joiners, leavers, permission audit)
- Performance report to partner (uptime, incidents, recommendations)
- Patch review (identify upcoming updates)

**Quarterly**:
- Disaster recovery test (restore from backup)
- Security audit (permissions, external access, compliance)
- Infrastructure review (licensing, software usage, costs)
- Account manager business review meeting (strategy, budget, security roadmap)

**Annually**:
- Full security assessment
- Compliance audit (SOC 2 Type II report)
- Software license optimization
- Technology roadmap planning

---

## Cost Breakdown

### Monthly Fees

| Component | Cost |
|-----------|------|
| 30 users × £100/user/month (managed services) | £3,000 |
| Email security add-on | £150 |
| M365 backup (30 users × £3) | £90 |
| Disaster recovery retainer | £200 |
| **Total Monthly** | **£3,440** |

### Annual Cost
- **Monthly Services**: £3,440 × 12 = **£41,280**
- **Project/Ad-Hoc Work**: ~£5,000-£10,000 (new device setup, migrations, training)
- **Total Year 1 Estimate**: **£46,000-£51,000**

### Comparison: In-House Alternative
- Full-time IT manager: £40,000-£50,000/year (salary only)
- Benefits & on-costs: +£8,000-£12,000/year
- Tools/software licenses: +£3,000-£5,000/year
- Training/professional development: +£2,000/year
- **Total: £53,000-£67,000/year** (plus management burden on CFO/partner)

**Verdict**: MSP saves cost AND provides access to specialists (security, cloud architects, compliance experts) that a single IT person cannot provide.

---

## Why This Setup Was Chosen

### Business Requirements
1. **Compliance**: Client data must be retained 7 years; regulatory audit trail required
2. **Security**: Client confidentiality is paramount; data breach = reputation + legal risk
3. **Uptime**: Firm cannot afford downtime during busy seasons (January, April, September)
4. **Growth**: Adding staff monthly; need scalable infrastructure

### Technical Requirements
1. **Collaboration**: Partners work from office, home, and client sites; need seamless file sync
2. **Integration**: Need to sync accounting software notifications with Teams
3. **Multi-device**: Staff use laptops, tablets, phones; need device management
4. **Performance**: Large spreadsheets and document repositories; need reliable bandwidth

### Cost Constraints
- Budget: £3,000-£4,000/month for IT services
- Could not justify dedicated full-time IT hire
- Needed access to specialized expertise (security, Azure, compliance)

---

## Typical Year 1 Timeline

| Month | Activity |
|-------|----------|
| Month 1 | Infrastructure assessment; licensing review |
| Month 2 | Migrate old email system to Exchange Online; establish backup process |
| Month 3 | Roll out Intune device management; enroll all laptops |
| Month 4 | Set up Teams and SharePoint; migrate shared drives |
| Month 5 | Implement MFA and Conditional Access policies |
| Month 6 | Conduct security training; penetration test |
| Month 7-12 | Ongoing optimization, quarterly reviews, new hires onboarded via playbook |

---

## Key Takeaways

This 30-person accountancy firm exemplifies a **mid-market MSP client** (Silver/Gold tier):
- **Not simple enough** for basic cloud setup (some regulatory complexity)
- **Not large enough** to need 24/7 SOC or architect team
- **Right fit** for managed services with dedicated account manager and proactive security
- **Cost-effective** compared to hiring in-house IT staff
- **Scalable** - can grow to 50-100 users without major infrastructure changes

The MSP maintains not just the technology, but the **governance, compliance, and operational health** of the firm's entire IT function.
