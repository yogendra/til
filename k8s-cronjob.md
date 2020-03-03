# Running Cronjob in K8s


## Simple CronJob
Running cronjob is straghtforwardin kubernetes. Just create one using a `CronJob` object. Example: 


```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

**OR** you can create same thing with following command:

```bash
kubectl run hello --restart=OnFailure --schedule "*/1 * * * *" --image=busybox -- /bin/sh -c "date; echo Hello from the Kubernetes cluster"
```


## CronJob for Managing K8s

So I was asked this question on how to run a command on another pod, regularly? Thought about it. Apparatly there is nothing on the k8s to allow this natively. 

I read thrrough regular places (so, google, k8s docs, etc.) and stumbled upon bit and pieces of it all. 

Turns out I need to execute kubectl in the job to do what I need. 

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: nginx-appender
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: cronjob
          containers:
            - name: hello
              image: bitnami/kubectl
              args:
                - exec
                - pod/nginx-5578584966-982k6
                - --
                - sh
                - -c
                - "date >>  /usr/share/nginx/html/index.html"
          restartPolicy: OnFailure
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cronjob-role
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create"]

---
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: null
  name: cronjob
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: cronjob-rolebinding
subjects:
  - kind: ServiceAccount
    name: cronjob
roleRef:
  kind: Role
  name: cronjob-role
  apiGroup: ""
```

* Create a service account (`cronjob`)
* Create a role to allow`pods::get, pods/exec::create` (`cronjob-role`)
* Bind account to role (`cronjob-rolebinding`)
* Create job is service account (`cronjob`)



