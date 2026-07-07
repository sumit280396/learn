# MLOps Interview Mastery Guide — EPAM Round 2
### For a Platform Engineer Supporting Domino Data Lab on EKS (4-Day Sprint)

---

## Part 0 — The Mental Model That Makes Everything Click

**MLOps = DevOps + two new artifact types: data and models.**

In DevOps, the only artifact that changes is **code**. You version it (Git), build it (CI), package it (Docker), deploy it (CD), and monitor it (Prometheus/Grafana).

In ML, THREE things change independently:
1. **Code** (training scripts, feature engineering, serving code)
2. **Data** (the training data changes, upstream schemas change, real-world data drifts)
3. **Model** (a binary artifact produced by code + data + hyperparameters + randomness)

Everything in MLOps exists to bring the same discipline you already apply to code to these other two artifacts:

| DevOps concept you know | MLOps equivalent | Tool examples |
|---|---|---|
| Git for code | Data versioning + experiment tracking | DVC, MLflow Tracking |
| Artifact repo (ECR, Nexus) | **Model Registry** | MLflow Registry, SageMaker Model Registry |
| CI pipeline (build/test) | **Training pipeline** | Kubeflow Pipelines, SageMaker Pipelines, Domino Flows |
| CD (deploy app) | **Model serving / deployment** | SageMaker Endpoints, KServe, Domino Model APIs |
| Monitoring (CPU, latency, errors) | Model monitoring (**drift**, accuracy decay) + the usual infra metrics | SageMaker Model Monitor, Evidently, Prometheus |
| Config drift | **Data drift / concept drift** | statistical tests on live traffic |
| Rollback a bad release | Roll back to previous model version in registry | registry stage transitions |
| "Works on my machine" | **Reproducibility problem** (code+data+env+seed) | Docker envs, pinned datasets, logged params |

One extra letter to remember: **CI/CD/CT** — MLOps adds **Continuous Training**: pipelines that retrain models automatically when data drifts or on a schedule.

**Your positioning sentence (memorize it):**
> "I come at MLOps from the platform side. I've supported Domino Data Lab on EKS — which is an enterprise MLOps platform — handling workspace provisioning, job orchestration, compute environments, and model API deployments. So I understand the ML lifecycle from the infrastructure that runs it, and I've been building depth on the surrounding toolchain: SageMaker, MLflow, and model monitoring."

---

## Part 1 — The ML Lifecycle (the backbone of every MLOps interview)

Learn this as a story you can narrate in 90 seconds. Every question you get will live at one of these stages.

```
 Data Ingestion → Data Prep/Feature Engineering → Experimentation →
 Training → Evaluation → Model Registry → Deployment → Monitoring →
 (drift detected) → Retraining → back to Training  ... a LOOP, not a line
```

**1. Data ingestion & preparation.** Raw data from databases, S3, APIs. Cleaned, transformed, features computed. Tools: Spark, pandas, AWS Glue (you know Glue!). Key concept: **feature engineering** = turning raw data into model inputs (e.g., "days since last purchase" from a timestamp column).

**2. Experimentation.** Data scientists try many combinations of algorithms, features, and **hyperparameters** (settings you choose before training — learning rate, tree depth, number of layers). This happens in notebooks/workspaces — exactly what Domino provisions. The problem: after 200 experiments, which one was best and can you reproduce it? → **experiment tracking**.

**3. Training.** Running the training script at scale — often on GPUs, often distributed. From a platform view: it's a batch job with heavy resource requirements. Domino Jobs, SageMaker Training Jobs, or Kubernetes Jobs on GPU node pools.

**4. Evaluation.** Test the model on held-out data. Metrics: accuracy, precision, recall, F1, AUC (know these names; definitions below). The model must beat the current production model to be promoted.

**5. Model Registry.** Central versioned store of approved models with metadata + lineage (which code, data, params produced it). Models move through stages: `None → Staging → Production → Archived`. This is your change-management gate — huge in GxP environments.

