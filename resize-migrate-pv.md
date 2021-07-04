# Resize and Migrating Persistent Volumes
## Create a temp pod to migrate the data
1. Scale down the MQ STS:
```
oc scale statefulset mq-dev-is1-ibm-mq --replicas=0
kubectl scale statefulset qm3-ibm-mq --replicas=0
```
2. Create a temp deployment to be used to create a new large PV and migrate the data from the small PV:
```
docker pull registry.access.redhat.com/rhel7/rhel-tools
docker tag registry.access.redhat.com/rhel7/rhel-tools default-route-openshift-image-registry.apps.ocpdev.devc.local/mq/rhel-tools
docker login default-route-openshift-image-registry.apps.ocpdev.devc.local -u ocpadmin -p $(oc whoami -t)
docker push default-route-openshift-image-registry.apps.ocpdev.devc.local/mq/rhel-tools
oc run tools --image=image-registry.openshift-image-registry.svc:5000/mq/rhel-tools -- tail -f /dev/null
oc set volume dc/tools --add -t pvc --name=data-qm3-ibm-mq-0 --claim-name=data-qm3-ibm-mq-0 --mount-path=/old-pv
oc set volume dc/tools --add -t pvc --name=data-qm3-ibm-mq-c-0 --claim-name=data-qm3-ibm-mq-c-0 --mount-path=/new-pv --claim-class=ceph-sc --claim-mode=ReadWriteOnce --claim-size=20Gi

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: tools
  name: tools
  namespace: mq
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      run: tools
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: tools
    spec:
      containers:
      - args:
        - tail
        - -f
        - /dev/null
        image: icp-cluster.integration.info:8500/mq/rhel-tools
        imagePullPolicy: Always
        name: tools
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /old-pv
          name: data-qm3-ibm-mq-0
        - mountPath: /new-pv
          name: data-qm3-ibm-mq-c-0
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: data-qm3-ibm-mq-0
        persistentVolumeClaim:
          claimName: data-qm3-ibm-mq-0
      - name: data-qm3-ibm-mq-c-0
        persistentVolumeClaim:
          claimName: data-qm3-ibm-mq-x-0
		
cat << EOF > mq-new-pvc.yaml		
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-qm3-ibm-mq-x-0
spec:
  storageClassName: ceph-sc
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
EOF
oc apply -f mq-new-pvc.yaml	  
```
3. Open a shell inside the created deployment and migrate the data:
```
oc rsh tools-3-hgxvm
rsync -avxHAX --progress /old-pv/* /new-pv
```
4. Delete the temp deployment
```
oc delete dc tools
kubectl delete deployment/tools
```

5. As you can see a new PVC/PV of 20Gi has been created:
* New PVC/PV (20Gi) data-mq-dev-is1a-ibm-mq-0/pvc-64755164-9524-4fe9-a610-39f60c0f498d
* Old PVC/PV (2Gi) data-mq-dev-is1-ibm-mq-0/pvc-eaa19393-ad1c-4345-a1b0-f6927f61bdfe

