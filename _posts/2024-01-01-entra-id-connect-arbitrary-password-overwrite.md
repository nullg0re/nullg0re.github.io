Screenshots can be found here: https://web.archive.org/web/20241013082556/https://nullg0re.com/2024/01/entra-id-connect-arbitrary-password-overwrite/


As always, jams….


DCSync Hijacking
I recently published a blog post detailing an attack where we could hijack AADConnect’s DCSync dataset to steal credentials and crack the credentials offline. The foundational knowledge here is as follows:

AADConnect performs DCSync operations to collect on-premises user’s credentials
Each NT hash is encrypted 1,000 times
The resulting encrypted credential dataset is sent off to Entra ID for processing
We learned that by injecting a malicious DLL into the miiserver.exe process, we could actually siphon the DCSync dataset to a file on disk for later exfiltration and cracking purposes, effectively allowing us to perform DCSync operations without triggering any traditional alerts.

What if I told you that there’s more that we could do here?

AADConnect Arbitrary Password Overwrite
Leveraging the same Microsoft.Online.PasswordSynchronization.dll DLL, same PasswordUtility class, and the same DecryptNtOwfPwdWithIndex method, we can actually overwrite any users NT hash with an attacker-controlled value, giving ourselves access to the organizations Azure subscription as the compromised user, with a pretty easy and effective ripcord for resetting the users password to the original value.

We can even re-utilize the Proof of Concept exploit from the previous post, but instead of writing the credentials to disk, we target a specific user and overwrite the user’s NT hash:

The below screenshot depicts the code necessary to overwrite victimuser2‘s NT Hash. AADConnects will initiate the DCSync and DecryptNtOwfPwdWithIndex will iterate through the results, encrypting each NT Hash 1,000 times before packaging it and sending it off to Entra ID. Once DecryptNtOwfPwdWithIndex hits the victimuser2 credential information, it will convert the MD4 hash of TacoTacoTaco123!! into the correct .NET format (e9-f8-c7-40-etc-etc) and then assign the result variable with the overwritten NT hash. AADConnect will then encrypt the overwritten hash 1,000 times and ship it off to Entra ID which will consider the overwritten hash the hash of record for the user’s Azure account.


This will result in the on-premises account’s NT hash being out of sync with the Entra ID NT hash, but as long as the miiserver.exe process isn’t restarted, the attacker will be able to log into the user’s Azure account indefinitely.

Here is my proposed Attack Path:

Attack Path
Attacker gains access to the AADConnect Server as Local Administrator.
Attacker builds malicious DLL that targets a Global Administrator.
Attacker injects malicious DLL into the miiserver.exe process.
AADConnect initiates DCSync to collect on-premises credentials.
AADConnect begins the process of encrypting each NT hash 1,000 times. When the AADConnect function reaches the targeted Global Administrator account, the DLL code overwrites the Global Admins NT Hash with a value the attacker controls.
AADConnect proceeds to encrypt the overwritten hash and continues operations.
AADConnect ships the encrypted hashes to Entra ID.
Entra ID receives the NT hashes and processes them.
Attacker pivots from on-premises to Entra ID, taking the Global Administrator account with it.
Attacker collects Access Tokens, Refresh Tokens, and Primary Refresh Tokens, sets up backdoor accounts, etc… all before restarting the miiserver.exe service without triggering any alerts.
It is important to call out that if an adversary has performed this attack, you can reset the compromised users credential to it’s original value by restarting the miiserver.exe process and then performing a full sync between on-premises and Entra ID. This will reset the password in Azure’s records to the value stored on-prem.

Microsoft’s Response
As with the previous blog post, this attack requires Local Admin privileges on an AADConnect server, which are considered Tier 0 devices. Microsoft’s documentation clearly states that these types of devices must have the highest security as they can be severely abused or exploited, as shown above and in the previous blog post.

Conclusion
AADConnect / Entra ID Connect continues to be a target rich environment and is ripe for additional exploitation and research.


