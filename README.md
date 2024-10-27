# Encrypting Confidential Data at Rest – Kubernetes

## Introduction

Encryption at rest ensures that sensitive data stored in Kubernetes clusters is secure and protected from unauthorized access. By encrypting data before writing it to disk, you can safeguard against potential data breaches. This guide explains how to enable encryption at rest for Kubernetes clusters.

## Before You Begin

- Ensure you have a **Kubernetes cluster** set up.
- The **kubectl** command-line tool should be configured to communicate with your cluster.

## Determine if Encryption at Rest is Enabled

By default, the Kubernetes API server stores resources as plain-text in etcd, without encryption. To check if encryption is enabled, verify the presence of an `--encryption-provider-config` argument in the `kube-apiserver` manifest:

```bash
controlplane $ cd /etc/kubernetes/manifests/
controlplane $ ls
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
controlplane $ cat kube-apiserver.yaml | grep "encryption-provider-config"
```
![image](https://github.com/user-attachments/assets/8faf8135-3156-441f-9bee-064a2fcfb65e)

## Create and Store a Secret
### To test encryption, create a sample secret and store it:
```bash
controlplane $ kubectl create secret generic mysecret --from-literal=mykey1=password
controlplane $ kubectl get secrets
controlplane $ kubectl describe secret mysecret
```
![image](https://github.com/user-attachments/assets/c95760ee-4f9a-40b0-970e-71e584b9e0c6)

## Read the Stored Secret in ETCD Without Encryption
### Use the etcdctl command to view the stored secret:
```bash
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/mysecret | hexdump -C
```
![image](https://github.com/user-attachments/assets/f25383cc-1ce5-428f-bd06-3ea36bb29ee9)

## Generate a New Encryption Key
### Generate a 32-byte random key and encode it using base64:
```bash
head -c 32 /dev/urandom | base64
```

![image](https://github.com/user-attachments/assets/9b49678a-6e29-447d-9201-b0b34bd6cb4e)

## Write an Encryption Configuration File
### Create an encryption configuration file (enc.yaml) as follows:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              # Copy the base64-encoded key generated from the previous step
              secret: <BASE64_ENCODED_SECRET>
      - identity: {} # Allows reading unencrypted secrets during migration
```
![image](https://github.com/user-attachments/assets/1c2c3edb-b92f-458b-8704-34a21744b519)

## Use the New Encryption Configuration File
### Mount the new encryption configuration file to the kube-apiserver static pod:

### Save enc.yaml to /etc/kubernetes/enc/enc.yaml on the control-plane node.
### Edit the kube-apiserver manifest located at /etc/kubernetes/manifests/kube-apiserver.yaml:

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    ...
    - --encryption-provider-config=/etc/kubernetes/enc/enc.yaml  # add this line
    volumeMounts:
    ...
    - name: enc                           # add this line
      mountPath: /etc/kubernetes/enc      # add this line
      readOnly: true                      # add this line
  ...
  volumes:
  ...
  - name: enc                             # add this line
    hostPath:                             # add this line
      path: /etc/kubernetes/enc           # add this line
      type: DirectoryOrCreate             # add this line
  ...
```
### Wait till restart happens of API server.

## Verify That Newly Written Data is Encrypted
### After restarting kube-apiserver, create a new secret to ensure encryption is working:

```bash
kubectl create secret generic secret1 -n default --from-literal=mykey=mydata
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/secret1 | hexdump -C
```
![image](https://github.com/user-attachments/assets/cc1799cd-f9fc-4dfd-b812-a69017c7936c)

## Verify the Secret is Decrypted via the API
```bash
kubectl get secret secret1 -n default -o yaml
```
![image](https://github.com/user-attachments/assets/e8698196-951c-4b14-9a89-6174ccf26ae8)

## Ensure All Relevant Data are Encrypted
### Run this command as an administrator to re-encrypt existing secrets:
```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```
![image](https://github.com/user-attachments/assets/9cc93f2c-63f0-49e1-adf9-cbfeccbd0ade)

## Conclusion
### By following these steps, you can enable encryption at rest in Kubernetes, ensuring that sensitive data stored in your clusters remains secure. Proper encryption configurations help in meeting compliance requirements and strengthen the overall security posture of your systems.

```vbnet
This guide includes step-by-step instructions for checking, setting up, and verifying encryption at rest in a Kubernetes environment.
```
