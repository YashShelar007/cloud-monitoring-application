# cloud-monitoring-application

A small Flask page that shows the CPU and memory use of the machine it runs on as two Plotly gauges, plus two Python scripts that push the app to AWS: `ecr.py` creates an Elastic Container Registry repository and `eks.py` deploys the image to an existing EKS cluster. It is a worked example of taking a Python app from local to Docker to Kubernetes on AWS.

## What it does not do

- It does not monitor EKS or ECR. The gauges read the host or container the app runs in (`psutil`), not cluster or registry state.
- No history, no persistence and no authentication. A page load takes one reading.
- It does not create the EKS cluster or node group. `eks.py` assumes both exist.
- No tests.

## Quickstart

Verified locally: the pinned `requirements.txt` installs on Python 3.13 and the page renders with live CPU and memory values.

```bash
pip install -r requirements.txt
python3 app.py          # http://localhost:5000 (Flask debug mode, binds 0.0.0.0)
```

Docker build, ECR and EKS steps are not verified in this cleanup (no Docker, AWS account or cluster available). The Dockerfile uses `python:3.9-slim-buster`, an end-of-life Debian base whose package repositories may no longer resolve, so the build may fail as written.

```bash
docker build -t cloud-monitor .
docker run -p 5000:5000 cloud-monitor

python3 ecr.py          # creates repo "cloud_native_monitoring_app_repo", prints its URI
docker tag cloud-monitor <ecr_repo_uri>:<tag> && docker push <ecr_repo_uri>:<tag>

# edit image="Your Image" in eks.py (line 26) to that URI, then
aws eks update-kubeconfig --region <region> --name <cluster>
python3 eks.py          # creates a Deployment and a Service in "default"
kubectl port-forward service/cloud-native-monitoring-service 5000:5000
```

## How it works

```
browser -> Flask app.py -> psutil (cpu_percent, virtual_memory)
                        -> templates/index.html (two Plotly gauge indicators)
```

- `app.py` has one route, `/`. It reads CPU and memory percentages and passes them to the template, along with a warning string when either is above 80.
- `templates/index.html` draws the gauges with Plotly loaded from a CDN, with bands at 0 to 50, 50 to 85 and 85 to 100.
- `ecr.py` calls `boto3` to create the repository and prints the URI. `eks.py` uses the `kubernetes` client with your local kubeconfig to create a one-replica Deployment and a Service on port 5000.
- The `Dockerfile` installs the requirements and starts the app with `flask run` on port 5000.

## Status

Built from September 2024 to April 2025 (first and last commit dates). Archived: no further changes planned.

## Known limits

- The dependency pins are old (`boto3==1.9.148`, `kubernetes==10.0.1`, `plotly==5.5.0`). The template loads `plotly-latest.min.js` from a CDN, an unpinned URL.
- `app.py` runs with `debug=True` on all interfaces. Do not expose it publicly.
- The Service in `eks.py` has no type, so it is cluster-internal; the instructions use `kubectl port-forward`.

## License

MIT, see [LICENSE](LICENSE).
