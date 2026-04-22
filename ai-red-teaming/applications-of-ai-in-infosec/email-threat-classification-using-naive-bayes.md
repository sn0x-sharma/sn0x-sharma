---
icon: envelope-open
cover: ../../.gitbook/assets/2av3tgbewbfz-vritrasec.png
coverY: 73.98742928975487
---

# Email Threat Classification using Naive Bayes

#### Core Concept

Spam detection is one of the oldest ML problems in infosec. The idea is simple: given an incoming message, figure out if it's legit or junk. The reason this matters from a security angle is that spam isn't just annoying — it's the primary delivery vehicle for phishing, malware droppers, and social engineering attacks. Your email gateway is your first line of defense, and if it's dumb, attackers walk right through it.

Naive Bayes is the classic algorithm for this. It's based on Bayes' Theorem — a way to calculate the probability of something being true given evidence you've already seen.

<figure><img src="../../.gitbook/assets/image (42).png" alt=""><figcaption></figcaption></figure>

### How It Works - Bayes' Theorem

The formula is:

```
P(Spam | Features) = ( P(Features | Spam) * P(Spam) ) / P(Features)
```

Break that down:

* `P(Spam | Features)` — given the words in this email, what's the probability it's spam? That's what we want.
* `P(Features | Spam)` — how likely are these words to appear in a spam email? We learn this from training data.
* `P(Spam)` — what fraction of all emails are spam? Just a historical baseline.
* `P(Features)` — how common are these words overall? Used to normalize the score.

The "Naive" part is the assumption that every word is independent of every other word. That's obviously not true in real language — "free" and "money" together are way more suspicious than either alone — but it works well enough in practice and makes the math extremely fast.

So the likelihood of all features together becomes:

```
P(Features | Spam) = P(word1 | Spam) * P(word2 | Spam) * ... * P(wordN | Spam)
```

#### Worked Example

Say an email has two features: F1 and F2.

```
P(Spam)             = 0.3   # 30% of all emails are spam historically
P(Not Spam)         = 0.7
P(F1 | Spam)        = 0.4   # F1 appears in 40% of spam emails
P(F2 | Spam)        = 0.5
P(F1 | Not Spam)    = 0.2
P(F2 | Not Spam)    = 0.3
```

Step 1: joint likelihood

```
P(F1,F2 | Spam)     = 0.4 * 0.5 = 0.2
P(F1,F2 | Not Spam) = 0.2 * 0.3 = 0.06
```

Step 2: total probability of these features (law of total probability)

```
P(F1,F2) = (0.2 * 0.3) + (0.06 * 0.7)
         = 0.06 + 0.042
         = 0.102
```

Step 3: posterior probability

```
P(Spam | F1,F2) = (0.2 * 0.3) / 0.102 ≈ 0.588
P(Not Spam | F1,F2) = (0.06 * 0.7) / 0.102 ≈ 0.412
```

0.588 > 0.412 → Email is classified as spam. Simple.

### The Dataset  SMS Spam Collection

The dataset used here is the SMS Spam Collection dataset — 5,574 text messages labeled as either `ham` (legitimate) or `spam`. It was built for SMS spam specifically because most existing datasets at the time focused on email.

Downloading and loading it:

```python
import requests, zipfile, io, pandas as pd

url = "<https://archive.ics.uci.edu/static/public/228/sms+spam+collection.zip>"
response = requests.get(url)
with zipfile.ZipFile(io.BytesIO(response.content)) as z:
    z.extractall("sms_spam_collection")

df = pd.read_csv(
    "sms_spam_collection/SMSSpamCollection",
    sep="\\t",
    header=None,
    names=["label", "message"]
)

# Quick sanity checks
print(df.head())
print(df.isnull().sum())
print("Duplicates:", df.duplicated().sum())
df = df.drop_duplicates()
```

Always check for nulls and duplicates before training anything. Duplicates in particular can leak data from your training set into your test set and inflate your accuracy numbers — a classic trap that makes your model look great in the notebook and garbage in production.

