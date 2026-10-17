# Lab runbook - do these in order

Everything is already built and unit-tested. In the lab you only **run it, capture proof, and record the video**.
Prereqs on the lab machine: Python 3.11, Docker, kubectl + Minikube (or Docker Desktop Kubernetes), git, a GitHub account.

## 0. Personalise (assignment's academic-integrity note)
Change something that is genuinely yours before submitting: e.g. add a third model (GradientBoosting) in
`src/heart/train.py::candidate_models`, tweak the grids, rename the API fields' log format, restyle the Grafana panels.
Be ready to explain every file in the viva/video.

## 1. Local run (tasks 1-4)
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
python scripts/download_data.py
python -m src.heart.eda
python -m src.heart.train          # re-train HERE so the model matches the lab's sklearn version
flake8 src api tests scripts && pytest -v
mlflow ui                           # open http://127.0.0.1:5000
```
Screenshots: `mlflow_runs.png` (experiment table), `mlflow_run_detail.png` (one run: params/metrics/artifacts).

## 2. GitHub + CI/CD (task 5)
```bash
git init && git add . && git commit -m "MLOps assignment"
git branch -M main && git remote add origin <your-repo-url> && git push -u origin main
```
- Actions tab -> green run -> `github_actions_success.png` (show all 3 jobs + artifacts).
- Fail-fast proof: edit a test (e.g. `assert 1 == 2` in `tests/test_data.py`), push, screenshot the red run as
  `github_actions_failure.png`, then **revert and push again** so the final run is green.

## 3. Docker (task 6)
```bash
docker build -t heart-api:latest .
docker run -d --name heart -p 8000:8000 heart-api:latest
curl localhost:8000/health
curl -X POST localhost:8000/predict -H "Content-Type: application/json" -d @scripts/sample_request.json
docker rm -f heart
```
Screenshot build + run + curl output together: `docker_run.png`.

## 4. Kubernetes (task 7)
```bash
minikube start --driver=docker
eval $(minikube docker-env) && docker build -t heart-api:latest .      # Docker Desktop: skip the eval
kubectl apply -f k8s/deployment.yaml -f k8s/hpa.yaml
kubectl rollout status deploy/heart-api
minikube tunnel                      # separate terminal, keep open (Docker Desktop: not needed)
kubectl get pods,svc,hpa             # -> kubectl_get_all.png
curl http://<EXTERNAL-IP>/health
curl -X POST http://<EXTERNAL-IP>/predict -H "Content-Type: application/json" -d @scripts/sample_request.json   # -> predict_curl.png
```
If EXTERNAL-IP stays `<pending>`: use `minikube service heart-api --url` and note that URL in the report.

## 5. Monitoring (task 8)
```bash
kubectl apply -f k8s/monitoring.yaml
kubectl port-forward svc/prometheus 9090:9090 &
kubectl port-forward svc/grafana 3000:3000 &          # admin / admin
python scripts/load_test.py --url http://<EXTERNAL-IP> -n 200
kubectl logs -l app=heart-api --tail=20                # -> api_logs.png
```
Grafana: Dashboards -> Import -> `monitoring/grafana-dashboard.json` -> add Prometheus data source
(`http://prometheus:9090`) -> screenshot as `grafana_dashboard.png`. (Fallback: `docker compose up --build`.)

## 6. Finish the report (task 9)
1. Overwrite the 9 red-bordered files in `screenshots/` with your real screenshots (same filenames, PNG).
2. Edit `docs/REPORT.md`: fill the `<...>` fields (name, BITS ID, repo URL, video link, cluster used, API access).
   Update numbers if you changed models or re-trained (see `reports/model_comparison.csv`).
3. Rebuild: `cd docs && pandoc REPORT.md -o REPORT.docx --resource-path=.:..`
   (no pandoc? open `REPORT.docx` in Word and use Change Picture on each placeholder). Export PDF too.
4. Put the repo URL in `README.md`, commit and push.

## 7. Video (~3-4 min) - suggested script
1. Repo tour (30s): folders, README. 2. Run `pytest` + show MLflow UI (45s). 3. Push a commit, show Actions run (45s).
4. `docker build/run` + curl (30s). 5. `kubectl get pods,svc`, curl `/predict` via service (45s).
6. Run load test, show Grafana + logs (45s).

## Final checklist (marks)
- [ ] EDA figures, cleaned CSV, download script (5)  - [ ] 2+ models, CV metrics (8)  - [ ] MLflow runs (5)
- [ ] model.joblib + requirements + pipeline (7)  - [ ] tests + green Actions + artifacts (8)  - [ ] Docker proof (5)
- [ ] K8s manifests + screenshots (7)  - [ ] logs + Grafana (3)  - [ ] ~10-page report + repo link (2)  - [ ] video