**6. Deployment / Serving.** Getting predictions out of the model. Three modes:
   - **Real-time (online) inference**: model behind a REST/gRPC endpoint, low latency, always running. (Domino Model APIs, SageMaker real-time endpoints, KServe.)
   - **Batch inference**: score a big dataset on a schedule, write results to a table/S3. Cheaper, no always-on infra. (SageMaker Batch Transform.)
   - **Streaming**: score events from Kafka/Kinesis as they arrive.

**7. Monitoring.** Two layers:
   - **Infra/ops layer** (you already own this): latency, throughput, error rate, saturation — Prometheus/Grafana, CloudWatch.
   - **ML layer** (new): data drift, prediction drift, model performance decay. Details in Part 4.

**8. Retraining.** Triggered by schedule, drift alerts, or new labeled data. Automating this loop is **Continuous Training** — the pinnacle of MLOps maturity.

**Maturity levels (Google's framing — interviewers love this):**
- **Level 0**: manual — data scientist trains in a notebook, hands a model file to engineering. Slow, unreproducible.
- **Level 1**: automated training pipeline — the whole train/evaluate/register flow is a pipeline that can be re-run on new data.
- **Level 2**: full CI/CD/CT — code changes trigger pipeline tests and deployments; drift triggers retraining; everything is automated and monitored.

---

## Part 2 — Core Concepts, Explained Properly

### 2.1 Experiment Tracking
**Problem:** hundreds of training runs, each with different code versions, data, hyperparameters, and results. Which produced the best model, and can we reproduce it?
**Solution:** log every run's parameters, metrics, artifacts, and environment to a central server.
**Tool to know: MLflow Tracking.** A data scientist adds a few lines:
```python
import mlflow
with mlflow.start_run():
    mlflow.log_param("learning_rate", 0.01)
    mlflow.log_param("max_depth", 6)
    mlflow.log_metric("auc", 0.91)
    mlflow.sklearn.log_model(model, "model")
```
Runs appear in a UI where you compare experiments side by side. Domino has built-in experiment tracking (it embeds MLflow), and SageMaker has SageMaker Experiments.
**Platform angle:** an MLflow tracking server is just an app with a backend DB (Postgres) and an artifact store (S3) — you'd deploy and operate it like any service.

### 2.2 Model Registry
The "ECR for models." Each registered model has **versions**; each version has a **stage** (Staging/Production/Archived), lineage back to the run that created it, and approval metadata.
Promotion = a stage transition, which can require approval — this is where **change/release management and GxP** plug in. Deployment tooling pulls "the current Production version of model X" from the registry instead of a hardcoded file path.
Tools: MLflow Model Registry, SageMaker Model Registry (model package groups + approval status), Azure ML Model Registry, Domino's model catalog.

### 2.3 Feature Store
**Problem 1 — reuse:** ten teams each computing "customer_lifetime_value" slightly differently.
**Problem 2 — training/serving skew:** features computed one way in the training pipeline (Spark batch) and a different way in the serving code (Python at request time). Model behaves differently in production than in testing.
**Solution:** a feature store — a central catalog of feature definitions with two access paths:
- **Offline store** (S3/Parquet/warehouse): large historical feature sets for training.
- **Online store** (DynamoDB/Redis): the latest feature values at millisecond latency for real-time inference.
Same feature definition feeds both → skew eliminated. Tools: SageMaker Feature Store, Feast (open source), Databricks/Tecton.

### 2.4 Data Versioning & Reproducibility
Reproducing a model requires the exact **code + data + environment + hyperparameters (+ random seed)**.
- Code → Git.
- Environment → Docker images (Domino "compute environments" are exactly this — you know this cold).
- Hyperparameters → experiment tracker.
- Data → **DVC** (Data Version Control): Git-like tool that stores hashes/pointers in Git while the actual large files live in S3. `dvc pull` fetches the exact dataset version for a given commit. Alternatives: S3 versioning, Delta Lake time travel, LakeFS.

### 2.5 Model Packaging & Serving Mechanics
A trained model is a serialized file: pickle/joblib (sklearn), SavedModel (TensorFlow), .pt (PyTorch), or the portable **ONNX** format. Serving = wrapping that file in a web service:
- Hand-rolled: Flask/FastAPI app in a container — `POST /predict` → load model, run `model.predict()`, return JSON.
- Dedicated servers: **TensorFlow Serving, TorchServe, NVIDIA Triton** (high-performance, multi-framework).
- Kubernetes-native: **KServe** (formerly KFServing) and **Seldon Core** — CRDs where you declare an `InferenceService` pointing at a model in S3, and the controller creates the deployment, autoscaling (including scale-to-zero), canary routing, and monitoring hooks. If you know how an Ingress controller or any operator works, you understand KServe: it's an operator for model servers.
- Managed: SageMaker endpoints, Azure ML endpoints, **Domino Model APIs** (Domino builds the image, deploys pods on EKS, exposes a versioned REST endpoint — you can describe this from real experience).

**GPU serving notes (platform gold):** GPUs on K8s need the NVIDIA device plugin; pods request `nvidia.com/gpu: 1`; GPUs are not shareable by default (fractional GPUs via MIG/time-slicing); GPU nodes are usually tainted so only ML workloads with tolerations land there; cluster-autoscaler scales expensive GPU node groups from zero.

### 2.6 Hyperparameter Tuning (know the term)
Automated search over hyperparameter combinations: grid search, random search, **Bayesian optimization**. Managed versions: SageMaker Automatic Model Tuning, Katib (Kubeflow). Platform impact: tuning jobs launch MANY parallel training jobs — quota, autoscaling, and cost implications.

### 2.7 Evaluation Metrics (30-second literacy)
- **Accuracy**: % of predictions correct. Misleading on imbalanced data (99% "no fraud" data → a model that always says "no fraud" is 99% accurate and useless).
- **Precision**: of the things flagged positive, how many really were? (Cost of false alarms.)
- **Recall**: of the real positives, how many did we catch? (Cost of misses.)
- **F1**: harmonic mean of precision and recall.
- **AUC-ROC**: how well the model separates classes across all thresholds; 0.5 = coin flip, 1.0 = perfect.
- Regression: **RMSE/MAE** — average size of prediction errors.
You don't need the math — you need to not blink when the interviewer says "the model's recall dropped."

---

## Part 3 — The Toolchain (what to say about each tool)

### 3.1 MLflow (learn this best — it's the lingua franca)
Four components:
1. **Tracking** — log/compare runs (params, metrics, artifacts). Server = app + Postgres + S3.
2. **Models** — a standard packaging format ("MLflow Model" directory with `MLmodel` metadata file, conda/requirements env spec, and the serialized model). Any MLflow model can be served with `mlflow models serve` or built into a Docker image.
3. **Model Registry** — versions, stages, approvals (see 2.2).
4. **Projects** — package training code + env so anyone can `mlflow run` it reproducibly.
Domino embeds MLflow for experiment tracking — say this; it connects your real experience to the standard tool.

### 3.2 Kubeflow (the Kubernetes-native ML platform)
An umbrella of ML components running as operators/CRDs on Kubernetes:
- **Kubeflow Pipelines (KFP)**: define ML workflows as DAGs in Python; each step runs as a container/pod. (Argo Workflows underneath.) Think "Jenkins pipeline where every stage is a pod and the artifacts are datasets/models."
- **Notebooks**: Jupyter servers as pods — conceptually identical to Domino workspaces.
- **Katib**: hyperparameter tuning.
- **KServe**: model serving (see 2.5).
- **Training operators**: `TFJob`, `PyTorchJob` CRDs for distributed training across pods.
Your line: "I haven't run Kubeflow in production, but architecturally it's CRDs and operators on Kubernetes — notebooks as pods, pipelines as Argo DAGs, KServe as a serving operator — so it maps directly onto what I operate daily. Domino solves the same problems as a commercial product."

### 3.3 Amazon SageMaker (in the JD — go deep)
Managed ML platform. Map each piece to what you know:
- **Studio / Notebook instances**: managed JupyterLab (≈ Domino workspaces).
- **Training Jobs**: you give it a container + data location in S3 + instance type; AWS spins up instances, trains, writes model.tar.gz to S3, tears down. Ephemeral batch compute — pay only during training. Bring-your-own-container via ECR.
- **Processing Jobs**: same pattern for data prep/evaluation steps.
- **Automatic Model Tuning**: managed hyperparameter search running many training jobs.
- **Model Registry**: model package groups + versions + approval status (PendingManualApproval → Approved) — gates deployment.
- **Endpoints (real-time)**: managed HTTPS endpoint; you pick instance type/count; supports auto-scaling, **multi-model endpoints** (many models on one endpoint to save cost), and **production variants** with traffic weights → canary/A-B testing built in.
- **Serverless Inference**: scale-to-zero endpoints for spiky traffic.
- **Async Inference**: queue-based, for large payloads/long-running inference.
- **Batch Transform**: batch scoring job — no persistent endpoint.
- **SageMaker Pipelines**: native CI/CT pipeline service (steps: processing → training → evaluation → conditional register → deploy).
- **Model Monitor**: captures endpoint request/response data to S3, compares live data statistics against a training-data baseline on a schedule, raises CloudWatch alarms on drift.
- **Feature Store**: offline (S3) + online (low-latency) stores.
Bonus link to your experience: **Domino can export/publish models to SageMaker endpoints** — you studied this integration.

### 3.4 Azure Machine Learning (in the JD — conversational level is enough)
Same concepts, Azure names: **Workspace** (top-level container) → **Compute Instances** (dev boxes) & **Compute Clusters** (autoscaling training clusters, can scale to 0) → **Jobs/Pipelines** (training workflows) → **Model Registry** → **Endpoints**: *managed online endpoints* (real-time, supports blue-green via multiple *deployments* under one endpoint with traffic split) and *batch endpoints*. Environments = Docker + conda specs. Integrates with AKS for K8s-based serving. Say: "I know the concepts and the AWS implementations deeply; Azure ML is the same architecture with different names, and I can come up to speed fast."

### 3.5 Domino Data Lab — reframed as YOUR MLOps credential
Practice saying Domino in MLOps vocabulary:
- Workspaces = managed experimentation environments (Jupyter/RStudio/VS Code pods on EKS).
- Compute Environments = reproducible, versioned Docker images → solves environment reproducibility.
- Jobs = training/batch execution with tracked code, data, results → reproducible training.
- Experiment tracking = embedded MLflow.
- Domino Flows = pipeline orchestration (DAGs of tasks).
- Model APIs = model serving with versioned REST endpoints on K8s.
- Datasets = versioned shared data → data reproducibility.
- SageMaker export = hybrid deployment.
- Everything auditable → why pharma/GxP shops buy it.
You supported an MLOps platform. Own it.

### 3.6 Orchestrators you may be asked to compare
- **Airflow**: general-purpose scheduler/orchestrator (Python DAGs), widely used for data + retraining pipelines.
- **Argo Workflows**: K8s-native DAGs (each step a pod) — underlies KFP.
- **Step Functions**: AWS serverless orchestration (you know this) — commonly orchestrates SageMaker steps.
- **Jenkins/GitLab CI**: still used for the CI half (tests, image builds) with an ML orchestrator for the training half.

### 3.7 Other names to recognize (one-liners)
**DVC** (data versioning) · **Evidently AI** (open-source drift monitoring) · **Great Expectations** (data quality tests in pipelines) · **Weights & Biases** (SaaS experiment tracking) · **BentoML** (model packaging/serving) · **Triton** (NVIDIA inference server) · **Feast** (open-source feature store) · **ONNX** (portable model format) · **LakeFS/Delta Lake** (data versioning at lake scale).

---

## Part 4 — Monitoring, Drift, and Retraining (very likely interview focus for a support role)

### 4.1 Why ML monitoring is different
A normal service fails loudly (5xx, latency). **A model fails silently**: the endpoint returns 200 OK with confident, wrong predictions. The world changed; the model didn't. That's why MLOps adds a monitoring layer on top of infra monitoring.

### 4.2 The vocabulary
- **Data drift (covariate shift)**: the distribution of *input features* changes vs. training data. Example: model trained on pre-2024 customer data; a new market launch shifts age/income distributions. Detected by comparing live feature statistics to a training baseline (statistical distance tests — KS test, PSI; you don't need the math, just the idea).
- **Concept drift**: the *relationship* between inputs and target changes. Example: fraud patterns evolve — the same transaction features now mean something different. Data can look identical while the model still decays.
- **Prediction drift**: the distribution of the model's *outputs* changes (e.g., fraud-flag rate jumps from 2% to 9%). Cheap early-warning signal because it needs no ground truth.
- **Ground truth lag**: true labels arrive late (you learn whether a loan defaulted months later), so real accuracy can't be measured in real time — hence proxy signals like drift.
- **Training/serving skew**: features computed differently in training vs. serving (see feature store).

### 4.3 What a monitoring stack looks like
1. Infra: Prometheus/Grafana or CloudWatch — latency p50/p95/p99, error rate, throughput, GPU/CPU/memory saturation. (Your home turf.)
2. Data capture: log inference requests + predictions (SageMaker Model Monitor data capture, or app-level logging to S3).
3. Drift jobs: scheduled jobs compare live distributions to the training baseline (Model Monitor schedules, or Evidently reports in a cron/Airflow job).
4. Alerting: CloudWatch alarms / Alertmanager → ticket or retraining trigger.
5. Performance tracking: once delayed ground truth arrives, compute real accuracy/precision/recall over time.

### 4.4 Retraining triggers & strategy
Scheduled (weekly/monthly) · drift-alert-driven · new-labeled-data-driven · performance-threshold-driven. Automated retraining pipeline: pull fresh data → validate data quality → train → evaluate against current prod model (champion/challenger) → register if better → (manual approval in GxP) → deploy.

### 4.5 Deployment/rollout patterns for models (learn all five)
- **Blue-green**: two environments; flip traffic; instant rollback.
- **Canary**: send 5–10% of traffic to the new model version, watch metrics, ramp up. (SageMaker production variants / KServe canary do this natively.)
- **Shadow (silent) deployment**: new model receives a copy of live traffic and logs predictions, but responses come from the old model. Zero user risk — compare offline. Very common for models.
- **A/B testing**: split traffic between model versions and compare *business* metrics statistically.
- **Champion/Challenger**: the standing pattern where the production model (champion) is continuously compared against candidates (challengers) — usually via shadow or offline evaluation.
- **Rollback**: registry stage transition back to the previous version + redeploy. Always know which model version is live (version pinning, model metadata in responses/logs).

---

## Part 5 — MLOps in Life Sciences / GxP (differentiator for THIS job)

Pharma regulated environments (GxP = Good Practice regulations: GLP/GCP/GMP; think 21 CFR Part 11) demand:
- **Validation & documentation**: systems/models used in regulated processes must be validated (IQ/OQ/PQ mindset); model changes go through change control, not casual redeploys.
- **Audit trails**: who trained what, on which data, who approved promotion, when deployed — the model registry + experiment tracking + Domino's audit logs ARE the audit trail.
- **Reproducibility**: must be able to reproduce a model/result years later → versioned data, code, environments (again: Domino compute environments).
- **Access control & data privacy**: patient/clinical data → RBAC, SSO (your Keycloak knowledge), least privilege, data residency.
- **Change/release management**: CAB approvals, release windows, documented rollback plans; ITSM tools (ServiceNow etc.) for incidents/changes — standard enterprise support motion you know from TCS/Accenture.
Interview line: "In a GxP context, the MLOps stack isn't just convenience — the registry, experiment tracking, and environment versioning are the compliance evidence. That's exactly why platforms like Domino are chosen in pharma."

---

## Part 6 — Interview Question Bank (with strong answers)

### Conceptual
**Q1. What is MLOps and how is it different from DevOps?**
"MLOps applies DevOps discipline to machine learning, but ML has three changing artifacts instead of one — code, data, and models. So on top of CI/CD you add data versioning, experiment tracking, a model registry as the artifact store, training pipelines as a new kind of CI, and a monitoring layer for silent failures like data drift. DevOps has CI/CD; MLOps has CI/CD/CT — continuous training."

**Q2. Walk me through the ML lifecycle.**
Narrate Part 1 in 90 seconds. End with: "…and critically it's a loop, not a line — production monitoring feeds retraining."

**Q3. What is a model registry and why does it matter?**
"It's the artifact repository for models — like ECR for images. Versioning, stage transitions (staging/production/archived), lineage back to the exact run, data, and code, and approval gates. It's the control point for change management, and in a GxP environment it doubles as audit evidence."

**Q4. Explain data drift vs concept drift.**
Use 4.2. Add the practical bit: "Data drift you can detect without labels by comparing feature distributions to the training baseline; concept drift is nastier because inputs look normal — you catch it through prediction drift and delayed ground-truth performance tracking."

**Q5. What is a feature store and what problem does it solve?**
Reuse + training/serving skew; offline vs online stores (2.3).

**Q6. How do you ensure reproducibility of an ML model?**
"Pin all four inputs: code in Git, environment as a versioned Docker image, data via versioned datasets or DVC, and hyperparameters/metrics in an experiment tracker, plus random seeds. In Domino this is native — every job records the code commit, the compute environment revision, and the dataset version."

**Q7. Real-time vs batch inference — when would you choose each?**
"Real-time when a user or system is waiting on the prediction — fraud check at transaction time; needs an always-on endpoint, low latency, autoscaling. Batch when predictions can be precomputed — nightly churn scores; run a job, write to a table, no persistent infra, much cheaper. There's a middle option — async endpoints — for large payloads or slow models."

**Q8. How would you deploy a new model version safely?**
"Never big-bang. Ideally shadow it first — mirror live traffic, compare predictions offline with zero user impact. Then canary 5–10% of traffic with monitoring on both infra and prediction metrics, ramp gradually, and keep the previous registry version ready for instant rollback. SageMaker production variants or KServe canary give this natively; in a GxP shop the promotion itself goes through change control."

**Q9. What is CI/CD for ML? What do you test?**
"Three test layers: normal code tests (unit tests on feature/serving code), data tests (schema and quality checks — Great Expectations style), and model tests (does the retrained model beat the champion on held-out data; latency within SLO; no degradation on critical segments). The pipeline: code merge → CI tests + image build → training pipeline → evaluation gate → register model → staged deployment. The key difference from app CI/CD is that the pipeline's output artifact — the model — depends on data, not just code, so the pipeline must be triggerable by new data as well as new code."

**Q10. How do you monitor a model in production?**
Layered answer from 4.3 — lead with "two layers: infra and ML," describe both, mention silent failure. This is YOUR question — a support-role interviewer wants exactly this.

### Platform / troubleshooting scenarios (your comfort zone — dressed in ML clothes)
**Q11. A training job that used to finish in 2 hours now takes 8. Troubleshoot.**
"Same USE-method discipline as any perf issue, plus ML-specific suspects. First: what changed — code, data volume, instance type, library versions? Check data size growth (most common cause). Check resource saturation — is the GPU actually utilized (`nvidia-smi`) or is it starved by a data-loading/IO bottleneck (CPU-bound preprocessing, S3 throughput, EBS IOPS)? Check for spot interruptions/retries, node pressure, noisy neighbors on shared nodes, and whether the job landed on the intended node pool (taints/tolerations, instance type). Compare against the previous run's logged params in the experiment tracker — that's what tracking is for."

**Q12. A model endpoint is returning 5xx / high latency. Walk through it.**
"Standard service triage first: pod status, OOMKills, restarts, recent deploys, HPA behavior, node pressure, dependency health. ML-specific additions: model loading time on cold start (big models can take minutes — health/readiness probes must account for it), payload size spikes, GPU memory exhaustion, batch-size settings on the server, and whether a new model version was just promoted — correlate incident start with registry/deploy events. If the new version is the cause: instant rollback to the previous version, then RCA."

**Q13. Data scientists say 'the model works in the workspace but predictions differ in production.'**
"Classic training/serving skew or environment mismatch. Diff the environments — library versions between the workspace image and serving image (this is why versioned compute environments matter). Diff the feature computation path — are production features computed by different code than training features? Check preprocessing order, missing-value handling, and input schema. A feature store or shared preprocessing library is the permanent fix — that's the RCA-and-permanent-fix pattern the JD asks for."

**Q14. Workspace/notebook won't start (Domino-flavored).**
Answer from your real Domino knowledge: pod scheduling (resources, quotas, node availability, taints), image pull failures on the compute environment, PVC/storage mounting, and control-plane state issues — e.g., platform state desync where the underlying pod is gone but the platform still shows the workspace as running; resolve via platform admin actions and restore consistency, then RCA the cause.

**Q15. GPU job is Pending forever on Kubernetes. Why might that be?**
"No schedulable GPU node: check `nvidia.com/gpu` requests vs allocatable, device plugin daemonset healthy, taints/tolerations match, cluster-autoscaler able to scale the GPU node group (quotas! GPU instance quotas are a classic), and resource quota limits in the namespace. `kubectl describe pod` events tell you which."

**Q16. How would you reduce the cost of ML infrastructure?**
"Training: spot instances with checkpointing, right-size instance types, scale training clusters to zero when idle, schedule off-hours. Serving: autoscaling with sensible minimums, serverless/scale-to-zero endpoints for spiky traffic, multi-model endpoints to consolidate low-traffic models, batch instead of real-time where latency permits, and GPU sharing (MIG/time-slicing) where supported. Plus the boring wins: idle workspace reaping, storage lifecycle policies on artifacts."

**Q17. Design an automated retraining pipeline.**
"Trigger (schedule or drift alarm) → data pull + data validation gate → training job → evaluation vs current champion on a fixed test set → if better, register new version → manual approval gate (GxP) → staged deploy (shadow/canary) → monitor. Orchestrate with SageMaker Pipelines / Step Functions / Airflow / Domino Flows; every step containerized; every artifact versioned; alerts on every failure. Idempotent and re-runnable."

**Q18. How do SageMaker training jobs work under the hood?**
From 3.3 — emphasize ephemerality: "provision → pull container from ECR → mount data from S3 → train → write model.tar.gz to S3 → terminate. It's ephemeral batch compute with a contract defined by the container interface."

### Honesty-handler questions
**Q19. "Have you worked on MLOps in production?"**
"My production experience is on the platform side of MLOps — supporting Domino Data Lab on EKS: workspace provisioning, job orchestration, compute environments, model API deployments, and the Kubernetes/AWS layer under it. I haven't owned a data-science team's pipelines end-to-end, so over the last weeks I've gone deep on the surrounding toolchain — MLflow, SageMaker pipelines and endpoints, drift monitoring, deployment patterns — and because the concepts map directly onto the DevOps discipline I've practiced for 7 years, I ramp fast. For a platform support role, the failure modes are pods, images, storage, networking, and state — that's exactly my strength."
Never claim tools you can't back up. Confidence + honest framing beats bluffing every time.

**Q20. "Which MLOps tools have you used hands-on?"**
"Hands-on: Domino end-to-end — which embeds MLflow tracking — plus the AWS services around it: S3, ECR, Step Functions, Glue, CloudWatch. I've studied and labbed SageMaker and MLflow standalone; Kubeflow I know architecturally because it's CRDs and operators on Kubernetes, which I run daily."
(Make this true before Friday — do the Day 3 labs below.)

---

## Part 7 — Rapid-Fire Glossary (last-night review)

Inference = making predictions · Endpoint = deployed model behind an API · Artifact = any output file (model, dataset, report) · Pipeline = DAG of ML steps · DAG = directed acyclic graph · Hyperparameters = pre-training settings · Training vs Inference = building vs using the model · Ground truth = the real correct answers · Labeled data = inputs paired with correct outputs · Supervised learning = learning from labeled data · Feature = an input variable to a model · Feature engineering = creating features from raw data · Model decay = performance dropping over time · Champion/challenger = prod model vs candidates · Shadow deployment = new model sees traffic, doesn't answer · Canary = small % of real traffic · Registry stage = None/Staging/Production/Archived · CT = continuous training · Skew = train/serve feature mismatch · PSI/KS = drift statistics (names only) · ONNX = portable model format · Quantization = shrinking models for faster/cheaper inference · Fine-tuning = adapting a pretrained model · GPU MIG = partitioning a GPU · Data lineage = where data came from · Model lineage = code+data+params that produced a model · Model card = documentation of a model's intended use/limits · IQ/OQ/PQ = installation/operational/performance qualification (GxP validation).

---

## Part 8 — The 4-Day Battle Plan

**Day 1 (today) — Concepts.** Read Parts 0–2 twice. Then close the doc and narrate the ML lifecycle aloud in 90 seconds, five times, until fluent. Write the Part 0 table from memory. Memorize the positioning sentence. Evening: read Part 4 (monitoring/drift) — as a support engineer this is your highest-yield section.

**Day 2 — Tools.** Parts 3.1–3.4. For each tool, be able to say: what it is, its main components, and one sentence connecting it to something you already know. SageMaker gets double time (it's in the JD and it's AWS — your ground). Evening: re-describe every Domino feature in MLOps vocabulary (3.5) — this is your interview ammunition.

**Day 3 — Hands-on (make Q20's answer true).** 
- MLflow locally (30 min): `pip install mlflow scikit-learn`, train any toy sklearn model logging params/metrics, run `mlflow ui`, register the model, transition it to Staging. Now you've USED MLflow.
- Serve it (20 min): `mlflow models serve -m models:/<name>/1 -p 5000`, curl a prediction. Now you've DEPLOYED a model.
- SageMaker (60–90 min): if you have an AWS account, run one built-in-algorithm training job and deploy a serverless endpoint from the console, then delete it (watch cost). No account? Watch a "SageMaker end-to-end demo" video at 1.5x and click through the console screens mentally.
- Evening: Parts 5 + 6 (Q1–Q10). Answer aloud, not in your head.

**Day 4 — Interview simulation.** Morning: Q11–Q20 aloud; for each troubleshooting scenario, structure answers as: clarify → check what changed → systematic layer-by-layer → root cause → permanent fix (the JD literally asks for RCA + permanent fixes — use those words). Afternoon: glossary (Part 7), then one full self-mock: lifecycle narration, 3 concept questions, 3 scenarios, the honesty-handlers. Evening: STOP studying by 8pm. Sleep. A rested brain retrieving 85% beats an exhausted brain holding 100%.

**In the interview:**
1. Bridge every answer to platform reality — interviewers for support roles reward operational thinking over theory.
2. When you don't know a tool: "I haven't used X hands-on, but based on what it does, it's solving <problem> — architecturally similar to <thing I know>." That answer has saved more interviews than any memorized fact.
3. Use the JD's own words: RCA, permanent fixes, workspace provisioning, job orchestration, ML lifecycle, platform reliability.
4. Ask them one good question at the end, e.g.: "What does the retraining and model-promotion workflow look like today — is it automated, and where do you want to take it?" — that question alone signals MLOps maturity.
