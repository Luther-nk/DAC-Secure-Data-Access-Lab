# DAC-Secure-Data-Access-Lab

> **Lab Type:** Windows Server — Dynamic Access Control (DAC) Implementation  
> **Author:** Luther NKAPNANG TCHATCHOUANG  
> **Role:** IT Support & Cybersecurity Analyst | Willis College, Sudbury, Ontario  
> **Course:** Willis College — Cybersecurity Analyst Diploma  
> **Assignment:** Assignment 3-1: Implementing Secure Data Access  
> **Certification Context:** CCNA | AZ-700 (in progress) | CompTIA Security+ (planned)

---

## Table of Contents

1. [Lab Overview](#1-lab-overview)
2. [Environment & Virtual Machines](#2-environment--virtual-machines)
3. [Lab Objectives](#3-lab-objectives)
4. [Exercise 1 — Preparing for DAC Deployment](#4-exercise-1--preparing-for-dac-deployment)
   - [Task 1 — Prepare AD DS for DAC Deployment](#task-1--prepare-ad-ds-for-dac-deployment)
   - [Task 2 — Configure User and Device Claims](#task-2--configure-user-and-device-claims)
   - [Task 3 — Configure Resource Properties and Resource Property Lists](#task-3--configure-resource-properties-and-resource-property-lists)
   - [Task 4 — Install File Server Resource Manager](#task-4--install-file-server-resource-manager)
   - [Task 5 — Implement File Classifications](#task-5--implement-file-classifications)
   - [Task 6 — Assign Department Property to the Docs Folder](#task-6--assign-department-property-to-the-docs-folder)
5. [Key Concepts Demonstrated](#5-key-concepts-demonstrated)
6. [Security Analysis](#6-security-analysis)
7. [Screenshots Index](#7-screenshots-index)
8. [References](#8-references)

---

## 1. Lab Overview

This laboratory demonstrates the implementation of **Dynamic Access Control (DAC)** in a Windows Server environment for a fictional enterprise, **Progreso Corporation**. DAC is a file access control technology that extends traditional NTFS permissions by incorporating user and device claims, resource classifications, and central access policies — enabling fine-grained, attribute-based access control (ABAC).

The scenario addresses the following business requirements:

| Requirement | DAC Solution Applied |
|---|---|
| Restrict Research team folder access to Research members only | User claim based on department attribute |
| Restrict confidential documents to Executive managers only | Confidentiality resource classification + Central Access Policy |
| Prevent managers from accessing files from unsecured home computers | Device claim based on ManagersWKS group membership |
| Help users understand access denied errors and request access automatically | Access Denied Assistance feature |
| Enable efficient data synchronization across personal devices | Work Folders feature |

---

## 2. Environment & Virtual Machines

| VM Name | Role | Domain |
|---|---|---|
| EDM-DC1 | Primary Domain Controller / DNS | Progreso.com |
| EDM-SVR1 | File Server / FSRM Host | Progreso.com |
| EDM-SVR2 | Secondary Server | Progreso.com |
| EDM-CL1 | Client Workstation (authorized) | Progreso.com |
| EDM-CL2 | Client Workstation | Progreso.com |

- **Hypervisor:** Microsoft Hyper-V
- **Operating System:** Windows Server 2025
- **Domain:** Progreso.com

---

## 3. Lab Objectives

After completing this lab, the following competencies are demonstrated:

- Preparing Active Directory Domain Services (AD DS) for DAC deployment
- Configuring the KDC to support claims and compound authentication
- Creating user and device claim types using the Active Directory Administrative Center (ADAC)
- Enabling and configuring resource properties (Department, Confidentiality)
- Installing and configuring the File Server Resource Manager (FSRM) role
- Creating content-based classification rules to automatically tag files
- Applying resource property classifications to folders

---

## 4. Exercise 1 — Preparing for DAC Deployment

---

### Task 1 — Prepare AD DS for DAC Deployment

**Performed on: EDM-DC1**

#### Step 1 — Create the DACProtected Organizational Unit

A dedicated OU named `DACProtected` was created under `Progreso.com` to house the computer objects subject to DAC policies.
![DACProtected OU Created](DACProtected%20Organizational%20Unit.jpg)

---

#### Step 2 — Move EDM-SVR1 and EDM-CL1 into DACProtected

The EDM-SVR1 and EDM-CL1 computer objects were moved from the default Computers container into the DACProtected OU to bring them under the scope of DAC group policies.

![Computers Moved to DACProtected](Move%20EDM-SVR1%20and%20EDM-CL1%20into%20DACProtected.jpg)

---

#### Step 3 — Enable KDC Support for Claims via Group Policy

The **Default Domain Controllers Policy** was edited to enable Kerberos Claims support — a prerequisite for DAC. The KDC must be configured to issue claims as part of Kerberos tickets.
Group Policy Management Console

→ Progreso.com → Group Policy Objects

→ Default Domain Controllers Policy → Edit

→ Computer Configuration → Policies → Administrative Templates → System → KDC

→ KDC support for claims, compound authentication and Kerberos armoring

→ Enabled | Always provide claims
![KDC Policy Enabled](Enable%20KDC%20Support%20for%20Claims%20via%20Group%20Policy.png)

---

#### Step 4 — Refresh Group Policy

Group Policy was refreshed on EDM-DC1 to apply the KDC claims policy immediately.

```powershell
gpupdate /force
```

![Group Policy Refreshed](Refresh%20Group%20Policy.png)

---

#### Step 5 — Create the ManagersWKS Security Group

A security group named `ManagersWKS` was created in the Users container. This group is used as a device claim to restrict file access to authorized manager workstations only.
Active Directory Users and Computers

→ Users container → New → Group

→ Group name: ManagersWKS | Scope: Global | Type: Security
![ManagersWKS Security Group Created](Create%20the%20ManagersWKS%20Security%20Group.png)

---

#### Step 6 — Add EDM-CL1 to ManagersWKS

The EDM-CL1 computer object was added to the ManagersWKS security group, designating it as an authorized manager workstation for DAC device claim evaluation.

![EDM-CL1 Added to ManagersWKS](Add%20EDM-CL1%20to%20ManagersWKS.png)

---

#### Step 7 — Create Users and Organizational Units

The following users were created in their respective OUs. The Marketing and Managers OUs were created first, then users were provisioned accordingly.

| First Name | Last Name | Logon Name | OU |
|---|---|---|---|
| Thompson | Ben | TBen | Marketing |
| Jude | Teddy | JTeddy | Marketing |
| Praiz | Timi | PTimi | Managers |
| John | Zeha | JZeha | Managers |

![Users and OUs Created](Create%20Users%20and%20Ous.png)

![Users and OUs Created — Additional View](Create%20Users%20and%20Ous_2.png)

---

### Task 2 — Configure User and Device Claims

**Performed on: EDM-DC1 via Active Directory Administrative Center (ADAC)**

#### Step 2 — Create Claim Type: Company Department

A claim type was created sourced from the `department` Active Directory attribute. This enables the KDC to embed the user's department value into their Kerberos ticket, which is then evaluated against file classification properties.

| Setting | Value |
|---|---|
| Source Attribute | department |
| Display Name | Company Department |
| Issued for | User and Computer |
| Suggested Values | Managers, Marketing |

![Company Department Claim Type](Create%20Claim%20Type_Company%20Department.png)

---

#### Step 3 — Create Claim Type: Description (Computer only)

A second claim type was created sourced from the `description` attribute, scoped to Computer objects only. This enables device-based claim evaluation.

| Setting | Value |
|---|---|
| Source Attribute | description |
| Display Name | description |
| Issued for | Computer only |

![Description Claim Type](Create%20Claim%20Type_%20Description%20_Computer%20only.png)

---

### Task 3 — Configure Resource Properties and Resource Property Lists

**Performed on: EDM-DC1**

#### Step 1 — Enable Department and Confidentiality Properties

The built-in `Department` and `Confidentiality` resource properties were enabled in the Dynamic Access Control → Resource Properties container.

![Properties Enabled](Enable%20Department%20and%20Confidentiality%20Properties.png)

---

#### Step 2 — Add Marketing as a Suggested Value for Department

The Department resource property was configured to include `Marketing` as a suggested classification value.

![Marketing Added to Department](Add%20Marketing%20to%20Department%20Property.png)

---

### Task 4 — Install File Server Resource Manager

**Performed on: EDM-SVR1**

The File Server Resource Manager (FSRM) role was installed via Server Manager. FSRM provides the classification engine required to automatically apply resource properties to files based on content analysis.
Server Manager → Manage → Add Roles and Features

→ File and Storage Services → File and iSCSI Services

→ File Server Resource Manager
---

### Task 5 — Implement File Classifications

**Performed on: EDM-SVR1**

#### Step 1 — Create C:\Docs Folder and Sample Files

The `C:\Docs` folder was created and populated with three text files (`Doc1.txt`, `Doc2.txt`, `Doc3.txt`), each containing the keyword `secret` to trigger the content-based classification rule.

![Docs Folder and Files Created](Create%20the%20Docs%20folder%20and%20files%20On%20EDM-SVR1.png)

---

#### Step 2 — Open FSRM and Refresh Classification Properties

FSRM Classification Properties were refreshed to confirm that `Department` and `Confidentiality` properties — synchronized from Active Directory — are available for use in classification rules.

![FSRM Classification Properties](Open%20FSRM%20and%20Refresh%20Classification%20Properties.png)

---

#### Step 3 — Create the Set Confidentiality Classification Rule

A content-based classification rule was created to automatically scan files in `C:\Docs` and assign `Confidentiality = High` to any file containing the string `secret`.

| Setting | Value |
|---|---|
| Rule Name | Set Confidentiality |
| Scope | C:\Docs |
| Classification Method | Content Classifier |
| Property | Confidentiality |
| Value | High |
| Classification Parameter | String: `secret` |
| Evaluation Type | Re-evaluate — Overwrite existing value |

![Classification Rule Created](Create%20Classification%20Rule.png)

---

#### Step 4 — Run the Classification Rule

The classification rule was executed immediately using **Run Classification with All Rules Now** from the FSRM Actions pane.

![Classification Rule Running](Run%20the%20Classification%20Rule.png)

---

#### Step 5 — Verify Classification Results

The `C:\Docs` folder and individual files were inspected on the Classification tab to confirm `Confidentiality = High` was applied successfully.

![Verify Classification Results](Verify%20Classification%20Results.png)

![Verify Classification Results — Files](Verify%20Classification%20Results2.png)

---

### Task 6 — Assign Department Property to the Docs Folder

**Performed on: EDM-SVR1**

The `C:\Docs` folder was manually classified with `Department = Marketing` using the Classification tab in folder properties. This enables Central Access Policies to enforce department-based access restrictions.

![Department = Marketing Assigned to Docs Folder](Assign%20Department%20Property%20to%20the%20Docs%20Folder%20on%20EDM-SVR1.png)

---

## 5. Key Concepts Demonstrated

### Dynamic Access Control Architecture

DAC operates through three interdependent components:
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────────┐

│   User Claims    │    │  Device Claims   │    │ Resource Properties  │

│  (department,    │ +  │ (ManagersWKS     │ +  │ (Confidentiality,    │

│   job title)     │    │  group member)   │    │  Department)         │

└──────────────────┘    └──────────────────┘    └──────────────────────┘

│                      │                         │

└──────────────────────┴─────────────────────────┘

│

┌───────────▼────────────┐

│  Central Access Policy  │

│  (evaluated at runtime) │

└────────────────────────┘

### Claim-Based Access Control Flow

1. User authenticates → KDC issues Kerberos ticket **with embedded claims** (e.g., `department = Marketing`)
2. User's device authenticates → KDC issues device claims (e.g., `member of ManagersWKS`)
3. User accesses a file → File server evaluates **user claims + device claims** against **resource classification**
4. Access is granted or denied based on Central Access Policy rules

### Content-Based File Classification

FSRM's Content Classifier scans file content and automatically applies resource properties based on keyword matches. This enables:
- Automatic tagging of files containing sensitive keywords (`secret`, `confidential`, `PII`)
- Consistent classification without relying on manual user action
- Enforceable access policies that follow the data regardless of file location

---

## 6. Security Analysis

| Security Control | Implementation | Principle |
|---|---|---|
| **Attribute-Based Access Control** | DAC user and device claims | Beyond discretionary access — context-aware |
| **Device Trust Verification** | ManagersWKS group device claim | Ensures access only from authorized workstations |
| **Data Classification** | FSRM content-based classification | Confidentiality tagging independent of location |
| **Kerberos Claims** | KDC policy — Always provide claims | Claims embedded in authentication tickets |
| **Least Privilege** | Department-scoped resource properties | Users access only department-relevant resources |
| **Centralized Policy Management** | Global Resource Property List | Single point of control for all file servers |

> **Key Security Insight:** Traditional NTFS permissions control access based on *who you are* (identity). DAC extends this to control access based on *who you are + what device you are using + what the file contains* — a significant improvement for sensitive enterprise environments.

---

## 7. Screenshots Index

| Filename | Task | Description |
|---|---|---|
| `DACProtected Organizational Unit.jpg` | Task 1 Step 1 | DACProtected OU created |
| `Move EDM-SVR1 and EDM-CL1 into DACProtected.jpg` | Task 1 Step 2 | Computers moved to DACProtected |
| `Enable KDC Support for Claims via Group Policy.png` | Task 1 Step 3 | KDC policy enabled |
| `Refresh Group Policy.png` | Task 1 Step 4 | gpupdate /force output |
| `Create the ManagersWKS Security Group.png` | Task 1 Step 5 | ManagersWKS group created |
| `Add EDM-CL1 to ManagersWKS.png` | Task 1 Step 6 | EDM-CL1 added to ManagersWKS |
| `Create Users and Ous.png` | Task 1 Step 7 | Users and OUs created |
| `Create Users and Ous_2.png` | Task 1 Step 7 | Users and OUs — additional view |
| `Create Claim Type_Company Department.png` | Task 2 Step 2 | Company Department claim type |
| `Create Claim Type_ Description _Computer only.png` | Task 2 Step 3 | Description claim type (Computer) |
| `Enable Department and Confidentiality Properties.png` | Task 3 Step 1 | Properties enabled in ADAC |
| `Add Marketing to Department Property.png` | Task 3 Step 2 | Marketing added to Department |
| `Create the Docs folder and files On EDM-SVR1.png` | Task 5 Step 1 | C:\Docs folder and files |
| `Open FSRM and Refresh Classification Properties.png` | Task 5 Step 2 | FSRM Classification Properties |
| `Create Classification Rule.png` | Task 5 Step 3 | Classification rule configured |
| `Run the Classification Rule.png` | Task 5 Step 4 | Classification rule executed |
| `Verify Classification Results.png` | Task 5 Step 5 | Folder Confidentiality = High |
| `Verify Classification Results2.png` | Task 5 Step 5 | Files Confidentiality = High |
| `Assign Department Property to the Docs Folder on EDM-SVR1.png` | Task 6 | Department = Marketing assigned |

---

## 8. References

- [Microsoft Docs — Dynamic Access Control Overview](https://learn.microsoft.com/en-us/windows-server/identity/solution-guides/dynamic-access-control-overview)
- [Microsoft Docs — Deploy Claims-Based Access Control](https://learn.microsoft.com/en-us/windows-server/identity/solution-guides/deploy-a-central-access-policy)
- [Microsoft Docs — File Server Resource Manager](https://learn.microsoft.com/en-us/windows-server/storage/fsrm/fsrm-overview)
- [Microsoft Docs — KDC Support for Claims](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-claims-based-access-control)
- [Microsoft Docs — Classification Rules in FSRM](https://learn.microsoft.com/en-us/windows-server/storage/fsrm/create-classification-rule)

---

*This lab is part of a portfolio of IT infrastructure and cybersecurity labs developed during the Cybersecurity Analyst Diploma program at Willis College, Sudbury, Ontario.*
