# Kustomize Blue/Green Lab: NGINX (blue) vs Apache (green)

Both versions run side by side at all times. The `web` Service selects one of them
via the `color` label. Switching traffic = applying the other overlay, which only
changes that selector. It is instant and needs no pod restarts.

| Color | Server | Image             | Page             |
|-------|--------|-------------------|------------------|
| blue  | NGINX  | nginx:1.27-alpine | blue background  |
| green | Apache | httpd:2.4-alpine  | green background |

```
.
├── base/                 # namespace, 2 deployments, service, html configmaps
└── overlays/
    ├── blue/             # Service -> color: blue  (NGINX)
    └── green/            # Service -> color: green (Apache)
```

## Prerequisites

- A running cluster (kind, minikube, k3s, RKE2, ...)
- `kubectl` v1.14+ (built-in kustomize)

## 1. Preview (optional)

```bash
kubectl kustomize overlays/blue
```

## 2. Deploy BLUE (NGINX is live)

```bash
kubectl apply -k overlays/blue
kubectl -n bluegreen-lab rollout status deploy/web-blue
kubectl -n bluegreen-lab rollout status deploy/web-green
kubectl -n bluegreen-lab get deploy,pods,svc -L color
```

## 3. Verify who is serving

Port-forward in a separate terminal:

```bash
kubectl -n bluegreen-lab port-forward svc/web 8080:80
```

Then:

```bash
curl -s localhost:8080 | grep -E "BLUE|GREEN|NGINX|APACHE"
curl -sI localhost:8080 | grep -i "^server"     # nginx/1.27.x  or  Apache/2.4.x
```

Or test from inside the cluster (no port-forward):

```bash
kubectl -n bluegreen-lab run curl --rm -it --restart=Never \
  --image=curlimages/curl -- curl -sI http://web
```

NodePort is also available at `http://<node-ip>:30080`.

## 4. Switch to GREEN (Apache)

```bash
kubectl apply -k overlays/green
kubectl -n bluegreen-lab get svc web -o jsonpath="{.spec.selector}{\"\n\"}"
kubectl -n bluegreen-lab get endpoints web
curl -s localhost:8080 | grep -E "BLUE|GREEN"
```

Note: `kubectl port-forward` pins to one pod when it starts. After switching,
stop it (Ctrl+C) and start it again, or test via the in-cluster curl / NodePort.

## 5. Roll back to BLUE (NGINX)

```bash
kubectl apply -k overlays/blue
curl -s localhost:8080 | grep -E "BLUE|GREEN"
```

## Handy one-liners

Watch the live server while you switch (use NodePort or in-cluster curl):

```bash
while true; do curl -sI http://<node-ip>:30080 | grep -i "^server"; sleep 1; done
```

Show which color the Service points to:

```bash
kubectl -n bluegreen-lab get svc web -o jsonpath="{.spec.selector.color}{\"\n\"}"
```

Quick switch without kustomize (same effect, but drifts from Git):

```bash
kubectl -n bluegreen-lab patch svc web \
  -p "{\"spec\":{\"selector\":{\"app\":\"web\",\"color\":\"green\"}}}"
```

## Cleanup

```bash
kubectl delete -k overlays/blue
# or: kubectl delete ns bluegreen-lab
```

## How it works

- `base/` defines `web-blue` (NGINX) and `web-green` (Apache), each with its own
  `color` label and its own HTML served from a ConfigMap.
- `base/service.yaml` selects `app=web, color=blue` by default.
- Each overlay applies a JSON patch that replaces `/spec/selector/color`.
- `kubectl apply -k overlays/<color>` therefore flips only the Service selector.
