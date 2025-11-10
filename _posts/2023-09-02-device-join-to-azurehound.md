

Screenshots can be found at: https://web.archive.org/web/20240803200454/https://nullg0re.com/2023/09/device-join-to-azurehound/

But first, tunes:


## Introduction
Microsoft Azure is an incredibly powerful tool that enables organizations to manage resources, identities, applications, installation of software to on-premises systems, and even manage devices. One of the ways an organization can manage devices is through the use of Device Join.

One of the most well known attacks against Azure Joined devices is the Pass-The-PRT Attack and has been published extensively by legends like Dirk-Jan Mollema, Lee Christensen, Dr. Nestori Syynimaa, and Benjamin Delpy.

This blog is going to cover the Pass-The-PRT Attack using Dirk-Jan’s ROADToken and ROADtools projects, as well as how to leverage the stolen PRT to obtain a FOCI (Family of Client ID) Refresh Token, use that token with AzureHound, and lastly, leverage a tool called TokenMan written by my good friend 0xZDH at Secureworks that will allow us to steal credentials from Teams Messages and send Phishing pretexts through a compromised users Teams account.

## Let’s get into it:

### Joining a Device To Azure AD
First thing we need to talk about is creating our Device Joined system.

Taking a Windows 10 / Windows Server 2016 system or newer, we’ll want to open Settings -> Accounts -> Access Work or School


From there, we click Connect


We enter our users UserPrincipalName (email address):


A Microsoft 365 Login Page pop-up window will appear and you are asked for the password for this account:


At this stage, the user will be asked for their MFA / secondary form of authentication. Once successfully logged in, Azure AD will issue your device a Primary Refresh Token that is stored inside the Web Account Manager (WAM) service in memory on your device.

### Low Severity MSRC Finding
The following scenario requires some understanding of Conditional Access Policies, MFA Enforcement, and Single-Factor Primary Refresh Tokens.

Conditional Access Policies are policies created by an organization that allows the organization to control who can access “things”, at what times, and in what ways. These policies can be sweeping policies that affect the whole organization or are very granular and affect users of a particular group within the organizations Azure subscription.

These policies can be configured to enforce MFA when accessing any number of Azure’s many APIs, and presents an interesting scenario where a misconfiguration of these policies can be abused.

Let’s present a scenario:

An organization leveraging Azure AD has configured MFA requirements for Windows Graph, and the Azure Core Management API, but has forgotten to configure MFA requirements for Microsoft Graph API. The organization has also forgotten to enforce MFA for the Azure Device Join process.

In this scenario, a user, who has MFA configured for their account, is joining their own device to the organizations subscription. The user is presented with the M365 Login Page and enters their UserPrincipalName and password.

Because MFA isn’t enforced on the Device Join process, when the user enters their credentials, Azure issues their device a Single-Factor Primary Refresh Token and the user is then asked for their MFA tokens. Once the MFA tokens have been received by Azure AD, Azure will reissue the Primary Refresh Token with an MFA imprint inside the token.

It is the space between the Single-Factor PRT being issued and the user (or adversary) is asked to present MFA tokens where this misconfiguration becomes an issue. This allows an adversary to steal a Single-Factor PRT and use this PRT to obtain an access token for the Microsoft Graph API, which, as we remember, does not have MFA enforced by the organization.

Let’s see an example of an abuse-case for this scenario:

### Stealing the PRT – ROADTools
This method requires two tools:

- ROADToken
- ROADTools
First thing we’ll need to is grab a required nonce from Microsoft, leveraging roadrecon from ROADTools:


We’ll then take our nonce and ROADToken.exe over to our Azure Joined device and steal the Primary Refresh Token:


We’ll take the blurred out data field, which is the Single Factor Primary Refresh Token, and pass that off to roadrecon in order to obtain our Azure Access Token and FOCI Refresh Token:

- ROADTools Command:
```bash
roadrecon auth --prt-cookie '--data-element-from-last-step--' -c 00b41c95-dab0-4487-9791-b9d2c32c80f2 --tokens-stdout
```
Note: The -c 00b41c95-dab0-4487-9791-b9d2c32c80f2 portion of the above command is a client ID for the Microsoft Graph API, a FOCI application. This client ID can be used to obtain refresh and access tokens for any of the FOCI applications and will become important later on.


With the “refreshToken” element from the JSON string outputted by roadrecon, we can use this Refresh Token directly with AzureHound.

-AzureHound Command:
```bash
azurehound list az-ad -r '--refresh-token-from-last-step--' --tenant '--clients-tenant-id--' -o azurehound-output.json
```
As you can see by the error message (red text) in the above output, there was an issue obtaining data from an Azure resource. This is because the Primary Refresh Token I stole was a Single-Factor Primary Refresh Token. In my Azure lab, I configured some Conditional Access Policies to prevent Single-Factor PRTs from accessing certain Azure resources.

While this does prevent us from getting a complete picture for BloodHound, we can see from the output, that we are still able to get a large portion of data from Azure AD, which aids us in continuing our attack against the client organization.

This is the importance of ensuring that MFA For Device Join is Enforced by organizations leveraging Azure AD. That being said, this attack is still viable, and albeit better, when the PRT is an MFA imprinted token.

Continuing on, and adding insult to injury, we still have a valid Primary Refresh Token that we can use for other purposes:

### TokenMan – Collecting Intel From Teams
Let’s take a couple steps back… We have an Azure Joined device with a Single Factor PRT. We’ve stolen the PRT and collected data with AzureHound. Can we leverage the PRT to steal credentials from Teams?

Using TokenMan, we’re going to want to swap our Windows Graph API FOCI Refresh Token (-c 00b41c95-dab0-4487-9791-b9d2c32c80f2) and request the Microsoft Teams FOCI access and refresh tokens (1fec8e78-bce4-4aaf-ab1b-5451cc387264):


We’re going to take this new access token and open up AADInternals so that we can start stealing credentials users have shared amongst themselves over Teams.

First we’ll need to create the $at variable and assign that variable the access token we just obtained.


And then we just request all the Teams messages with the Get-AADIntTeamsMessages commandlet from AADInternals:


After we’ve stolen credentials, we can even send Teams messages while masquerading as the compromised user account:



## Conclusion
You should always enforce MFA on Azure Device Join and enforce MFA on all Azure APIs that your organization uses to contain sensitive information or APIs that allow for a user to perform actions within the Azure subscription.

This blog post demonstrated what an adversary could do with an Azure Joined Device and can even be extended to demonstrate what an adversary can do with a stolen laptop, where the victim’s device is joined to an organization Azure subscription.

