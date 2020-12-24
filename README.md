# Disaster Recovery Red Hat Openshift Container Platform 4 From Expired Control-Plane Nodes Certificates
*Written by Kerem ÇELİKER*
- Linkedin: **`linkedin.com/in/keremceliker`**
- Twitter: **`@CloudRss`**
- Blog: **`www.keremceliker.com`**

Sometimes teams have to shutdown the openshift cluster for planned work, and if this is not done appropriately, there are unexpected problems.  

 

However, there is a point where the teams managing many Container infrastructures in general are overlooked due to their work intensives. In this case, after such a process, the core-certificates that they have already forgotten expire and after this situation, the long periods of downtime and the occurrence of money losses in different area- online customer systems  

 

 

- In such a problem, must first be determined issue with the following commands. 

 

`[keremceliker@bastion ~]$ export KUBECONFIG=/2021GonnaBeGood/auth/kubeconfig` 

 
```yaml
[keremceliker@bastion ~]$ oc get nodes  

**Unable to connect to the server: x509: certificate has expired or is not yet valid** 
```
 

- Check the logs for the kubelet.service and and you must be getting an certificate error like the one below  

 
`[core@kcmaster01 ~]$ sudo journalctl -a --no-pager -u kubelet -b0 | grep "E" | head -10`

```yaml
Dec 24 09:35:03 kcmaster0 hyperkube[1680]: E1024 09:35:03.175362 1680 bootstrap.go:264] 
Part of the existing bootstrap client certificate is expired: 2020-12-23 03:40:12 +0000 UTC 
=====================================================
Dec 24 09:35:03 master0 hyperkube[1680]: E1024 09:35:03.193155 1680 certificate_manager.go:385] 
Failed while requesting a signed certificate from the master: cannot create certificate signing request: Post https://api-int.keremceliker.com:6443/apis/certificates.k8s.io/v1beta1/certificatesigningrequests: EOF 
```
 

- Check which kubelet certificates are in expire state 

 
 
```yaml
[core@kcmaster01 ~]$ sudo cat /var/lib/kubelet/pki/kubelet-client-current.pem | openssl x509 -noout -dates  

notBefore=Oct 22 15:37:00 2020 GMT  
notAfter=Oct 23 15:20:32 2020 GMT 
```
 

 

- Check the validity of bootstrap certificate for kubelet authentication 

 

 
```shell
[core@kcmaster01 ~]$ sudo cat /etc/kubernetes/kubeconfig | grep client-certificate-data | awk '{print $2}' | base64 -d | openssl x509 -noout -dates  

notBefore=Oct 22 15:37:00 2020 GMT 

notAfter=Oct 23 15:20:32 2020 GMT 
```
 

 

 

***In the last case if everything shows that the certificate has expired you must follow these steps in order***

 

- Log on to a Master Node as kcmaster01 as root to use temp. kube-apiserver and follow the step-by-step below one-to-one. 

 

 
```yaml
[keremceliker@bastion ~]$  ssh kcmaster01 

[core@kcmaster01 ~]$ sudo su - 
 
[root@kcmaster01 ~]$ RELEASE_IMAGE=quay.io/openshift-release-dev/ocp-release:4.6.0  

[root@kcmaster01 ~]$ KAO_IMAGE=$( oc adm release info --registry-config='/var/lib/kubelet/config.json' "${RELEASE_IMAGE}" --image-for=cluster-kube-apiserver-operator )
 
[root@kcmaster01~]$ podman pull --authfile=/var/lib/kubelet/config.json "${KAO_IMAGE}"  
 
[root@kcmaster01 ~]$ podman run -it --network=host -v /etc/kubernetes/:/etc/kubernetes/:F--entrypoint=/usr/bin/cluster-kube-apiserver-operator "${KAO_IMAGE}" recovery-apiserver create
 
[root@kcmaster01 ~]$ export KUBECONFIG=/etc/kubernetes/static-pod-resources/recovery-kube-apiserver-pod/admin.kubeconfig  
 
[root@kcmaster01 ~]$ until oc get namespace kube-system 2>/dev/null 1>&2; do echo 'Waiting for recovery apiserver to come up.'; sleep 1; done 
```
 

- Re-create control-plane certificates by running this podman command over master0 with root user 

 
```yaml
[root@kcmaster01 ~]$ podman run -it --network=host -v /etc/kubernetes/:/etc/kubernetes/:F --entrypoint=/usr/bin/cluster-kube-apiserver-operator "${KAO_IMAGE}" regenerate-certificates  
```
 
 

