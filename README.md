## How to jump from vCenter to Supervisor to Guest Cluster


(1)	vCenter
```
	https://192.168.1.50:5480			#Enable bash shell!!!
```

(2)	vCenter
```
	ssh root@192.168.1.50   
	shell
	/usr/lib/vmware-wcp/decryptK8Pwd.py
	ssh root@192.168.1.180				#use IP/Password from above command
```

(3)	On Supervisor VM get guest cluster credentials
```
	kubectl get cluster -A  #select cluster and namespace and set variables
	export NAMESPACE=namespace1000
	export CLUSTER=cluster3

	kubectl get secrets -A | grep $NAMESPACE | grep ssh
	kubectl -n $NAMESPACE get secret $CLUSTER-ssh -o jsonpath='{.data.ssh-privatekey}' | base64 -d > cluster-ssh-key
	chmod 600 cluster-ssh-key
```

(3.1)	How to ssh to Worker nodes
```
	kubectl -n $NAMESPACE get virtualmachines
	kubectl -n $NAMESPACE get virtualmachines | grep node
	export NODE=`kubectl -n $NAMESPACE get virtualmachines | grep node | tail -1 | awk '{print $1}'`

 	export VMIP=`kubectl -n $NAMESPACE get virtualmachine $NODE  -o jsonpath='{$.status..primaryIP4}'`
	echo $VMIP
	ssh -i cluster-ssh-key vmware-system-user@$VMIP

	sudo cat /var/log/cloud-init-output.log (log for cluster join)
	df -h
```

(3.2)	How to ssh to Control nodes
```
	kubectl -n $NAMESPACE get virtualmachines
	kubectl -n $NAMESPACE get virtualmachines | grep -v node
	export WNODE=`kubectl -n $NAMESPACE get virtualmachines | grep -v node | tail -1 | awk '{print $1}'`

 	export VMIPW=`kubectl -n $NAMESPACE get virtualmachine $WNODE  -o jsonpath='{$.status..primaryIP4}'`
	echo $VMIPW

	ssh -i cluster-ssh-key vmware-system-user@$VMIPW

	sudo kubectl --kubeconfig /etc/kubernetes/admin.conf  get pods -A

	sudo kubectl --kubeconfig /etc/kubernetes/admin.conf  get events -A  | grep -i evicted
	sudo kubectl --kubeconfig /etc/kubernetes/admin.conf  get events -A  | grep -i pressure
	sudo kubectl --kubeconfig /etc/kubernetes/admin.conf  get events -A  | grep -i error
	sudo kubectl --kubeconfig /etc/kubernetes/admin.conf  get events -A  | grep -i failed
	sudo kubectl logs --kubeconfig /etc/kubernetes/admin.conf -n kube-system etcd-cluster3-rn59z-xtw6t 

	# get all logs

	for f in `sudo kubectl --kubeconfig /etc/kubernetes/admin.conf  get pods -A | awk '{print $1"_" $2}'`; do
	sudo kubectl logs --kubeconfig /etc/kubernetes/admin.conf -n ${f/_/ };
	done

	or

	for f in `sudo kubectl --kubeconfig /etc/kubernetes/admin.conf  get pods -A | awk '{print $1"_" $2}'`; do  sudo kubectl logs --kubeconfig /etc/kubernetes/admin.conf -n ${f/_/ }; done | grep -i error


```

# Supervisor content lib
```
https://wp-content.vmware.com/supervisor/v1/latest/lib.json
```
# TKR content lib
```
https://wp-content.vmware.com/v2/latest/lib.json
```

# Clusterclass Documentation

```
k get clusterclass
k describe clusterclass builtin-generic-v3.3.0


https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vsphere-supervisor-services-and-standalone-components/latest/managing-vsphere-kuberenetes-service-clusters-and-workloads/provisioning-tkg-service-clusters/using-the-cluster-v1beta1-api/using-the-versioned-clusterclass/usingbuiltingenericv330.html
```
# K8 login

```
kubectl vsphere login --server=192.168.2.201 --vsphere-username administrator@vsphere.local --insecure-skip-tls-verify

kubectl vsphere login --server=192.168.2.201 --vsphere-username administrator@vsphere.local --insecure-skip-tls-verify --tanzu-kubernetes-cluster-namespace namespace1000 --tanzu-kubernetes-cluster-name cluster3 --insecure-skip-tls-verify

```

