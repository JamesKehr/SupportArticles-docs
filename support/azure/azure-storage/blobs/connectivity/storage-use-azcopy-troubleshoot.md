---
title: Troubleshoot issues in AzCopy (Azure Storage)
description: Understand how to apply workarounds for common Azure Storage issues that occur in AzCopy version 10.
ms.service: azure-storage
ms.date: 05/08/2024
editor: v-jsitser
ms.reviewer: abail, broder, schoag, dafalkne, shaas, raharina, normesta, azuresamfilesmigcic, azurestocic, v-weizhu, v-leedennis
ms.custom: sap:Connectivity
---
# Troubleshoot issues in AzCopy v10

This article discusses common issues that you might encounter when you use AzCopy. This article also helps you identify the causes of the issues, and suggests how to resolve them.

## Identifying issues

You can determine whether a job succeeds by looking at the exit code.

If the exit code is `0-success`, the job finished successfully.

If the exit code is `1-error`, examine the log file. After you understand the exact error message, you can more easily search for the right keywords and determine the solution. To learn more, see [Find errors and resume jobs by using log and plan files in AzCopy](/azure/storage/common/storage-use-azcopy-configure).

If the exit code is `2-panic`, check whether the log file exists. If the file doesn't exist, file a bug or reach out to support.

Any other nonzero exit code (such as `OOMKilled`) might be generated by the system. Check your operating system documentation for special exit codes.

## "403" errors

"403" errors are common. Sometimes they're benign and don't cause a failed transfer. For example, in AzCopy logs, you might see that a `HEAD` request received "403" errors. Those errors appear when AzCopy checks whether a resource is public. In most cases, you can ignore those instances.

In some cases, "403" errors can cause a failed transfer. If this issue occurs, other attempts to transfer files are likely to fail until you resolve the issue. "403" errors can be caused by authentication and authorization issues. They can also occur if requests are blocked by the storage account firewall configuration.

### Authentication and authorization issues

"403" errors that prevent data transfer occur because of issues that involve SAS tokens, role-based access control (Azure RBAC) roles, and access control list (ACL) configurations.

#### SAS tokens

If you use a shared access signature (SAS) token, make sure that the following statements are true:

- The expiration and start times of the SAS token are appropriate.

- You selected all the necessary permissions for the token.

- You generated the token by using an official SDK or tool. Try Storage Explorer if you haven't already.

#### Azure RBAC

If you use Azure RBAC roles through the `azcopy login` command, verify that you have the appropriate Azure roles assigned to your identity (for example, the Storage Blob Data Contributor role).

To learn more about Azure roles, see [Assign an Azure role for access to blob data](/azure/storage/blobs/assign-azure-role-data-access).

#### ACLs

If you use access control lists (ACLs), verify that your identity appears in an ACL entry for each file or directory that you intend to access. Also, make sure that each ACL entry reflects the appropriate permission level.

To learn more about ACLs and ACL entries, see [Access control lists (ACLs) in Azure Data Lake Storage Gen2](/azure/storage/blobs/data-lake-storage-access-control).

To learn more about how to incorporate Azure roles together with ACLs and how the system evaluates them to make authorization decisions, see [Access control model in Azure Data Lake Storage Gen2](/azure/storage/blobs/data-lake-storage-access-control-model).

### Firewall and private endpoint issues

If the storage firewall configuration doesn't allow access from the hosting component on which AzCopy is running, AzCopy operations return an HTTP "403" error code.

> [!NOTE]
> In this article, the term *hosting component* refers to a physical computer, virtual machine (VM), or container.

### Permitted scope for copy operations

The `AllowedCopyScope` property of a storage account is used to specify the environments from which you can copy data to the destination account. This property is displayed in the Azure portal as the **Permitted scope for copy operations (preview)** configuration setting. By default, the property isn't given a value. The property doesn't return a value until you explicitly set it. The `AllowedCopyScope` property has three possible values, as shown in the following table.

