# The Complete Beginner's MLOps-on-AWS Lab
## Every step, every command, every "why" — assuming you've created nothing and know nothing yet

This is the same one-hour project as before, rewritten so that **nothing is assumed**. Setup will take you 30–45 minutes the first time (it's one-time work — every future AWS project skips it), and the lab itself about an hour. Read the "Why" before every command; typing commands you don't understand teaches you nothing.

---

# PART 0 — THE CONCEPTS YOU NEED BEFORE TOUCHING ANYTHING (10 min read)

### What is AWS, really?
Amazon Web Services is a landlord for computers. Amazon owns warehouses full of servers ("data centers"); you rent slices of them by the second and control everything through **APIs** — web requests that say "create me a storage bucket," "start me a server." Everything you'll do today — every console click, every command — is ultimately an API call.

### What is a Region?
AWS's data centers are grouped into **Regions** — independent geographic clusters like `us-east-1` (N. Virginia) or `ap-south-1` (Mumbai). Resources live in ONE region. If you create a bucket in `us-east-1` and then look at the console while it's set to `ap-south-1`, you'll see nothing and panic. **Rule for this lab: pick one region and never change it.** We'll use `us-east-1` throughout (it's the default in most documentation and has every service). The region selector is in the top-right of the AWS web console.

### Root user vs IAM user — the first security lesson
When you create an AWS account, you get the **root user** (the email you signed up with). Root can do *anything*, including delete the account — so the universal rule is: **log in as root once, create a working identity, then never use root again.** The working identities live in **IAM** (Identity and Access Management), AWS's system for *who* can do *what*. Everything in AWS is deny-by-default: an identity can do nothing until a **policy** (a JSON document listing allowed actions) is attached to it.

### Console vs CLI — why we'll use a terminal
The **AWS Console** is the website — good for looking at things. The **AWS CLI** is a terminal program (`aws ...` commands) that makes the same API calls from your keyboard — good for *doing* things, because commands are precise, repeatable, and copy-pasteable (a tutorial can't misclick). Real platform work is CLI/code, not clicking. We'll use the console to *verify* and the CLI to *act*.

### The cast of services in this lab
- **S3** (Simple Storage Service) — stores files ("objects") in "buckets." Infinitely large, absurdly durable. Our datasets and model files live here.
- **IAM** — identities and permissions, as above.
- **SageMaker** — AWS's machine-learning service. We use one slice of it: it can take a model file and run it behind an HTTPS endpoint (a URL that answers predictions).
- **CloudWatch** — AWS's built-in metrics/monitoring. Every service reports numbers here automatically.
- **MLflow** — NOT an AWS service; a free open-source tool that runs on your laptop and records your training experiments. Industry standard.

### What "MLOps" means (the one-paragraph version)
Machine learning in production is three things changing independently: **code, data, and the model**. MLOps is the discipline of versioning all three, tracking which combination produced which model, deploying models through a controlled path (never "email me the file"), watching them in production (they rot as the world changes — "drift"), and retraining/rolling back safely. Today you will do every one of those things once, by hand, in miniature. Every automated pipeline you ever build later is just freezing these manual steps into a script.

---

# PART 1 — ONE-TIME SETUP (30–45 min)

## Step 1.1 — The AWS account

If you already have an AWS account, skip to 1.2. If not: go to https://aws.amazon.com → "Create an AWS Account." You'll need an email, a phone number, and a credit/debit card (AWS bills you only for what you use; this lab costs a few cents). Choose the free "Basic support" plan. When done, you can sign in as the **root user**.

**Why the card if it's cents?** AWS has no prepaid mode; the card is how they bill actual usage. This is exactly why the teardown step at the end matters.

## Step 1.2 — Create your working IAM identity (as root, the last time you'll use it)

**Why:** the root-user rule from Part 0. We create an "admin" IAM user for daily work.

1. Sign in to the console as root → search "IAM" in the top search bar → open IAM.
2. Left menu → **Users** → **Create user**.
3. User name: `sumit-admin`. Tick **"Provide user access to the AWS Management Console"** → choose "I want to create an IAM user" → set a password.
4. Next → **Attach policies directly** → search and tick **`AdministratorAccess`**.
   - **Why AdministratorAccess?** It's a built-in policy allowing everything except root-only actions. For a personal learning account this is normal. In a company, you'd get narrow permissions — least privilege — but you can't learn AWS while fighting permission denials on day one.
5. Create user. Note the console sign-in URL shown (it contains your 12-digit account ID).
6. **Sign out of root. Sign in as `sumit-admin`.** From now on, always this user.

## Step 1.3 — Give your terminal access: access keys

**Why:** the console knows who you are from your login. Your *terminal* needs credentials too. An **access key** is a username/password pair for programs: an Access Key ID (public-ish) and a Secret Access Key (a password — treat it like one).

1. IAM → Users → `sumit-admin` → **Security credentials** tab → **Create access key**.
2. Use case: "Command Line Interface (CLI)" → tick the confirmation → Create.
3. **Copy both values now** (the secret is shown exactly once). Store them in a password manager.

⚠️ **The rule you already know from interview prep:** these keys must NEVER go into code or Git. They live in one place only — the AWS CLI's config file, which the next step creates. (Leaked keys = bots mining crypto on your card within hours. This is guide Q35 in real life.)

## Step 1.4 — Install and configure the AWS CLI

**Why:** this is the terminal program that turns `aws s3 mb ...` into signed API calls using your keys.

Install (pick your OS):
```bash
# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# macOS
brew install awscli

# Windows: download and run the MSI from
# https://awscli.amazonaws.com/AWSCLIV2.msi   (then use PowerShell for the lab)
```

Verify, then configure:
```bash
aws --version          # should print aws-cli/2.x.x

aws configure
# AWS Access Key ID:     <paste from 1.3>
# AWS Secret Access Key: <paste from 1.3>
# Default region name:   us-east-1        <- our one region, see Part 0
# Default output format: json
```

**What `aws configure` did:** wrote two small files, `~/.aws/credentials` (your keys) and `~/.aws/config` (region/format). Every `aws` command now reads them automatically.

**The verification command you'll use forever:**
```bash
aws sts get-caller-identity
```
**What it does:** asks AWS "who am I?" — returns your account ID and user ARN. If this works, your terminal is correctly authenticated. (STS = Security Token Service, the identity part of AWS. This command is the first line of every credential debugging session in your career.)

## Step 1.5 — Python environment

**Why a "virtual environment":** Python projects need specific library versions; installing everything globally creates conflicts between projects. A **venv** is a private, disposable set of libraries for this project only — the same isolation idea as containers, one level lighter.

```bash
mkdir mlops-lab && cd mlops-lab      # our project directory — stay in it for the whole lab
python3 -m venv .venv                # create the private environment in a hidden folder
source .venv/bin/activate            # activate it (Windows PowerShell: .venv\Scripts\Activate.ps1)
# your prompt now shows (.venv) — pip installs go into the venv, not your system

pip install mlflow scikit-learn pandas boto3 sagemaker joblib
```

**What each library is for:**
- `scikit-learn` — the ML library; provides the dataset and the model algorithm.
- `pandas` — tables (DataFrames) in Python; how we hold the dataset.
- `mlflow` — experiment tracking + model registry (Part 0).
- `boto3` — the AWS SDK for Python: like the CLI, but callable from code. Reads the same `~/.aws` credentials.
- `sagemaker` — a convenience layer over boto3 specifically for SageMaker.
- `joblib` — saves/loads sklearn models to a file.

The install takes a few minutes (sagemaker pulls a lot). Meanwhile, read 1.6.

## Step 1.6 — Create the SageMaker execution role

**The concept — and it's a big one:** when SageMaker later runs your model, *SageMaker's servers* (not your laptop) must download the model file from *your* S3 bucket. What identity do those servers use? Not your user keys — you never hand your keys to a service. Instead, AWS uses **roles**: an identity with permissions that a *service* is allowed to temporarily assume. We'll create a role that says: "the SageMaker service may wear this identity, and while wearing it, it may use SageMaker and read/write S3."

Two parts of every role:
1. **Trust policy** — WHO may assume it (here: the service `sagemaker.amazonaws.com`).
2. **Permission policies** — WHAT the wearer may do (here: SageMaker + S3).

```bash
aws iam create-role --role-name sagemaker-lab-role \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow",
                  "Principal":{"Service":"sagemaker.amazonaws.com"},
                  "Action":"sts:AssumeRole"}]}'
```
**Reading that JSON:** "Allow the principal (actor) sagemaker.amazonaws.com to perform sts:AssumeRole" — i.e., the SageMaker service may put on this costume. That's the trust policy.

```bash
aws iam attach-role-policy --role-name sagemaker-lab-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess
aws iam attach-role-policy --role-name sagemaker-lab-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```
**What these do:** attach two AWS-managed permission policies (prewritten by AWS — note `::aws:` in the ARN means "AWS's own policy, not mine") granting full SageMaker and full S3 access. Broad, fine for a one-hour personal lab; in production you'd write a policy naming the exact bucket. An **ARN** (Amazon Resource Name), by the way, is just AWS's globally-unique ID format for any resource — you'll see them everywhere.

Verify:
```bash
aws iam get-role --role-name sagemaker-lab-role --query 'Role.Arn'
```
You should see something like `arn:aws:iam::123456789012:role/sagemaker-lab-role`. Setup done — everything past this point is the actual MLOps lab.

---

# PART 2 — THE MLOPS LOOP

## Step 2.1 — Versioned data in S3 (~10 min)

**What we're doing:** creating a storage bucket, turning on versioning, and uploading our dataset.

**Why versioning is the whole point:** an ML model is produced by code + data. Code is versioned in Git — but the 2 GB CSV isn't in Git. If the file gets overwritten next month, you can *never* reproduce or debug today's model ("was the model bad, or did the data change?" becomes unanswerable). S3 **bucket versioning** solves this in one switch: every time an object is overwritten, S3 silently keeps the old copy under a **VersionId**. Record that ID with your training run → you can always retrieve the exact bytes it trained on.

```bash
export BUCKET=mlops-lab-$(aws sts get-caller-identity --query Account --output text)
```
**What this does:** sets a shell variable `BUCKET`. Bucket names are **globally unique across all AWS customers** (they become URLs), so `mlops-lab` alone would collide with someone else's; appending your 12-digit account ID makes it unique. The `$( ... )` runs the identity command and pastes its output into the name. `--query Account` filters the JSON down to just the account number, `--output text` strips the quotes.

⚠️ `export` lasts only for the current terminal window. If you open a new terminal later, re-run this line first — "bucket does not exist" errors mid-lab are almost always an empty `$BUCKET`. Check anytime with `echo $BUCKET`.

```bash
aws s3 mb s3://$BUCKET
```
**What:** `mb` = make bucket. `s3://` is the URI scheme S3 commands use.

```bash
aws s3api put-bucket-versioning --bucket $BUCKET \
  --versioning-configuration Status=Enabled
```
**What:** flips the versioning switch. (Why `s3api` not `s3`? The CLI has two S3 command sets: `aws s3` = friendly shortcuts like `cp`/`mb`; `aws s3api` = the raw API, one command per API call, needed for less-common operations like this.)

**Now the dataset.** We use **Iris**, a famous tiny dataset: 150 flowers, 4 measurements each (sepal/petal length/width), and a label 0/1/2 for the species. Our model will predict the species from the measurements. It's deliberately toy-sized — *the model is not the lesson; the machinery is.*

Create a file named `make_data.py` (use any editor — `nano make_data.py`, VS Code, whatever) with exactly this content:

```python
import pandas as pd
from sklearn.datasets import load_iris   # sklearn ships Iris built-in, no download

iris = load_iris(as_frame=True)          # as_frame=True -> give me a pandas DataFrame
df = iris.frame                          # 150 rows x 5 columns (4 features + 'target')
df.to_csv("iris.csv", index=False)       # write to CSV; index=False = no extra row-number column
print(df.shape, "written to iris.csv")
print(df.head())                         # show the first 5 rows so you SEE the data
```

Run it, then upload and inspect versions:

```bash
python make_data.py
aws s3 cp iris.csv s3://$BUCKET/data/iris.csv
```
**What:** `cp` copies local→S3. The path `data/iris.csv` inside the bucket is called the **key** (S3 has no real folders — keys just contain slashes, and the console draws them as folders).

```bash
aws s3api list-object-versions --bucket $BUCKET --prefix data/iris.csv \
  --query 'Versions[].{v:VersionId,t:LastModified}'
```
**What:** lists every stored version of that key. You'll see one entry with a long random `VersionId`. **That string is your data version.** Re-upload the file and run this again — two entries. Nothing is ever lost.

**Verify in the console:** S3 → your bucket → data/ → iris.csv. Seeing your CLI work reflected in the console builds the mental map. (Region top-right must say N. Virginia / us-east-1!)

> **MLOps concept unlocked: data versioning.** You can now name the exact input of any training run.

---

## Step 2.2 — Tracked training with MLflow (~15 min)

**What we're doing:** training a model twice with different settings, and letting MLflow record every run.

**Why tracking exists:** a data scientist tries 50 experiments in a week. Without tracking: "which settings produced the good one? which data? can we rebuild it?" — nobody knows; you get `model_final_v2_REAL.pkl` in an email. With tracking, every run automatically records its **parameters** (the knobs you set), **metrics** (how good it was), **artifacts** (the saved model file), and — because we log it explicitly — the **data VersionId**. That complete record is called **lineage**.

**Three ML terms, quickly:**
- **Hyperparameter** — a setting *you* choose before training (we'll vary one called `C`, which controls how strongly the model is regularized — its exact meaning doesn't matter today; what matters is that it's a knob and different values give different quality).
- **Train/test split** — we hide 20% of the data during training and grade the model on that unseen 20%. Grading on data the model saw is cheating (memorization looks like intelligence).
- **Accuracy** — fraction of test flowers classified correctly. Our metric.

Create `train.py`:

```python
import sys, os
import mlflow, mlflow.sklearn
import pandas as pd, boto3
from sklearn.model_selection import train_test_split
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score

# hyperparameter from the command line: `python train.py 0.01` -> C=0.01
C = float(sys.argv[1]) if len(sys.argv) > 1 else 1.0
BUCKET = os.environ["BUCKET"]            # reads the shell variable you exported

# --- Download the data FROM S3, not from the local file ---
# Why: production habit. Training must consume the versioned source of truth,
# so the run's recorded VersionId genuinely describes what it trained on.
s3 = boto3.client("s3")
s3.download_file(BUCKET, "data/iris.csv", "iris_train.csv")
data_version = s3.head_object(Bucket=BUCKET, Key="data/iris.csv")["VersionId"]
# head_object = fetch metadata without downloading; includes the current VersionId

df = pd.read_csv("iris_train.csv")
X = df.drop(columns=["target"])          # the 4 measurement columns (features)
y = df["target"]                         # the species label we want to predict
X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2, random_state=42)
# random_state=42 fixes the shuffle so reruns split identically (reproducibility)

with mlflow.start_run() as run:          # everything inside is recorded as ONE run
    # A Pipeline chains preprocessing + model into a single object:
    #   StandardScaler rescales each feature to mean 0 / std 1 (many algorithms need this)
    #   LogisticRegression is the classifier
    # Why one object? The scaler is FITTED ONLY ON TRAINING DATA (no test-data
    # statistics leak in), and later the SAME object serves in production, so
    # serving preprocessing can never drift from training preprocessing.
    model = Pipeline([
        ("scaler", StandardScaler()),
        ("clf", LogisticRegression(C=C, max_iter=200)),
    ])
    model.fit(X_tr, y_tr)                            # learn from the 80%
    acc = accuracy_score(y_te, model.predict(X_te))  # grade on the hidden 20%

    mlflow.log_param("C", C)                         # record the knob
    mlflow.log_param("data_version", data_version)   # record the data lineage
    mlflow.log_metric("accuracy", acc)               # record the grade
    mlflow.sklearn.log_model(model, "model")         # store the model artifact itself
    print(f"run_id={run.info.run_id}  C={C}  accuracy={acc:.4f}")
```

Run it **twice** — two experiments:

```bash
python train.py 0.01     # a deliberately poor setting
python train.py 1.0      # a sensible setting
```

Each prints a `run_id` (a long hex string) and an accuracy. Expect roughly ~0.83 for C=0.01 and ~0.97–1.0 for C=1.0. **Copy the run_id of the better run somewhere — you need it in the next step.**

Now look at what MLflow recorded:

```bash
mlflow ui --port 5000
```
Open http://localhost:5000 in a browser. You'll see an experiment table with both runs: click around — parameters, metrics, and under Artifacts, the stored model. Tick both runs → **Compare**. *This table is the answer to "which model was good and what produced it," forever.* Press `Ctrl+C` in the terminal to stop the UI when done.

(Where did MLflow put all this? A local folder called `mlruns/` it created next to your scripts — MLflow's simplest storage mode. In a company you'd run one shared MLflow server backed by S3 and a database; identical API and UI, just multi-user.)

> **MLOps concepts unlocked:** experiment tracking, lineage, reproducible splits, leakage-safe pipelines, training/serving-skew prevention.

---

## Step 2.3 — Register the winning model (~7 min)

**What we're doing:** promoting the better run's model into a **registry**, and packaging it in the format SageMaker needs.

**Why a registry:** deployment must never start from "a file someone had." A **model registry** is the versioned catalog of blessed models — `iris-classifier` version 1, version 2, … — each traceable to its run. Deployments pull *from the registry*; rollbacks repoint to an older version. We do it in two layers: MLflow's registry (the concept, with version numbers) and S3 (the immutable artifact SageMaker will actually download).

Create `register.py`:

```python
import sys, os, tarfile
import mlflow, mlflow.sklearn
import boto3, joblib

run_id = sys.argv[1]                     # you pass the winning run_id on the command line
BUCKET = os.environ["BUCKET"]

# 1) Register in MLflow: creates 'iris-classifier' version N, linked to the run
result = mlflow.register_model(f"runs:/{run_id}/model", "iris-classifier")
print("registered as version", result.version)

# 2) Package for SageMaker.
# SageMaker's convention: the model must be a gzipped tar archive (model.tar.gz)
# containing the model file. We load the model back from the MLflow run,
# save it as model.joblib, and tar it.
model = mlflow.sklearn.load_model(f"runs:/{run_id}/model")
joblib.dump(model, "model.joblib")
with tarfile.open("model.tar.gz", "w:gz") as t:
    t.add("model.joblib")

# 3) Upload to a VERSIONED PATH in S3. v1 and v2 will live side by side —
# never overwritten. Immutable artifacts are what make rollback trivial.
key = f"models/iris-classifier/v{result.version}/model.tar.gz"
boto3.client("s3").upload_file("model.tar.gz", BUCKET, key)
print(f"artifact: s3://{BUCKET}/{key}")
```

```bash
python register.py <paste_the_winning_run_id_here>
```

Verify: `aws s3 ls s3://$BUCKET/models/iris-classifier/ --recursive` — you should see `v1/model.tar.gz`.

> **MLOps concept unlocked: model registry with immutable versioned artifacts.** From here on, nothing references your laptop.

---

## Step 2.4 — Deploy a real-time endpoint on SageMaker (~15 min, mostly waiting)

**What we're doing:** turning the registered model file into a live HTTPS URL that answers predictions.

**Why this shape:** a model is useful when *other software* can ask it questions — an app sends flower measurements, gets a species back, in milliseconds. That requires the model loaded in memory inside a running server. You could build that server yourself (a Docker container with a web framework — everything from Section 1 of your guide). **SageMaker endpoints** are AWS doing it for you: AWS provides a prebuilt scikit-learn serving container, injects your `model.tar.gz` from S3 into it, runs it on a managed instance, health-checks it, and gives you an invokable endpoint. Conceptually identical to pods-behind-a-service in Kubernetes — just managed.

**The one piece SageMaker can't guess:** how to load *your* model file. You supply a tiny script with a function named exactly `model_fn` — the container calls it at startup. Create `inference.py`:

```python
import os, joblib

def model_fn(model_dir):
    # SageMaker extracts your model.tar.gz into model_dir, then calls this.
    # We saved model.joblib in Step 2.3, so we load and return it.
    return joblib.load(os.path.join(model_dir, "model.joblib"))
```

That's genuinely all — the container's built-in defaults handle the rest (parse incoming CSV → call model.predict → return the result). And because the loaded object is our *Pipeline*, the scaler runs automatically on every request: training preprocessing = serving preprocessing, by construction.

Create `deploy.py`:

```python
import os, boto3
from sagemaker.sklearn import SKLearnModel

BUCKET = os.environ["BUCKET"]
VERSION = os.environ.get("MODEL_VERSION", "1")

# Fetch the role ARN we created in setup — SageMaker's servers will assume it
# to download the model from YOUR bucket (the "why roles exist" from Step 1.6).
role = boto3.client("iam").get_role(RoleName="sagemaker-lab-role")["Role"]["Arn"]

model = SKLearnModel(
    model_data=f"s3://{BUCKET}/models/iris-classifier/v{VERSION}/model.tar.gz",
    role=role,
    entry_point="inference.py",     # our loader script, uploaded alongside
    framework_version="1.2-1",      # which prebuilt sklearn container image to use
    name=f"iris-classifier-v{VERSION}",
)
predictor = model.deploy(
    initial_instance_count=1,        # one serving instance (prod would run 2+ for HA)
    instance_type="ml.m5.large",     # the machine size — THIS is what bills hourly
    endpoint_name="iris-endpoint",   # the stable name clients will call
)
print("endpoint InService: iris-endpoint")
```

```bash
export MODEL_VERSION=1
python deploy.py
```

**This takes 5–8 minutes.** Use the wait productively — watch it happen in the console: **SageMaker → Inference → Endpoints** → `iris-endpoint` shows status `Creating`. Behind the scenes, in order: SageMaker created a **Model** object (pointer to your S3 artifact + container image), an **Endpoint Config** (which model, what instance type, how many), and an **Endpoint** (the running thing) — three separate objects, and that separation is exactly what makes the version-2 update in Step 2.7 clean. It then launched the instance, pulled the container, downloaded your model, called your `model_fn`, and is health-checking before flipping status to `InService` — the same readiness-probe idea as Kubernetes deployments.

**If it fails:** the error almost always names the cause. `ValidationException ... role` → the role from 1.6 wasn't created or lacks S3 access. `NoSuchKey` → the S3 path is wrong (did `register.py` print v1, and did you export MODEL_VERSION=1?). This is IAM/S3 debugging — your bread and butter.

> **MLOps concept unlocked: registry-driven deployment.** The endpoint was constructed from a versioned artifact at a known S3 path — an auditable, repeatable, laptop-free path to production.

---

## Step 2.5 — Invoke it like a client would (~5 min)

**What we're doing:** sending real prediction requests over the network.

**Why this step matters conceptually:** notice what the client needs — the endpoint *name* and AWS credentials. Not the model, not sklearn, not the data. The model is now a *service*. Also notice authentication: requests are signed with your AWS identity (boto3 does it invisibly) — IAM controls who may invoke, with no API keys to leak.

Create `invoke.py`:

```python
import boto3

rt = boto3.client("sagemaker-runtime")   # the runtime client is for CALLING endpoints
                                          # (plain "sagemaker" client is for managing them)

# Two flowers, one per line: sepal_len, sepal_wid, petal_len, petal_wid
payload = "5.1,3.5,1.4,0.2\n6.7,3.0,5.2,2.3"

resp = rt.invoke_endpoint(
    EndpointName="iris-endpoint",
    ContentType="text/csv",              # tells the container how to parse the body
    Body=payload,
)
print("predictions:", resp["Body"].read().decode())
```

```bash
python invoke.py
```

Expected output: `predictions: [0. 2.]` — the first flower is species 0 (setosa: tiny petals), the second species 2 (virginica: large petals). Your model, live, over HTTPS. **Run it 5–10 times** — you're generating traffic for the monitoring step.

---

## Step 2.6 — Monitoring: ops metrics + drift (~10 min)

**Why monitoring is different for ML:** normal software fails loudly (errors, 500s). **Models fail silently** — the endpoint stays green and fast while predictions quietly become garbage, because the *world* changed since training (guide Q56: drift). So you watch two layers.

**Layer 1 — operational metrics (CloudWatch, automatic).** Every invocation you just made was counted:

```bash
# Linux date syntax; on macOS replace the two --start/--end values with e.g.
#   $(date -u -v-30M +%Y-%m-%dT%H:%M:%S) and $(date -u +%Y-%m-%dT%H:%M:%S)
aws cloudwatch get-metric-statistics \
  --namespace AWS/SageMaker --metric-name Invocations \
  --dimensions Name=EndpointName,Value=iris-endpoint Name=VariantName,Value=AllTraffic \
  --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time   $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 --statistics Sum
```
**Reading the command:** metrics live in namespaces (`AWS/SageMaker`); dimensions filter to *your* endpoint; `--period 300` buckets counts into 5-minute windows; `Sum` totals each bucket. You should see your invocations. Friendlier view: console → CloudWatch → Metrics → SageMaker — look at `Invocations`, `ModelLatency`, `Invocation4XXErrors`. In production you'd attach **alarms** ("latency > 200ms for 5 min → page someone").

**Layer 2 — data drift (the ML-specific layer).** The trap: you usually can't compute live accuracy, because truth arrives late or never (you don't learn a loan defaulted until months pass). The workaround: **compare the distribution of live inputs against the training data.** If today's inputs look statistically different from what the model learned on, its answers are suspect — *no labels needed*. Create `drift_check.py`:

```python
import pandas as pd

train = pd.read_csv("iris_train.csv").drop(columns=["target"])

# Simulate this week's live traffic: suppose a new flower population appeared
# and petal lengths shifted upward. (In prod you'd load logged real requests.)
live = train.copy()
live["petal length (cm)"] = live["petal length (cm)"] + 1.5

# For each feature: how far did the live mean move, measured in units of the
# training standard deviation? ("Standardized mean difference" — the simplest
# real drift statistic. >0.5 std devs = worth an alert.)
for col in train.columns:
    shift = abs(live[col].mean() - train[col].mean()) / train[col].std()
    flag = "  <-- DRIFT" if shift > 0.5 else ""
    print(f"{col:25s} shift={shift:.2f} std devs{flag}")
```

```bash
python drift_check.py
```

Three features near 0.00; petal length screams. That alert is your trigger to investigate and retrain — which is precisely the next step. (AWS's productized version of this exact comparison is SageMaker Model Monitor; tools like Evidently do it open-source. You just hand-rolled the core idea.)

> **MLOps concepts unlocked:** operational monitoring, and label-free drift detection as the early-warning system.

---

## Step 2.7 — Retrain v2 and update the live endpoint, blue/green (~10 min)

**What we're doing:** producing model version 2 and swapping it into the running endpoint **without downtime**, then rolling back.

**Why "update" is the scary part of MLOps:** the endpoint is serving users *right now*. Kill-then-replace means an outage; replace-in-place risks serving a broken model to everyone instantly. The safe pattern is **blue/green**: stand up the new version (green) *alongside* the old (blue), health-check it, shift traffic, and only then retire blue. SageMaker's `update-endpoint` does this automatically — and you'll watch it.

**Produce v2** (pretend the drift alert triggered a retrain — same commands as before, new hyperparameter):

```bash
python train.py 10.0
# copy the printed run_id, then:
python register.py <new_run_id>
# it will print "registered as version 2" and upload .../v2/model.tar.gz
aws s3 ls s3://$BUCKET/models/iris-classifier/ --recursive   # v1 AND v2, side by side
```

**Now the swap.** Remember the three objects from Step 2.4 (Model, Endpoint Config, Endpoint)? We create a *new* Model and a *new* Config pointing at v2, then tell the *existing* Endpoint to move to the new config. Create `update.py`:

```python
import os, boto3
from sagemaker.sklearn import SKLearnModel

BUCKET = os.environ["BUCKET"]
role = boto3.client("iam").get_role(RoleName="sagemaker-lab-role")["Role"]["Arn"]

# 1) Define the v2 model (create() registers it with SageMaker WITHOUT deploying)
m = SKLearnModel(
    model_data=f"s3://{BUCKET}/models/iris-classifier/v2/model.tar.gz",
    role=role, entry_point="inference.py",
    framework_version="1.2-1", name="iris-classifier-v2",
)
m.create(instance_type="ml.m5.large")

sm = boto3.client("sagemaker")

# 2) New endpoint config: same instance type, but pointing at the v2 model
sm.create_endpoint_config(
    EndpointConfigName="iris-config-v2",
    ProductionVariants=[{
        "VariantName": "AllTraffic",
        "ModelName": "iris-classifier-v2",
        "InstanceType": "ml.m5.large",
        "InitialInstanceCount": 1,
    }],
)

# 3) Point the LIVE endpoint at the new config -> SageMaker performs blue/green
sm.update_endpoint(EndpointName="iris-endpoint",
                   EndpointConfigName="iris-config-v2")
print("updating iris-endpoint to v2 (blue/green in progress)...")
```

```bash
python update.py
```

**The important experiment — do this during the update (it takes ~5 min):**

```bash
python invoke.py    # run it several times WHILE the console shows status "Updating"
```

It keeps answering. Every time. In the console (SageMaker → Endpoints → iris-endpoint) the status is `Updating`: AWS launched a second instance with v2, waited for it to pass health checks, shifted traffic to it, and only then terminated the v1 instance. Blue/green, zero downtime, and you served requests straight through it. When status returns to `InService`, the endpoint is v2.

**Rollback — the payoff of everything you versioned.** Suppose v2 turns out worse in production. The v1 endpoint config still exists:

```bash
aws sagemaker list-endpoint-configs --query 'EndpointConfigs[].EndpointConfigName'
# find the original config name — the SDK auto-generated it, it contains "iris-classifier-v1"
aws sagemaker update-endpoint --endpoint-name iris-endpoint \
  --endpoint-config-name <that-v1-config-name>
```

One command. Another blue/green, in reverse, zero downtime, and production is back on v1 — while you diagnose v2 offline at leisure. *This* is why models flow through registries and immutable artifacts: rollback becomes a pointer change, not a rebuild.

> **MLOps concepts unlocked:** the retraining loop, blue/green deployment, instant rollback.

---

## Step 2.8 — What a pipeline would automate (~5 min, reading)

Look back at what you did. You *were* the pipeline:

| You did, by hand | Production automates it with |
|---|---|
| Noticed drift in `drift_check.py` | Model Monitor on a schedule → CloudWatch alarm → triggers retraining |
| `python train.py` | A training pipeline (SageMaker Pipelines, Airflow, Step Functions) on schedule/trigger |
| Compared runs in the MLflow UI, picked the winner | A champion/challenger gate: register only if metrics beat current production |
| `python register.py` | Automatic registration on gate pass |
| `python update.py` | CD triggered by registry promotion — usually canary (5% of traffic first), not a full flip |
| The rollback command | Automatic rollback on alarm during the canary |

Nothing in a "real" MLOps pipeline is a new *concept* — it's these exact steps, frozen into code, with humans removed from the loop. You now understand what every box in every MLOps architecture diagram is *for*, because you've been each box.

---

## Step 2.9 — TEARDOWN (~5 min) — NOT OPTIONAL

**Why:** everything else in this lab is free or fractions of a cent, but the **endpoint is a running instance billing every hour, 24/7, forever, until deleted** (~$0.115/hr for ml.m5.large ≈ $83/month). Forgotten endpoints are the #1 way learners get a shock bill.

```bash
# 1) The endpoint — the only hourly-billing resource. Delete FIRST.
aws sagemaker delete-endpoint --endpoint-name iris-endpoint

# 2) Endpoint configs (free, but be tidy). List, then delete each:
aws sagemaker list-endpoint-configs --query 'EndpointConfigs[].EndpointConfigName'
aws sagemaker delete-endpoint-config --endpoint-config-name iris-config-v2
aws sagemaker delete-endpoint-config --endpoint-config-name <the-v1-config-name>

# 3) Model objects. List, then delete each (names contain iris-classifier):
aws sagemaker list-models --query 'Models[].ModelName'
aws sagemaker delete-model --model-name iris-classifier-v2
aws sagemaker delete-model --model-name <the-v1-model-name>

# 4) The S3 bucket. Versioned buckets refuse simple deletion (that's the feature!),
#    so first delete all versions, then the bucket:
aws s3api delete-objects --bucket $BUCKET --delete "$(aws s3api list-object-versions \
  --bucket $BUCKET --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' --output json)"
aws s3 rb s3://$BUCKET
```

**Verify with your eyes:** console → SageMaker → Endpoints must be **empty**. That's the page that costs money. (The IAM role and user cost nothing — keep them for the next lab.)

---

# WHAT YOU CAN NOW SAY IN AN INTERVIEW — TRUTHFULLY

"I've built the full MLOps loop hands-on on AWS from scratch: IAM users and service roles, versioned training data in S3, MLflow experiment tracking with data-version lineage, a model registry with immutable versioned artifacts, registry-driven deployment to a SageMaker real-time endpoint, IAM-authenticated invocation, CloudWatch operational monitoring plus a statistical drift check against the training baseline, and a live blue/green model update with a one-command rollback."

Every clause is something you literally just did — including the parts most candidates only describe from slides (the blue/green update while traffic flowed, the rollback, the drift check without labels).

**Next extensions, in order of value:**
1. Wrap Steps 2.2–2.3 in a GitHub Actions job (you know Actions from your book-sharing platform) → "training triggered from CI."
2. Recreate the bucket + role in **Terraform** instead of CLI → connects this lab to your Terraform curriculum.
3. Replace the endpoint with your own Flask/FastAPI container on your EKS knowledge → proves the "SageMaker is just managed pods-behind-a-service" claim to yourself.
