Screenshots can be found: https://web.archive.org/web/20240917160441/https://nullg0re.com/2023/09/hijacking-someone-else-dcsync/


# Jams all day:


## Introduction
So you’re on a network doing a pentest. You’ve escalated privileges and it’s time to extract hashes from the DC. DCSync attacks are widely known and widely detected on.

What if there was a way to perform a DCSync without executing the attack or triggering an alert for the activity?

We’d first have to look into what devices and services perform DCSync as part of their standard operating procedures. Like Domain Controller replica sets. A Backup DC is of course going to perform a DCSync to the primary DC in order to back up all of its data.

What if there was another device or service out their that we could get access too without needing Domain Admin privileges to access? Enter AADConnect servers.

## AADConnect
AADConnect servers are an interesting concept. They should follow the Tier model and be treated as Tier 0 assets, but not all organizations treat them that way. Or in most cases, they do, but through the course of a pentest or compromise, the adversary is able to obtain Local Administrator access to the AADConnect servers themselves.

AADConnect is also an interesting piece of software. It contains a large amount of code to essentially perform the following actions:

- Perform a DCSync against a local DC to obtain user data and credentials
- AADConnect rehashes the users NTHashes 1,000 times
- AADConnect ships the hashes off to Azure AD for synchronization
- More information about the synchronization protocol can be found here.

What if there was a way to backdoor AADConnect so that AADConnect performs the DCSync and we just steal the credentials its supposed to obtain for it’s normal operations?

## DnSpy to the Rescue
To get started, I opened all of the EXE’s and DLLs inside the C:\Program Files\Microsoft Azure AD Sync directory with DnSpy and started hunting for .NET classes and methods that might be of any value. The goal here would be to find a method that I could overwrite that would allow me to steal the credentials from AADConnect after its DCSync, before it encrypts or after it decrypts the NT hashes during the sync.

Nestori Syynimaa helped me narrow down the list to Microsoft.Online.PasswordSynchronization.dll which contained the class PasswordUtility and the method DecryptNtOwfPwdWithIndex. This function is called during step one in the above list.

Here is a screenshot of the decompiled class:


This is a great method to overwrite because it had limited .NET functionality and the majority of the functions are native functions. This means we can keep the DLL relatively small and don’t have to add a bunch of unnecessary code to make the final product functional!

## Cooking Up Some DLLs
So now we have the method to overwrite, next we need to build the DLL.

The first thing that we’ll need to do is create a method inside of our DLL that will load the method we want to overwrite in memory, load the malicious method, and the overwrite the method in memory.

Below shows the Patch() method used to overwrite the existing code with our malicious code in memory:


The remainder of the DLL is our malicious method. This method contains all of the code from the legitimate method, so that we don’t crash the service, as well as the functionality to write the stolen DCSync hashes in the secretsdump.py format to disk at C:\Windows\Temp\Hashes.txt:


## Execution
The way this attack would work is that the adversary or tester would need to become a Local Admin of the AADConnect server and then inject the malicious DLL into the miiserver.exe process. This process is the service executable of the AADConnect service.

The adversary would need to identify the PID of the miiserver.exe process. Task Manager, Process Explorer, Process Hacker, and even the command line are great options:

CMD:

tasklist | findstr miiserver.exe
The adversary would then need to inject the malicious DLL:


With the malicious DLL injected into the miiserver.exe process, the next step would be for the adversary to log into Azure AD as any user and just reset a compromised hybrid user’s password. This will kick off the synchronization between on-prem and Azure AD and our malicious code will sit back and wait until miiserver.exe needs to decrypt the users credentials with the AADConnect Session Key, and our code will just write the credentials to disk at C:\Windows\Temp\Hashes.txt!


The adversary would then just need to exfil that file off the disk and they’ve successfully performed a DCSync without ever actually executing an out-of-band DCSync!


## Additional Context
While performing this research, researchers at SYGNIA found an alternate path to achieve the same objective. They have an incredibly detailed write-up that can be found here.

## Microsoft Communications
I submitted this research to MSRC on September 15th, 2023 expecting Microsoft to say that this was “by design” or to classify this as “Not a vulnerability” because there is no Security Boundaries being crossed with this abuse scenario. MSRC responded on September 21st, 2023 agreeing to the same:


## Conclusion
DCSync is an essential task when conducting a pentest, but it can be very noisy. We’ve discussed a way where we can perform a DCSync without triggering any traditional alarms.