#### Training - Pipeline + GridSearchCV

```python
from sklearn.model_selection import train_test_split, GridSearchCV
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline
from sklearn.feature_extraction.text import CountVectorizer

vectorizer = CountVectorizer()
pipeline = Pipeline([
    ("vectorizer", vectorizer),
    ("classifier", MultinomialNB())
])

# Tune the alpha smoothing parameter
param_grid = {
    "classifier__alpha": [0.01, 0.1, 0.15, 0.2, 0.25, 0.5, 0.75, 1.0]
}

grid_search = GridSearchCV(pipeline, param_grid, cv=5, scoring="f1")
grid_search.fit(df["message"], y)

best_model = grid_search.best_estimator_
print("Best alpha:", grid_search.best_params_)
```

The alpha parameter is Laplace smoothing — it stops the model from assigning zero probability to words it hasn't seen in training. Without it, a single unseen word kills the whole probability product (anything times zero is zero). Alpha adds a small baseline count to every word so the math doesn't break on new vocabulary.

F1-score is used as the evaluation metric here because we care about both false positives (legitimate emails flagged as spam — user loses important email) and false negatives (spam emails that slip through — user gets phished). F1 balances precision and recall.

#### Evaluation

<figure><img src="../../.gitbook/assets/image (41).png" alt=""><figcaption></figcaption></figure>

```python
new_messages = [
    "Congratulations! You've won a $1000 Walmart gift card. Go to <http://bit.ly/1234> to claim now.",
    "Hey, are we still meeting up for lunch today?",
    "Urgent! Your account has been compromised. Verify your details here: www.fakebank.com/verify",
    "Reminder: Your appointment is scheduled for tomorrow at 10am.",
    "FREE entry in a weekly competition to win an iPad. Just text WIN to 80085 now!",
]
```

Output:

```
Message: Congratulations! You've won a $1000 Walmart gift card...
Prediction: Spam | Spam Probability: 1.00

Message: Hey, are we still meeting up for lunch today?
Prediction: Not-Spam | Spam Probability: 0.00

Message: Urgent! Your account has been compromised...
Prediction: Spam | Spam Probability: 0.94

Message: Reminder: Your appointment is scheduled for tomorrow at 10am.
Prediction: Not-Spam | Spam Probability: 0.00

Message: FREE entry in a weekly competition to win an iPad...
Prediction: Spam | Spam Probability: 1.00
```

Saving the model with joblib so you don't have to retrain every restart:

```python
import joblib

joblib.dump(best_model, 'spam_detection_model.joblib')

# Later, load and use directly
loaded_model = joblib.load('spam_detection_model.joblib')
predictions = loaded_model.predict(new_data_processed)
```

#### Attack Scenario |  "NovaCorp Internal Mail Gateway"

Imagine a mid-sized company called **NovaCorp** running an email gateway with a Naive Bayes spam filter trained on their historical email data. The SOC team is proud of it , it catches 95% of spam.

An attacker doing a targeted phishing campaign against NovaCorp employees knows about these kinds of filters. They don't send the classic "CONGRATULATIONS YOU WON" email that gets caught instantly.

Instead, they:

1. **Study NovaCorp's language patterns** — scrape LinkedIn, look at job postings, get the internal vocabulary (project names, department names, tools they use).
2. **Craft a phishing email that looks like internal comms** — "Hi team, please review the attached Q3 budget approval in the OneDrive link below." Subject line: "FW: Budget Sign-off Needed - Action Required."
3. **Avoid spam trigger words** — no "FREE", no "WINNER", no urgency phrases that Naive Bayes learned to flag.
4. **Use a legitimate-looking domain** — `novacorp-internal-docs.com` or an actual compromised vendor's domain.

The Naive Bayes filter never saw this vocabulary in spam training data. It looks like a normal internal email. It sails right through — `P(Spam | Features) ≈ 0.03`.

**The defense here isn't just the spam filter.** It's DMARC/DKIM/SPF for domain validation, link scanning, sandboxing attachments, and user training. ML is one layer, not the whole wall.

***
