# Windows Endpoint Management and Security Baseline with Microsoft Intune

A hands-on lab in a personal Microsoft 365 trial tenant. I joined a physical Windows 11 laptop to Microsoft Entra ID, enrolled it in Microsoft Intune, applied a Microsoft Defender Antivirus policy and a Windows compliance policy through a security group, and checked the results in the Intune admin center. Later the device flipped to Noncompliant, so I traced the failing setting and fixed it. That is documented below too.

**Skills shown:** Microsoft Intune, Microsoft Entra ID, device join and MDM enrollment, security groups, endpoint security policy, device compliance, DNS and custom domain setup, troubleshooting.

> Status: built 14 November 2026. Compliance fault found and fixed 21 November 2026. This is a learning lab in a trial tenant, and the limitations are listed at the end.

## Environment

| Item | Detail |
|---|---|
| Tenant | Microsoft 365 trial tenant ("Sink State") with a custom domain registered at IONOS |
| Test device | HP ProBook 640 G5, a physical decommissioned laptop (not a VM) |
| OS | Fully wiped, then a clean install of Windows 11 26H2 from a USB stick |
| Identity | Joined to Microsoft Entra ID (Settings > Accounts > Access work or school > Join this device to Microsoft Entra ID), signed in with a lab user |
| Management | Microsoft Intune (MDM), enrolled automatically on join because the MDM user scope was set to All |
| Ownership shown in Intune | Personal |
| Targeting | Security group `SG-LAB-DEVICES` |

```mermaid
flowchart LR
  A[Custom domain verified in Entra] --> B[Lab user]
  B --> C[Windows laptop: Entra joined + Intune enrolled]
  C --> D[Security group SG-LAB-DEVICES]
  D --> E[Defender Antivirus policy]
  D --> F[Windows compliance policy]
  F --> G{Compliant or Noncompliant}
```

## What I built

### 1. Custom domain

I bought a domain at IONOS, added the Microsoft 365 records (TXT verification, MX, the SPF TXT, autodiscover, and the `enterpriseenrollment` and `enterpriseregistration` CNAMEs that let Windows find the Intune enrollment service), and verified it in Entra, where it is set as the primary domain.

![IONOS DNS records](screenshots/01-IONOS-DNS-Records.png)
![IONOS domain overview](screenshots/02-IONOS-Domain-Overview.png)
![Custom domain verified in Entra](screenshots/03-Entra-Custom-Domain-Verified.png)

### 2. Checked the enrollment settings

MDM user scope was set to All, so lab users can enroll devices, and the default enrollment restriction allows Windows (MDM), including personally owned devices.

![MDM user scope](screenshots/04-Intune-Enrollment-Configuration.png)
![Enrollment restrictions](screenshots/05-Enrollment-Restrictions.png)
![Enrollment restriction properties](screenshots/06-Enrollment-Restriction-Properties.png)

### 3. Created a device security group

`SG-LAB-DEVICES` holds the test device. Assigning policy to a group is how this is done in practice, because it scales and is easy to audit. Before enrollment, the Windows devices list was empty.

![No Windows devices before enrollment](screenshots/07-Windows-Devices-Before-Enrollment.png)
![Adding the device to the group](screenshots/08-SG-LAB-DEVICES-Add-Device.png)

### 4. Created and assigned a Microsoft Defender Antivirus policy

Created under Endpoint security > Antivirus and assigned to `SG-LAB-DEVICES`. The configured settings include real-time monitoring, script scanning, on-access protection, and real-time scanning of all files in both directions. The policy reports Succeeded on the device with no errors or conflicts.

![Policy assignment](screenshots/09-Defender-Policy-Assignment.png)
![Policy assigned](screenshots/10-Defender-Policy-Assigned.png)
![Configured settings](screenshots/11-Defender-Policy-Configured-Settings.png)
![Policy status: Succeeded](screenshots/12-Defender-Policy-Status-Succeeded.png)

### 5. Joined and enrolled the Windows device

After the clean install I joined the laptop to Entra ID, and it enrolled in Intune. In Entra the laptop shows a join type of Microsoft Entra joined, with Intune as the MDM. (The iPhone in the same list is from my separate iOS MDM lab and shows as registered, which is a different relationship. The difference is covered in "What I learned".)

![Entra devices: join type](screenshots/13-Entra-Devices-Join-Type.png)
![First check-in in Intune](screenshots/14-Device-Overview-First-Check-In.png)

