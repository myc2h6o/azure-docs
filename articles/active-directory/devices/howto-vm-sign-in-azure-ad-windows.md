---
title: Sign in to Windows virtual machine in Azure using Azure Active Directory
description: Azure AD sign in to an Azure VM running Windows

services: active-directory
ms.service: active-directory
ms.subservice: devices
ms.topic: how-to
ms.date: 08/19/2021

ms.author: joflore
author: MicrosoftGuyJFlo
manager: karenhoran
ms.reviewer: sandeo
ms.custom: references_regions, devx-track-azurecli, subject-rbac-steps
ms.collection: M365-identity-device-management
---
# Login to Windows virtual machine in Azure using Azure Active Directory authentication

Organizations can now improve the security of Windows virtual machines (VMs) in Azure by integrating with Azure Active Directory (AD) authentication. You can now use Azure AD as a core authentication platform to RDP into a **Windows Server 2019 Datacenter edition** or **Windows 10 1809** and later. Additionally, you will be able to centrally control and enforce Azure RBAC and Conditional Access policies that allow or deny access to the VMs. This article shows you how to create and configure a Windows VM and login with Azure AD based authentication.

There are many security benefits of using Azure AD based authentication to login to Windows VMs in Azure, including:
- Use your corporate AD credentials to login to Windows VMs in Azure.
- Reduce your reliance on local administrator accounts, you do not need to worry about credential loss/theft, users configuring weak credentials etc.
- Password complexity and password lifetime policies configured for your Azure AD directory help secure Windows VMs as well.
- With Azure role-based access control (Azure RBAC), specify who can login to a VM as a regular user or with administrator privileges. When users join or leave your team, you can update the Azure RBAC policy for the VM to grant access as appropriate. When employees leave your organization and their user account is disabled or removed from Azure AD, they no longer have access to your resources.
- With Conditional Access, configure policies to require multi-factor authentication and other signals such as low user and sign in risk before you can RDP to Windows VMs. 
- Use Azure deploy and audit policies to require Azure AD login for Windows VMs and to flag use of no approved local account on the VMs.
- Login to Windows VMs with Azure Active Directory also works for customers that use Federation Services.
- Automate and scale Azure AD join with MDM auto enrollment with Intune of Azure Windows VMs that are part for your VDI deployments. Auto MDM enrollment requires Azure AD P1 license. Windows Server 2019 VMs do not support MDM enrollment.

> [!NOTE]
> Once you enable this capability, your Windows VMs in Azure will be Azure AD joined. You cannot join it to other domain like on-premises AD or Azure AD DS. If you need to do so, you will need to disconnect the VM from your Azure AD tenant by uninstalling the extension.

## Requirements

### Supported Azure regions and Windows distributions

The following Windows distributions are currently supported for this feature:

- Windows Server 2019 Datacenter
- Windows 10 1809 and later

> [!IMPORTANT]
> Remote connection to VMs joined to Azure AD is only allowed from Windows 10 PCs that are either Azure AD registered (starting Windows 10 20H1), Azure AD joined or hybrid Azure AD joined to the **same** directory as the VM. 

This feature is now available in the following Azure clouds:

- Azure Global
- Azure Government
- Azure China

### Network requirements

To enable Azure AD authentication for your Windows VMs in Azure, you need to ensure your VMs network configuration permits outbound access to the following endpoints over TCP port 443:

For Azure Global
- `https://enterpriseregistration.windows.net` - For device registration.
- `http://169.254.169.254` - Azure Instance Metadata Service endpoint.
- `https://login.microsoftonline.com` - For authentication flows.
- `https://pas.windows.net` - For Azure RBAC flows.

For Azure Government
- `https://enterpriseregistration.microsoftonline.us` - For device registration.
- `http://169.254.169.254` - Azure Instance Metadata Service.
- `https://login.microsoftonline.us` - For authentication flows.
- `https://pasff.usgovcloudapi.net` - For Azure RBAC flows.

For Azure China
- `https://enterpriseregistration.partner.microsoftonline.cn` - For device registration.
- `http://169.254.169.254` - Azure Instance Metadata Service endpoint.
- `https://login.chinacloudapi.cn` - For authentication flows.
- `https://pas.chinacloudapi.cn` - For Azure RBAC flows.

