Screenshots can be found at:  https://web.archive.org/web/20240917162314/https://nullg0re.com/2023/09/automating-osint-with-gcp/

First:


# What is OSINT
OSINT is an abbreviation that stands for Open Source Intelligence Gathering. It consists of techniques used by adversaries and pentesters alike to identify information about a target organization that provides information to the operator such as:

Live hosts
Cloud presence
Leaked secrets
Previously breached credentials / disclosures
Technology stacks leaked in job postings
etc…
OSINT is passive in nature and primarily relies on information obtained without actually sending traffic downrange to the target organizations assets.

# OSINT Automation
OSINT operators who regularly perform Open Source Intelligence Gathering will often times perform the same series of tasks over and over again and need some way of automating those activities so that they are repeatable and scalable.

An operator would likely use a Cloud technology like AWS, Azure, or GCP to facilitate this automation for the following reasons:

Cloud technologies like GCP’s CloudRun allow you to create small containers with only the functionality that you’d need for a given task, making them fast, small, and accessible via the open internet enabling OSINT teams to work together.
GCP CloudRun cycles IP addresses for every deployed instance of the OSINT container. This is great because, if, for whatever reason, you have a tool or workflow that does require sending packets downrange, it keeps the traffic somewhat anonymous.
An OSINT operator, or team, might deploy multiple containers to CloudRun and chain them together so that the output of one container feeds into the input of another container, making the process of obtaining OSINT faster and more efficient.
I wrote a tool called Nuclear OSINT that does just this.

# Nuclear OSINT
Nuclear is a python script that sends data to a series of Google Cloud CloudRun containers that performs the following actions:

Track domain name and target LinkedIn URL for billing purposes
Send domain name to CloudRun container running Amass for subdomain enumeration
Send domain name and LinkedIn URL to CloudRun container running a tool used to collect email addresses and check them against the HaveIBeenPwned API to identify any public data breach disclosures.
Send domain name to CloudRun container running AADInternals to identify Azure subscription ownership.
Send the Amass CloudRun containers output to three separate containers:
CloudRun container running HTTPX to simulate “browsing” and helping find web applications
CloudRun container running DNS lookups to find “live” hosts utilizing DNS records
CloudRun container running Subjack to identify any subdomain takeover opportunities
Send the DNS Lookup CloudRun container output to two additional containers:
CloudRun container running WhoIs lookups to find domain ownership (which is a great way to identify Cloud hosting provider ownership)
CloudRun container running Shodan to passively find open ports on the target organization as well as get an insight on potential CVEs
All of this data is requested by the operators python script, which goes out and collects the above data, then parses and manages the data so that it can be used in reporting and action-on-objectives later on.

The above items are all passive in nature except for HTTPX which is the only “Active” sequencing in this process. This, however, is a necessary step in some cases as it allows the operator to perform a domain flyover and is considered an additional data collection point where the information gathered here can aid in pre-emptively confirming or rejecting some of the additional data found.

It must be stated that all activities between an ethical hacker and a target organization must be talked about and consented to ahead of time. In the cases that I’ve needed to include the HTTPX traffic in my OSINT collection activities, the activity was agreed upon ahead of time.

# But How Fast Is This?
Before creating the Nuclear CloudRun framework, I was running this as a linear python script, that in some cases, due to the size of the target domain, could take anywhere from 5 to 15 hours to complete. After building Nuclear in CloudRun, the longest runtime for this script that I’ve had has been under an hour because there are multiple tasks being executed at the same time. Even where one task’s dependencies required an upstream task to finish, the speed at which the combined CloudRun activities finished is a win in my book, allowing me to perform more OSINT operations in a day being the sole operator.

Cloud automation, if done properly, can save time and money. To have this framework running in GCP, it cost me less than a cup of coffee per month to keep the framework running in GCP, as you are only charged for the CloudRun containers run time. The following table is taken from GCP’s CloudRun pricing page and demonstrates how cheap this kind of a setup can be:


Source: https://cloud.google.com/run/pricing

# Additional Reading Material
I’ve left the creation of a Nuclear style OSINT collection service as an exercise for the reader. Here’s a couple of helpful links to help you get started:

https://cloud.google.com/run
https://developers.google.com/learn/pathways/cloud-run-serverless-computing
https://k21academy.com/google-cloud/google-cloud-run/
https://medium.com/google-cloud/deploy-serverless-container-google-cloud-run-68d716af7716
# Conclusion
OSINT at scale is a challenge but leveraging the power of Cloud technologies like GCP’s CloudRun, or similar products in AWS or Azure can really help in the creation of OSINT (or other) products that operate quickly and in masse.


