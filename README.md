# 3-Tier Helm — PostgreSQL Chart

Helm দিয়ে Kubernetes-এ PostgreSQL (StatefulSet) deploy করার একটা প্র্যাকটিস প্রজেক্ট।
এই README-তে প্রজেক্টের গঠন, secrets কীভাবে রাখতে হয়, আর Helm-এর প্রায় সব দরকারি কমান্ড বাংলায় ব্যাখ্যা করা আছে।

---

## সূচিপত্র

- [প্রজেক্টের গঠন](#প্রজেক্টের-গঠন)
- [Helm-এর মূল ধারণা](#helm-এর-মূল-ধারণা)
- [Secrets সেটআপ](#secrets-সেটআপ)
- [Values ফাইল কীভাবে merge হয়](#values-ফাইল-কীভাবে-merge-হয়)
- [Quick Start](#quick-start)
- [Helm কমান্ড — একটা একটা করে](#helm-কমান্ড--একটা-একটা-করে)
- [প্র্যাকটিস প্ল্যান](#প্র্যাকটিস-প্ল্যান)
- [সতর্কতা](#সতর্কতা)

---

## প্রজেক্টের গঠন

```
3-tire-helm/
├── .gitignore                 # secrets-values*.yaml, *.env — GitHub-এ যাবে না
├── secrets-values.yaml        # DB user/password (git-এ commit হবে না)
└── myapp/                     # Helm chart
    ├── Chart.yaml             # চার্টের নাম, version
    ├── values.yaml            # default values
    ├── values-staging.yaml    # staging-এর জন্য values
    ├── values-prod.yaml       # production-এর জন্য values
    └── templates/
        ├── db.yaml            # Namespace, Headless Service, Service, StatefulSet
        └── secrets.yaml       # Secret (মান আসে secrets-values.yaml থেকে)
```

চার্ট যা তৈরি করে:

| রিসোর্স | নাম (staging উদাহরণ) | কাজ |
|---|---|---|
| Namespace | `db-staging` | সব রিসোর্স এখানে থাকে |
| Secret | `postgres-secret` | `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` |
| Service (headless) | `staging-db-headless` | StatefulSet pod-এর stable DNS |
| Service | `postgres-staging` | অ্যাপ এখান দিয়ে DB-তে connect করবে (port 5432) |
| StatefulSet | `postgres` | PostgreSQL pod + PVC (`data`) |

---

## Helm-এর মূল ধারণা

- **Chart** — টেমপ্লেট আর ভ্যালুর প্যাকেজ (এখানে `myapp/` ফোল্ডার)।
- **Values** — টেমপ্লেটের `{{ .Values... }}`-এর জায়গায় যে মান বসে।
- **Release** — একটা চার্ট ক্লাস্টারে install করলে যেটা চলে (যেমন `db`)। একই চার্ট থেকে একাধিক release বানানো যায় (`db`, `db-prod`)।
- **Revision** — প্রতিটা install/upgrade/rollback-এ নতুন version (1, 2, 3...)। এর ফলে rollback করা যায়।

Helm install করার সময় রিসোর্সগুলো kind অনুযায়ী ক্রমে তৈরি করে — **Namespace → Secret → Service → StatefulSet**। তাই Secret তৈরির আগেই Namespace তৈরি হয়ে যায়।

---

## Secrets সেটআপ

পাসওয়ার্ড কখনো চার্টে বা GitHub-এ রাখা যাবে না। তাই:

1. `templates/secrets.yaml` শুধু টেমপ্লেট — এটা commit করা নিরাপদ:

   ```yaml
   stringData:
     POSTGRES_USER: {{ required "db.user is required" .Values.db.user | quote }}
     POSTGRES_PASSWORD: {{ required "db.password is required" .Values.db.password | quote }}
     POSTGRES_DB: {{ required "db.name is required" .Values.db.name | quote }}
   ```

2. রিপোর রুটে `secrets-values.yaml` বানান (এটা `.gitignore`-এ আছে):

   ```yaml
   db:
     user: myuser
     password: super-strong-password
     name: mydb
   ```

3. install/upgrade-এর সময় `-f secrets-values.yaml` দিন।

`required` থাকার কারণে secrets ফাইল দিতে ভুলে গেলে Helm error দিয়ে থেমে যাবে।

> **প্রথম commit-এর আগে** `git status` চালিয়ে নিশ্চিত হোন যে `secrets-values.yaml` লিস্টে **নেই**।

Production-এর জন্য আরো ভালো উপায়: **Sealed Secrets**, **helm-secrets + SOPS**, বা **External Secrets Operator**।

---

## Values ফাইল কীভাবে merge হয়

`-f` দিলে `values.yaml` বাদ পড়ে না — সব ফাইল একটার উপর আরেকটা merge হয়। **পরেরটা আগেরটাকে override করে**:

```
1. myapp/values.yaml          (সবসময় লোড হয়)
2. -f myapp/values-prod.yaml
3. -f secrets-values.yaml
4. --set key=value            (সবার উপরে)
```

যে key শুধু একটা ফাইলে আছে সেটা থেকে যায়; একই key দুই জায়গায় থাকলে পরেরটা জেতে।

---

## Quick Start

```bash
# staging
helm upgrade --install db ./myapp -f myapp/values-staging.yaml -f secrets-values.yaml

# production
helm upgrade --install db-prod ./myapp -f myapp/values-prod.yaml -f secrets-values.yaml

# চেক
kubectl get all,secret,pvc -n db-staging
```

---

## Helm কমান্ড — একটা একটা করে

সব কমান্ড রিপোর রুট (`3-tire-helm/`) থেকে চালাবেন।

### ১. Install-এর আগে চেক

#### `helm version`
```bash
helm version
```
Helm ইনস্টল আছে কিনা আর কোন version — দেখায়।

#### `helm lint`
```bash
helm lint ./myapp -f secrets-values.yaml
```
চার্টের syntax/structure চেক করে।
⚠️ lint পাস মানেই Kubernetes মেনে নেবে তা নয় — যেমন `containerPort:5432` (স্পেস ছাড়া) lint পাস করে কিন্তু Kubernetes reject করে।

#### `helm template`
```bash
helm template db ./myapp -f myapp/values-staging.yaml -f secrets-values.yaml

# শুধু একটা ফাইল দেখতে
helm template db ./myapp -f secrets-values.yaml -s templates/secrets.yaml
```
ক্লাস্টারে কিছু না পাঠিয়ে **রেন্ডার করা YAML** দেখায়। `{{ }}`-এর জায়গায় কী বসলো, তা এখানে দেখা যায়।

#### `helm install --dry-run=server`
```bash
helm install db ./myapp -f secrets-values.yaml --dry-run=server
```
ক্লাস্টারের API server দিয়ে **validate** করায়, কিন্তু কিছু তৈরি করে না। `helm template`-এর চেয়ে নির্ভরযোগ্য।
👉 আসল install-এর আগে সবসময় এটা চালান।

---

### ২. Install আর দেখা

#### `helm install`
```bash
helm install db ./myapp -f myapp/values-staging.yaml -f secrets-values.yaml
```
- `db` — release-এর নাম
- `./myapp` — চার্টের ফোল্ডার
- `-f` — বাড়তি values ফাইল

#### `helm list`
```bash
helm list        # current namespace-এর release
helm list -A     # সব namespace-এর release
```
📌 রিসোর্সগুলো `db-staging`-এ তৈরি হলেও release-এর তথ্য থাকে যে namespace থেকে helm চালিয়েছেন সেখানে (`-n` না দিলে `default`)।

#### `helm status`
```bash
helm status db
```
release-এর অবস্থা (deployed / failed / pending) আর বর্তমান revision।

---

### ৩. Release-এর ভেতরে দেখা

#### `helm get values`
```bash
helm get values db          # শুধু আপনি যা দিয়েছেন (-f / --set)
helm get values db --all    # values.yaml সহ চূড়ান্ত সব values
```

#### `helm get manifest`
```bash
helm get manifest db
```
ক্লাস্টারে **আসলে যে YAML পাঠানো হয়েছে** সেটা দেখায়।

#### `helm get all`
```bash
helm get all db
```
values, manifest, notes — সব একসাথে।

---

### ৪. পরিবর্তন করা (Upgrade)

#### `helm upgrade`
```bash
helm upgrade db ./myapp -f myapp/values-staging.yaml -f secrets-values.yaml
```
যা বদলেছে শুধু সেটা আপডেট করে, revision এক বাড়ে।
- ⚠️ `-f secrets-values.yaml` আবার দিতে হবে, নাহলে `required` error দেবে।
- ⚠️ release আগে থেকে না থাকলে error: `"db" has no deployed releases`।

#### `--set`
```bash
helm upgrade db ./myapp -f secrets-values.yaml --set app.limits.memory=512Mi
```
ফাইল না বদলে কমান্ড থেকে value দেওয়া। `--set` সবার উপরে প্রাধান্য পায়।

#### `helm upgrade --install`
```bash
helm upgrade --install db ./myapp -f myapp/values-staging.yaml -f secrets-values.yaml
```
- release **না থাকলে** → install (revision 1)
- release **থাকলে** → upgrade (revision 2, 3...)

CI/CD-তে এটাই সবচেয়ে বেশি ব্যবহার হয়, কারণ প্রতিবার একই কমান্ড চালানো যায় — "আগে install হয়েছে কিনা" চেক করার দরকার নেই।

---

### ৫. ইতিহাস আর Rollback

#### `helm history`
```bash
helm history db
```
সব revision-এর তালিকা — কবে, কী status (deployed / superseded / failed)।

#### `helm rollback`
```bash
helm rollback db 1
```
release-কে revision 1-এর অবস্থায় ফেরায়। এটা নতুন একটা revision তৈরি করে (যেমন 4), যার কনটেন্ট revision 1-এর মতো।

---

### ৬. মুছে ফেলা

#### `helm uninstall`
```bash
helm uninstall db
```
release-এর সব রিসোর্স মুছে দেয়।
⚠️ এই চার্ট নিজেই Namespace তৈরি করে, তাই uninstall করলে **পুরো namespace — PVC আর ডেটাবেসের ডেটা সহ — মুছে যাবে**।

---

### ৭. চার্ট নিয়ে কাজ

#### `helm show`
```bash
helm show values ./myapp    # default values
helm show chart ./myapp     # Chart.yaml-এর তথ্য
```

#### `helm package`
```bash
helm package ./myapp
```
চার্টকে `myapp-<version>.tgz` বানায়। `Chart.yaml`-এর `version` বাড়ালে package version-ও বাড়ে।
📌 `myapp/` ফোল্ডারের ভেতরের সব ফাইল package-এ ঢোকে — তাই secrets ফাইল কখনো `myapp/`-এর ভেতরে রাখবেন না।

#### `helm create`
```bash
helm create testchart
```
নতুন চার্টের খালি কাঠামো তৈরি করে।

---

### ৮. অন্যের চার্ট ব্যবহার (Repository)

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo postgresql
helm show values bitnami/postgresql
helm pull bitnami/postgresql --untar
```

| কমান্ড | কাজ |
|---|---|
| `repo add` | chart repository যোগ করে |
| `repo update` | repo-র সর্বশেষ তালিকা নামায় |
| `search repo` | repo-তে চার্ট খোঁজে |
| `show values` | ওই চার্টের default values দেখায় |
| `pull --untar` | চার্ট ডাউনলোড করে খুলে দেয় |

👉 bitnami-র postgresql চার্ট খুলে নিজের `db.yaml`-এর সাথে তুলনা করুন — প্রফেশনাল চার্ট কীভাবে লেখা হয় শিখতে পারবেন।

---

### কমান্ড চিটশিট

| কমান্ড | এক লাইনে কাজ |
|---|---|
| `helm lint` | চার্টের syntax চেক |
| `helm template` | লোকালি YAML রেন্ডার |
| `helm install --dry-run=server` | ক্লাস্টার দিয়ে validate, কিছু তৈরি না করে |
| `helm install` | নতুন release তৈরি |
| `helm upgrade` | release আপডেট |
| `helm upgrade --install` | না থাকলে install, থাকলে upgrade |
| `helm list` | release-এর তালিকা |
| `helm status` | release-এর অবস্থা |
| `helm get values / manifest / all` | release-এর values / YAML / সব |
| `helm history` | revision-এর তালিকা |
| `helm rollback` | আগের revision-এ ফেরা |
| `helm uninstall` | release মুছে ফেলা |
| `helm show values / chart` | চার্টের তথ্য |
| `helm package` | চার্টকে `.tgz` বানানো |
| `helm repo add / update` | repository যোগ / আপডেট |
| `helm search repo` | চার্ট খোঁজা |
| `helm pull` | চার্ট ডাউনলোড |

---

## প্র্যাকটিস প্ল্যান

1. `helm lint` → `helm template` → `helm install --dry-run=server`
2. `helm install db ./myapp -f myapp/values-staging.yaml -f secrets-values.yaml`
   → `kubectl get all -n db-staging`
3. `helm list`, `helm status db`, `helm get values db --all`
4. `values-staging.yaml`-এ memory বদলে `helm upgrade` → `helm history db`
5. ইচ্ছা করে ভুল image দিন:
   `helm upgrade db ./myapp -f secrets-values.yaml --set app.image=postgres:wrong`
   → pod-এ `ImagePullBackOff` দেখুন → `helm rollback db <আগের revision>`
6. production release:
   `helm install db-prod ./myapp -f myapp/values-prod.yaml -f secrets-values.yaml`
7. `helm list -A` — দুটো release দেখুন
8. `helm uninstall` দিয়ে release মুছে ফেলার আগে `helm uninstall --help` পড়ুন, তারপর `helm uninstall db-prod`
9. `helm uninstall db` → `helm upgrade db ...` (error দেখুন) → `helm upgrade --install db ...` (install হবে)
10. `helm package ./myapp`

---

## সতর্কতা

- `secrets-values.yaml` কখনো commit করবেন না — commit-এর আগে `git status` দেখুন।
- secrets ফাইল `myapp/` ফোল্ডারের ভেতরে রাখবেন না (`helm package`-এ ঢুকে যাবে)।
- `helm uninstall` করলে namespace সহ ডেটাবেসের ডেটা মুছে যায়।
- `templates/` ফোল্ডারে `.env` বা অন্য non-YAML ফাইল রাখবেন না — Helm সেগুলোও রেন্ডার করার চেষ্টা করে error দেয়।
- `replicas`-এর মতো সংখ্যার মান quote (`"..."`) করবেন না — string হয়ে যাবে, Kubernetes reject করবে।
