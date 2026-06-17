# Sample Spring Boot App — Jenkins + kind Pipeline

## What's in this repo

- `src/`, `pom.xml` — minimal Spring Boot REST app (endpoints: `/`, `/health`, `/version`)
- `Dockerfile` — multi-stage build (Maven build -> JRE runtime)
- `Jenkinsfile` — declarative pipeline: checkout -> build/test -> build & push image (Kaniko) -> deploy to kind
- `k8s/registry.yaml` — in-cluster Docker registry (one-time setup, not part of the pipeline)
- `k8s/deployment.yaml` — Deployment + Service for the app (templated by the pipeline)

## One-time cluster setup

1. Deploy the in-cluster registry (only needs to be done once):

   ```bash
   kubectl apply -f k8s/registry.yaml
   kubectl rollout status deployment/registry -n jenkins
   ```

2. Confirm Jenkins agent pods can reach it:

   ```bash
   kubectl run test-curl --rm -it --image=curlimages/curl -n jenkins -- \
     curl -s registry.jenkins.svc.cluster.local:5000/v2/_catalog
   ```

   You should get back `{"repositories":[]}`.

3. Push this repo to your private GitHub repo, with `Jenkinsfile` at the root.

## Jenkins job setup

1. **New Item** -> name it `demo-app-pipeline` -> select **Pipeline**
2. Under **Pipeline**, choose **Pipeline script from SCM**
3. SCM: **Git**
   - Repository URL: your GitHub repo URL
   - Credentials: select your `github-creds` (must already exist in Jenkins)
   - Branch: `main` (or your default branch)
   - Script Path: `Jenkinsfile`
4. Save -> **Build Now**

## Verifying the deployment

```bash
kubectl get pods -n jenkins -l app=demo-app
kubectl port-forward -n jenkins svc/demo-app 8081:8080
```

Then visit `http://localhost:8081/` — you should see:
`Hello from Spring Boot running in kind cluster!`

## Notes

- Each pipeline run builds a new image tagged with the Jenkins `BUILD_NUMBER`, so every run produces a uniquely tagged image — nothing gets silently overwritten.
- The registry uses `emptyDir` storage, so images are lost if the registry pod restarts. Fine for local practice; not meant for anything persistent.
- Kaniko is used instead of Docker-in-Docker so the build doesn't need privileged access to the host Docker daemon.
