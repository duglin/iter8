---
template: main.html
---

# SLO Validation

!!! tip "Scenario: SLO validation with progressive traffic shift"
    This tutorial illustrates an [SLO validation experiment with two versions](../../../concepts/buildingblocks.md#slo-validation); the candidate version will be promoted after Iter8 validates that it satisfies service-level objectives (SLOs). You will:

    1. Specify *latency* and *error-rate* based service-level objectives (SLOs). If the candidate version satisfies SLOs, Iter8 will declare it as the winner.
    2. Use Prometheus as the provider for latency and error-rate metrics.
    3. Combine SLO validation with [progressive traffic shifting](../../../concepts/buildingblocks.md#progressive-traffic-shift).
    
    ![SLO validation with progressive traffic shift](../../../images/slovalidationprogressive.png)

???+ warning "Platform setup"
    Follow [these steps](../platform-setup.md) to install Iter8 and Istio in your K8s cluster.    

## Steps 1 to 3
Follow [Steps 1 to 3 of the Iter8 quick start tutorial](../quick-start.md).

## 4. Launch experiment
Launch the SLO validation experiment.
```shell
kubectl apply -f $ITER8/samples/istio/slovalidation/experiment.yaml
```

??? info "Look inside experiment.yaml"
    ```yaml linenums="1"
    apiVersion: iter8.tools/v2alpha2
    kind: Experiment
    metadata:
      name: slovalidation-exp
    spec:
      # target identifies the service under experimentation using its fully qualified name
      target: bookinfo-iter8/productpage
      strategy:
        # this experiment will perform a Canary test
        testingPattern: Canary
        # this experiment will progressively shift traffic to the winning version
        deploymentPattern: Progressive
        actions:
          # when the experiment completes, promote the winning version using kubectl apply
          finish:
          - if: CandidateWon()
            run: kubectl -n bookinfo-iter8 apply -f https://raw.githubusercontent.com/iter8-tools/iter8/master/samples/istio/quickstart/vs-for-v2.yaml
          - if: not CandidateWon()
            run: kubectl -n bookinfo-iter8 apply -f https://raw.githubusercontent.com/iter8-tools/iter8/master/samples/istio/quickstart/vs-for-v1.yaml
      criteria:
        objectives: # metrics used to validate versions
        - metric: iter8-istio/mean-latency
          upperLimit: 300
        - metric: iter8-istio/error-rate
          upperLimit: "0.01"
        requestCount: iter8-istio/request-count
      duration: # product of fields determines length of the experiment
        intervalSeconds: 10
        iterationsPerLoop: 5
      versionInfo:
        # information about the app versions used in this experiment
        baseline:
          name: productpage-v1
          variables:
          - name: namespace # used in metric queries
            value: bookinfo-iter8
          weightObjRef:
            apiVersion: networking.istio.io/v1beta1
            kind: VirtualService
            namespace: bookinfo-iter8
            name: bookinfo
            fieldPath: .spec.http[0].route[0].weight
        candidates:
        - name: productpage-v2
          variables:
          - name: namespace # used in metric queries
            value: bookinfo-iter8
          weightObjRef:
            apiVersion: networking.istio.io/v1beta1
            kind: VirtualService
            namespace: bookinfo-iter8
            name: bookinfo
            fieldPath: .spec.http[0].route[1].weight
    ```

## 5. Observe experiment
Follow [these steps](../../../getting-started/first-experiment.md#3-observe-experiment) to observe your experiment.

## 6. Cleanup
```shell
kubectl delete -f $ITER8/samples/istio/quickstart/fortio.yaml
kubectl delete -f $ITER8/samples/istio/slovalidation/experiment.yaml
kubectl delete namespace bookinfo-iter8
```
