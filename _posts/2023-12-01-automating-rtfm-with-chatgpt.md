Screenshots can be found at: https://web.archive.org/web/20240917153006/https://nullg0re.com/2023/12/automating-rtfm-with-chatgpt-a-security-researchers-guide-to-vulnerability-discovery-with-the-help-of-llms/


But first – MUSIC 😛


# BlueHat October 2023
I had the pleasure and privilege of attending Microsoft’s BlueHat October 2023 conference this year and was able to see and enjoy many of the talks that were presented. There are several that stood out but there is one that I specifically want to talk about (and is the inspiration for this post and the work the post details).

A Security Researcher by the name of Dor Dali had a brilliant idea of “going back to the basics” and just reading the Microsoft documentation regarding the Remote Desktop Protocol. This approached enabled him and his team to identify three CVE’s including a Remote Code Execution bug.

His talk was titled “From RTFM to RCE: An Unexpected Dive into the Remote Desktop Protocol” and the foundation of the talk was simply Read The F***ing Manual. Here is the link to his talk if you’d like to watch it for yourself:


# Personal Challenges
I have multiple reverse engineering and exploit development certifications from Offensive Security and the concept of reading the documentation before starting to develop exploits isn’t lost on me. But in practice, at least from my personal experience, I like to tinker first and then read the documentation if something doesn’t make sense.

This approach leads to a TON of long nights, working weekends, banging my head against the wall until something clicks… Needless to say, Dor inspired me to take a different approach.

I believe, one of the biggest challenges in bug hunting platforms as large and complex as Microsoft Azure or Google Cloud Platform, is not having enough time in a lifetime to be able to test every single function or parameter or environment that can be deployed. I’m 34 now and I believe my grandchildren would have grandchildren by the time the Gore family line finished reading all of the current documentation out there. Not even talking about the actual tinker time.

How can we be more efficient in our vulnerability discovery and analysis, while not skimping on the basics (reading the manual) and not skimping on the heavy lifting done during tinkering.

# Enter ChatGPT
Over the past two weeks I started developing a tool with my good friend 0xZDH. Here’s the general breakdown of what the tool does:

Take a Microsoft Documentation URL as an argument
Using Selenium, connect to the documentation URL and collect all chapter and subchapter links found in the documentation page’s Table of Contents
Download all of the HTML content and convert them all to Markdown
Extract the raw text from the Markdown
Split the extracted text into chunks due to ChatGPT prompt size restrictions
Convince ChatGPT it’s a helpful assistant, its a Cybersecurity Expert, and is an expert in all known vulnerabilities
Pass each chunk to ChatGPT and politely ask it to identify any and all vulnerabilities that are described by the protocol / documentation
Convert ChatGPT responses, documentation title page, and chunk number into a Markdown table
Render the final Markdown table in GitHub, GitLab, Obsidian, or some other Markdown rendering tool.
Begin analysis
Tool Output

Final Output

# Good Approach?
I don’t know. And won’t know until this process yields results. But, it’s important to call out here that an operator using this approach would use ChatGPT’s responses to quickly identify potential vulnerabilities, go back and read the surrounding documentation for context clues, and then proceed to build out a lab for tinkering / vulnerability discovery. This is where I believe we could speed up our research / RTFM time.

# Improvements?
We could use a self-developed LLM with Microsoft’s Azure Machine Learning Studio and train our own model to perform this specific task.

We could modify the system prompts sent to ChatGPT to make it more helpful.

We could identify the patterns in ChatGPT’s response to filter out “Not a vulnerability” responses.

This tool is a work in progress and if it nets any vulnerabilities, I’ll write a blog post about it and tag this post in some way.

# Conclusion
Working on a team whose responsibility it is to hunt and find vulnerabilities in Microsoft products, we have to be efficient with our time without forgoing the basics. I believe this approach will help us in achieving that goal and with all things this tool will improve over time. As for now, I have a bunch of new work to do thanks to ChatGPT!

Hope everyone had wonderful holidays! See you in the next post!

