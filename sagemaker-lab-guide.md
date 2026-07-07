# SageMaker Mini-Lab — Managed Training, Registry, and Endpoints on Real AWS
### Extension to the local MLOps lab. Time: ~2–2.5 hours. Cost if you follow teardown: well under $1.

**What you'll truthfully be able to say afterward:** "I've run SageMaker training jobs in script mode, debugged a failed job through CloudWatch logs, registered models in the SageMaker Model Registry with approval status, deployed a serverless endpoint, invoked it, run batch transform, and torn it all down."

---

## Part 0 — Cost Model First (read before touching anything)

Know what bills BEFORE you create it — that's platform discipline.

| Resource | Price | Your usage | Cost |
|---|---|---|---|
| Training job, ml.m5.large | ~$0.115/hr, billed per second | 2–3 runs × ~4 min | ~$0.05 |
| Serverless endpoint | per-second compute only when invoked; $0 idle | a few dozen invocations | < $0.05 |
| Batch transform, ml.m5.large | ~$0.115/hr | one ~4 min job | ~$0.01 |
| S3 + CloudWatch logs | fractions of a cent | tiny files | ~$0.00 |

**The one thing that can actually cost you money: a REAL-TIME endpoint left running.** A real-time endpoint on ml.m5.large is an EC2 instance billing 24/7 (~$80+/month). That's why this lab uses a **serverless** endpoint (scales to zero, bills per invocation) and why Part 6 (teardown) is not optional. Rule: **never end a SageMaker session without checking the Endpoints page is empty.**

---

## Part 1 — IAM Setup (15 min)

SageMaker jobs don't run as *you* — they run as an **execution role** that SageMaker assumes. The service needs permission to pull your data from S3, write model artifacts back, and emit logs. Misconfigured execution roles are a top-3 source of real SageMaker support tickets, so pay attention to what you're building here.

