# Convert-Registry-from-NFS-to-S3
This document is prepared for converting Registry storage from NFS to S3 on Openshift 3.6.

```
It looks like the data is there, I think it's safe to do the migration from NFS to S3.

Before doing the procedure attach: oc get dc docker-registry -n default -o yaml to the case so that in case something goes wrong we have a procedure to rollback.

You may do the procedure:

These are the steps:
1- copy the files from the pod to the s3. To do so mount the volume in a host that has epel (preferably not an ocp node)
mount <pvc mounting options> <mount path>
yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
yum -y install s3cmd
cd <mount path>
s3cmd --config (configure it as expected)
s3cmd put -r docker s3://<s3 bucket>


2- create the configmap.
**WARNING**: In the s3 section replace the appropriate values:
cat > config.yaml <<EOF  
version: 0.1
log:
  level: debug
http:
  addr: :5000
storage:
  s3:
    region: <region>
    bucket: <s3 bucket>
    encrypt: false
    secure: true
  delete: true
    enabled: true
auth:
  openshift:
    realm: openshift
middleware:
  registry:
  - name: openshift
  repository:
  - name: openshift
    options:
      pullthrough: True
      acceptschema2: True
      enforcequota: False
  storage:
  - name: openshift
EOF


3-  oc create configmap registry-config --from-file config.yml

4- oc volume dc/docker-registry --add --type=configmap \
    --configmap-name=registry-config -m /etc/docker/registry/

5- oc set env dc/docker-registry \
    REGISTRY_CONFIGURATION_PATH=/etc/docker/registry/config.yml

6- oc volume dc/docker-registry --add --type=emptyDir \
    --overwrite -m /registry/


```
