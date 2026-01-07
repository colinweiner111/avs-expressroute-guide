# 🧩 Azure VMware Solution (AVS) – ExpressRoute, Identity & HCX Setup Checklist (v7.2)

## 🌐 1. ExpressRoute Connectivity to On-Premises

📘 **References:**  
[Connect AVS Gen 1 to on-premises via ExpressRoute Global Reach](https://learn.microsoft.com/en-us/azure/azure-vmware/tutorial-expressroute-global-reach-private-cloud) | [Create an ExpressRoute circuit](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-howto-circuit-portal-resource-manager) | [AVS Networking Concepts](https://learn.microsoft.com/en-us/azure/azure-vmware/concepts-networking)

> 🧱 _This section applies to AVS Gen 1. For Gen 2, on-premises connectivity uses standard ExpressRoute connection through the Virtual Network gateway, not Global Reach._

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
- Confirm that **Global Reach** connectivity is enabled and active (if used).  
- From a jump box or on-prem VM, run `tracert` / `Test-NetConnection` to confirm reachability to AVS vCenter and NSX Manager FQDNs.  
- Review **Effective Routes** on your Azure VM NIC to ensure AVS subnets are advertised through ExpressRoute.  
- Ensure both AVS private clouds (old and new) show Connected status under **Connectivity → ExpressRoute** before proceeding with HCX.

---

## ✅ 2. Identity Configuration (Active Directory / LDAP)

📘 **Reference:** [Configure an external identity source in AVS (Gen 1)](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-identity-source-vcenter)

> 🧱 _This section applies to AVS Gen 1. For Gen 2, DNS is configured using native DNS forward lookup zones (public or private), and identity configuration follows a similar LDAPS/LDAP integration process._

### **Task Checklist**
- [ ] Validate DNS resolution between AVS and your AD domain (e.g., `corp.contoso.com`)
- [ ] Create and configure LDAPS service account with proper permissions
- [ ] Verify LDAPS connectivity and certificate validity
- [ ] Export LDAPS certificate from a domain controller (`.cer`) and upload to Azure Blob Storage
- [ ] Add Identity Source using Run Command → AddIdentitySource
- [ ] Verify identity source in vCenter
- [ ] Assign roles to AD groups or users (CloudAdmin / custom roles)
- [ ] Test login with `domain\user` credentials

---

### **🔍 Pre-Flight Troubleshooting & Validation**

**Run these tests BEFORE attempting AddIdentitySource to avoid common failures:**

| Area | Test Command | Expected Result | If Failed |
|------|-------------|----------------|-----------|
| **DNS** | `nslookup dc01.corp.contoso.com` | Returns DC IP address | Configure DNS forwarders in AVS |
| **DNS Reverse** | `nslookup <DC_IP>` | Returns DC FQDN | Configure reverse DNS zones |
| **Connectivity** | `Test-NetConnection -ComputerName dc01.corp.contoso.com -Port 636` | `TcpTestSucceeded : True` | Check NSG, firewall, ExpressRoute routing |
| **Certificate** | `openssl s_client -connect dc01.corp.contoso.com:636` | Displays cert chain, no errors | Install valid LDAPS cert on DC |
| **TLS Version** | Check output from openssl command above | Shows `TLSv1.2` or `TLSv1.3` | Enable TLS 1.2+ on Domain Controllers |
| **Auth Test** | `ldp.exe` (Windows) - bind to `ldaps://dc01.corp.contoso.com:636` | Successful bind | Verify service account credentials |

---

### **🔑 Service Account Requirements**

Create a dedicated service account for vCenter LDAPS binding:

#### **Active Directory Configuration**

```powershell
# Create service account in AD (run on Domain Controller)
New-ADUser -Name "svc-vcenter-ldap" `
  -SamAccountName "svc-vcenter-ldap" `
  -UserPrincipalName "svc-vcenter-ldap@corp.contoso.com" `
  -Path "OU=ServiceAccounts,DC=corp,DC=contoso,DC=com" `
  -AccountPassword (ConvertTo-SecureString "YourSecurePassword!" -AsPlainText -Force) `
  -Enabled $true `
  -PasswordNeverExpires $true `
  -CannotChangePassword $true

# Verify account was created
Get-ADUser -Identity "svc-vcenter-ldap" | Select-Object Name,Enabled,PasswordNeverExpires
```

#### **Required Permissions**

| Permission | Scope | Purpose |
|------------|-------|---------|
| **Read** | Base DN (e.g., `DC=corp,DC=contoso,DC=com`) | Search users and groups |
| **Member of Domain Users** | Default group membership | Standard read access |

> ✅ **No admin rights required** - service account only needs read access to AD  
> ⚠️ **Do NOT** make this account a Domain Admin

#### **Security Best Practices**

- [ ] Use a dedicated service account (not a user account)
- [ ] Set `PasswordNeverExpires = true` to avoid auth failures
- [ ] Document password in secure vault (Azure Key Vault, password manager)
- [ ] Restrict account to "Logon To" specific DCs only (optional)
- [ ] Enable "Account is sensitive and cannot be delegated" (optional security hardening)

---

### **Detailed Steps**

| Step | Action | Location | Notes / Inputs | Reference |
|------|--------|-----------|----------------|------------|
| **2.1** | **Validate DNS resolution** between AVS and your AD domain | **Azure Portal** | Configure DNS forwarders so vCenter can resolve DCs | [Configure DNS forwarder for AVS (Gen 1)](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-dns-azure-vmware-solution) |
| **2.1a** | **Test DNS from AVS** | Run Command → `Invoke-PreflightDnsTest` | Verify DC FQDN resolves correctly | See validation commands below |
| **2.2** | **Create LDAPS service account** | Active Directory | See Service Account Requirements section above | — |
| **2.3** | **Verify LDAPS certificate on DC** | Domain Controller | Cert must have Server Auth EKU, valid SAN, not expired | See certificate validation below |
| **2.4** | **Export LDAPS certificate** from a domain controller and upload to Azure Blob Storage | **Azure Portal / Storage Account** | Export full chain as Base64 `.cer` | [Step 2 – Export certificate](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-identity-source-vcenter#step-2-export-the-domain-controller-certificate) |
| **2.5** | **Generate SAS URL for certificate** | Azure Storage Account | Use 24-48 hour expiration | See command syntax below |
| **2.6** | **Add Identity Source** using *Run Command → AddIdentitySource* | **Azure Portal → AVS → Run Command** | See complete command example below | [Step 3 – AddIdentitySource](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-identity-source-vcenter#step-3-run-the-addidentitysource-command) |
| **2.7** | **Verify identity source in vCenter** | **vCenter UI** | *Administration → SSO → Configuration → Identity Sources* | [Step 4 – Verify](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-identity-source-vcenter#step-4-verify-the-identity-source) |
| **2.8** | **Test AD search functionality** | **vCenter UI** | Administration → Access Control → Add → Search for AD user/group | Confirms LDAPS bind is working |
| **2.9** | **Assign roles** to AD groups or users** | **vCenter UI** | Use least privilege and governance best practices | [AVS Identity Architecture](https://learn.microsoft.com/en-us/azure/azure-vmware/architecture-identity) |
| **2.10** | **Test login** with `domain\user` credentials | **vCenter UI** | Validate successful AD authentication | — |

---

### **📋 Complete Command Examples**

#### **Step 2.5: Generate SAS URL for Certificate**

```powershell
# Upload certificate to Azure Blob Storage
$storageAccount = "yourstorageaccount"
$containerName = "certs"
$certFile = "C:\Temp\dc-ldaps-cert.cer"
$blobName = "dc-ldaps-cert.cer"

# Upload file
az storage blob upload `
  --account-name $storageAccount `
  --container-name $containerName `
  --name $blobName `
  --file $certFile

# Generate SAS URL (valid for 24 hours)
$sasToken = az storage blob generate-sas `
  --account-name $storageAccount `
  --container-name $containerName `
  --name $blobName `
  --permissions r `
  --expiry (Get-Date).AddHours(24).ToString("yyyy-MM-ddTHH:mm:ssZ") `
  --https-only `
  --output tsv

$sasUrl = "https://$storageAccount.blob.core.windows.net/$containerName/$blobName?$sasToken"
Write-Host "SAS URL: $sasUrl"
```

#### **Step 2.6: Complete AddIdentitySource Command**

```powershell
# AVS Run Command → AddIdentitySource
# Replace values with your environment

-Name "corp.contoso.com"
-DomainName "corp.contoso.com"
-DomainAlias "CORP"
-PrimaryUrl "ldaps://dc01.corp.contoso.com:636"
-SecondaryUrl "ldaps://dc02.corp.contoso.com:636"  # Optional but recommended
-BaseDNUsers "DC=corp,DC=contoso,DC=com"
-BaseDNGroups "DC=corp,DC=contoso,DC=com"
-Credential (New-Object PSCredential("svc-vcenter-ldap", (ConvertTo-SecureString "YourServiceAccountPassword" -AsPlainText -Force)))
-SSLCertificatesSasUrl "https://yourstorageaccount.blob.core.windows.net/certs/dc-ldaps-cert.cer?sv=2021..."
-GroupName "vsphere-admins"  # Optional: AD group to auto-assign CloudAdmin role
```

**Parameter Breakdown:**

| Parameter | Value | Notes |
|-----------|-------|-------|
| `-Name` | `corp.contoso.com` | Display name for identity source |
| `-DomainName` | `corp.contoso.com` | **Case-sensitive** - must match exactly |
| `-DomainAlias` | `CORP` | NetBIOS name (for `CORP\username` login format) |
| `-PrimaryUrl` | `ldaps://dc01.corp.contoso.com:636` | Primary DC FQDN with LDAPS port |
| `-SecondaryUrl` | `ldaps://dc02.corp.contoso.com:636` | Optional secondary DC for redundancy |
| `-BaseDNUsers` | `DC=corp,DC=contoso,DC=com` | Where to search for users |
| `-BaseDNGroups` | `DC=corp,DC=contoso,DC=com` | Where to search for groups |
| `-Credential` | Service account username & password | Use dedicated LDAPS service account |
| `-SSLCertificatesSasUrl` | SAS URL from Step 2.5 | Must be accessible from AVS |
| `-GroupName` | `vsphere-admins` | Optional: auto-assign CloudAdmin to this AD group |

---

### **✅ Validation Commands (After Each Step)**

#### **Validate DNS Resolution (Step 2.1)**

```powershell
# From jump box or Azure VM with access to AVS
nslookup dc01.corp.contoso.com
nslookup dc02.corp.contoso.com

# Expected output: Returns IP address(es)
```

#### **Validate LDAPS Connectivity (Step 2.3)**

```powershell
# Test LDAPS port connectivity
Test-NetConnection -ComputerName dc01.corp.contoso.com -Port 636

# Expected output: TcpTestSucceeded : True
```

#### **Validate LDAPS Certificate (Step 2.3)**

```bash
# Using OpenSSL (Git Bash, WSL, or Linux)
openssl s_client -connect dc01.corp.contoso.com:636 -showcerts

# Check for:
# ✅ Certificate chain displayed
# ✅ "Verification: OK" or similar
# ✅ "subject=CN=dc01.corp.contoso.com" matches FQDN
# ✅ "TLS handshake" completes
# ❌ No "certificate verify failed" errors
```

#### **Validate Identity Source Added (Step 2.7)**

```powershell
# In vCenter PowerCLI
Get-IdentitySource | Where-Object {$_.Name -eq "corp.contoso.com"}

# Expected output: Returns identity source details
```

#### **Validate AD Search (Step 2.8)**

In vCenter UI:
1. **Administration → Access Control → Global Permissions**
2. Click **+ Add**
3. Search for a known AD user (e.g., `jsmith`)
4. ✅ **Expected:** User appears in results
5. ❌ **If no results:** LDAPS bind issue - check service account credentials

---

### **🚨 Common Error Messages & Resolutions**

| Error Message | Cause | Resolution |
|---------------|-------|------------|
| `"Cannot resolve host"` | DNS failure | Verify DNS forwarders in AVS pointing to AD DNS servers |
| `"Connection timed out"` | Port 636 blocked | Check NSG, Azure Firewall, on-prem firewall rules |
| `"Certificate verification failed"` | Cert issue | Ensure cert has Server Auth EKU, valid SAN, not expired |
| `"Invalid credentials"` | Bind account password wrong | Verify service account password, check account not locked |
| `"Unable to search"` | Base DN incorrect | Verify `DC=corp,DC=contoso,DC=com` matches AD structure |
| `"SSL handshake failed"` | TLS version mismatch | Enable TLS 1.2+ on Domain Controllers |
| `"Users not found"` | Wrong search base | Ensure users are within Base DN scope |
| `"Run Command timed out"` | SAS URL expired/inaccessible | Regenerate SAS URL with longer expiration |

---

## � 2.x vCenter LDAPS Troubleshooting & Best Practices

📘 **References:**  
[Configure external identity source in AVS](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-identity-source-vcenter) | [vCenter identity architecture](https://learn.microsoft.com/en-us/azure/azure-vmware/architecture-identity)

> ✅ **Scope:** This section applies when customers require **vCenter authentication via LDAPS to Active Directory**. This is **separate** from Microsoft Entra ID (OIDC) authentication and **does not apply to NSX‑T Manager**.

### **Supported Authentication Options in AVS**

| Component | Supported Identity Sources |
|-----------|---------------------------|
| **vCenter Server** | ✅ Local SSO users (`vsphere.local`)<br>✅ LDAP / LDAPS to Windows Active Directory<br>✅ Microsoft Entra ID (OIDC) |
| **NSX‑T Manager** | ✅ Local NSX users<br>✅ LDAP / LDAPS to Windows Active Directory<br>❌ Microsoft Entra ID **not supported** |

> ⚠️ **Important Design Decision:**  
> vCenter supports **either** Entra ID (OIDC) **or** LDAP/LDAPS as the primary external identity provider.  
> They can coexist but should **not be mixed** during initial configuration or troubleshooting.

---

### **🧯 Mandatory Safety Check (Break‑Glass Access)**

**Before modifying any identity source:**

- ✅ Confirm access to **`cloudadmin@vsphere.local`**
- ✅ Do **not** proceed if this account cannot log in
- ✅ This account is **not affected** by LDAPS failures and prevents total lockout

---

### **🔄 Reset vCenter to a Known‑Good State (When LDAPS Fails)**

If LDAPS authentication is failing or producing errors:

| Step | Action | Location | Purpose |
|------|--------|----------|---------|
| **2.x.1** | Log in to vCenter as `cloudadmin@vsphere.local` | **vCenter UI** | Bypass LDAPS authentication |
| **2.x.2** | Navigate to **Administration → Single Sign‑On → Configuration → Identity Sources** | **vCenter UI** | View current identity providers |
| **2.x.3** | Remove or disable existing **LDAP / LDAPS** identity sources | **vCenter UI** | Isolate the issue |
| **2.x.4** | Confirm vCenter login works using local SSO users only | **vCenter UI** | Establish baseline |

> ✅ This isolates the issue and prevents lockouts during troubleshooting.

---

### **✅ Validate Active Directory LDAPS Prerequisites (Before Re‑Adding)**

> 🚨 **Most vCenter LDAPS failures are caused by Active Directory misconfiguration.**

#### **Domain Controller Requirements**

Each Domain Controller **must** have:

- ✅ Valid **LDAPS certificate** installed on DC
- ✅ Certificate includes:
  - **Server Authentication EKU** (Extended Key Usage)
  - **SAN = DC FQDN** (Subject Alternative Name)
  - Not expired or revoked
- ✅ **TLS 1.2 enabled** (TLS 1.0/1.1 will cause silent failures)

> ❗ **Critical:** If the DC certificate was **renewed**, vCenter must be updated with the new certificate chain.

#### **LDAPS Connectivity Test (Before vCenter Configuration)**

From a system with network visibility to AVS (e.g., jump box or Azure VM):

```powershell
# Test LDAPS connectivity
Test-NetConnection -ComputerName dc01.corp.contoso.com -Port 636

# Validate certificate (requires OpenSSL or similar)
openssl s_client -connect dc01.corp.contoso.com:636
```

✅ **Expected result:** TLS handshake succeeds with no certificate validation errors  
❌ **If this fails:** Do not proceed to vCenter configuration until resolved

---

### **➕ Add LDAPS Identity Source to vCenter (Correct Configuration)**

Use **Run Command → AddIdentitySource** (AVS Gen 1) or manual vCenter configuration:

| Field | Value | Notes |
|-------|-------|-------|
| **Identity Source Type** | LDAP over SSL | Always use LDAPS (636), not LDAP (389) |
| **Primary Server URL** | `ldaps://dc01.corp.contoso.com:636` | Use FQDN, not IP address |
| **Secondary Server** | `ldaps://dc02.corp.contoso.com:636` | Optional but recommended |
| **Base DN** | `DC=corp,DC=contoso,DC=com` | Must match your AD structure |
| **Bind DN** | `CN=svc-vcenter-bind,OU=ServiceAccounts,DC=corp,DC=contoso,DC=com` | Service account with AD read permissions |
| **Bind Password** | *(Service account password)* | Avoid special characters during initial testing |
| **Domain Name** | `corp.contoso.com` | **Case-sensitive** — must match exactly |
| **Certificate** | LDAPS cert chain from DC | Upload `.cer` file to Azure Blob + SAS URL (Gen 1) |

> 💡 **Tip:** Avoid special characters in the bind account password during initial testing.

---

### **✅ Validation Steps (Confirm LDAPS is Working)**

| Step | Action | Location | Expected Result |
|------|--------|----------|----------------|
| **2.x.5** | Verify identity source appears | **Administration → SSO → Identity Sources** | Domain listed as identity source |
| **2.x.6** | Search for AD users/groups | **Administration → Access Control** | Users/groups return results |
| **2.x.7** | Assign a role to an AD group | **Administration → Access Control** | CloudAdmin or custom role assigned |
| **2.x.8** | Log out and log in with `domain\user` | **vCenter UI** | Successful authentication |

✅ **Success:** AD users can authenticate to vCenter via LDAPS

---

### **🚨 Common LDAPS Failure Scenarios (90% of Issues)**

| Issue | Symptom | Resolution |
|-------|---------|-----------|
| **DC certificate renewed** | "Invalid credentials" or silent auth failures | Export new cert from DC and update vCenter |
| **TLS 1.0 / 1.1 only on DC** | Connection timeout or handshake failure | Enable TLS 1.2 on Domain Controllers |
| **Wrong Base DN** | Users/groups not found in search | Verify `DC=corp,DC=contoso,DC=com` matches AD |
| **Bind password expired** | Authentication errors in vCenter | Reset service account password |
| **Domain name case mismatch** | Intermittent auth failures | Ensure domain name matches exact case |
| **Firewall blocking port 636** | Connection timeout | Verify NSG / Azure Firewall rules allow 636 |
| **Certificate SAN mismatch** | LDAP bind failures | Cert SAN must match DC FQDN |
| **LDAPS cert chain incomplete** | Certificate trust errors | Include root + intermediate certs |

---

### **🔑 Key Takeaways**

- ✅ vCenter LDAPS is **fully supported** in AVS (Gen 1 and Gen 2)
- ❌ Microsoft Entra ID **does not replace LDAPS** for NSX‑T
- ✅ Always **validate LDAPS outside vCenter first** (using `openssl` or `Test-NetConnection`)
- ✅ Never modify identity sources **without confirmed break‑glass access**
- ✅ NSX‑T authentication is **separate** and unaffected by vCenter LDAPS configuration

---

### **📍 AD Location-Specific Considerations**

| AD Location | Key Considerations |
|-------------|-------------------|
| **Inside AVS** | ✅ No ExpressRoute dependency<br>✅ Lowest latency<br>⚠️ Requires AVS-hosted domain controllers |
| **Azure IaaS (VNet)** | ✅ Private connectivity via VNet peering<br>⚠️ Verify NSG rules allow 636<br>⚠️ DNS resolution required |
| **On-Premises (ExpressRoute)** | ⚠️ TLS inspection risk (proxies/firewalls)<br>⚠️ ExpressRoute latency<br>⚠️ Certificate trust boundaries |

> 💡 **Tip:** For on-premises AD over ExpressRoute, ensure no TLS inspection is occurring on port 636, as this will break certificate validation.

---

## 🔧 Troubleshooting LDAPS Login Failures (After Run Command Succeeds)

> **Scenario:** AVS Run Command → AddIdentitySource completed successfully, but users still cannot log in to vCenter with AD credentials.

### **Phase 1: Verify LDAPS Configuration is Active**

| Step | Action | Expected Result | If Failed |
|------|--------|----------------|-----------|
| **1** | Log in to vCenter as `cloudadmin@vsphere.local` | Successful login | Cannot proceed - verify break-glass access |
| **2** | Navigate to **Administration → Single Sign-On → Configuration → Identity Sources** | Domain appears in list | Re-run AddIdentitySource command |
| **3** | Note the **exact domain name** shown (including case) | Example: `corp.contoso.com` or `CORP.CONTOSO.COM` | This is critical for login format |
| **4** | Click **+ Add** in **Global Permissions** → search for a known AD user or group | Returns AD search results | LDAPS bind issue - verify bind account password |

✅ If Step 4 returns results, LDAPS is working correctly. Proceed to Phase 2.

---

### **Phase 2: Verify User Permissions**

| Step | Action | Location | Notes |
|------|--------|----------|-------|
| **5** | Search for the user's AD group | **Administration → Access Control → Global Permissions** | Example: `Domain Admins` or security group |
| **6** | Check if the group has assigned roles | **Global Permissions** list | Must have CloudAdmin or custom role assigned |
| **7** | If no permissions exist, add them | Click **+ Add** → search for AD group → assign role | Most common cause of login failures |
| **8** | Verify user is member of the AD group | Check in Active Directory | User must be in the group assigned permissions |

---

### **Phase 3: Test Login with Correct Format**

| Login Format | Example | When to Use |
|--------------|---------|-------------|
| **DOMAIN\username** | `CORP\jsmith` | NetBIOS domain name (most common) |
| **domain.fqdn\username** | `corp.contoso.com\jsmith` | Fully qualified domain name |
| **username@domain.fqdn** | `jsmith@corp.contoso.com` | UPN format (if configured in AD) |

> ⚠️ **Critical:** Domain name is **case-sensitive**. Use the exact case shown in **Identity Sources** (Step 3 above).

**Test Procedure:**
1. Log out of vCenter
2. Try login using format: `DOMAIN\username` (replace DOMAIN with exact case from Identity Sources)
3. If failed, try format: `domain.fqdn\username`
4. If failed, try format: `username@domain.fqdn`

---

### **Phase 4: Validate Active Directory Account Status**

| Check | How to Verify | Resolution |
|-------|---------------|------------|
| **Account enabled** | AD Users and Computers → user properties | Enable the account |
| **Account not locked** | AD Users and Computers → Account tab | Unlock the account |
| **Password not expired** | AD Users and Computers → Account tab | Reset password |
| **"User must change password at next logon"** | AD Users and Computers → Account tab | Uncheck this option |
| **User in correct OU** | Verify user is within the Base DN scope | Move user or adjust Base DN |

---

### **Phase 5: Test with Known Admin Account**

**Purpose:** Isolate whether the issue is configuration-wide or user-specific.

1. Identify a known working account (e.g., Domain Administrator)
2. Ensure this account has vCenter permissions assigned (Step 6-7)
3. Attempt login using exact format from Phase 3

**Results:**
- ✅ **Admin account works** → Original user needs permissions or has account issues
- ❌ **Admin account fails** → Domain name case mismatch or configuration error

---

### **Phase 6: Advanced Diagnostics**

#### **Check vCenter Identity Source Details**

1. **Administration → Single Sign-On → Configuration → Identity Sources**
2. Select the domain → click **Edit**
3. Verify:

| Field | What to Check |
|-------|---------------|
| **Primary URL** | Must match DC FQDN: `ldaps://dc01.corp.contoso.com:636` |
| **Base DN** | Must match AD structure: `DC=corp,DC=contoso,DC=com` |
| **Domain Name** | Exact case matches what users are typing |

#### **Verify Certificate is Current**

If DC certificate was recently renewed:
1. Export **new certificate** from Domain Controller
2. Upload to Azure Blob Storage with SAS URL
3. Re-run **AddIdentitySource** command with updated certificate

#### **Check AVS Run Command History**

1. Azure Portal → AVS → **Run Command → Execution History**
2. Find the **AddIdentitySource** execution
3. Review **Input Parameters** to confirm:
   - DomainName
   - PrimaryURL
   - BaseDN
   - Username (bind account)

---

### **🎯 Common Resolution Checklist**

| Issue | Symptoms | Fix |
|-------|----------|-----|
| **No permissions assigned** | All users fail to log in | Assign CloudAdmin role to AD group in Global Permissions |
| **Domain case mismatch** | Login fails but search works | Use exact case from Identity Sources list |
| **Wrong login format** | "Invalid credentials" error | Try all three formats from Phase 3 |
| **Account locked in AD** | Specific user fails | Unlock account in Active Directory |
| **User not in assigned group** | Specific user fails | Add user to AD group with vCenter permissions |
| **Bind account password changed** | Search fails to return results | Update identity source with new password |
| **Certificate expired/renewed** | Sudden failures after working | Update certificate in identity source |

---

### **✅ Success Criteria**

After completing troubleshooting, verify:

- [ ] Identity source appears in vCenter
- [ ] AD user/group search returns results
- [ ] At least one AD group has vCenter role assigned
- [ ] Test user is member of assigned AD group
- [ ] Test user account is enabled and not locked in AD
- [ ] Domain name case matches exactly between vCenter and login
- [ ] Successful login with AD credentials using correct format

---

## �🚀 3. VMware HCX Deployment & Configuration

📘 **References:**  
[Install VMware HCX in AVS (Gen 1)](https://learn.microsoft.com/en-us/azure/azure-vmware/install-vmware-hcx) | [Configure VMware HCX](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-vmware-hcx) | [HCX Network Extension](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-hcx-network-extension)

> 🧱 _This section applies to AVS Gen 1. For Gen 2, HCX configuration follows the same process but uses native connectivity through Azure Virtual Network peering._

### **Task Checklist**
- [ ] Enable HCX Cloud Manager  
- [ ] Copy HCX license key and cloud URL  
- [ ] Deploy HCX Connector OVA in source SDDC vCenter  
- [ ] Activate HCX Connector with license key  
- [ ] Create AVS Interconnect between both SDDCs (if same region)  
- [ ] Pair source Connector with destination HCX Cloud Manager  
- [ ] Create Network Profiles & Compute Profiles  
- [ ] Build Service Mesh between both SDDCs  
- [ ] (Optional) Extend L2 Networks  
- [ ] Validate migration (Cold, vMotion, Bulk)

> 💡 **Tip:** For same-region migrations, **AVS Interconnect** replaces the need for ExpressRoute Global Reach and should be configured **before** HCX pairing.

### **Detailed Steps**

| Step | Action | Location | Notes / Inputs | Reference |
|------|--------|-----------|----------------|------------|
| **3.1** | **Enable HCX Cloud Manager** | **Azure Portal → Manage → Add-ons → HCX → Enable** | Deployment takes ~30 min | [Install HCX](https://learn.microsoft.com/en-us/azure/azure-vmware/install-vmware-hcx) |
| **3.2** | **Copy HCX license key and cloud URL** | **Azure Portal → Manage → Add-ons → HCX** | Needed for HCX Connector activation | same doc |
| **3.3** | **Deploy HCX Connector OVA** | **vSphere UI (source SDDC)** | Configure management and service networks | [Configure HCX](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-vmware-hcx) |
| **3.4** | **Activate HCX Connector** | **HCX UI** | Validate SSO and network connectivity | same doc |
| **3.5** | **Create AVS Interconnect between both SDDCs (same region)** | **Azure Portal → AVS → Connectivity → AVS Interconnect** | Establishes private routing between SDDCs in the same region for HCX pairing | [Connect multiple AVS private clouds (Gen 1 Interconnect)](https://learn.microsoft.com/en-us/azure/azure-vmware/connect-multiple-private-clouds-same-region#add-connection-between-private-clouds) |
| **📘 Prerequisite Note:** | If both SDDCs are in the same region and not connected via a shared ExpressRoute circuit or Global Reach, complete **Step 3.5 – AVS Interconnect** **before** pairing HCX sites. This ensures private connectivity for HCX Service Mesh. | — | — | — |
| **3.6** | **Pair sites** – Source HCX Connector → Destination HCX Cloud Manager | **HCX UI** | Authenticate with `cloudadmin@vsphere.local` credentials | [Configure HCX](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-vmware-hcx) |
| **3.7** | **Create Network Profiles & Compute Profiles** | **HCX UI** | Enables Service Mesh creation | same doc |
| **3.8** | **Build Service Mesh** | **HCX UI** | Ensure all services show *Healthy* | same doc |
| **3.9** | *(Optional)* **Extend L2 Networks** to AVS | **HCX UI** | For live migration or DR validation | [Network Extension](https://learn.microsoft.com/en-us/azure/azure-vmware/configure-hcx-network-extension) |
| **3.10** | **Validate migrations** (Cold, vMotion, Bulk) | **HCX UI** | Confirm vCenter and replication visibility | [Migration Architecture](https://learn.microsoft.com/en-us/azure/azure-vmware/architecture-migrate) |

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
**Last Updated:** October 21, 2025