* New PVC/PV (20Gi) data-qm3-ibm-mq-c-0/pvc-902d2faa-db82-11eb-8aaa-02e7f1fc1018
* Old PVC/PV (2Gi)  data-qm3-ibm-mq-0/pvc-3dbaf5bb-6d4b-11ea-8aaa-02e7f1fc1018
```
[root@ocp-inst mq-deploy]# oc get pvc
NAME                           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
data-mq-dev-is1-ibm-mq-0       Bound    pvc-eaa19393-ad1c-4345-a1b0-f6927f61bdfe   5Gi        RWO            rook-cephfs       4d20h
data-mq-dev-is1-ibm-mq-1       Bound    pvc-f2f9f5ce-17bd-4895-93e6-dee748de0a64   2Gi        RWO            rook-cephfs       22d
data-mq-dev-is1a-ibm-mq-0      Bound    pvc-64755164-9524-4fe9-a610-39f60c0f498d   20Gi       RWO            rook-cephfs       17m
data-mq-dev-is2-ibm-mq-0       Bound    pvc-bd4f3b39-0409-4d18-8370-a27aefd488c9   2Gi        RWO            rook-cephfs       4d11h

NAME                       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-qm3-ibm-mq-0          Bound    pvc-3dbaf5bb-6d4b-11ea-8aaa-02e7f1fc1018   2Gi        RWO            glusterfs      467d
data-qm3-ibm-mq-x-0        Bound    pvc-902d2faa-db82-11eb-8aaa-02e7f1fc1018   20Gi       RWO            ceph-sc        34h
```
6. change the Reclaim policy of both the old and the new PVs to Retain instead of delete:
```
[root@ocp-inst mq-deploy]# oc patch pv pvc-eaa19393-ad1c-4345-a1b0-f6927f61bdfe -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
persistentvolume/pvc-eaa19393-ad1c-4345-a1b0-f6927f61bdfe patched
[root@ocp-inst mq-deploy]# oc patch pv pvc-64755164-9524-4fe9-a610-39f60c0f498d -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
persistentvolume/pvc-64755164-9524-4fe9-a610-39f60c0f498d patched

#kubectl patch pv pvc-3dbaf5bb-6d4b-11ea-8aaa-02e7f1fc1018 -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
#kubectl patch pv pvc-902d2faa-db82-11eb-8aaa-02e7f1fc1018 -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```
7. Delete the old and new PVCs:
```
[root@ocp-inst mq-deploy]# oc delete pvc data-mq-dev-is1-ibm-mq-0
persistentvolumeclaim "data-mq-dev-is1-ibm-mq-0" deleted
[root@ocp-inst mq-deploy]# oc delete pvc data-mq-dev-is1a-ibm-mq-0
persistentvolumeclaim "data-mq-dev-is1a-ibm-mq-0" deleted
```
8. Ensure the status of the PVs is now released:
```
[root@ocp-inst mq-deploy]# oc get pv pvc-eaa19393-ad1c-4345-a1b0-f6927f61bdfe
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                         STORAGECLASS   REASON   AGE
pvc-eaa19393-ad1c-4345-a1b0-f6927f61bdfe   5Gi        RWO            Retain           Released   mq/data-mq-dev-is1-ibm-mq-0   rook-cephfs             34d
[root@ocp-inst mq-deploy]# oc get pv pvc-64755164-9524-4fe9-a610-39f60c0f498d
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                          STORAGECLASS   REASON   AGE
pvc-64755164-9524-4fe9-a610-39f60c0f498d   20Gi       RWO            Retain           Released   mq/data-mq-dev-is1a-ibm-mq-0   rook-cephfs             27m
```
8. Edit the new PV "pvc-64755164-9524-4fe9-a610-39f60c0f498d" and Remove claimRef section.
9. Create a new PVC manifest with the old PVC name but pointing to the new PV:
```
cat << EOF > new-mq-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-qm3-ibm-mq-0
  namespace: mq
spec:
  accessModes:
    - ReadWriteOnce
  volumeName: pvc-902d2faa-db82-11eb-8aaa-02e7f1fc1018
  storageClassName: ceph-sc
  resources:
    requests:
      storage: 20Gi
EOF
kubectl apply -f new-mq-pvc.yaml
```
10. Ensure new PVC bounded to the right PV:
```
[root@ocp-inst mq-deploy]# oc get pvc
NAME                           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
data-mq-dev-is1-ibm-mq-0       Bound    pvc-64755164-9524-4fe9-a610-39f60c0f498d   20Gi       RWO            rook-cephfs       37m
``` 
9. Scale up the MQ STS:
```
c scale statefulset mq-dev-is1-ibm-mq --replicas=1
```
10. Ensure MQ is up and running:
```
[root@ocp-inst mq-deploy]# oc get po
NAME                                   READY   STATUS      RESTARTS   AGE
mq-dev-is1-ibm-mq-0                    1/1     Running     0          33m
```
