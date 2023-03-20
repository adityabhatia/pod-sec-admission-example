# pod-sec-admission-example

Pod Security Admission (PSA) example.

1. Create `kind` cluster with 2 nodes    
    ```kind create cluster --config=./cluster-config.yaml```

2. Apply policy at NS level for a custom namespace
   ```
        kubectl create namespace psa
   
        kubectl label --overwrite ns psa \
        pod-security.kubernetes.io/warn=baseline \
        pod-security.kubernetes.io/warn-version=v1.25
    ```

3. Try enforcing `baseline` policy on all namespaces. The following command will notify the workloads which violate the policy across namespaces.
   ```
    kubectl label --dry-run=server --overwrite ns --all \
    pod-security.kubernetes.io/enforce=baseline
    ```   

4. Create another custom namespace with multiple policies. 
The following command will not allow pods which violate the `baseline` policy and warn / audit for pods which violate the `restricted` policy.
   ```
        kubectl create namespace psa-2
   
        kubectl label --overwrite ns psa-2 \
         pod-security.kubernetes.io/enforce: baseline \
         pod-security.kubernetes.io/audit: restricted \
         pod-security.kubernetes.io/warn: restricted
    ```

5. Create a pod in the namespace `psa-2`. 
Since the pod doesn't violate the baseline policy it will be allowed. 
However, it will warn / add to audit log, since it violates the restricted policy.
   ```kubectl apply -f pod.yaml```

6. To apply a cluster-wide policy create the following resource as an example and refer it via the `--admission-control-config-file` flag on the API server.
    ```
    cat admission_configuration.yaml
    apiVersion: apiserver.config.k8s.io/v1
    kind: AdmissionConfiguration
    plugins:
    - name: DefaultPodSecurity
      configuration:
        apiVersion: pod-security.admission.config.k8s.io/v1alpha1
        kind: PodSecurityConfiguration
        defaults:
          enforce: "baseline"
          enforce-version: "latest"
          audit: "restricted"
          audit-version: "latest"
          warn: "restricted"
          warn-version: "latest"
        exemptions:
          usernames: []
          runtimeClassNames: []
          namespaces: [kube-system]
```