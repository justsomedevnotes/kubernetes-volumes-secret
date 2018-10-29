# Introduction
There are a couple ways to add secrets to containers in Kubernetes; as volumes, or as environmental variables.  This post describes adding the secret using volumes.  We will create a Kubernetes Secret and then add that secret to a pod as a volume mount.  One interesting bit of information I discovered while doing this is if the secret gets updated then the secret mounted in the container gets updated automatically as well.  This only happens if you use volumes and not environment variables to add secrets though.

## Create a Secret
There are several ways to create a secret in Kubernetes.  For this example we will be creating a secret in a yaml manifest.  In order to do this we need to encode the data in base64.  Below are commands to encode a user, 'jsmith', and a password, 'mysupersecurepassword'.

```console
echo -n "jsmith" | base64
```
```console
echo -n "mysupersecurepassword" | base64
```
The result of the previous commands need to be copied and pasted to the secret manifest.  The example should look something like the below.  The key 'user' will be the name of the file that is mounted in the Kubernetes container and the base64 encoded value will be decoded into the contents of the file inside the container.  
```console
kind: Secret
apiVersion: v1
metadata:
  name: my-secret
  namespace: default
type: Opaque
data:
  user: anNtaXRo
  password: bXlzdXBlcnNlY3VyZXBhc3N3b3Jk
``

Now create the secret.
```console
kubectl apply -f my-secret.yml
secret/my-secret configured
```

## Create the Pod
There are two sections that need to be added to the pod spec when mounting a secret volume; 1) volumes, 2) volumeMounts.  The volumes element is part of the pod.spec element and the volumeMounts is part of the pod.spec.containers[] element.  The example below illustrates adding a secret.
```console
kind: Pod
apiVersion: v1
metadata:
  name: my-pod
  namespace: default
spec:
  volumes:
  - name: vol-secret
    secret:
      secretName: my-secret
  containers:
  - name: c1
    image: busybox
    imagePullPolicy: IfNotPresent
    command:
    - sleep
    - '3600'
    volumeMounts:
    - name: vol-secret
      mountPath: /etc/app/secrets
  restartPolicy: Always
```
```console
kubectl apply -f my-pod.yml
```
Once the pod has been created exec into the pod to see the secrets.
```console
kubectl exec -it my-pod -- /bin/sh
```
```console
/ # ls /etc/app/secrets
password  user
/ # cat /etc/app/secrets/password
mysupersecurepassword
/ # cat /etc/app/secrets/user
jsmith
```

## Update the Secret
Occasionally a secret value will need to change.  There are various techniques that can be used to update secrets in pods; for example deployments, but I wanted to demostrate one feature that is built into Kubernetes.  If you mount a secret (ie..not used environment varialbes for secrets) if the secret changes it automagically gets updated in the container.  Lets demonstrate.
```console
echo -n "mydoublesecurepassword" | base64
```
Update the my-secret.yml manifest with the newly generated value and then apply the new file.
```console
kubectl apply -f my-secret.yml
secret/my-secret configured
```
Now exec into the running container (it might take a few seconds for the change to take place)
```console
kubectl exec -it my-pod -- /bin/sh
```
```console
/ # cat /etc/app/secrets/password
mydoublesecurepassword
```






