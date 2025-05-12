# setting up mysql with statefulset

## Configuration files

`mysql-configmap.yml`
- stores the mysql configration for primary and replica
- it controls whether it's a writer or reader
- replicas are configured to be read only

`mysql-service.yml`
- we define two services
- first is a headless service, it's just to get dns to each of the pod (e.g. `mysql-0.mysql`)
    - set need to set `clusterIP: None`
- second is a normal clusterIP which will load balance the read requests between all the pods.

`mysql-statefulset.yml`
- statefulset here
- init-mysql creates custom configuration for each mysql pod
    - it generates a server-id based on pod index
    - it index is 0, it applies the writer (primary) conf
    - else, it applies the reader conf
- clone-mysql copies data from existing pod, to prevent replication from scratch.
    - skip if it's writer pod
- there are two containers
    - **mysql**: pretty standard config, volumes, readiness, liveness, etc
    - **extrabackup**: a sidecar container that starts the replication in readers
- volumeClaimTemplates
    - tells kubernetes to allocate a 1GB storage to each pod

## Deploying

Simply do 
```sh
kubectl -f apply mysql-configmap.yml
kubectl -f apply mysql-service.yml
kubectl -f apply mysql-statefulset.yml

# get the pods and watch for changes
kubectl get pods -l app=mysql -w

# at the end you would see 
# mysql-0   2/2     Running       0          42m
# mysql-1   2/2     Running       0          42m
# mysql-2   2/2     Running       0          41m
```

## Testing shit out

There are two ways to execute command or start a interactive mysql client
```sh
# first, run a temporary container 
kubectl run mysql-client --image=mysql:5.7 -it --rm -- mysql -h mysql-read

# second, use exec -it and get into the running containers
kubectl exec -it mysql-0 mysql
```

Write some data in the writer node (mysql-0)
```sql
-- sql here
CREATE DATABASE hello;
CREATE TABLE hello.messages (message VARCHAR(250));
INSERT INTO messages VALUES ('hello world');

-- then check the table
SELECT * FROM hello.messages;

-- quit
exit
```

Then, check the reader nodes
```sql
SELECT @@hostname;

-- output is something like this:
-- +------------+
-- | @@hostname |
-- +------------+
-- | mysql-1    |
-- +------------+

-- if you exit and quit, you might get different hostname
-- mysql> SELECT @@hostname;
-- +------------+
-- | @@hostname |
-- +------------+
-- | mysql-0    |
-- +------------+

-- Check if the message is synced
SELECT * FROM hello.messages;
```

## Scale shit out
```sh
kubectl scale statefulset mysql --replicas=4

# check it out
kubectl get pods -l app=mysql -w
# mysql-0   2/2     Running   0          28m
# mysql-1   2/2     Running   0          28m
# mysql-2   2/2     Running   0          27m
# mysql-3   2/2     Running   0          3m54s

# note that even after you scale down
kubectl scale statefulset mysql --replicas=3
# the PV will still be there and it's expected
# next time you scale up again, it will reuse the PV
# NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
# pvc-0aa9651a-a64e-43df-ab24-f32eafe2e8c7   1Gi        RWO            Delete           Bound    default/data-mysql-0   longhorn       <unset>                          45m
# pvc-0acccbbd-e605-4298-84ac-f1a93a93ae14   1Gi        RWO            Delete           Bound    default/data-mysql-3   longhorn       <unset>                          21m
# pvc-0e0e9d15-f5ff-47ae-ab59-639431d7272c   1Gi        RWO            Delete           Bound    default/data-mysql-2   longhorn       <unset>                          44m
# pvc-f2b51130-d28e-4d81-9a4d-3ddb6f17d9a9   1Gi        RWO            Delete           Bound    default/data-mysql-1   longhorn       <unset>                          45m
```

## Delete the pod and see?
```sh
kubectl delete pod mysql-2

# it will get deleted
# but a replacement will be automatically created
# it will share the same name, and the same pv
# mysql-0   2/2     Running   0          44m
# mysql-1   2/2     Running   0          43m
# mysql-2   2/2     Running   0          43m
# mysql-2   2/2     Terminating   0          43m
# mysql-2   0/2     Completed     0          43m
# mysql-2   0/2     Completed     0          43m
# mysql-2   0/2     Completed     0          43m
# mysql-2   0/2     Pending       0          0s
# mysql-2   0/2     Pending       0          0s
# mysql-2   0/2     Init:0/2      0          0s
# mysql-2   0/2     Init:1/2      0          3s
# mysql-2   0/2     PodInitializing   0          4s
# mysql-2   1/2     Running           0          5s
# mysql-2   2/2     Running           0          9s
```

## Tear shits down
```sh
kubectl delete -f mysql-statefulset.yml
kubectl delete -f mysql-service.yml
kubectl delete -f mysql-configmap.yml
kubectl delete pvc -l app=mysql
```