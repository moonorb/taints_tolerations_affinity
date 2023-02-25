# taints_tolerations_affinity
The aim of this exercise is to practice taints/tolerations/affinity by applying with a simple example.

There is 1 master and 3 workers in our K8s cluster. 
For the purpose of testing:
- worker1->back-api-redis(backend)
- worker2->front-api-flask(frontend)
- worker3->everything else

Our sample application consist of 2 deployments where each has 3 replicas with a total of 6 pods. Deployments are redis/backend and flask-api/frontend. 
Normally when deployed 6 pods should be randomly distributed to 2 workers but we will modify the deployments so the backend will be on worker1 and frontend will be scheduled on worker2.

![screenshot-1-1](https://user-images.githubusercontent.com/46006590/221326246-7c1fbcb5-858f-40ab-b238-87dadcb81765.PNG)

### Scenario-1
- Worker[1,2] tainted.
- Worker3 without taint
- No Toleration

If any of the workers were without any taint, all the pods would be scheduled on that node.  
All the pods getting scheduled on worker3 because its the only one withouto taint.  

```
kubectl taint nodes worker1.moonorb.cloud workload=DBgroup:NoSchedule
kubectl taint nodes worker2.moonorb.cloud workload=DBgroup:NoSchedule
```
```
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints --no-headers
master1.moonorb.cloud   [map[effect:NoSchedule key:node-role.kubernetes.io/control-plane]]
worker1.moonorb.cloud   [map[effect:NoSchedule key:workload value:DBgroup]]
worker2.moonorb.cloud   [map[effect:NoSchedule key:workload value:DBgroup]]
worker3.moonorb.cloud   <none>
```
```
kubectl create -f redis.deployment.yaml  -nflask
kubectl create -f frontend.deployment.yaml  -nflask
```

```

k get po -n flask -o wide
NAME                         READY   STATUS              RESTARTS   AGE   IP       NODE                 NOMINATED NODE   READINESS GATES
back-api-5b595dc89f-jns58    0/1     ContainerCreating   0          51s   <none>   worker3.moonorb.cloud   <none>           <none>
back-api-5b595dc89f-rsd9g    0/1     ContainerCreating   0          51s   <none>   worker3.moonorb.cloud   <none>           <none>
back-api-5b595dc89f-tlbpm    0/1     ContainerCreating   0          51s   <none>   worker3.moonorb.cloud   <none>           <none>
front-api-6549c65dc9-dcfwp   0/1     ContainerCreating   0          51s   <none>   worker3.moonorb.cloud   <none>           <none>
front-api-6549c65dc9-g2mn2   0/1     ContainerCreating   0          51s   <none>   worker3.moonorb.cloud   <none>           <none>
front-api-6549c65dc9-mx42n   0/1     ContainerCreating   0          51s   <none>   worker3.moonorb.cloud   <none>           <none>
```
---------------------------------------------------------------------------------------------------------
### Scenario-2
- Worker[1,2,3] tainted.
- No Toleration.
```
kubectl taint nodes worker3.moonorb.cloud workload=all_else:NoSchedule
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints --no-headers
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints --no-headers
master1.moonorb.cloud   [map[effect:NoSchedule key:node-role.kubernetes.io/control-plane]]
worker1.moonorb.cloud   [map[effect:NoSchedule key:workload value:DBgroup]]
worker2.moonorb.cloud   [map[effect:NoSchedule key:workload value:DBgroup]]
worker3.moonorb.cloud   [map[effect:NoSchedule key:workload value:all_else]]
k delete -f . -nflask
k create -f . -nflask
``` 
Inless pods have a matched toleration, they can't be scheduled to any of the workers due to the taint. 
``` 
k get po -n flask -o wide
NAME                         READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
back-api-5b595dc89f-jns58    0/1     Pending   0          6s    <none>   <none>   <none>           <none>
back-api-5b595dc89f-rsd9g    0/1     Pending   0          6s    <none>   <none>   <none>           <none>
back-api-5b595dc89f-tlbpm    0/1     Pending   0          6s    <none>   <none>   <none>           <none>
front-api-6549c65dc9-dcfwp   0/1     Pending   0          6s    <none>   <none>   <none>           <none>
front-api-6549c65dc9-g2mn2   0/1     Pending   0          6s    <none>   <none>   <none>           <none>
front-api-6549c65dc9-mx42n   0/1     Pending   0          6s    <none>   <none>   <none>           <none>
``` 
---------------------------------------------------------------------------------------------------------
### Scenario-3
- Worker[1,2,3] still all tainted.
- Only Frontend deployment has Toleration with Operator:Equal

Add toleration to one of the deployments(frontend) so the pods CAN be scheduled to the workers. But which one? 

![screenshot-2-2](https://user-images.githubusercontent.com/46006590/221327103-2bf40e0b-1af0-4348-9fa1-adf6a34f9dd5.PNG)

After recreating both deployments only frontend is deployed but not on any particular worker. Its distributed randomly between worker1 and worker2 only but not on worker3.
Why? because the toleration key for worker3 was "all_else" which does not match what we have in the deployment config "DBgroup" which matches with taint value for worker1 and worker2. 
``` 
john@master1:~/x/flask$ k get po -n flask -o wide
NAME                         READY   STATUS    RESTARTS   AGE   IP               NODE                 NOMINATED NODE   READINESS GATES
back-api-5b595dc89f-chh5m    0/1     Pending   0          8s    <none>           <none>               <none>           <none>
back-api-5b595dc89f-jm6x7    0/1     Pending   0          8s    <none>           <none>               <none>           <none>
back-api-5b595dc89f-mvpcp    0/1     Pending   0          8s    <none>           <none>               <none>           <none>
front-api-7454995d66-2gspd   1/1     Running   0          8s    172.16.108.205   worker2.moonorb.cloud   <none>           <none>
front-api-7454995d66-9lssd   1/1     Running   0          8s    172.16.251.134   worker1.moonorb.cloud   <none>           <none>
front-api-7454995d66-mccmd   1/1     Running   0          8s    172.16.108.206   worker2.moonorb.cloud   <none>           <none>
``` 
Summary: All 3 Nodes->Tainted, Pods which belonged to Frontend Deployment tolerated the taint but Redis did not have toleration therefore it was not scheduled anywhere. 
---------------------------------------------------------------------------------------------------------
### Scenario4
- Worker[1,2,3] still all tainted. 
- Frontend deployment has Toleration(Equal)
- Backend deployment also has Toleration(Exists)

We add Toleration to Redis/Backend deployment in a slightly different way where the key:value pair of the toleration is not explicitly defined as before but only key(workload) is defined with an operator of "Equal" which is sufficient. 

![screenshot-3-3](https://user-images.githubusercontent.com/46006590/221327114-331f9ead-3d40-451f-912d-a40ea0fa8620.PNG)

After recreating the pods they are all scheduled but randomly. In We can see now back-api pods are also scheduled to worker3 because we did not specify the value so its acceptable. 
``` 
john@master1:~/x/flask$ k get po -n flask -o wide
NAME                         READY   STATUS    RESTARTS   AGE   IP               NODE                 NOMINATED NODE   READINESS GATES
back-api-7bd4bbb6f-s8l7t     1/1     Running   0          2s    172.16.238.27    worker3.moonorb.cloud   <none>           <none>
back-api-7bd4bbb6f-wmqkc     1/1     Running   0          2s    172.16.108.207   worker2.moonorb.cloud   <none>           <none>
back-api-7bd4bbb6f-zxgbp     1/1     Running   0          2s    172.16.251.136   worker1.moonorb.cloud   <none>           <none>
front-api-7454995d66-249nq   1/1     Running   0          2s    172.16.108.208   worker2.moonorb.cloud   <none>           <none>
front-api-7454995d66-8kcj6   1/1     Running   0          2s    172.16.251.137   worker1.moonorb.cloud   <none>           <none>
front-api-7454995d66-xpdz7   1/1     Running   0          2s    172.16.251.135   worker1.moonorb.cloud   <none>           <none>
``` 
All the pods have tolerants but we are all over the place next step is to make sure back-api is ONLY scheduled to worker1 and worker2 and Frontend is scheduled to worker3. 
---------------------------------------------------------------------------------------------------------
### Scenario-5
- Worker[1,2,3] tainted.
- Both deployments have Tolerations(operator:Exists).
- Nodes Worker[1,2] labelled. Worker3 has no label.
- Backend deployment has affinity with key value pair of workload=database. 

```
kubectl get nodes --show-labels (no labels)
kubectl label nodes worker1.moonorb.cloud workload=database
kubectl label nodes worker2.moonorb.cloud workload=api
kubectl get nodes -l workload

ohn@master1:~/x/flask$ kubectl get nodes -l workload
NAME                 STATUS   ROLES    AGE     VERSION
worker1.moonorb.cloud   Ready    <none>   4d23h   v1.25.0
worker2.moonorb.cloud   Ready    <none>   3h36m   v1.25.0
```
In order to utilize node labels we have to add affinity configuration to the deployment. Add the config below to ONLY backend deployment.  

```
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: workload
          operator: In
          values:
          - database
```

![screenshot-4](https://user-images.githubusercontent.com/46006590/221327308-1dfd95c4-8b35-4fa7-8be2-b8abb6a04cb4.PNG)

While all the backend pods are scheduled to worker1 frontapi pods are distributed randomly across all workers. 
```
john@master1:~/x/flask$ k get po -n flask -o wide
NAME                         READY   STATUS    RESTARTS   AGE   IP               NODE                 NOMINATED NODE   READINESS GATES
back-api-79c6874546-c94f7    1/1     Running   0          3s    172.16.251.165   worker1.moonorb.cloud   <none>           <none>
back-api-79c6874546-cjt2g    1/1     Running   0          3s    172.16.251.168   worker1.moonorb.cloud   <none>           <none>
back-api-79c6874546-sdtgp    1/1     Running   0          3s    172.16.251.167   worker1.moonorb.cloud   <none>           <none>
front-api-5b46fffb7c-dkdms   1/1     Running   0          3s    172.16.238.30    worker3.moonorb.cloud   <none>           <none>
front-api-5b46fffb7c-shbqt   1/1     Running   0          3s    172.16.108.219   worker2.moonorb.cloud   <none>           <none>
front-api-5b46fffb7c-w4sl6   1/1     Running   0          3s    172.16.251.166   worker1.moonorb.cloud   <none>           <none>
```

### Scenario-6

- Worker[1,2,3] still all tainted. 
- Both deployments have Tolerations(operator:Exists). 
- Worker[1,2] nodes labelled. Worker3 has no label. 
- Both deployments have affinity configuration to match the node labels. 

![screenshot-5](https://user-images.githubusercontent.com/46006590/221327373-9768058c-3e7c-4853-a3b0-8d10125d5bde.PNG)

```
kubectl get nodes --show-labels

john@master1:~/x/flask$ kubectl get nodes --show-labels
NAME                 STATUS   ROLES           AGE     VERSION   LABELS
master1.moonorb.cloud   Ready    control-plane   5d1h    v1.25.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master1.moonorb.cloud,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
worker1.moonorb.cloud   Ready    <none>          5d1h    v1.25.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker1.moonorb.cloud,kubernetes.io/os=linux,workload=database
worker2.moonorb.cloud   Ready    <none>          5d1h    v1.25.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker2.moonorb.cloud,kubernetes.io/os=linux,workload=api
worker3.moonorb.cloud   Ready    <none>          5h36m   v1.25.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker3.moonorb.cloud,kubernetes.io/os=linux
```
```
john@master1:~/x/flask$ k get po -n flask -o wide
NAME                         READY   STATUS    RESTARTS   AGE     IP               NODE                 NOMINATED NODE   READINESS GATES
back-api-79c6874546-c94f7    1/1     Running   0          7m25s   172.16.251.165   worker1.moonorb.cloud   <none>           <none>
back-api-79c6874546-cjt2g    1/1     Running   0          7m25s   172.16.251.168   worker1.moonorb.cloud   <none>           <none>
back-api-79c6874546-sdtgp    1/1     Running   0          7m25s   172.16.251.167   worker1.moonorb.cloud   <none>           <none>
front-api-67699cc7cf-ct27j   1/1     Running   0          5s      172.16.108.220   worker2.moonorb.cloud   <none>           <none>
front-api-67699cc7cf-dhcsx   1/1     Running   0          3s      172.16.108.222   worker2.moonorb.cloud   <none>           <none>
front-api-67699cc7cf-qf25p   1/1     Running   0          4s      172.16.108.221   worker2.moonorb.cloud   <none>           <none>
```

![taints-affinity](https://user-images.githubusercontent.com/46006590/221327443-6c536239-9a1c-443a-80ec-5fd06224fe77.PNG)


