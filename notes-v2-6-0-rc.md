```
kind create cluster --name argo-cd-v2.6.0-rc
kubectl create namespace argocd
# kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.6.0-rc2/manifests/install.yaml
kubectl kustomize ./argo-cd | kubectl apply -n argocd -f -
kubectl apply -n argocd -f argo-cd/argo-cd-app.yaml
kubectl get pods -w -n argocd
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

https://localhost:8080/login

1. Multiple sources for Applications
    - Helm chart in one place, values in another.
    - https://argo-cd.readthedocs.io/en/release-2.6/user-guide/multiple_sources/
    - Must add directly to the cluster, can't use UI or CLI yet. Beta feature.
      ```
      kubectl apply -f ./helm-guestbook-values/multi-source-app.yaml
      ```
2. A Rollout Strategy for ApplicationSets
   - Deploy to dev, qa, prod namespaces.
   - https://argo-cd.readthedocs.io/en/release-2.6/operator-manual/applicationset/Progressive-Rollouts/
   - https://github.com/crenshaw-dev/appset-progressive
   - Must add directly to the cluster, can't use UI or CLI yet. Alpha feature.
     ```
     kubectl apply -f ./applicationsets/progressive-rollout.yaml
     ```
   - make change to image tag in helm values of app template.
3. Parameter support in CMPs
   - Deploy simple CMP with parameters.
   - https://github.com/argoproj/argo-cd/pull/9216
   - https://github.com/argoproj/argo-cd/blob/master/docs/proposals/parameterized-config-management-plugins.md