## Enabling Azure AD login in for Windows VM in Azure

To use Azure AD login in for Windows VM in Azure, you need to first enable Azure AD login option for your Windows VM and then you need to configure Azure role assignments for users who are authorized to login in to the VM.
There are multiple ways you can enable Azure AD login for your Windows VM:

- Using the Azure portal experience when creating a Windows VM
- Using the Azure Cloud Shell experience when creating a Windows VM **or for an existing Windows VM**

### Using Azure portal create VM experience to enable Azure AD login

You can enable Azure AD login for Windows Server 2019 Datacenter or Windows 10 1809 and later VM images. 

To create a Windows Server 2019 Datacenter VM in Azure with Azure AD logon: 

1. Sign in to the [Azure portal](https://portal.azure.com), with an account that has access to create VMs, and select **+ Create a resource**.
1. Type **Windows Server** in Search the Marketplace search bar.
   1. Click **Windows Server** and choose **Windows Server 2019 Datacenter** from Select a software plan dropdown.
   1. Click on **Create**.
1. On the "Management" tab, enable the option to **Login with AAD credentials** under the Azure Active Directory section from Off to **On**.
1. Make sure **System assigned managed identity** under the Identity section is set to **On**. This action should happen automatically once you enable Login with Azure AD credentials.
1. Go through the rest of the experience of creating a virtual machine. You will have to create an administrator username and password for the VM.

![Login with Azure AD credentials create a VM](./media/howto-vm-sign-in-azure-ad-windows/azure-portal-login-with-azure-ad.png)

> [!NOTE]
> In order to log in to the VM using your Azure AD credential, you will first need to configure role assignments for the VM as described in one of the sections below.

### Using the Azure Cloud Shell experience to enable Azure AD login

Azure Cloud Shell is a free, interactive shell that you can use to run the steps in this article. Common Azure tools are preinstalled and configured in Cloud Shell for you to use with your account. Just select the Copy button to copy the code, paste it in Cloud Shell, and then press Enter to run it. There are a few ways to open Cloud Shell:

- Select **Try It** in the upper-right corner of a code block.
- Open Cloud Shell in your browser.
- Select the Cloud Shell button on the menu in the upper-right corner of the [Azure portal](https://portal.azure.com).

If you choose to install and use the CLI locally, this article requires that you are running the Azure CLI version 2.0.31 or later. Run az --version to find the version. If you need to install or upgrade, see the article [Install Azure CLI](/cli/azure/install-azure-cli).

1. Create a resource group with [az group create](/cli/azure/group#az_group_create). 
1. Create a VM with [az vm create](/cli/azure/vm#az_vm_create) using a supported distribution in a supported region. 
1. Install the Azure AD login VM extension. 

The following example deploys a VM named myVM that uses Win2019Datacenter, into a resource group named myResourceGroup, in the southcentralus region. In the following examples, you can provide your own resource group and VM names as needed.

```AzureCLI
az group create --name myResourceGroup --location southcentralus

az vm create \
    --resource-group myResourceGroup \
    --name myVM \
    --image Win2019Datacenter \
    --assign-identity \
    --admin-username azureuser \
    --admin-password yourpassword
```

> [!NOTE]
> It is required that you enable System assigned managed identity on your virtual machine before you install the Azure AD login VM extension.

It takes a few minutes to create the VM and supporting resources.

Finally, install the Azure AD login VM extension to enable Azure AD login for Windows VM. VM extensions are small applications that provide post-deployment configuration and automation tasks on Azure virtual machines. Use [az vm extension](/cli/azure/vm/extension#az_vm_extension_set) set to install the AADLoginForWindows extension on the VM named `myVM` in the `myResourceGroup` resource group:

> [!NOTE]
> You can install AADLoginForWindows extension on an existing Windows Server 2019 or Windows 10 1809 and later VM to enable it for Azure AD authentication. An example of AZ CLI is shown below.

```AzureCLI
az vm extension set \
    --publisher Microsoft.Azure.ActiveDirectory \
    --name AADLoginForWindows \
    --resource-group myResourceGroup \
    --vm-name myVM
```

The `provisioningState` of `Succeeded` is shown, once the extension is installed on the VM.

## Configure role assignments for the VM

Now that you have created the VM, you need to configure Azure RBAC policy to determine who can log in to the VM. Two Azure roles are used to authorize VM login:

- **Virtual Machine Administrator Login**: Users with this role assigned can log in to an Azure virtual machine with administrator privileges.
- **Virtual Machine User Login**: Users with this role assigned can log in to an Azure virtual machine with regular user privileges.

> [!NOTE]
> To allow a user to log in to the VM over RDP, you must assign either the Virtual Machine Administrator Login or Virtual Machine User Login role. An Azure user with the Owner or Contributor roles assigned for a VM do not automatically have privileges to log in to the VM over RDP. This is to provide audited separation between the set of people who control virtual machines versus the set of people who can access virtual machines.

There are multiple ways you can configure role assignments for VM:

- Using the Azure AD Portal experience
- Using the Azure Cloud Shell experience

> [!NOTE]
> The Virtual Machine Administrator Login and Virtual Machine User Login roles use dataActions and thus cannot be assigned at management group scope. Currently these roles can only be assigned at the subscription, resource group or resource scope.

### Using Azure AD Portal experience

To configure role assignments for your Azure AD enabled Windows Server 2019 Datacenter VMs:

1. Select **Access control (IAM)**.

1. Select **Add** > **Add role assignment** to open the Add role assignment page.

1. Assign the following role. For detailed steps, see [Assign Azure roles using the Azure portal](../../role-based-access-control/role-assignments-portal.md).
    
    | Setting | Value |
    | --- | --- |
    | Role | **Virtual Machine Administrator Login** or **Virtual Machine User Login** |
    | Assign access to | User, group, service principal, or managed identity |

    ![Add role assignment page in Azure portal.](../../../includes/role-based-access-control/media/add-role-assignment-page.png)

### Using the Azure Cloud Shell experience

The following example uses [az role assignment create](/cli/azure/role/assignment#az_role_assignment_create) to assign the Virtual Machine Administrator Login role to the VM for your current Azure user. The username of your active Azure account is obtained with [az account show](/cli/azure/account#az_account_show), and the scope is set to the VM created in a previous step with [az vm show](/cli/azure/vm#az_vm_show). The scope could also be assigned at a resource group or subscription level, and normal Azure RBAC inheritance permissions apply. For more information, see [Log in to a Linux virtual machine in Azure using Azure Active Directory authentication](../../virtual-machines/linux/login-using-aad.md).

```   AzureCLI
$username=$(az account show --query user.name --output tsv)
$vm=$(az vm show --resource-group myResourceGroup --name myVM --query id -o tsv)

az role assignment create \
    --role "Virtual Machine Administrator Login" \
    --assignee $username \
    --scope $vm
```

> [!NOTE]
> If your AAD domain and logon username domain do not match, you must specify the object ID of your user account with the `--assignee-object-id`, not just the username for `--assignee`. You can obtain the object ID for your user account with [az ad user list](/cli/azure/ad/user#az_ad_user_list).

For more information on how to use Azure RBAC to manage access to your Azure subscription resources, see the following articles:

- [Assign Azure roles using Azure CLI](../../role-based-access-control/role-assignments-cli.md)
- [Assign Azure roles using the Azure portal](../../role-based-access-control/role-assignments-portal.md)
- [Assign Azure roles using Azure PowerShell](../../role-based-access-control/role-assignments-powershell.md).

## Using Conditional Access

You can enforce Conditional Access policies such as multi-factor authentication or user sign-in risk check before authorizing access to Windows VMs in Azure that are enabled with Azure AD sign in. To apply Conditional Access policy, you must select the "**Azure Windows VM Sign-In**" app from the cloud apps or actions assignment option and then use Sign-in risk as a condition and/or 
require multi-factor authentication as a grant access control. 

> [!NOTE]
> If you use "Require multi-factor authentication" as a grant access control for requesting access to the "Azure Windows VM Sign-In" app, then you must supply multi-factor authentication claim as part of the client that initiates the RDP session to the target Windows VM in Azure. The only way to achieve this on a Windows 10 client is to use Windows Hello for Business PIN or biometric authentication with the RDP client. Support for biometric authentication was added to the RDP client in Windows 10 version 1809. Remote desktop using Windows Hello for Business authentication is only available for deployments that use cert trust model and currently not available for key trust model.

> [!WARNING]
> Per-user Enabled/Enforced Azure AD Multi-Factor Authentication is not supported for VM Sign-In.

## Log in using Azure AD credentials to a Windows VM

> [!IMPORTANT]
> Remote connection to VMs joined to Azure AD is only allowed from Windows 10 PCs that are either Azure AD registered (minimum required build is 20H1) or Azure AD joined or hybrid Azure AD joined to the **same** directory as the VM. Additionally, to RDP using Azure AD credentials, the user must belong to one of the two Azure roles, Virtual Machine Administrator Login or Virtual Machine User Login. If using an Azure AD registered Windows 10 PC, you must enter credentials in the `AzureAD\UPN` format (for example, `AzureAD\john@contoso.com`). At this time, Azure Bastion can't be used to log in by using Azure Active Directory authentication with the AADLoginForWindows extension; only direct RDP is supported.

To log in to your Windows Server 2019 virtual machine using Azure AD: 

1. Navigate to the overview page of the virtual machine that has been enabled with Azure AD logon.
1. Select **Connect** to open the Connect to virtual machine blade.
1. Select **Download RDP File**.
1. Select **Open** to launch the Remote Desktop Connection client.
1. Select **Connect** to launch the Windows logon dialog.
1. Logon using your Azure AD credentials.

You are now signed in to the Windows Server 2019 Azure virtual machine with the role permissions as assigned, such as VM User or VM Administrator. 

> [!NOTE]
> You can save the .RDP file locally on your computer to launch future remote desktop connections to your virtual machine instead of having to navigate to virtual machine overview page in the Azure portal and using the connect option.

## Using Azure Policy to ensure standards and assess compliance

Use Azure Policy to ensure Azure AD login is enabled for your new and existing Windows virtual machines and assess compliance of your environment at scale on your Azure Policy compliance dashboard. With this capability, you can use many levels of enforcement: you can flag new and existing Windows VMs within your environment that do not have Azure AD login enabled. You can also use Azure Policy to deploy the Azure AD extension on new Windows VMs that do not have Azure AD login enabled, as well as remediate existing Windows VMs to the same standard. In addition to these capabilities, you can also use Azure Policy to detect and flag Windows VMs that have non-approved local accounts created on their machines. To learn more, review [Azure Policy](../../governance/policy/overview.md).

## Troubleshoot

### Troubleshoot deployment issues

The AADLoginForWindows extension must install successfully in order for the VM to complete the Azure AD join process. Perform the following steps if the VM extension fails to install correctly.

1. RDP to the VM using the local administrator account and examine the `CommandExecution.log` file under:
   
   `C:\WindowsAzure\Logs\Plugins\Microsoft.Azure.ActiveDirectory.AADLoginForWindows\0.3.1.0.`

   > [!NOTE]
   > If the extension restarts after the initial failure, the log with the deployment error will be saved as `CommandExecution_YYYYMMDDHHMMSSSSS.log`. 
"
1. Open a PowerShell window on the VM and verify these queries against the Instance Metadata Service (IMDS) Endpoint running on the Azure host returns:

   | Command to run | Expected output |
   | --- | --- |
   | `curl -H @{"Metadata"="true"} "http://169.254.169.254/metadata/instance?api-version=2017-08-01"` | Correct information about the Azure VM |
   | `curl -H @{"Metadata"="true"} "http://169.254.169.254/metadata/identity/info?api-version=2018-02-01"` | Valid Tenant ID associated with the Azure Subscription |
   | `curl -H @{"Metadata"="true"} "http://169.254.169.254/metadata/identity/oauth2/token?resource=urn:ms-drs:enterpriseregistration.windows.net&api-version=2018-02-01"` | Valid access token issued by Azure Active Directory for the managed identity that is assigned to this VM |

   > [!NOTE]
   > The access token can be decoded using a tool like [calebb.net](http://calebb.net/). Verify the `appid` in the access token matches the managed identity assigned to the VM.

1. Ensure the required endpoints are accessible from the VM using PowerShell:
   
   - `curl https://login.microsoftonline.com/ -D -`
   - `curl https://login.microsoftonline.com/<TenantID>/ -D -`
   - `curl https://enterpriseregistration.windows.net/ -D -`
   - `curl https://device.login.microsoftonline.com/ -D -`
   - `curl https://pas.windows.net/ -D -`

   > [!NOTE]
   > Replace `<TenantID>` with the Azure AD Tenant ID that is associated with the Azure subscription.<br/> `enterpriseregistration.windows.net` and `pas.windows.net` should return 404 Not Found, which is expected behavior.
            
1. The Device State can be viewed by running `dsregcmd /status`. The goal is for Device State to show as `AzureAdJoined : YES`.

   > [!NOTE]
   > Azure AD join activity is captured in Event viewer under the `User Device Registration\Admin` log.

If AADLoginForWindows extension fails with certain error code, you can perform the following steps:

#### Issue 1: AADLoginForWindows extension fails to install with terminal error code '1007' and exit code: -2145648574.

This exit code translates to `DSREG_E_MSI_TENANTID_UNAVAILABLE` because the extension is unable to query the Azure AD Tenant information.

1. Verify the Azure VM can retrieve the TenantID from the Instance Metadata Service.

   - RDP to the VM as a local administrator and verify the endpoint returns valid Tenant ID by running this command from an elevated PowerShell window on the VM:
      
      - `curl -H Metadata:true http://169.254.169.254/metadata/identity/info?api-version=2018-02-01`

1. The VM admin attempts to install the AADLoginForWindows extension, but a system assigned managed identity has not enabled the VM first. Navigate to the Identity blade of the VM. From the System assigned tab, verify Status is toggled to On.

#### Issue 2: AADLoginForWindows extension fails to install with Exit code: -2145648607

This Exit code translates to `DSREG_AUTOJOIN_DISC_FAILED` because the extension is not able to reach the `https://enterpriseregistration.windows.net` endpoint.

1. Verify the required endpoints are accessible from the VM using PowerShell:

   - `curl https://login.microsoftonline.com/ -D -`
   - `curl https://login.microsoftonline.com/<TenantID>/ -D -`
   - `curl https://enterpriseregistration.windows.net/ -D -`
   - `curl https://device.login.microsoftonline.com/ -D -`
   - `curl https://pas.windows.net/ -D -`
   
   > [!NOTE]
   > Replace `<TenantID>` with the Azure AD Tenant ID that is associated with the Azure subscription. If you need to find the tenant ID, you can hover over your account name to get the directory / tenant ID, or select **Azure Active Directory > Properties > Directory ID** in the Azure portal.<br/>`enterpriseregistration.windows.net` and `pas.windows.net` should return 404 Not Found, which is expected behavior.

1. If any of the commands fails with "Could not resolve host `<URL>`", try running this command to determine the DNS server that is being used by the VM.
   
   `nslookup <URL>`

   > [!NOTE] 
   > Replace `<URL>` with the fully qualified domain names used by the endpoints, such as `login.microsoftonline.com`.

1. Next, see if specifying a public DNS server allows the command to succeed:

   `nslookup <URL> 208.67.222.222`

1. If necessary, change the DNS server that is assigned to the network security group that the Azure VM belongs to.

#### Issue 3: AADLoginForWindows extension fails to install with Exit code: 51

Exit code 51 translates to "This extension is not supported on the VM's operating system".

The AADLoginForWindows extension is only intended to be installed on Windows Server 2019 or Windows 10 (Build 1809 or later). Ensure the version of Windows is supported. If the build of Windows is not supported, uninstall the VM Extension.

### Troubleshoot sign-in issues

Some common errors when you try to RDP with Azure AD credentials include no Azure roles assigned, unauthorized client, or 2FA sign-in method required. Use the following information to correct these issues.

The Device and SSO State can be viewed by running `dsregcmd /status`. The goal is for Device State to show as `AzureAdJoined : YES` and `SSO State` to show `AzureAdPrt : YES`.

Also, RDP Sign-in using Azure AD accounts is captured in Event viewer under the `AAD\Operational` event logs.

#### Azure role not assigned

If you see the following error message when you initiate a remote desktop connection to your VM: 

- Your account is configured to prevent you from using this device. For more info, contact your system administrator.

![Your account is configured to prevent you from using this device.](./media/howto-vm-sign-in-azure-ad-windows/rbac-role-not-assigned.png)

Verify that you have [configured Azure RBAC policies](../../virtual-machines/linux/login-using-aad.md) for the VM that grants the user either the Virtual Machine Administrator Login or Virtual Machine User Login role:

> [!NOTE]
> If you are running into issues with Azure role assignments, see [Troubleshoot Azure RBAC](../../role-based-access-control/troubleshooting.md#azure-role-assignments-limit).
 
#### Unauthorized client

If you see the following error message when you initiate a remote desktop connection to your VM: 

- Your credentials did not work.

![Your credentials did not work](./media/howto-vm-sign-in-azure-ad-windows/your-credentials-did-not-work.png)

Verify that the Windows 10 PC you are using to initiate the remote desktop connection is one that is either Azure AD joined, or hybrid Azure AD joined to the same Azure AD directory where your VM is joined to. For more information about device identity, see the article [What is a device identity](./overview.md).

> [!NOTE]
> Windows 10 Build 20H1 added support for an Azure AD registered PC to initiate RDP connection to your VM. When using an Azure AD registered (not Azure AD joined or hybrid Azure AD joined) PC as the RDP client to initiate connections to your VM, you must enter credentials in the format `AzureAD\UPN` (for example, `AzureAD\john@contoso.com`).

Verify that the AADLoginForWindows extension was not uninstalled after the Azure AD join finished.

Also, make sure that the security policy "Network security: Allow PKU2U authentication requests to this computer to use online identities" is enabled on both the server **and** the client.
 
#### MFA sign-in method required

If you see the following error message when you initiate a remote desktop connection to your VM: 

- The sign-in method you're trying to use isn't allowed. Try a different sign-in method or contact your system administrator.

![The sign-in method you're trying to use isn't allowed.](./media/howto-vm-sign-in-azure-ad-windows/mfa-sign-in-method-required.png)

If you have configured a Conditional Access policy that requires multi-factor authentication (MFA) before you can access the resource, then you need to ensure that the Windows 10 PC initiating the remote desktop connection to your VM signs in using a strong authentication method such as Windows Hello. If you do not use a strong authentication method for your remote desktop connection, you will see the previous error.

- Your credentials did not work.

![Your credentials did not work](./media/howto-vm-sign-in-azure-ad-windows/your-credentials-did-not-work.png)

> [!WARNING]
> Per-user Enabled/Enforced Azure AD Multi-Factor Authentication is not supported for VM Sign-In. This setting causes Sign-in to fail with “Your credentials do not work.” error message.

You can resolve the above issue by removing the per user MFA setting, by following these steps:

```

# Get StrongAuthenticationRequirements configure on a user
(Get-MsolUser -UserPrincipalName username@contoso.com).StrongAuthenticationRequirements
 
# Clear StrongAuthenticationRequirements from a user
$mfa = @()
Set-MsolUser -UserPrincipalName username@contoso.com -StrongAuthenticationRequirements $mfa
 
# Verify StrongAuthenticationRequirements are cleared from the user
(Get-MsolUser -UserPrincipalName username@contoso.com).StrongAuthenticationRequirements

```

If you have not deployed Windows Hello for Business and if that is not an option for now, you can exclude MFA requirement by configuring Conditional Access policy that excludes "**Azure Windows VM Sign-In**" app from the list of cloud apps that require MFA. To learn more about Windows Hello for Business, see [Windows Hello for Business Overview](/windows/security/identity-protection/hello-for-business/hello-identity-verification).

> [!NOTE]
> Windows Hello for Business PIN authentication with RDP has been supported by Windows 10 for several versions, however support for Biometric authentication with RDP was added in Windows 10 version 1809. Using Windows Hello for Business authentication during RDP is only available for deployments that use cert trust model and currently not available for key trust model.
 
Share your feedback about this feature or report issues using it on the [Azure AD feedback forum](https://feedback.azure.com/forums/169401-azure-active-directory?category_id=166032).

## Next steps

For more information on Azure Active Directory, see [What is Azure Active Directory](../fundamentals/active-directory-whatis.md).
