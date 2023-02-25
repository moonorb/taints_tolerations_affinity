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
Worker[1,3] tainted.
Worker3 without taint
No Toleration

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