- Quickly publish by forced this for Nodes across all Control Planes: 

 
```yaml
[root@kcmaster01 ~]$ oc patch kubeapiserver cluster -p='{"spec": {"forceRedeploymentReason": "recovery-'"$( date --rfc-3339=ns )"'"}}' --type=merge  
 
[root@kcmaster01 ~]$ oc patch kubecontrollermanager cluster -p='{"spec": {"forceRedeploymentReason": "recovery-'"$( date --rfc-3339=ns )"'"}}' --type=merge  
 
[root@kcmaster01 ~]$ oc patch kubescheduler cluster -p='{"spec": {"forceRedeploymentReason": "recovery-'"$( date --rfc-3339=ns )"'"}}' --type=merge 
```
 

 

- Create a new bootstrap certificate to use kubeconfig bash. 

 

`[root@kcmaster01 ~]$ recover-kubeconfig.sh > /2021GonnaBeGood/kubeconfig`
 

 

 

- Check and to have the CA certificate to verify getting connections from the API server 

 
```
[root@kcmaster01 ~]$ oc get configmap kube-apiserver-to-kubelet-client-ca -n openshift-kube-apiserver-operator --template='{{ index .data "ca-bundle.crt" }}' > /2021GonnaBeGood/kubelet-ca.crt  
```
 
 

 

- Deploy newly created boot certificate and CA certificate to all Nodes 

 

 
`[keremceliker@bastion ~]$ scp core@kcmaster01:/2021GonnaBeGood/kubeconfig .`

`[keremceliker@bastion ~]$ scp core@kcmaster01:/2021GonnaBeGood/kubelet-ca.crt .`

 
```yaml
[keremceliker@bastion ~]$ for node in core@kcmaster{01,02,03} core@kcworker{01,02,03}  
do  
if [[ ! "$node" =~ "kcmaster01" ]] ; then scp kubeconfig $node:/2021GonnaBeGood; fi  
ssh $node sudo cp /2021GonnaBeGood/kubeconfig /etc/kubernetes/kubeconfig  
done 
```
 
```yaml
[keremceliker@bastion ~]$ for node in core@kcmaster{01,02,03} core@kcworker{01,02,03} 
do 
if [[ ! "$node" =~ "kcmaster01" ]] ; then scp kubelet-ca.crt $node:/2021GonnaBeGood; fi  
ssh $node sudo cp /2021GonnaBeGood/kubelet-ca.crt /etc/kubernetes/kubelet-ca.crt  
done 
```
 

 

- Add machine-config-daemon-force files for all hosts and then run the following command to configure Machine-Config and accept the new updated certificate in Nodes 

 

 
```
[keremceliker@bastion ~]$ for node in core@kcmaster{01,02,03} core@kcworker{01,02,03}  

do  

ssh $node sudo touch /run/machine-config-daemon-force  

done 
```
 

- Restart the "kubelet.service" from Master nodes and remove all previous/former expired certificates: 

 
```shell
[keremceliker@bastion ~]$ for node in core@kcmaster{01,02,02}  

do  

ssh $node sudo systemctl stop kubelet  

ssh $node sudo rm -rf /var/lib/kubelet/pki /var/lib/kubelet/kubeconfig  

ssh $node sudo systemctl start kubelet  

done 
```
 

 

- Restart the "kubelet.service" from Worker nodes and remove all previous/former expired certificates: 

 
```shell
[keremceliker@bastion ~]$ for node in core@kcworker{01,02,03}  

do  

ssh $node sudo systemctl stop kubelet  

ssh $node sudo rm -rf /var/lib/kubelet/pki /var/lib/kubelet/kubeconfig  

ssh $node sudo systemctl start kubelet  

done 
```
 

- Confirm all newly "Pending State" CSR certificates by running the following commands and have nodes receive them. Thus, all graded operators will be running perfectly again (: 

 
```
[root@kcmaster01 ~]$ oc get csr or watch "oc get csr" 
```
```
[root@kcmaster01 ~]$ for id in `oc get csr | grep Pend | awk '{print $1}'`;  

do  

oc adm certificate approve $id;  

done  
```
 

- You will need to do this several times. so wait a while for all the certificates to arrive 

 

Sometimes you need to run repeatly to oc adm certificate approve "csr name"  

 

- Verify that the cluster is up and healthy again by running oc get nodes: 

`[root@kcmaster01 ~]# oc get co && oc get nodes -o wide`
Or 

`[keremceliker@bastion ~]$ oc get co && oc get nodes -o wide`
 

 

**References:**

 

- how to set up TLS client certificate bootstrapping for kubelets 

 

https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/ 

 

- Recovering from expired control plane certificates 

 

https://access.redhat.com/documentation/en-us/openshift_container_platform/4.2/html/backup_and_restore/disaster-recovery#dr-scenario-3-recovering-expired-certs_dr-recovering-expired-certs 

 

 

- Configure Certificate Rotation for the Kubelet 

 

 

https://kubernetes.io/docs/tasks/tls/certificate-rotation/ 
