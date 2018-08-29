# deploy a webhook as a kubernetes service

This is a cluster admin instruction about how to deploy a webhook as a kubernetes service.
It's recommend to read [this document](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) before you going on.

## Prepare the node(s)

1. Kubernetes 1.9.0 or above with the admissionregistration.k8s.io/v1beta1 API enabled.

```shell
kubectl api-versions | grep admissionregistration.k8s.io/v1beta1
# The result should be:
admissionregistration.k8s.io/v1beta1
```

2. lable the node

```shell
kubectl label nodes nodeName webhook=true
```

3. install godep if you need to develop


```shell
go get -u -v github.com/tools/godep`
```

## Prepare the docker image

1. build the docker image

```shell
make build-container
```

2. push the docker image to REPOSITORY

```shell
docker push chaowang95/mutating-webhook:v0.1
```

**Note that before pushing the image, you may need to tag the image with your name. e.g. `docker tag <local-image-name> <your-username>/mutating-webhook:v0.1`**

## Prepare Certificates

run the following command:

```shell
$ cd gencerts
$ ./make-ca-cert.sh "webhook.default.svc" # in such format: <service-name>.<namespace>.svc
$ ls output
ca.crt  client.crt  client.key  server.cert  server.key
```

For more information about certificates: <https://kubernetes.io/docs/concepts/cluster-administration/certificates/>

## Config the kube-apiserver

1. enable the MutatingAdmissionWebhook admission controller, add the follow options to kube-apiserver

```
--enable-admission-plugins=...,MutatingAdmissionWebhook,... --admission-control-config-file=/home/chaowang/gocode/src/k8s.io/Example-AdmissionWebhook/admission_config.yaml
```

2. in the file admission-config.yaml, you should have:

```yaml
apiVersion: apiserver.k8s.io/v1alpha1
kind: AdmissionConfiguration
plugins:
- name: MutatingAdmissionWebhook
  configuration:
    apiVersion: apiserver.config.k8s.io/v1alpha1
    kind: WebhookAdmission
    kubeConfigFile: /home/chaowang/gocode/src/k8s.io/Example-AdmissionWebhook/webhook_admission_config.yaml
# other admission configs
# ...
```

3. in the file webhook_admission_config.yaml, you should have:

```yaml
apiVersion: v1
kind: Config
users:
- name: "webhook.default.svc" # replace this with "*" or "<service-name>.<namespace>.svc"
  user:
    client-certificate: /home/chaowang/gocode/src/k8s.io/Example-AdmissionWebhook/gencerts/output/client.crt  # this is the client.crt you generated before
    client-key: /home/chaowang/gocode/src/k8s.io/Example-AdmissionWebhook/gencerts/output/client.key  # this is the client.key you generated before
```

4. start the kube-apiserver

## Create a service for webhook admission controller

kubectl create -f mutatingwebhook_service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webhook
spec:
  ports:
  - port: 443 #service暴露在cluster ip上的端口，通过<cluster ip>:port访问服务
    targetPort: 443 #Pod的外部访问端口，port和nodePort的数据通过这个端口进入到Pod内部，Pod里面的containers的端口映射到这个端口，提供服务
  clusterIP: 20.1.0.2
    app: admission-webhook
```

## deploy the webhook

1. create the secret

```shell
kubectl create secret generic webhook-pod-secret --from-file=ca.crt=/home/chaowang/gocode/src/k8s.io/Example-AdmissionWebhook/gencerts/output/ca.crt --from-file=server.key=/home/chaowang/gocode/src/k8s.io/Example-AdmissionWebhook/gencerts/output/server.key --from-file=server.cert=/home/chaowang/gocode/src/k8s.io/Example-AdmissionWebhook/gencerts/output/server.cert
# note that the ca.crt,server.key and server.cert are generated before
```

2. create the deployment

kubectl create -f mutatingwebhook_deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admission-webhook
  labels:
    app: admission-webhook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: admission-webhook
  template:
    metadata:
      labels:
        app: admission-webhook
    spec:
      volumes:
      - name: secret-volume-certs
        secret:
          secretName: webhook-pod-secret
      containers:
      - name: admission-webhook-container
        image: chaowang95/mutating-webhook:v0.1
        imagePullPolicy: IfNotPresent
        command: ["/admission-webhook", "--v=10", "--alsologtostderr", "--client-ca-file=/etc/secret-volume/ca.crt", "--tls-cert-file=/etc/secret-volume/server.cert", "--tls-private-key-file=/etc/secret-volume/server.key"]
        volumeMounts:
        - name: secret-volume-certs
          readOnly: true
          mountPath: "/etc/secret-volume"
        ports:
          - containerPort: 443
