# Kserve Demonstration for Iris model

### Install Cert Manager

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

### Install KServe CRDs

```
kubectl create namespace kserve

helm install kserve-crd oci://ghcr.io/kserve/charts/kserve-crd \
  --version v0.16.0 \
  -n kserve \
  --wait
```

### Install KServe controller

```
helm install kserve oci://ghcr.io/kserve/charts/kserve \
  --version v0.16.0 \
  -n kserve \
  --set kserve.controller.deploymentMode=RawDeployment \
  --wait
```

### Deploy the sklearn iris model

```
kubectl create namespace ml

cat <<EOF | kubectl apply -n ml -f -
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: sklearn-iris
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: "gs://kfserving-examples/models/sklearn/1.0/model"
      resources:
        requests:
          cpu: "100m"
          memory: "512Mi"
        limits:
          cpu: "1"
          memory: "1Gi"
EOF

kubectl get inferenceservice sklearn-iris -n ml
```

### Port-forward to access the model

```
kubectl -n ml port-forward svc/sklearn-iris-predictor 8080:80
```

### Inference the Model

```
curl -s -H "Content-Type: application/json" \
  -d '{"instances":[[5.9,3.0,5.1,1.8]]}' \
  http://localhost:8080/v1/models/sklearn-iris:predict
```

### Cleanup

```
kubectl delete inferenceservice sklearn-iris -n ml
helm uninstall kserve -n kserve
helm uninstall kserve-crd -n kserve
kubectl delete ns ml kserve
```

