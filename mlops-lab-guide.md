# MLOps Mini-Project Lab — Build, Break, and Fix the Full Model Lifecycle
### Train → Track → Register → Serve → Containerize → Deploy on Kubernetes → Roll out v2 → Roll back
**Time: ~4 hours. Everything runs locally. No cloud cost.**

What you'll have at the end, truthfully: *"I've tracked experiments in MLflow, compared runs, registered and promoted model versions, served a model as a REST endpoint, containerized it, deployed it on Kubernetes with health probes, rolled out a new model version, and rolled it back."*

---

## Part 0 — Environment Setup (15 min)

**Prerequisites:** Python 3.10+, Docker, and minikube (or kind). You have all three skills; install minikube if it's not on this machine: https://minikube.sigs.k8s.io/docs/start/

```bash
mkdir -p ~/mlops-lab && cd ~/mlops-lab
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install mlflow scikit-learn pandas
```

**VERIFY:**
```bash
mlflow --version        # expect 2.x or 3.x
python -c "import sklearn; print(sklearn.__version__)"
```

**WHY a venv:** the exact library versions in this venv become part of your model's reproducibility story. MLflow will record them automatically when you log a model. This is the same problem Domino solves with versioned compute environments — hold that thought, it comes back in Part 5.

---

## Part 1 — Start a Real Tracking Server (10 min)

You could let MLflow write to a local `./mlruns` folder, but we'll run it the way it's deployed in companies: a **server process + database backend + artifact store**. That architecture is itself an interview answer.

```bash
cd ~/mlops-lab
mlflow server \
  --backend-store-uri sqlite:///mlflow.db \
  --default-artifact-root ./mlartifacts \
  --host 127.0.0.1 --port 5000
```

Leave this terminal running. Open http://127.0.0.1:5000 in a browser — you should see the MLflow UI with an empty "Default" experiment.

**WHY each flag:**
- `--backend-store-uri sqlite:///mlflow.db` → **metadata** (params, metrics, run info, registry entries) goes into a SQL database. In production this is Postgres/RDS. The Model Registry requires a database backend — that's why we're not using the plain file store.
- `--default-artifact-root ./mlartifacts` → **artifacts** (the model files themselves) go to a file/object store. In production this is S3.
- **The interview point:** "An MLflow tracking server is just a stateless app + a SQL DB for metadata + object storage for artifacts. Operating it is ordinary platform work — backups, storage lifecycle, auth in front of it." You've now personally run that architecture.

**WHAT TO CHECK:** `ls ~/mlops-lab` → you should see `mlflow.db` appear after first use, and `mlartifacts/` once you log a model.

---

## Part 2 — Train a Model with Experiment Tracking (30 min)

Open a **second terminal** (keep the server running in the first):
```bash
cd ~/mlops-lab && source venv/bin/activate
```

Create `train.py`:

```python
import sys
import mlflow
import mlflow.sklearn
from mlflow.models import infer_signature
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, f1_score

# Hyperparameters from CLI args (so we can run many experiments easily)
n_estimators = int(sys.argv[1]) if len(sys.argv) > 1 else 100
max_depth = int(sys.argv[2]) if len(sys.argv) > 2 else 5

# Point the client at our tracking server
mlflow.set_tracking_uri("http://127.0.0.1:5000")
mlflow.set_experiment("iris-classifier")

# Load data: 150 flowers, 4 numeric features, 3 species (classes 0/1/2)
X, y = load_iris(return_X_y=True, as_frame=True)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.3, random_state=42
)

with mlflow.start_run():
    # 1. Log the INPUTS of the experiment
    mlflow.log_param("n_estimators", n_estimators)
    mlflow.log_param("max_depth", max_depth)

    # 2. Train
    model = RandomForestClassifier(
        n_estimators=n_estimators, max_depth=max_depth, random_state=42
    )
    model.fit(X_train, y_train)

    # 3. Evaluate and log the OUTPUTS
    preds = model.predict(X_test)
    acc = accuracy_score(y_test, preds)
    f1 = f1_score(y_test, preds, average="macro")
    mlflow.log_metric("accuracy", acc)
    mlflow.log_metric("f1_macro", f1)

    # 4. Log the model ARTIFACT with a signature (input/output schema)
    signature = infer_signature(X_train, preds)
    mlflow.sklearn.log_model(
        model, "model", signature=signature, input_example=X_train.iloc[:2]
    )

    print(f"n_estimators={n_estimators} max_depth={max_depth} "
          f"accuracy={acc:.4f} f1={f1:.4f}")
```

