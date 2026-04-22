---
icon: tree-deciduous
cover: ../../.gitbook/assets/orig_m9zpj9.jpg
coverY: -38.61722187303582
---

# Random Forest for Identifying Suspicious Network Activity

### Core Concept

Network anomaly detection is about finding traffic that doesn't look like normal traffic. You train a model on what "normal" looks like, and then anything that deviates gets flagged. Random Forests are a solid choice for this because they handle high-dimensional tabular data well, are resistant to overfitting, and you can extract feature importance to understand what signals the model actually learned.

The NSL-KDD dataset is the standard benchmark for this kind of task — it contains labeled network traffic samples covering normal activity and four categories of attacks: DoS, Probe, Privilege Escalation, and Access attacks.

#### How Random Forests Work

A Random Forest is an ensemble of decision trees. Each tree is trained on a random bootstrap sample of the training data, and at each split it only considers a random subset of features. This randomness creates diversity  each tree makes different errors, and when you average their predictions (voting for classification), the errors cancel out.

<figure><img src="../../.gitbook/assets/image (43).png" alt=""><figcaption></figcaption></figure>

The reason this beats a single decision tree is variance reduction. One tree can memorize the training data. Hundreds of slightly different trees can't all memorize the same noise  their aggregate is smoother and generalizes better.

### Dataset NSL-KDD

Downloading:

```python
import requests, zipfile, io

url = "<https://academy.hackthebox.com/storage/modules/292/KDD_dataset.zip>"
response = requests.get(url)
z = zipfile.ZipFile(io.BytesIO(response.content))
z.extractall('.')
```

Loading with column names:

```python
import pandas as pd

columns = [
    'duration', 'protocol_type', 'service', 'flag', 'src_bytes', 'dst_bytes',
    'land', 'wrong_fragment', 'urgent', 'hot', 'num_failed_logins', 'logged_in',
    'num_compromised', 'root_shell', 'su_attempted', 'num_root', 'num_file_creations',
    'num_shells', 'num_access_files', 'num_outbound_cmds', 'is_host_login', 'is_guest_login',
    'count', 'srv_count', 'serror_rate', 'srv_serror_rate', 'rerror_rate', 'srv_rerror_rate',
    'same_srv_rate', 'diff_srv_rate', 'srv_diff_host_rate', 'dst_host_count', 'dst_host_srv_count',
    'dst_host_same_srv_rate', 'dst_host_diff_srv_rate', 'dst_host_same_src_port_rate',
    'dst_host_srv_diff_host_rate', 'dst_host_serror_rate', 'dst_host_srv_serror_rate',
    'dst_host_rerror_rate', 'dst_host_srv_rerror_rate', 'attack', 'level'
]

df = pd.read_csv('KDD+.txt', names=columns)
print(df.head())
```

### Preprocessing

**Step 1  Binary classification target** (normal vs attack)

```python
df['attack_flag'] = df['attack'].apply(lambda a: 0 if a == 'normal' else 1)
```

**Step 2 Multi-class target** (which kind of attack?)

```python
dos_attacks = ['apache2', 'back', 'land', 'neptune', 'mailbomb', 'pod',
               'processtable', 'smurf', 'teardrop', 'udpstorm', 'worm']
probe_attacks = ['ipsweep', 'mscan', 'nmap', 'portsweep', 'saint', 'satan']
privilege_attacks = ['buffer_overflow', 'loadmdoule', 'perl', 'ps',
                     'rootkit', 'sqlattack', 'xterm']
access_attacks = ['ftp_write', 'guess_passwd', 'http_tunnel', 'imap',
                  'multihop', 'named', 'phf', 'sendmail', 'snmpgetattack',
                  'snmpguess', 'spy', 'warezclient', 'warezmaster',
                  'xclock', 'xsnoop']

def map_attack(attack):
    if attack in dos_attacks:
        return 1
    elif attack in probe_attacks:
        return 2
    elif attack in privilege_attacks:
        return 3
    elif attack in access_attacks:
        return 4
    else:
        return 0   # normal

df['attack_map'] = df['attack'].apply(map_attack)
```

**Step 3  Encode categorical features**

`protocol_type` and `service` are strings — ML algorithms need numbers. One-hot encoding turns each category into its own binary column. This avoids implying any ordinal relationship (like "tcp > udp" which would be meaningless).

