# Setup

The delegate is the only thing that needs to reach into your cluster. Harness's SaaS control plane never connects to Docker Desktop directly, the delegate calls out to it instead.

## Defaults used below

- org `default`, project `yt_dlp_ci`
- build pods in namespace `harness-ci`
- Python 3.12, branch built: `master` (yt-dlp's default branch)
- tests: `python3 -m devscripts.run_tests core`, yt-dlp's own offline test target. The full "download" suite hits real video sites and isn't suitable for CI.

Change any of these by editing the corresponding field under `.harness/orgs/default/projects/yt_dlp_ci/` and the commands below.

## Requirements

- Docker Desktop with Kubernetes enabled (`kubectl config current-context` → `docker-desktop`)
- A Harness account

## 1. Fork yt-dlp

Fork https://github.com/yt-dlp/yt-dlp on GitHub.

## 2. Create this repo

Create an empty GitHub repo named `harness-python-yt-dlp-example`, clone it, copy this file and `.harness/` into it.

## 3. GitHub token

Fine-grained token scoped to your `yt-dlp` fork and `harness-python-yt-dlp-example`: Contents (read/write), Metadata (read), Webhooks (read/write). Or a classic token with `repo` scope. Copy it now. Don't put it in any file in this repo.

## 4. Install the Harness CLI

Per https://github.com/harness/cli , install harness cli via:

```bash
curl -fsSL https://raw.githubusercontent.com/harness/cli/main/install.sh | sh
```

## 5. Log in, create the project

Account ID is in the URL when logged into the platform (`app.harness.io/ng/#/account/<id>`). Generate an API token from your user profile.

```bash
harness auth login --api-token <TOKEN> --account <ACCOUNT_ID> --org default --overwrite
harness create project yt_dlp_ci --set name="yt_dlp_ci"
harness auth login --api-token <TOKEN> --account <ACCOUNT_ID> --org default --project yt_dlp_ci --overwrite
```

## 6. Cluster: namespaces, delegate

```bash
kubectl config use-context docker-desktop
kubectl create namespace harness-ci
```

In the Harness UI: Project Setup > Delegates > New Delegate > Kubernetes > Manifest. Name it `yt-dlp-kubernetes-delegate`, generate, and download the YAML then apply via kubectl. It comes with your account ID, manager endpoint, and a real, correctly base64-encoded delegate token already filled in, don't edit it.

```bash
kubectl apply -f <downloaded file name>
kubectl get pods -n harness-delegate-ng
harness list delegate   # wait for CONNECTED
```

## 7. GitHub PAT secret

```bash
harness create secret github_pat --set value="<PAT>" --set name="github_pat"
```

## 8. Connectors

The GitHub connector files reference a GitHub username directly (`allyunion` in this repo). If you're reusing this repo under a different account, update `url` and `username` in both `github_harness_repo_connector.yaml` and `github_ytdlp_connector.yaml` first.

```bash
harness create connector github_harness_repo_connector -f .harness/orgs/default/projects/yt_dlp_ci/connectors/github_harness_repo_connector.yaml
harness create connector github_ytdlp_connector -f .harness/orgs/default/projects/yt_dlp_ci/connectors/github_ytdlp_connector.yaml
harness create connector dockerhub_anonymous_connector -f .harness/orgs/default/projects/yt_dlp_ci/connectors/dockerhub_anonymous_connector.yaml
harness create connector k8s_docker_desktop_connector -f .harness/orgs/default/projects/yt_dlp_ci/connectors/k8s_docker_desktop_connector.yaml
```

The `connectors/` folder should only ever contain connector files. A pipeline-shaped file placed there won't get treated as a connector correctly.

## 9. Create the pipeline

```bash
harness create pipeline -f .harness/orgs/default/projects/yt_dlp_ci/pipelines/buildytdlp.yaml \
  --connector github_harness_repo_connector \
  --repo harness-python-yt-dlp-example \
  --branch main \
  --file-path .harness/orgs/default/projects/yt_dlp_ci/pipelines/buildytdlp.yaml

git pull
```

This commits the pipeline YAML into this repo through the connector, it's git-sourced from here on, not just locally version-controlled.

## 10. Run it

```bash
harness execute pipeline build_yt_dlp --branch main --follow
```

`--branch main` is this repo's branch, where the pipeline YAML lives, not yt-dlp's `master`.

## 11. Commit

```bash
git add README.md SETUP.md .harness/orgs/default/projects/yt_dlp_ci/connectors/*.yaml .gitignore
git commit -m "Harness CI setup for yt-dlp build pipeline"
git push
```

The pipeline YAML was already committed by Harness in step 9, pull before this to avoid a conflict. Before this commit, double check `git status` doesn't show a Secret object with an actual token value staged, that class of mistake is easy to make with the manifest approach if you ever re-download it from the UI.

## Secrets

Only `tokenRef: github_pat` appears in any connector file. The delegate manifest downloaded in step 6 has a real token in it by design, it's applied directly from wherever you download it and never added to this repo. The PAT, the Harness API token, and the delegate token exist only in your shell history, that downloaded file on local disk, and the Harness/Kubernetes secret stores.

## Troubleshooting

- `connectorType: must not be null` on create: `harness create connector -f` wraps your file's content in a top-level `connector:` key itself before sending it. If the file already has that key, the body ends up double-nested and the API rejects it with this error. Notably, connector YAML exported directly from the Harness UI *always* includes that top-level key, so a UI export can't be applied as-is with this CLI. Possibly a mismatch between the UI's export format and what this CLI version expects rather than something wrong with the connector definition itself. If you regenerate any connector file from the UI, strip the top-level `connector:` line and de-indent one level before running `create connector` on it.
- A connector's `tokenRef`/`serviceRef`/`environmentRef` lookup fails: check `projectIdentifier` in that file matches `yt_dlp_ci`.
- Kubernetes connector can't find a delegate: check `delegateSelectors` matches the delegate's actual name exactly, a rename on one side without the other is the most common cause.
- Delegate `CrashLoopBackOff` right after `kubectl apply`: re-download the manifest from step 6 rather than reusing an old copy, an expired or already-scrubbed token is the most common cause.
- Build pods `Pending` in `harness-ci`: `kubectl describe pod -n harness-ci <pod>`, usually a resource ceiling on the single Docker Desktop node.
- A tool "not found" mid-pipeline: each Run step is a fresh container, nothing installed in one step carries to the next except what's inside the cloned repo directory.