**Console path:** IAM → Roles → Create role →
1. Trusted entity type: **AWS service** → use case: **SageMaker** (this sets the trust policy so `sagemaker.amazonaws.com` can assume the role).
2. It attaches `AmazonSageMakerFullAccess`. Add **`AmazonS3FullAccess`** too (lab shortcut — in production you'd scope this to specific buckets; say that in the interview).
3. Name: `SageMakerLabRole`. Create, then copy the **Role ARN** (`arn:aws:iam::<account>:role/SageMakerLabRole`).

**On your machine:**
```bash
cd ~/mlops-lab && source venv/bin/activate
pip install sagemaker boto3
aws configure   # if not already configured; use a region like us-east-1 or ap-south-1
aws sts get-caller-identity     # VERIFY: your account/user comes back
```

**WHY two identities:** *your* credentials (aws configure) create/control jobs from your laptop; the *execution role* is what the training container runs as inside AWS. When a training job fails with `AccessDenied` on S3, the fix is on the role, not the user — a distinction that separates people who've used SageMaker from people who've read about it.

---

## Part 2 — Data to S3 (10 min)

SageMaker's contract: **input data comes from S3, model artifacts go back to S3.** Create the dataset and bucket:

```bash
mkdir -p ~/mlops-lab/sagemaker && cd ~/mlops-lab/sagemaker

python - << 'PYEOF'
from sklearn.datasets import load_iris
X, y = load_iris(return_X_y=True, as_frame=True)
df = X.copy()
# simpler column names for CSV round-tripping
df.columns = ["sepal_length", "sepal_width", "petal_length", "petal_width"]
df["target"] = y
df.to_csv("iris.csv", index=False)
print(df.head())
PYEOF

# Bucket names are globally unique — suffix with your account id
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
BUCKET=mlops-lab-$ACCOUNT
aws s3 mb s3://$BUCKET
aws s3 cp iris.csv s3://$BUCKET/iris/train/iris.csv
aws s3 ls s3://$BUCKET/iris/train/          # VERIFY
echo $BUCKET   # note this down
```

---

## Part 3 — A Managed Training Job in Script Mode (40 min)

**The concept you're about to experience:** you hand SageMaker (a) a training script, (b) a framework version, (c) an instance type, and (d) an S3 data location. SageMaker then: provisions a fresh instance → pulls the matching framework container from AWS's ECR → downloads your data from S3 into the container → runs your script → uploads `model.tar.gz` to S3 → **terminates the instance**. Ephemeral batch compute — you pay for minutes, and there is nothing to patch or clean up. Compare this mentally to the Kubernetes Job + GPU node pool pattern you know.

**Step 3.1 — the training script.** Create `sagemaker/train_sm.py`:

```python
import argparse
import os
import joblib
import pandas as pd
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, f1_score

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    # Hyperparameters arrive as CLI args
    parser.add_argument("--n-estimators", type=int, default=100)
    parser.add_argument("--max-depth", type=int, default=5)
    # SageMaker injects these paths via environment variables:
    parser.add_argument("--train", type=str,
                        default=os.environ.get("SM_CHANNEL_TRAIN"))
    parser.add_argument("--model-dir", type=str,
                        default=os.environ.get("SM_MODEL_DIR"))
    args = parser.parse_args()

    df = pd.read_csv(os.path.join(args.train, "iris.csv"))
    X = df.drop("target", axis=1)
    y = df["target"]
    Xtr, Xte, ytr, yte = train_test_split(X, y, test_size=0.3, random_state=42)

    model = RandomForestClassifier(
        n_estimators=args.n_estimators, max_depth=args.max_depth, random_state=42
    )
    model.fit(Xtr, ytr)
    preds = model.predict(Xte)

    # Anything printed lands in CloudWatch logs — and can be scraped as metrics
    print(f"accuracy={accuracy_score(yte, preds):.4f};")
    print(f"f1_macro={f1_score(yte, preds, average='macro'):.4f};")

    # Whatever you write to SM_MODEL_DIR gets tarred and uploaded to S3
    joblib.dump(model, os.path.join(args.model_dir, "model.joblib"))

# Used later by the SageMaker inference container to load the model
def model_fn(model_dir):
    return joblib.load(os.path.join(model_dir, "model.joblib"))
```

**The contract in that script (interview material):**
- `SM_CHANNEL_TRAIN` → where SageMaker mounted your S3 data inside the container (`/opt/ml/input/data/train`).
- `SM_MODEL_DIR` → `/opt/ml/model`; everything written there becomes `model.tar.gz` in S3.
- `print()` → CloudWatch. That's your observability into a job you can't SSH into.
- `model_fn` → the hook the serving container calls to load the model at endpoint startup.

**Step 3.2 — the launcher.** Create `sagemaker/run_training.py`:

```python
import sagemaker
from sagemaker.sklearn.estimator import SKLearn

ROLE = "arn:aws:iam::<ACCOUNT_ID>:role/SageMakerLabRole"   # <-- paste your ARN
BUCKET = "<YOUR_BUCKET>"                                    # <-- from Part 2

session = sagemaker.Session()

estimator = SKLearn(
    entry_point="train_sm.py",
    framework_version="1.2-1",          # sklearn container version in AWS ECR
    instance_type="ml.m5.large",
    instance_count=1,
    role=ROLE,
    hyperparameters={"n-estimators": 100, "max-depth": 5},
    base_job_name="iris-rf",
)

estimator.fit({"train": f"s3://{BUCKET}/iris/train/"})
print("Model artifact:", estimator.model_data)
```

Run it:
```bash
cd ~/mlops-lab/sagemaker
python run_training.py
```

**WHAT TO WATCH (5–6 minutes):** the console streams the job lifecycle — `Starting` (provisioning the instance) → `Downloading` (your data + script) → `Training` (your script's prints appear!) → `Uploading` → `Completed`. Meanwhile open the AWS console → SageMaker → Training jobs → your `iris-rf-...` job: instance type, input channels, hyperparameters, and the output S3 path are all recorded. VERIFY the artifact: `aws s3 ls s3://<bucket-from-output>/ --recursive | grep model.tar.gz`.

**BREAK IT #1 — fail a training job and diagnose it via CloudWatch (THE support skill).**
Edit `train_sm.py`: change `df.drop("target", axis=1)` to `df.drop("targett", axis=1)` (typo on purpose). Run `python run_training.py` again.
- The job fails after a few minutes with `AlgorithmError`.
- Now diagnose it the way you would a ticket: Console → SageMaker → Training jobs → the failed job → **"Failure reason"** shows the exception summary → **View logs** → CloudWatch log group `/aws/sagemaker/TrainingJobs` → full Python traceback: `KeyError: "['targett'] not found in axis"`.
- **Lesson:** you cannot SSH into a training container. CloudWatch logs + the DescribeTrainingJob failure reason ARE the debugging surface. "A data scientist says their training job keeps failing" → this exact navigation is the answer. Fix the typo, rerun, confirm `Completed`. (Cost of the lesson: ~2 cents.)

---

## Part 4 — Model Registry with Approval Gate (20 min)

Register the trained model into the **SageMaker Model Registry**. Append to a new file `sagemaker/register.py`:

```python
import sagemaker
from sagemaker.sklearn.estimator import SKLearn
from sagemaker.sklearn.model import SKLearnModel

ROLE = "arn:aws:iam::<ACCOUNT_ID>:role/SageMakerLabRole"
MODEL_DATA = "<paste estimator.model_data S3 URI from the training output>"

model = SKLearnModel(
    model_data=MODEL_DATA,
    role=ROLE,
    entry_point="train_sm.py",       # contains model_fn
    framework_version="1.2-1",
)

package = model.register(
    model_package_group_name="iris-classifier",
    content_types=["application/json"],
    response_types=["application/json"],
    inference_instances=["ml.m5.large"],
    approval_status="PendingManualApproval",
)
print("Registered:", package.model_package_arn)
```

```bash
python register.py
```

**Now go look at what you built:** Console → SageMaker → Model registry (under "Inference") → model group `iris-classifier` → **Version 1, status: PendingManualApproval**.

**Approve it — this click IS the governance gate:** open the version → Actions → **Update approval status → Approved**. In real pharma/GxP setups, this transition is wired to change control: deployment pipelines (EventBridge → Pipeline) only fire on `Approved`, the approver identity is recorded, and that record is audit evidence. You've now personally operated the mechanism behind "model promotion requires approval."

**Say-it-aloud sentence:** "SageMaker's registry organizes models into package groups with versions and an approval status; deployments key off Approved status, which is where change management plugs in."

---

## Part 5 — Deploy and Invoke (30 min)

### 5.1 Serverless endpoint (the cost-safe kind)
Create `sagemaker/deploy.py`:

```python
import sagemaker
from sagemaker.sklearn.model import SKLearnModel
from sagemaker.serverless import ServerlessInferenceConfig

ROLE = "arn:aws:iam::<ACCOUNT_ID>:role/SageMakerLabRole"
MODEL_DATA = "<same model.tar.gz S3 URI>"

model = SKLearnModel(
    model_data=MODEL_DATA,
    role=ROLE,
    entry_point="train_sm.py",
    framework_version="1.2-1",
)

predictor = model.deploy(
    serverless_inference_config=ServerlessInferenceConfig(
        memory_size_in_mb=2048,
        max_concurrency=5,
    ),
    endpoint_name="iris-serverless",
)
print("Endpoint:", predictor.endpoint_name)
```

```bash
python deploy.py     # takes ~3-5 minutes to reach InService
```

**Invoke it** — `sagemaker/invoke.py`:
```python
import time
from sagemaker.sklearn.model import SKLearnPredictor
from sagemaker.serializers import NumpySerializer
from sagemaker.deserializers import NumpyDeserializer

predictor = SKLearnPredictor(
    endpoint_name="iris-serverless",
    serializer=NumpySerializer(),
    deserializer=NumpyDeserializer(),
)

t0 = time.time()
print("prediction:", predictor.predict([[5.1, 3.5, 1.4, 0.2]]),
      f"latency={time.time()-t0:.2f}s")

t0 = time.time()
print("prediction:", predictor.predict([[6.7, 3.0, 5.2, 2.3]]),
      f"latency={time.time()-t0:.2f}s")
```
```bash
python invoke.py
```
**Expected:** predictions `[0]` then `[2]` — and **look at the two latencies.** The first invocation is slow (often several seconds), the second fast. That's the **cold start**: serverless scaled from zero, and the container had to start and *load the model* — the exact model-loading cost you engineered readiness probes around in the Kubernetes lab. One concept, three appearances: FastAPI startup, K8s initialDelaySeconds, serverless cold start. Trade-off to articulate: serverless = zero idle cost but cold-start latency; real-time instances = constant cost but consistent latency; pick per workload.

**BREAK IT #2 — malformed invocation, and where to look:**
```bash
aws sagemaker-runtime invoke-endpoint \
  --endpoint-name iris-serverless \
  --content-type application/json \
  --body '{"wrong": "shape"}' \
  --cli-binary-format raw-in-base64-out /tmp/out.json ; cat /tmp/out.json
```
→ you get a `ModelError` (HTTP 4xx/5xx from the model container). Diagnose: Console → CloudWatch → Log groups → `/aws/sagemaker/Endpoints/iris-serverless` → the container's traceback is there. **Lesson:** endpoint invocation errors are debugged in the *endpoint's* CloudWatch logs; SageMaker also emits per-endpoint metrics (Invocations, ModelLatency, 4XX/5XX errors) to CloudWatch Metrics — that's the infra half of model monitoring, and it's the layer a platform support engineer owns.

### 5.2 Batch transform (10 min, optional but cheap ammunition)
Batch inference — no endpoint at all:
```bash
# Make an unlabeled batch input file
python - << 'PYEOF'
import pandas as pd
from sklearn.datasets import load_iris
X, _ = load_iris(return_X_y=True, as_frame=True)
X.columns = ["sepal_length","sepal_width","petal_length","petal_width"]
X.sample(20, random_state=1).to_csv("batch.csv", index=False, header=False)
PYEOF
aws s3 cp batch.csv s3://$BUCKET/iris/batch/batch.csv
```
`sagemaker/batch.py`:
```python
from sagemaker.sklearn.model import SKLearnModel

ROLE = "arn:aws:iam::<ACCOUNT_ID>:role/SageMakerLabRole"
MODEL_DATA = "<same model.tar.gz URI>"
BUCKET = "<your bucket>"

model = SKLearnModel(model_data=MODEL_DATA, role=ROLE,
                     entry_point="train_sm.py", framework_version="1.2-1")

transformer = model.transformer(
    instance_count=1,
    instance_type="ml.m5.large",
    output_path=f"s3://{BUCKET}/iris/batch-output/",
    strategy="MultiRecord",
    assemble_with="Line",
    accept="text/csv",
)
transformer.transform(
    data=f"s3://{BUCKET}/iris/batch/batch.csv",
    content_type="text/csv",
    split_type="Line",
)
transformer.wait()
```
```bash
python batch.py
aws s3 cp s3://$BUCKET/iris/batch-output/batch.csv.out -   # predictions printed
```
**Lesson:** batch transform spins up instances, scores the whole file, writes results to S3, terminates. No always-on infra, no endpoint. This is your concrete backing for the "real-time vs batch inference" question — you've now run both against the same model artifact.

---

## Part 6 — TEARDOWN (mandatory, 10 min)

```python
# sagemaker/teardown.py
import boto3
sm = boto3.client("sagemaker")

sm.delete_endpoint(EndpointName="iris-serverless")
sm.delete_endpoint_config(EndpointConfigName="iris-serverless")
# delete any Models created along the way:
for m in sm.list_models()["Models"]:
    sm.delete_model(ModelName=m["ModelName"])
print("done")
```
```bash
python teardown.py
```

**Then VERIFY in the console — trust nothing you didn't check:**
1. SageMaker → **Endpoints: must be EMPTY.** (The only resource here that bills while idle... except serverless doesn't, but verify anyway — the habit matters more than this instance.)
2. Endpoint configurations: empty. Models: empty.
3. Training jobs / transform jobs: completed jobs cost nothing; leave the history — it's your record.
4. Model registry: leave it; registry entries are metadata, effectively free, and nice to screenshot.
5. Optional full cleanup: `aws s3 rb s3://$BUCKET --force` and delete the IAM role.

Set a **billing alarm** if you don't have one (Console → Billing → Budgets → $5 budget) — 2 minutes, permanent peace of mind.

---

## Part 7 — Debrief: your new true sentences

1. "I've run SageMaker training jobs in script mode — the SM_CHANNEL/SM_MODEL_DIR contract, framework containers from ECR, ephemeral instances, artifacts to S3."
2. "I've debugged a failed training job through the failure reason and the CloudWatch training-job log group — that's the support workflow, since you can't SSH into managed training."
3. "I've registered models in the SageMaker Model Registry with a PendingManualApproval → Approved gate — the hook for change control."
4. "I've deployed a serverless inference endpoint, measured the cold start, and invoked it; I've also debugged a ModelError through the endpoint's CloudWatch logs and know the endpoint metrics (ModelLatency, invocation errors)."
5. "I've run batch transform against the same artifact — so I can speak to real-time vs batch trade-offs from having done both."
6. "And I understand the cost model: which resources are ephemeral, which bill while idle, and why real-time endpoints are the thing you never leave running."

Plus the connective sentence for THIS job: "Domino can publish models to SageMaker endpoints, so a pharma platform team realistically runs both — Domino for the governed workbench, SageMaker for AWS-native serving. I've now touched both sides of that integration."