# Offline image package move

```
https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/8-0/using-tkg-service-with-vsphere-supervisor/using-private-registries-with-tkg-service-clusters/push-standard-packages-to-a-private-harbor-registry.html
```

# VCF CLI Install + Plugins
```
Download VCF CLI:       https://packages.broadcom.com/artifactory/vcf-distro/vcf-cli/
Move to linux box:      scp vcf-cli.tar.gz orf@192.168.1.2:/tmp/.
Untar:                  cd /tmp
Untar:                  tar xvf vcf-cli.tar.gz
Install:                sudo install  vcf-cli-linux_amd64 /usr/local/bin/vcf
Test:                   vcf version
Harbor:                 In Harbor create a project called vcf_cli
Download all plugins:   vcf plugin download-bundle --to-tar /tmp/FILE-NAME.tar.gz
Download Harbor cert:   go to projects and vcf_cli and there is a button download cert
Copy cert into place:   sudo cp /tmp/ca.crt  /var/snap/docker/current/etc/docker/certs.d/harbor.lab.local
Restart docker:         sudo snap restart docker
Log into local Harbor:  docker login harbor.lab.local -u admin (Harbor12345)
Update VCF with cert:   vcf config cert add --host harbor.lab.local --ca-certificate /tmp/ca.crt
Upload to Harbor:       vcf plugin upload-bundle --tar /tmp/FILE-NAME.tar.gz --to-repo harbor.lab.local/vcf_cli/plugins
Update new Harbor:      vcf plugin source update default --uri harbor.lab.local/vcf_cli/plugins/plugin-inventory:latest
Clean up and validate:  vcf plugin clean && vcf plugin list
Install cluster plugin: vcf plugin install cluster
Check the install:      vcf plugin list installed
Install the bassics:    vcf plugin install --group vmware-vcfcli/essentials
```

# Bitnami - Harbor
```
Download: https://vcf.broadcom.com/vsc/services/details/harbor-singlevm?slug=true
Group: Emerging Solutions Tanzu SE

Generate a key for the OVA: ssh-keygen -t rsa -b 4096

Use this key for the OVA deploy: cat .ssh/id_rsa.pub

Log on: ssh -i .ssh/id_rsa bitnami@192.168.1.15

GUI: http://192.168.1.15/account/sign-in?redirect_url=%2Fharbor%2Fprojects
ID: admin
Password: This is located on the console (In my case: 2+C!aD+Y7Kla)
Change: Select upper right hand user and select change password. 
```


# Set pod security policy and regcred
```
# ns=default
kubectl label --overwrite ns default pod-security.kubernetes.io/enforce=privileged

# ns=opencart
kubectl create namespace opencart
kubectl label ns opencart pod-security.kubernetes.io/enforce=privileged
kubectl create secret docker-registry regcred --docker-server=harbor-01a.site-a.vcf.lab --docker-username="admin" --docker-password='mycoolpassword' --docker-email=administrator@vsphere.local -n opencart

```

# Standard packages pull and push (imgpkg install)
=====================================================
```
#Download the binary
curl -LO https://github.com/carvel-dev/imgpkg/releases/download/v0.47.1/imgpkg-linux-amd64

#Move the binary in to your PATH
mv imgpkg-linux-amd64 /usr/local/bin/imgpkg

#Make the binary executable
chmod +x /usr/local/bin/imgpkg

#Get the packages into a tar file:
imgpkg  copy --bundle projects.packages.broadcom.com/vsphere/supervisor/packages/2025.1.7/vks-standard-packages:v2025.1.7 --to-tar /tmp/temp2.tar --registry-response-header-timeout=600s

#Create VKS project in Harbor: 
GUI create vks public project

#Test file: 
ls -ltrah /tmp/temp2.tar
file /tmp/temp2.tar

#Copy From tar file to Harbor (ca.crt comes from harbor GUI):
imgpkg copy --tar=/tmp/temp2.tar --to-repo harbor.lab.local/vks/vks-standard-packages --registry-response-header-timeout=600s --registry-ca-cert-path /tmp/ca.crt --concurrency 1
```