```python
features_to_encode = ['protocol_type', 'service']
encoded = pd.get_dummies(df[features_to_encode])
```

**Step 4 — Select numeric features and combine**

```python
numeric_features = [
    'duration', 'src_bytes', 'dst_bytes', 'wrong_fragment', 'urgent', 'hot',
    'num_failed_logins', 'num_compromised', 'root_shell', 'su_attempted',
    'num_root', 'num_file_creations', 'num_shells', 'num_access_files',
    'num_outbound_cmds', 'count', 'srv_count', 'serror_rate',
    'srv_serror_rate', 'rerror_rate', 'srv_rerror_rate', 'same_srv_rate',
    'diff_srv_rate', 'srv_diff_host_rate', 'dst_host_count', 'dst_host_srv_count',
    'dst_host_same_srv_rate', 'dst_host_diff_srv_rate',
    'dst_host_same_src_port_rate', 'dst_host_srv_diff_host_rate',
    'dst_host_serror_rate', 'dst_host_srv_serror_rate', 'dst_host_rerror_rate',
    'dst_host_srv_rerror_rate'
]

train_set = encoded.join(df[numeric_features])
multi_y = df['attack_map']
```

#### Splitting the Dataset

Three splits  not two. This tripped me up at first. You need a train set to fit the model, a validation set to tune hyperparameters without touching your test data, and a test set to do the final honest evaluation. If you tune hyperparameters on your test set, you're cheating  your test set stops being "unseen" the moment any decision is influenced by it.

```python
from sklearn.model_selection import train_test_split

# First split: 80% train, 20% test
train_X, test_X, train_y, test_y = train_test_split(
    train_set, multi_y, test_size=0.2, random_state=1337
)

# Second split: carve validation from training
multi_train_X, multi_val_X, multi_train_y, multi_val_y = train_test_split(
    train_X, train_y, test_size=0.3, random_state=1337
)
```

What you end up with:

<figure><img src="../../.gitbook/assets/Screenshot 2026-03-13 144540.png" alt=""><figcaption></figcaption></figure>

#### Uploading the Model for Evaluation

After training and saving:

```python
import joblib
joblib.dump(best_model, "network_anomaly_detection_model.joblib")
```

Submit to HTB evaluation portal:

```python
import requests, json

url = "<http://localhost:8001/api/upload>"
model_file_path = "network_anomaly_detection_model.joblib"

with open(model_file_path, "rb") as model_file:
    files = {"model": model_file}
    response = requests.post(url, files=files)

print(json.dumps(response.json(), indent=4))
```

From your own machine over VPN: navigate to `http://<VM-IP>:8001/` and upload manually.

#### Attack Scenario | "DataStack Corp" Internal Network"

**DataStack Corp** runs a network intrusion detection system (NIDS) built on a Random Forest model trained on their historical traffic. The model was trained on "normal" — internal DNS queries, HTTP to known SaaS tools, AD authentication traffic, etc.

An attacker has initial access via a phishing email (bypassed the spam filter from Section 1). Now they're inside the network doing lateral movement.

The naive attacker runs `nmap -sV -p- 192.168.1.0/24` from the compromised host — this immediately triggers the model. The feature `srv_count` (number of connections to the same service from the same host) spikes to 255. The `serror_rate` (SYN error rate) blows up. The model flags this as a `Probe` attack (class 2). SOC gets an alert within seconds.

A smarter attacker does **low-and-slow recon**:

* Space scans out over hours instead of seconds
* Keep `count` and `srv_count` values low — stay within the distribution the model learned as "normal"
* Use legitimate protocols (HTTP, DNS) for C2 instead of raw TCP to weird ports
* Blend into the noise  if the office normally generates 1000 HTTP connections/hour, keep C2 traffic under that baseline

This is called **evasion by staying within the distribution.** The model can only flag what looks anomalous  if the attacker looks like the normal traffic it was trained on, it's invisible.

The deeper problem: NSL-KDD was created in 2009. Trained on 2009 traffic patterns. Real networks in 2024 look nothing like that  cloud traffic, encrypted payloads, modern protocols. A model trained on old data will have massive blind spots on modern attack techniques.