Now run **four experiments** with different hyperparameters:
```bash
python train.py 10 2
python train.py 50 3
python train.py 100 5
python train.py 200 10
```

**WHAT TO CHECK:**
1. Refresh the MLflow UI → experiment "iris-classifier" → 4 runs listed.
2. Click any run → see params, metrics, and under **Artifacts** the logged `model/` folder containing:
   - `MLmodel` — metadata file: how to load this model, its signature (schema), which "flavor" (sklearn)
   - `model.pkl` — the serialized model itself
   - `requirements.txt` / `conda.yaml` — the **environment captured automatically**. Open `requirements.txt` — your exact sklearn version is pinned. THIS is reproducibility being solved in front of you.
3. Select all 4 runs → click **Compare** → see the params-vs-metrics table. This screen is the answer to "how do you know which of 200 experiments was best?"

**WHY the signature matters:** `infer_signature` records the expected input schema (4 named float columns). MLflow serving will *enforce* it — you'll break this deliberately in Part 4.

**BREAK IT #1 — kill the tracking server** (Ctrl+C in terminal 1), then run `python train.py 100 5`:
- You get a connection error. **Lesson:** the tracking server is a hard dependency of every training job in the company. When a data scientist raises "my training runs are failing," tracking-server health is on your checklist. Restart the server before continuing.

---

## Part 3 — Model Registry: Register and Promote (20 min)

Right now you have 4 runs. The best one (probably `200/10` or `100/5` — check f1_macro) needs to become an **official, versioned, deployable model**.