```

## make MutatingWebhookConfiguration object in kube-apiserver

kubectl create -f mutatingwebhook_configuration.yaml

```yaml
apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingWebhookConfiguration
metadata:
  name: coredump
webhooks:
- clientConfig:
    # replace this with your caBundle
    # use this command to get it:
    # cat gencerts/output/ca.crt | base64 | tr '\n' '\0'
    caBundle: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURYRENDQWtTZ0F3SUJBZ0lKQUxaSmo5RGxFMCtYTUEwR0NTcUdTSWIzRFFFQkN3VUFNQ014SVRBZkJnTlYKQkFNTUdERXdMakUyTnk0eE16TXVNek5BTVRVeU9EZzFORE16T1RBZUZ3MHhPREEyTVRNd01UUTFNemxhRncweQpPREEyTVRBd01UUTFNemxhTUNNeElUQWZCZ05WQkFNTUdERXdMakUyTnk0eE16TXVNek5BTVRVeU9EZzFORE16Ck9UQ0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ0VCQU1BbjlLd2EvRVplUm9VcjhIaDcKLys2R3p2MllNdTljUEtuZ3JIYnM0aTVLcXhrRXdIT094QTdRdVdMUnRoTFlSay9xcWdJNUJyRDByMUJFVEFPdApGd1ZoMlFOV2hJYVEvUUFaaCtEWFFiM3V5RkFOUlpkTTNJZ0FNZDM3VUZLbFh0MDhPMzJ4eUFtOUhKa0VCbGJOCndhWXpnR01sYUZnZnFQajlWdGFYRVhjK3Jxd2p4MjFvM29lWkVCaEg3czMvMjFsS2ZycURORWt1NWpLeXdYSTcKY1JQK0JWR3JLaWphd1V0RGxZTktqeVo0allVdlRCMFR6YmVDYTNLT21IcTF4bUozeXJVUTcwdGFuOGs5VXRSdgpHa1dFYW5zM2dsK3psR0Q2dzA2NWpJWDkyeXdscEs3ajY2WUlBSGJySUdrMUJyUDJVSDR0emtyZy95SFFhc3VvCnJDa0NBd0VBQWFPQmtqQ0JqekFkQmdOVkhRNEVGZ1FVb3NSQUEzTmFIRm1nNDFhT0RQU3R0SHdFeDZJd1V3WUQKVlIwakJFd3dTb0FVb3NSQUEzTmFIRm1nNDFhT0RQU3R0SHdFeDZLaEo2UWxNQ014SVRBZkJnTlZCQU1NR0RFdwpMakUyTnk0eE16TXVNek5BTVRVeU9EZzFORE16T1lJSkFMWkpqOURsRTArWE1Bd0dBMVVkRXdRRk1BTUJBZjh3CkN3WURWUjBQQkFRREFnRUdNQTBHQ1NxR1NJYjNEUUVCQ3dVQUE0SUJBUUJVd2x0Z25INnVGUEZhQXFWQ2hneDkKSUhBR2RLSzM0SUVaQlZIVDhqYnVxRXJFTUdVbFZ0V21LSkIyQ29COXEyZUdRMUdDSlBOKzRhRFFORzRjOENpdQpzOVNMZFJ4bXYyRnBIMkRMQTNNZGtQTThra0xMcWRyK1BzNklUei92NEUwK3FMK3lQZE50SjdacVozMjRIeEZmCnFEVzJaUFJWRm96cW5wUVZoRHMyNEJWdklCTnpkbno0K0ZNSzRVVjUyblRaZTh2S0hGVzBLdGI0bWU4Q29NRWEKY05ENElZTzF6S2hJNWF2a3NPbG1hdzE2ZW1wNEs3aUFBSTRueFFWSi84UVZJOGRLY2NINmxBQTJmQ3AvOVdEVQorV1lRV1BRRjl1c2o0MVVMYitIMWtUcjdJcUNIVHREQVU2SERRcFdyWXkrOUpQR243Zmt1YXUvb2lXV2MySGZGCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    service:
      namespace: default
      name: webhook
  failurePolicy: Ignore
  name: mutatinggwebhook.fujitsu.com
  rules:
  - apiGroups:
    - ""
    apiVersions:
    - v1
    operations:
    - CREATE
    resources:
    - pods

```
