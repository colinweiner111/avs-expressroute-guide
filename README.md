---
title: Azure VMware Solution (AVS) – ExpressRoute, Identity & HCX Setup Checklist (Gen 1)
author: Microsoft Cloud Solution Architect
environment: Azure VMware Solution (Gen 1)
last_updated: 2025-10-20
---

# 🧩 Azure VMware Solution (AVS) – ExpressRoute, Identity & HCX Setup Checklist (v6)

**Environment:** Azure VMware Solution (Gen 1)  
**Scenario:** Post-deployment configuration for a new AVS Gen 1 private cloud  
**Focus:** Portal-first workflow for configuring ExpressRoute Connectivity to On-Premises, Identity (AD/LDAP), and VMware HCX  
**Docs validated:** October 2025  

---

## 🌐 1. ExpressRoute Connectivity to On-Premises

📘 **References:**  
[Connect AVS Gen 1 to on-premises via ExpressRoute Global Reach](https://learn.microsoft.com/en-us/azure/azure-vmware/tutorial-expressroute-global-reach-private-cloud) | [Create an ExpressRoute circuit](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-howto-circuit-portal-resource-manager) | [AVS Networking Concepts](https://learn.microsoft.com/en-us/azure/azure-vmware/concepts-networking)

> 🧱 _This section applies to AVS Gen 1. For Gen 2, native connectivity replaces ExpressRoute Global Reach._

> 💡 **Note:** If you already have an existing ExpressRoute circuit connected to another AVS private cloud, reuse that same circuit and simply create a new **Global Reach connection** from the new AVS private cloud’s ExpressRoute circuit to your existing customer-managed circuit. Each AVS private cloud has its own Microsoft-managed ExpressRoute circuit that must be linked individually.

### **Task Checklist**
- [ ] Obtain ExpressRoute authorization key from AVS portal  
- [ ] Create or identify an existing ExpressRoute circuit  
- [ ] Link AVS private cloud to the ExpressRoute circuit  
- [ ] Configure on-prem router for BGP peering  
- [ ] Validate ExpressRoute connectivity and route advertisement  

### **Detailed Steps**

| Step | Action | Location | Notes / Inputs | Reference |
|------|--------|-----------|----------------|------------|
| **1.1** | **Obtain ExpressRoute authorization key** | **Azure Portal → AVS → Connectivity → ExpressRoute** | Copy the `Authorization Key` and note the `Peer ASN` and `Circuit ID` | [Tutorial: Connect AVS Gen 1 to on-prem via ExpressRoute Global Reach](https://learn.microsoft.com/en-us/azure/azure-vmware/tutorial-expressroute-global-reach-private-cloud) |
| **1.2** | **Create ExpressRoute circuit** | **Azure Portal → Create a resource → Networking → ExpressRoute** | Choose bandwidth, provider (Equinix, AT&T, etc.), and SKU | [Create ExpressRoute circuit](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-howto-circuit-portal-resource-manager) |
| **1.3** | **Link AVS private cloud to ExpressRoute circuit** | **Azure Portal → AVS → Connectivity → Link to ExpressRoute circuit** | Paste the `Authorization Key` from Step 1.1 | same doc |
| **1.4** | **Configure on-prem router for BGP** | **Customer Edge Router (on-prem)** | Configure ASN, subnets, and BGP session parameters | [AVS Networking Concepts](https://learn.microsoft.com/en-us/azure/azure-vmware/concepts-networking) |
| **1.5** | **Validate ExpressRoute connectivity** | **Azure Portal / On-prem Router** | Check connection status = *Connected*; verify routes and ping vCenter/HCX endpoints | [Tutorial: Connect AVS Gen 1 to on-prem via ExpressRoute Global Reach](https://learn.microsoft.com/en-us/azure/azure-vmware/tutorial-expressroute-global-reach-private-cloud) |

### **Validation Tips**
- Verify **ExpressRoute connection status** shows *Connected* in both AVS and ExpressRoute circuit blades.  
- Confirm that **Global Reach** connectivity is enabled and active.  
- From a jump box or on-prem VM, run `tracert` or `Test-NetConnection` to confirm reachability to AVS vCenter and NSX Manager FQDNs.  
- Review **Effective Routes** on your Azure VM NIC to ensure AVS subnets are advertised through ExpressRoute. 
- Ensure both AVS private clouds (old and new) show Connected status under Connectivity → ExpressRoute before proceeding with HCX Interconnect. (This helps confirm readiness for       migration.) 

---

## ✅ 2. Identity Configuration (Active Directory / LDAP)

📘 **Reference:** [Configure an external identity source in AVS (Gen 1)](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-identity-source-vcenter)

> 🧱 _This section applies to AVS Gen 1. For Gen 2, refer to the updated “native DNS and identity” documentation._

### **Task Checklist**
- [ ] Validate DNS resolution between AVS and your AD domain (e.g., `corp.contoso.com`)
- [ ] Export LDAPS certificate from a domain controller (`.cer`) and upload to Azure Blob Storage
- [ ] Add Identity Source using Run Command → AddIdentitySource
- [ ] Verify identity source in vCenter
- [ ] Assign roles to AD groups or users (CloudAdmin / custom roles)
- [ ] Test login with `domain\\user` credentials

### **Detailed Steps**

| Step | Action | Location | Notes / Inputs | Reference |
|------|--------|-----------|----------------|------------|
| **2.1** | **Validate DNS resolution** between AVS and your AD domain | **Azure Portal** | Configure DNS forwarders so vCenter can resolve DCs | [Configure DNS forwarder for AVS (Gen 1)](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-dns-azure-vmware-solution) |
| **2.2** | **Export LDAPS certificate** from a domain controller and upload to Azure Blob Storage | **Azure Portal / Storage Account** | Generate a SAS URL for the certificate | [Step 2 – Export certificate](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-identity-source-vcenter#step-2-export-the-domain-controller-certificate) |
| **2.3** | **Add Identity Source** using *Run Command → AddIdentitySource* | **Azure Portal → AVS → Run Command** | Fill in DomainName, BaseDN, PrimaryURL, SAS URL, Username, and Password | [Step 3 – AddIdentitySource](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-identity-source-vcenter#step-3-run-the-addidentitysource-command) |
| **2.4** | **Verify identity source in vCenter** | **vCenter UI** | *Administration → SSO → Configuration → Identity Sources* | [Step 4 – Verify](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-identity-source-vcenter#step-4-verify-the-identity-source) |
| **2.5** | **Assign roles** to AD groups or users | **vCenter UI** | Use least privilege and governance best practices | [AVS Identity Architecture](https://learn.microsoft.com/en-us/azure/azure-vmware/architecture-identity) |
| **2.6** | **Test login** with `domain\\user` credentials | **vCenter UI** | Validate successful AD authentication | — |

---

## 🚀 3. VMware HCX Deployment & Configuration

📘 **References:**  
[Install VMware HCX in AVS (Gen 1)](https://learn.microsoft.com/en-us/azure/azure-vmware/install-vmware-hcx) | [Configure VMware HCX](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-vmware-hcx) | [HCX Network Extension](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-hcx-network-extension)

> 🧱 _This section applies to AVS Gen 1. For Gen 2, see the new HCX and networking integration model that uses native connectivity._

### **Task Checklist**
- [ ] Enable HCX Cloud Manager
- [ ] Copy HCX license key and cloud URL
- [ ] Deploy HCX Connector OVA in on-premises vCenter
- [ ] Activate HCX Connector with license key
- [ ] Pair on-prem Connector with AVS HCX Cloud Manager
- [ ] Create Network Profiles & Compute Profiles
- [ ] Build Service Mesh between AVS and on-prem
- [ ] (Optional) Extend L2 Networks to AVS
- [ ] Validate migration (Cold, vMotion, Bulk)
- [ ] (Optional) Migrate between AVS private clouds using Interconnect

> 💡 **Tip:** Typically, this is used for migrations between AVS private clouds (A→B) after both are linked to the same on-prem ExpressRoute circuit via Global Reach.


### **Detailed Steps**

| Step | Action | Location | Notes / Inputs | Reference |
|------|--------|-----------|----------------|------------|
| **3.1** | **Enable HCX Cloud Manager** | **Azure Portal → Manage → Add-ons → HCX → Enable** | Deployment takes ~30 min | [Install HCX](https://learn.microsoft.com/en-us/azure/azure-vmware/install-vmware-hcx) |
| **3.2** | **Copy HCX license key and cloud URL** | **Azure Portal → Manage → Add-ons → HCX** | Needed for HCX Connector activation | same doc |
| **3.3** | **Deploy HCX Connector OVA** | **vSphere UI (on-prem)** | Configure management and service networks | [Configure HCX](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-vmware-hcx) |
| **3.4** | **Activate HCX Connector** | **HCX UI** | Validate SSO and network connectivity | same doc |
| **3.5** | **Pair sites** – On-prem Connector → AVS HCX Cloud Manager | **HCX UI** | Authenticate with `cloudadmin@vsphere.local` credentials | same doc |
| **3.6** | **Create Network Profiles & Compute Profiles** | **HCX UI** | Enables Service Mesh creation | same doc |
| **3.7** | **Build Service Mesh** | **HCX UI** | Ensure all services show *Healthy* | same doc |
| **3.8** | *(Optional)* **Extend L2 Networks** to AVS | **HCX UI** | For live migration or DR validation | [Network Extension](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-hcx-network-extension) |
| **3.9** | **Validate migrations** (Cold, vMotion, Bulk) | **HCX UI** | Confirm vCenter and replication visibility | [Migration Architecture](https://learn.microsoft.com/en-us/azure/azure-vmware/architecture-migrate) |
| **3.10** | **(Optional)** If migrating from another AVS private cloud | **Azure Portal → Networking → Interconnect** | Use **AVS Interconnect (Gen 1)** to connect both clouds in same region | [Connect multiple AVS private clouds (Gen 1 Interconnect)](https://learn.microsoft.com/en-us/azure/azure-vmware/connect-multiple-private-clouds-same-region) |

---

## 🧾 Validation Checklist

| Area | Validation | Status |
|------|-------------|---------|
| **ExpressRoute** | Connection status shows *Connected* | [ ] |
|  | BGP routes and connectivity validated | [ ] |
| **Identity** | vCenter shows AD domain as an identity source | [ ] |
|  | AD user login succeeds via vCenter SSO | [ ] |
| **HCX** | HCX Cloud Manager shows *Active* in AVS Portal | [ ] |
|  | Site pairing and service mesh healthy | [ ] |
|  | Network extension (if used) operational | [ ] |
|  | Test migration completed successfully | [ ] |
| **AVS Interconnect** | Connection between old and new AVS private clouds validated | [ ] |

---

**Prepared by:** _Microsoft Cloud Solution Architect_  
**Environment:** Azure VMware Solution (Gen 1)  
**Last Updated:** October 20, 2025