### 6. Created a Windows 10/11 compliance policy

The policy requires:

- Secure Boot
- Firewall
- Trusted Platform Module (TPM)
- Antivirus
- Antispyware
- Microsoft Defender Antimalware
- Microsoft Defender Antimalware security intelligence up-to-date
- Real-time protection

The noncompliance action is "Mark device noncompliant" immediately, and the policy is assigned to `SG-LAB-DEVICES`.

![Required settings](screenshots/17-Compliance-Policy-Required-Settings.png)
![Actions and assignment](screenshots/18-Compliance-Policy-Actions-and-Assignment.png)

### 7. Synced and verified

I synced the device from the Intune admin center. Both policies applied, the device reported Compliant, and the Defender policy showed Succeeded.

![Sync and policy status](screenshots/19-Device-Sync-and-Policy-Status.png)
![Compliance policy: device status](screenshots/20-Compliance-Policy-Device-Status.png)
![Defender policy deployment succeeded](screenshots/21-Defender-Policy-Deployment-Succeeded.png)

## Troubleshooting

### The device went Noncompliant on 21 November

A week after the lab was working, the device list showed the laptop as Noncompliant.

![Devices list showing Noncompliant](screenshots/22-Devices-List-Noncompliant.png)

I opened the device, then Device compliance, then the policy's per-setting view. Seven settings were Compliant and one was Not compliant: **Microsoft Defender Antimalware security intelligence up-to-date**. The laptop's Defender definitions were out of date, so the compliance check correctly failed.

![The failing setting](screenshots/23-Compliance-Policy-Failing-Setting.png)

**Fix:** I updated the Defender security intelligence on the laptop, then used Sync on the device in Intune. After the next check-in all eight settings showed Compliant, and the device list and device overview went back to Compliant (check-in 21 Nov, 08:45).

![All settings Compliant after the fix](screenshots/24-Compliance-Policy-All-Compliant-After-Fix.png)
![Devices list: Compliant](screenshots/25-Devices-List-Compliant.png)
![Device overview: Compliant](screenshots/26-Windows-Device-Overview-Compliant.png)
![Devices overview: no noncompliant devices](screenshots/27-Devices-Overview-All-Compliant.png)

This is the compliance policy doing its job: a device that falls behind on protection gets flagged, and the per-setting report shows exactly why.

### Other issues from the build

- **Default compliance policy showed an error.** Before my policy reached the device, the built-in Default Device Compliance Policy showed "Has a compliance policy assigned: Error 65001 (Not applicable)". That is expected when no custom policy has applied yet, and it cleared once my policy was assigned and the device synced.

  ![Error 65001 on the default policy](screenshots/15-Default-Compliance-Policy-Error-65001.png)

- **Template error on the noncompliance action.** Adding a "Send email to end user" action failed with "Please select a template", because that action needs a notification message template. I kept the "Mark device noncompliant" action for this lab.

  ![Template error](screenshots/16-Compliance-Policy-Template-Error.png)

- **Reporting delay.** The summary tiles on the policy page showed 0 while the device row already showed Compliant. Intune warns that reports can lag, and the numbers caught up after a refresh.

Personal details such as email addresses, device IDs, serial numbers and device owner names have been redacted with black boxes.

## Limitations (and what I'd do next)

- **Single test device in a trial tenant.** This is a lab, not a production rollout.
- **The compliance baseline is not complete.** It requires Secure Boot, firewall, TPM and the Defender checks, but BitLocker encryption and code integrity are not required, and there is no minimum OS version. A realistic baseline would add them, with BitLocker recovery keys stored in Entra.
- **Noncompliance only marks the device.** There is no email to the user, because that needs a notification message template, and no grace period.
- **Next:** tighten the compliance policy, then use the compliance status as the gate for a Conditional Access policy (my next lab, starting in report-only mode).

## What I learned

- The difference between a **configuration or endpoint security policy** (applies a setting) and a **compliance policy** (evaluates the device and reports a status other services can act on).
- The difference between a device that is **registered** (personal device known to Entra), **joined** (device signs in with a work identity) and **enrolled** (managed by Intune). My laptop is joined and enrolled.
- Why policy is assigned to **groups**, not individual devices.
- How to read the per-setting compliance report to find exactly why a device failed, and how to read Intune reporting, including its delays.
