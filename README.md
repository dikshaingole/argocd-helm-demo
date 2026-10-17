# argocd-helm-demo

A small GitOps project: a **Helm chart** for a web page, deployed and kept up to date by **Argo CD**.

```
 you edit values.yaml ──git push──▶ GitHub repo ◀──watches── Argo CD ──deploys──▶ Kubernetes (kind)
```

You never run `kubectl apply` or `helm install` for the app. You change Git, and Argo CD changes the cluster to match.

## What's in here

```
argocd-helm-demo/
├── hello-app/                 ← the Helm chart
│   ├── Chart.yaml             ← chart name and version
│   ├── values.yaml            ← the settings you change (message, colour, replicas)
│   └── templates/             ← Kubernetes YAML with {{ placeholders }}
│       ├── configmap.yaml     ← the HTML page
│       ├── deployment.yaml    ← runs nginx pods that serve the page
│       ├── service.yaml       ← gives the pods one stable address
│       └── NOTES.txt          ← message printed after install
└── argocd/
    └── application.yaml       ← tells Argo CD where the chart is and where to deploy it
```

**Helm** turns `templates/` + `values.yaml` into plain Kubernetes YAML.
**Argo CD** does that rendering for you, applies the result, and keeps doing it every time Git changes.

## Before you start

- Docker Desktop is running.
- The local cluster exists: `kubectl config current-context` prints `kind-argocd`.
  If it shows your EKS cluster, switch with `kubectl config use-context kind-argocd`.
- The Argo CD UI port-forward is running (in its own terminal):
  `kubectl port-forward svc/argocd-server -n argocd 8080:443`

---

## Step 1 (optional): see what Helm generates

Run this from the `argocd-helm-demo` folder:

```powershell
helm lint hello-app
helm template hello-app hello-app
```

`helm template` prints the final YAML. Change something in `values.yaml`, run it again, and look at what changed. That's all Helm does: fill in templates.

## Step 2: put the project on GitHub

Argo CD reads from Git, so the chart has to be in a repository it can reach.

1. On github.com, create a new **public**, **empty** repository named `argocd-helm-demo`. Don't add a README.
2. Push this folder to it:

```powershell
cd D:\Devops\argocd-helm-demo
git init -b main
git add .
git commit -m "Helm chart and Argo CD application"
git remote add origin https://github.com/dikshaingole/argocd-helm-demo.git
git push -u origin main
```

> If your GitHub username or repo name is different, update `repoURL` in `argocd/application.yaml` to match before you push.
> If you make the repo private, Argo CD needs a token first:
> `argocd repo add https://github.com/<you>/argocd-helm-demo.git --username <you> --password <github-token>`

## Step 3: hand the app to Argo CD

```powershell
kubectl apply -f argocd/application.yaml
```

This is the only `kubectl apply` you run. It creates an Argo CD **Application**: a record that says "deploy the chart at `hello-app/` in this repo into the `hello` namespace, and keep it in sync".

Watch it come up:

```powershell
argocd app get hello-app
```

Or open https://localhost:8080 and click the **hello-app** tile. You'll see the tree: Application → Deployment → ReplicaSet → Pods, plus the Service and ConfigMap. Wait until it shows **Synced** and **Healthy**.

## Step 4: open your app

In a new terminal:

```powershell
kubectl port-forward svc/hello-app -n hello 8081:80
```

Go to http://localhost:8081. You'll see a blue page that says **Version 1 - deployed with GitOps**.

## Step 5: the GitOps loop (the main point)

Edit `hello-app/values.yaml`:

```yaml
replicaCount: 3

page:
  title: "Hello from Argo CD + Helm"
  message: "Version 2 - changed only in Git!"
  backgroundColor: "#2f855a"
```

Then push it:

```powershell
git commit -am "v2: green page, 3 replicas"
git push
```

Argo CD checks Git about every 3 minutes. To skip the wait, click **Refresh** in the UI or run:

```powershell
argocd app sync hello-app
```

Watch the UI: a third pod appears, and the old pods are replaced with new ones. The `checksum/page` annotation in `deployment.yaml` restarts the pods whenever the page content changes.
Refresh http://localhost:8081 and the page is green. (If it doesn't load, restart the port-forward from Step 4.)

## Step 6: self-healing

Try changing the cluster by hand, the way you're not supposed to:

```powershell
kubectl scale deployment hello-app -n hello --replicas=6
kubectl get pods -n hello -w
```

Within a few seconds Argo CD sees that the cluster no longer matches Git and scales back to 3. That's `selfHeal: true`. **Git is the source of truth.** Press `Ctrl+C` to stop watching.

## Step 7: roll back

With GitOps, rolling back means undoing the commit:

```powershell
git revert HEAD --no-edit
git push
argocd app sync hello-app
```

The page goes back to blue, version 1, with 2 replicas. `argocd app history hello-app` lists every revision that has been deployed.

## Step 8: clean up

```powershell
kubectl delete -f argocd/application.yaml   # Argo CD removes the app
kubectl delete namespace hello
```

To delete the entire local cluster, Argo CD included: `kind delete cluster --name argocd`

---

## Practice ideas

1. Add a `values-dev.yaml` with a different colour, and a second Application (`hello-app-dev`) that uses it in a `hello-dev` namespace. One chart, two environments.
2. Bump `version` in `Chart.yaml` and see how Argo CD shows the change.
3. Delete `service.yaml` from Git, push, and watch `prune: true` remove the Service.

## Troubleshooting

| Problem | Fix |
|---|---|
| App shows **Unknown** / "repository not found" | Check `repoURL`, and make sure the repo is public or added with `argocd repo add` |
| Change doesn't show up | Click **Refresh**, or run `argocd app sync hello-app` |
| `localhost:8080` / `8081` won't open | The port-forward stopped. Run it again |
| kubectl talks to the wrong cluster | `kubectl config use-context kind-argocd` |
| Pods stuck in `ImagePullBackOff` | Check your internet connection and `image.tag` in `values.yaml` |