| Value | Description |
|--|--|
| (`null`) | (Default value) Allows copying from any storage account to the destination account. |
| `Microsoft Entra ID` | Permits copying from only accounts that are within the same Microsoft Entra tenant as the destination account. |
| `PrivateLink` | Permits copying from only storage accounts that have private links to the same virtual network as the destination account. |

For more information about this property and its associated configuration setting, see [Restrict the source of copy operations to a storage account](/azure/storage/common/security-restrict-copy-operations).

#### Transfer data from or to a local hosting component

If you're uploading or downloading data between a storage account and an on-premises hosting component, make sure that the hosting component that runs AzCopy can access either the source or destination storage account. You might have to use IP network rules in the firewall settings of either the source or destination accounts to allow access from the public IP address of the hosting component.

#### Transfer data between storage accounts

"403" authorization errors can prevent you from transferring data between accounts by using the client hosting component on which AzCopy is running.

If you're copying data between storage accounts, make sure that the hosting component that runs AzCopy can access both the source and the destination account. You might have to use IP network rules in the firewall settings of both the source and destination accounts to allow access from the public IP address of the hosting component. The service uses the IP address of the AzCopy client hosting component to authorize the source to destination traffic. To learn how to add a public IP address to the firewall settings of a storage account, see [Grant access from an internet IP range](/azure/storage/common/storage-network-security#grant-access-from-an-internet-ip-range).

In case your VM doesn't or can't have a public IP address, consider using a private endpoint. See [Use private endpoints for Azure Storage](/azure/storage/common/storage-private-endpoints).

#### Use Private Link

[Private Link](/azure/private-link/private-link-overview) is at the virtual network/subnet level. If you want AzCopy requests to go through Private Link, then AzCopy must make those requests from a VM that's running in that virtual network/subnet. For example, suppose that you configure Private Link in VNet1/Subnet1, but the VM on which AzCopy runs is in VNet1/Subnet2. In this scenario, AzCopy requests don't use Private Link, and the requests are expected to fail.

## Proxy-related errors

If you encounter TCP errors such as "dial tcp: lookup proxy.x.x: no such host," this means that your environment isn't configured to use the correct proxy or you use an advanced proxy that AzCopy doesn't recognize.