**Via the UI (do it this way first — it's what approvers see):**
1. Open the best run → Artifacts → click the `model` folder → click **Register Model**.
2. Create new model: name it `iris-classifier`. → It becomes **Version 1**.
3. Go to the **Models** tab (top nav) → `iris-classifier` → you see Version 1 with lineage: click through — it links back to the exact run, params, metrics, and code environment that produced it. **That link is model lineage — audit-trail material in GxP.**
4. Give it an **alias**: on the version page, add alias `champion`. (Aliases like `champion`/`challenger` are the modern MLflow way; the older "stages" — Staging/Production/Archived — still exist but are deprecated. Know both words.)

**Register a second version via code** — cheaper hyperparameters, pretend it's a candidate:
```bash
python - << 'EOF'
import mlflow
mlflow.set_tracking_uri("http://127.0.0.1:5000")
mlflow.set_experiment("iris-classifier")
# Re-log the small model and register it in one step
import sys; sys.argv = ["", "10", "2"]
EOF
```
Simpler: just edit nothing and run a variant of the training that registers directly — add this run:
```bash
python - << 'EOF'
import mlflow, mlflow.sklearn
from mlflow.models import infer_signature
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, f1_score

mlflow.set_tracking_uri("http://127.0.0.1:5000")
mlflow.set_experiment("iris-classifier")
X, y = load_iris(return_X_y=True, as_frame=True)
Xtr, Xte, ytr, yte = train_test_split(X, y, test_size=0.3, random_state=42)
with mlflow.start_run():
    m = RandomForestClassifier(n_estimators=10, max_depth=2, random_state=42)
    m.fit(Xtr, ytr)
    p = m.predict(Xte)
    mlflow.log_param("n_estimators", 10); mlflow.log_param("max_depth", 2)
    mlflow.log_metric("accuracy", accuracy_score(yte, p))
    mlflow.log_metric("f1_macro", f1_score(yte, p, average="macro"))
    mlflow.sklearn.log_model(m, "model",
        signature=infer_signature(Xtr, p),
        registered_model_name="iris-classifier")   # <-- registers as Version 2
EOF
```

**WHAT TO CHECK:** Models tab → `iris-classifier` now shows **Version 1 (alias: champion)** and **Version 2**. Two versions, one name, full lineage each. That's a model registry — "ECR for models" is now something you've operated, not just read about.

**WHY this matters in interviews:** deployment tooling asks the registry for "iris-classifier @champion" instead of a hardcoded file path. Promotion = moving the alias. Rollback = moving it back. Approval gates sit on that transition. You'll use exactly this URI in the next part.

---

## Part 4 — Serve the Model as a REST Endpoint (25 min)

MLflow can wrap any registered model in a REST server with one command. In terminal 2:

```bash
export MLFLOW_TRACKING_URI=http://127.0.0.1:5000
mlflow models serve -m "models:/iris-classifier/1" -p 5001 --env-manager local
```

**WHY each piece:**
- `models:/iris-classifier/1` → a **registry URI**: model name + version. You could also use `models:/iris-classifier@champion` — deploy-by-alias. Nobody hardcodes file paths.
- `-p 5001` → the tracking server already owns 5000. (Port collision here is a classic first stumble — now it won't be yours.)
- `--env-manager local` → serve using your current venv instead of building a fresh isolated env. Fine for a lab; in production the env is baked into a container (Part 5).

**VERIFY — health probe first, like an SRE:**
```bash
curl -s http://127.0.0.1:5001/ping        # expect empty 200 OK
```

**Get a prediction:**
```bash
curl -s -X POST http://127.0.0.1:5001/invocations \
  -H "Content-Type: application/json" \
  -d '{
    "dataframe_split": {
      "columns": ["sepal length (cm)", "sepal width (cm)", "petal length (cm)", "petal width (cm)"],
      "data": [[5.1, 3.5, 1.4, 0.2], [6.7, 3.0, 5.2, 2.3]]
    }
  }'
```
**Expected:** `{"predictions": [0, 2]}` — species 0 (setosa) and species 2 (virginica). You are now doing **real-time (online) inference**: a model behind an HTTP endpoint, request in, prediction out.

**BREAK IT #2 — violate the schema.** Remove one column:
```bash
curl -s -X POST http://127.0.0.1:5001/invocations \
  -H "Content-Type: application/json" \
  -d '{"dataframe_split": {"columns": ["sepal length (cm)","sepal width (cm)","petal length (cm)"], "data": [[5.1,3.5,1.4]]}}'
```
→ 400 error naming the missing column. **Lesson:** the signature you logged in Part 2 is now enforcing an input contract at the API boundary. Without signatures, this would be a silent garbage-in/garbage-out prediction — the worst kind of ML failure. This is your concrete story for "training/serving skew" and "why do model signatures matter."

**BREAK IT #3 — request a version that doesn't exist.** Ctrl+C the server, then:
```bash
mlflow models serve -m "models:/iris-classifier/99" -p 5001 --env-manager local
```
→ `RESOURCE_DOES_NOT_EXIST`. **Lesson:** deployments reference registry entries; a deleted/renamed model version breaks deploys. When someone reports "model deployment failing," registry state is on the checklist. Restart with version 1 if you want, or move on — the endpoint's job is done.

---
## Part 5 — Containerize It Yourself (45 min)

`mlflow models build-docker` can do this in one command, but you'll build it by hand — because then you *own* every layer, and because a hand-built FastAPI server is exactly what Domino Model APIs / KServe generate for you under the hood.

**Step 5.1 — Download the model artifact out of the registry:**
```bash
mkdir -p ~/mlops-lab/serving && cd ~/mlops-lab/serving
python - << 'PYEOF'
import mlflow
mlflow.set_tracking_uri("http://127.0.0.1:5000")
path = mlflow.artifacts.download_artifacts(
    artifact_uri="models:/iris-classifier/1", dst_path="model"
)
print("Downloaded to:", path)
PYEOF
ls model            # VERIFY: you must see the MLmodel file here
```
If `MLmodel` landed one folder deeper (e.g. `model/model/MLmodel`), flatten it: `mv model/model/* model/ && rmdir model/model`. **Always verify the artifact layout before baking an image** — mismatched paths are a top cause of CrashLoopBackOff in model containers.

**Step 5.2 — the serving app.** Create `serving/app.py`:
```python
import mlflow.pyfunc
import pandas as pd
from fastapi import FastAPI
from pydantic import BaseModel

# Model is baked into the image at ./model (loaded ONCE at startup — see WHY)
model = mlflow.pyfunc.load_model("model")

app = FastAPI()

COLUMNS = ["sepal length (cm)", "sepal width (cm)",
           "petal length (cm)", "petal width (cm)"]

class IrisRequest(BaseModel):
    sepal_length: float
    sepal_width: float
    petal_length: float
    petal_width: float

@app.get("/health")
def health():
    return {"status": "ok"}

@app.post("/predict")
def predict(req: IrisRequest):
    df = pd.DataFrame(
        [[req.sepal_length, req.sepal_width, req.petal_length, req.petal_width]],
        columns=COLUMNS,
    )
    pred = model.predict(df)
    return {"prediction": int(pred[0]), "model": "iris-classifier", "version": 1}
```

**WHY load the model at import time, not per-request:** model loading is the expensive step (for real models: seconds to minutes, gigabytes of memory). Load once, serve many. **This is exactly why model pods start slowly and why readiness probes need generous initialDelaySeconds** — you'll prove it on Kubernetes in Part 6.

**Step 5.3 — requirements and Dockerfile.** Create `serving/requirements.txt`:
```
mlflow-skinny
scikit-learn
pandas
fastapi
uvicorn
```
(`mlflow-skinny` = the client/loading library without the server bloat — smaller image. Open `model/requirements.txt` and pin your scikit-learn version to match it if you want to be strict — that *is* the reproducibility discipline in action.)

Create `serving/Dockerfile`:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY model/ ./model/
COPY app.py .
EXPOSE 8080
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8080"]
```
**WHY model-in-image:** version 1 of the model is immutably baked into image tag v1 → **the image tag IS the model version** → model rollout/rollback becomes ordinary image rollout/rollback (Part 6 exploits this). The alternative — pulling the model from S3/registry at pod startup via an initContainer — is what KServe/SageMaker do. Know both and the trade-off: immutable + reproducible vs. smaller images + swap models without rebuilds.

**Step 5.4 — build and run:**
```bash
cd ~/mlops-lab/serving
docker build -t iris-model:v1 .
docker run -d --name iris -p 8080:8080 iris-model:v1
sleep 5
curl -s http://127.0.0.1:8080/health
curl -s -X POST http://127.0.0.1:8080/predict \
  -H "Content-Type: application/json" \
  -d '{"sepal_length": 5.1, "sepal_width": 3.5, "petal_length": 1.4, "petal_width": 0.2}'
```
**Expected:** `{"prediction": 0, "model": "iris-classifier", "version": 1}`

**BREAK IT #4 — starve it of memory:**
```bash
docker rm -f iris
docker run -d --name iris-oom -m 60m -p 8080:8080 iris-model:v1
sleep 8
docker ps -a --filter name=iris-oom     # STATUS: Exited (137)
docker inspect iris-oom --format '{{.State.OOMKilled}}'   # true
docker rm -f iris-oom
```
**Lesson:** exit code **137 = SIGKILL from the OOM killer**. Model containers OOM *at startup*, while loading the model, before serving a single request — on Kubernetes this appears as CrashLoopBackOff with reason OOMKilled, and the fix is sizing memory to model size + runtime overhead. You just manufactured the single most common model-serving incident. (If 60m doesn't kill it on your machine, try 40m.)

---

## Part 6 — Deploy on Kubernetes (60 min)

```bash
minikube start
# Make your local image visible inside the cluster:
minikube image load iris-model:v1
```
**WHY `image load`:** the cluster's container runtime has its own image cache and cannot see your host's Docker images. Forgetting this = `ErrImageNeverPull`/`ImagePullBackOff` — you'll trigger it deliberately in a minute.

Create `k8s/deploy.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iris-model
  labels: {app: iris-model}
spec:
  replicas: 2
  selector:
    matchLabels: {app: iris-model}
  template:
    metadata:
      labels: {app: iris-model}
    spec:
      containers:
      - name: iris-model
        image: iris-model:v1
        imagePullPolicy: Never          # local image; never ask a registry
        ports:
        - containerPort: 8080
        resources:
          requests: {cpu: 100m, memory: 200Mi}
          limits:   {cpu: 500m, memory: 512Mi}
        readinessProbe:
          httpGet: {path: /health, port: 8080}
          initialDelaySeconds: 5        # model load time buffer
          periodSeconds: 5
        livenessProbe:
          httpGet: {path: /health, port: 8080}
          initialDelaySeconds: 10
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: iris-model
spec:
  type: NodePort
  selector: {app: iris-model}
  ports:
  - port: 80
    targetPort: 8080
```

Deploy and test:
```bash
kubectl apply -f k8s/deploy.yaml
kubectl get pods -w                     # wait for 2/2 Running and READY 1/1
minikube service iris-model --url      # gives you a URL; keep it running or copy the URL
curl -s -X POST <URL>/predict -H "Content-Type: application/json" \
  -d '{"sepal_length": 6.7, "sepal_width": 3.0, "petal_length": 5.2, "petal_width": 2.3}'
```
**Expected:** `{"prediction": 2, ...}`. Your model is now running as a replicated, health-checked, resource-bounded Kubernetes service. **This is what a Domino Model API creates for you automatically** — you've now built the underlying thing by hand.

**WHY readiness vs liveness for models (interview gold):**
- **Readiness** gates traffic. A model pod that is up but still loading the model must NOT receive requests → readiness probe + generous initial delay. For huge models, a `startupProbe` is the proper tool.
- **Liveness** restarts hung processes. Too-aggressive liveness on a slow-loading model = restart loop of a healthy-but-loading pod — a real and classic production incident.

### BREAK IT #5 — the image that isn't there
```bash
kubectl set image deployment/iris-model iris-model=iris-model:v9
kubectl get pods                        # new pods stuck: ErrImageNeverPull / ImagePullBackOff
kubectl describe pod <new-pod-name>    # Events section names the exact cause
```
**Lesson 1:** `kubectl describe` → Events is always your first move.
**Lesson 2 — look at what did NOT break:** the old v1 pods are still Running and serving. The rolling update strategy refuses to kill healthy pods before new ones are Ready. **Deployments protect model availability during bad rollouts by design.** Fix it:
```bash
kubectl rollout undo deployment/iris-model
kubectl rollout status deployment/iris-model
```

### BREAK IT #6 — the pod that's Running but never Ready
Edit the readinessProbe path in `k8s/deploy.yaml` to `/healthz` (wrong on purpose), then:
```bash
kubectl apply -f k8s/deploy.yaml
kubectl get pods                        # new pod: Running but READY 0/1, forever
kubectl get endpoints iris-model       # the not-ready pod is ABSENT from endpoints
kubectl describe pod <pod>             # Events: "Readiness probe failed: 404"
```
**Lesson:** "pod is Running but the service returns errors / has no backends" is one of the most common tickets in existence. Running ≠ Ready; Services only route to Ready pods. Also note the rollout is stuck half-done — again protecting the old version. Fix the path back to `/health`, re-apply.

### BREAK IT #7 — OOMKilled, Kubernetes edition
Set `limits.memory: 64Mi`, re-apply, then:
```bash
kubectl get pods                        # CrashLoopBackOff
kubectl describe pod <pod> | grep -A3 "Last State"   # Reason: OOMKilled, Exit Code: 137
```
Same 137 you saw in Docker — now with Kubernetes dressing. Fix back to 512Mi, re-apply. **RCA + permanent fix language for the interview:** "root cause: memory limit below model working set; permanent fix: right-size limits from measured usage and alert on container memory saturation, not just node memory."

---

## Part 7 — The Payoff: Roll Out Model v2, Then Roll It Back (30 min)

This is the MLOps money-move: shipping a NEW MODEL VERSION with zero downtime, then reverting it.

**7.1 — build the v2 image.** Download registry Version 2 and rebuild:
```bash
cd ~/mlops-lab/serving
rm -rf model
python - << 'PYEOF'
import mlflow
mlflow.set_tracking_uri("http://127.0.0.1:5000")
mlflow.artifacts.download_artifacts(
    artifact_uri="models:/iris-classifier/2", dst_path="model")
PYEOF
ls model                                # verify MLmodel present (flatten if nested)
sed -i 's/"version": 1/"version": 2/' app.py
docker build -t iris-model:v2 .
minikube image load iris-model:v2
```

**7.2 — rolling update:**
```bash
kubectl set image deployment/iris-model iris-model=iris-model:v2
kubectl rollout status deployment/iris-model
# watch in another terminal: kubectl get pods -w
curl -s -X POST <URL>/predict -H "Content-Type: application/json" \
  -d '{"sepal_length": 5.1, "sepal_width": 3.5, "petal_length": 1.4, "petal_width": 0.2}'
# response now says "version": 2
```
Watch the choreography: new pod created → readiness probe passes → old pod terminated → repeat. **Zero downtime model upgrade.** With two Deployments and weighted routing this becomes canary; with a mirror of traffic it becomes shadow — same primitives.

**7.3 — the model is "worse"; roll back:**
Pretend monitoring shows prediction drift after v2 went live (v2 was the weak 10-tree model, remember):
```bash
kubectl rollout history deployment/iris-model
kubectl rollout undo deployment/iris-model
kubectl rollout status deployment/iris-model
curl ...   # "version": 1 again
```
**Say this in the interview:** "Because the image tag is the model version, model rollback is a one-command `rollout undo` — and the registry tells me exactly which lineage each version has. In a regulated environment the promotion and rollback are change-controlled, but mechanically it's this."

---

## Part 8 — Cleanup & Debrief (10 min)

```bash
kubectl delete -f k8s/deploy.yaml
minikube stop                 # or: minikube delete
docker rm -f iris 2>/dev/null; docker rmi iris-model:v1 iris-model:v2
# keep ~/mlops-lab — mlflow.db + mlartifacts are your experiment history; restart the UI anytime
```

**Map what you built to the interview vocabulary — say each sentence out loud:**
1. "I ran an MLflow tracking server — app + SQLite metadata store + artifact directory — the same architecture as production, just with Postgres and S3 there." (Part 1)
2. "I ran multiple experiments with different hyperparameters and compared them in the tracking UI; every run captured params, metrics, the model artifact, and the pinned environment." (Part 2)
3. "I registered the best run in the Model Registry, giving me versions, aliases, and lineage back to the exact run." (Part 3)
4. "I served a registry version as a REST endpoint, and saw the model signature enforce the input schema at the API boundary — that's the guard against training/serving skew." (Part 4)
5. "I containerized the model behind FastAPI, baking the model version into the image tag." (Part 5)
6. "I deployed it on Kubernetes with readiness/liveness probes and resource limits, and debugged ImagePull, not-Ready, and OOMKilled failure modes." (Part 6)
7. "I rolled out model v2 with a zero-downtime rolling update and rolled it back when it underperformed — the same primitives that give you canary and shadow deployments." (Part 7)

That's the ML deployment lifecycle end to end, and every word of it is now true.

**If something in the lab fails and you can't fix it in 15 minutes** — that's not lost time, that's a bonus troubleshooting rep. Diagnose it the way you would a ticket: what changed, describe/logs/events, layer by layer. Then bring me the symptom and I'll debug it with you.
