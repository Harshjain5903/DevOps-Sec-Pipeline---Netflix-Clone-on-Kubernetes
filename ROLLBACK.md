# Rollback

Each Jenkins build pushes `harshjain5903/netflix:<BUILD_NUMBER>` (plus `:latest`)
and sets that tag on the deployment, so every release shows up in rollout history.

## Check what's running

```bash
kubectl rollout status deployment/netflix-app
kubectl rollout history deployment/netflix-app
kubectl get deployment netflix-app -o jsonpath='{.spec.template.spec.containers[0].image}'
```

## Roll back to the previous release

```bash
kubectl rollout undo deployment/netflix-app
kubectl rollout status deployment/netflix-app
```

## Roll back to a specific release

```bash
kubectl rollout history deployment/netflix-app          # find the revision
kubectl rollout undo deployment/netflix-app --to-revision=<rev>
```

or pin a known good image directly:

```bash
kubectl set image deployment/netflix-app netflix-app=harshjain5903/netflix:<build-number>
```

## After rolling back

- Check the app on `NodeIP:30007` and the pods are Ready (`kubectl get pods -l app=netflix-app`).
- Readiness probe keeps bad pods out of the service, so if a rollout hangs, look at
  `kubectl describe pod <pod>` before rolling back.
- Note the bad build number and why it was rolled back before the next deploy.