You have to update the proxy settings to reflect the correct configurations. See [Configure proxy settings](/azure/storage/common/storage-ref-azcopy-configuration-settings?toc=/azure/storage/blobs/toc.json#configure-proxy-settings).

You can also bypass the proxy by setting the environment variable `NO_PROXY="*"`.

Here are the endpoints that AzCopy requires:

| Sign-in endpoints | Azure Storage endpoints |
|---|---|
| `login.microsoftonline.com` (global Azure) | `(blob | file | dfs).core.windows.net` (global Azure) |
| `login.chinacloudapi.cn` (Azure China) | `(blob | file | dfs).core.chinacloudapi.cn` (Azure China) |
| `login.microsoftonline.de` (Azure Germany) | `(blob | file | dfs).core.cloudapi.de` (Azure Germany) |
| `login.microsoftonline.us` (Azure US Government) | `(blob | file | dfs).core.usgovcloudapi.net` (Azure US Government) |

## x509: certificate signed by unknown authority

This error is often related to the use of a proxy that's using a Secure Sockets Layer (SSL) certificate that isn't trusted by the operating system. Verify your settings and make sure that the certificate is trusted at the operating system level.

We recommend that you add the certificate to your hosting component's root certificate store because that's where the trusted authorities are kept.

## Unrecognized parameters

If you receive an error message that states that your parameters aren't recognized, make sure that you're using the correct version of AzCopy. AzCopy v8 and earlier versions are deprecated. [AzCopy v10](/azure/storage/common/storage-use-azcopy-v10?toc=/azure/storage/blobs/toc.json) is the current version, and it's a complete rewrite that doesn't share any syntax with the previous versions. Refer to [AzCopy Migration Guide for v8 to v10](https://github.com/Azure/azure-storage-azcopy/blob/main/MigrationGuideV8toV10.md).

Also, make sure to use built-in help messages by using the `-h` switch together with any command (for example, `azcopy copy -h`). See [Get command help](/azure/storage/common/storage-use-azcopy-v10?toc=/azure/storage/blobs/toc.json#get-command-help). To view the same information online, see [azcopy copy](/azure/storage/common/storage-ref-azcopy-copy?toc=/azure/storage/blobs/toc.json).

To help you understand commands, we provide an education tool that's located in the [AzCopy command guide](https://azcopyvnextrelease.z22.web.core.windows.net/). This tool demonstrates the most popular AzCopy commands along with the most popular command flags. To find example commands, see [Transfer data](/azure/storage/common/storage-use-azcopy-v10?toc=/azure/storage/blobs/toc.json#transfer-data). If you have a question, try searching through existing [GitHub issues](https://github.com/Azure/azure-storage-azcopy/issues) first to see whether it was already answered.

## Conditional access policy error

You might receive the following error when you invoke the `azcopy login` command:

> Failed to perform login command:
failed to login with tenantID "common", Azure directory endpoint "https://login.microsoftonline.com", autorest/adal/devicetoken: -REDACTED- AADSTS50005: User tried to log in to a device from a platform (Unknown) that's currently not supported through Conditional Access policy. Supported device platforms are: iOS, Android, Mac, and Windows flavors.
Trace ID: -REDACTED-
Correlation ID: -REDACTED-
Timestamp: 2021-01-05 01:58:28Z

This error means that your administrator configured a conditional access policy that specifies the kind of device that you can sign in from. AzCopy uses the device code flow. The device code flow can't guarantee that the hosting component on which you're using the AzCopy tool is also where you're signing in from.

If your device is among the list of supported platforms, then you might be able to use Storage Explorer. Storage Explorer integrates AzCopy for all data transfers (it passes tokens to AzCopy through the secret store) but provides a sign-in workflow that supports passing device information. AzCopy itself also supports managed identities and service principals as a sign-in alternative.

If your device isn't on the list of supported platforms, contact your administrator for help.

## Server busy, network errors, or time-outs

If you see a large number of failed requests that have the "503 Server Busy" status, then the storage service is throttling your requests. If you see network errors or time-outs, you might be trying to push too much data for your infrastructure to handle. In all cases, the workaround is similar.

If you see a large file repeatedly fail to copy because certain chunks fail each time, try to limit the concurrent network connections or throughput limit depending on your specific case. We suggest that you lower the performance drastically at first, observe whether this action solved the initial problem, and then ramp up the performance again until you achieve an overall balance.

For more information, see [Optimize the performance of AzCopy with Azure Storage](/azure/storage/common/storage-use-azcopy-optimize).

If you're copying data between accounts by using AzCopy, the quality and reliability of the network from where you run AzCopy might affect the overall performance. Even though data transfers from server to server, AzCopy does initiate calls for each file to copy between service endpoints.

## Known constraints in AzCopy

- Copying data from government clouds to commercial clouds isn't supported. However, copying data from commercial clouds to government clouds is supported.

- Asynchronous service-side copy isn't supported. AzCopy performs synchronous copy only. In other words, by the time the job finishes, the data has been moved.

- When you copy to an Azure File share, if you forgot to specify the `--preserve-smb-permissions` flag and you don't want to transfer the data again, consider using Robocopy to bring over the permissions.

- Azure Functions has a different endpoint for MSI authentication. AzCopy doesn't yet support MSI authentication.

## See also

- [Get started with AzCopy](/azure/storage/common/storage-use-azcopy-v10)
- [Find errors and resume jobs by using log and plan files in AzCopy](/azure/storage/common/storage-use-azcopy-configure)

[!INCLUDE [Azure Help Support](../../../../includes/azure-help-support.md)]
