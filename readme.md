# Azure Kubernetes Service - Enhanced Kubernetes cluster pod security baseline standards for Linux-based workloads

This example builds upon existing Pod Security baseline with enhancements, and provides means to manage the initiative as code using GitHub actions.
(You can also create manually the initiative, but this guide does not have instructions for manual workflow)

**Scope** of this guide is to increase workload security by aligning the use of Azure Policy and resource specification together. First with audit controls, and then with deny controls.

**Original policy baseline policy** 

Description| source
-|-
|This initiative includes the policies for the Kubernetes cluster pod security baseline standards. This policy is generally available for Kubernetes Service (AKS), and preview for AKS Engine and Azure Arc enabled Kubernetes. For instructions on using this policy, visit https://aka.ms/kubepolicydoc.|Kubernetes cluster pod security baseline standards for Linux-based workloads


**Enhanced policy**

![img](img/front.png)

- [Azure Kubernetes Service - Enhanced Kubernetes cluster pod security baseline standards for Linux-based workloads](#azure-kubernetes-service---enhanced-kubernetes-cluster-pod-security-baseline-standards-for-linux-based-workloads)
  - [Differences compared to the built-in initiative:](#differences-compared-to-the-built-in-initiative)
  - [Disclaimer](#disclaimer)
  - [Utilizing the initiative](#utilizing-the-initiative)
    - [Example of which policy affects which specification in resource](#example-of-which-policy-affects-which-specification-in-resource)
    - [Example of healthy podspec](#example-of-healthy-podspec)
  - [Deployment guide:](#deployment-guide)
    - [Express installation with password based SPN (recommended only for testing)](#express-installation-with-password-based-spn-recommended-only-for-testing)
    - [Advanced with Workload federation (recommended for enterprise use)](#advanced-with-workload-federation-recommended-for-enterprise-use)
  - [References](#references)



---

## Differences compared to the built-in initiative:


✔️ Adds "debug" namespace to exclusions, so you don't have to run debugging workloads in the namespaces  ``["kube-system" "gatekeeper-system", "azure-arc"] ``

- Typical example of debug namespace use would be the SSH to AKS node

```bash
kubectl create namespace 'debug'
node=$(kubectl get nodes -o=jsonpath='{.items[0].metadata.name}')
val=$(echo $node | sed 's/"//g')
kubectl debug node/$val -it --image=mcr.microsoft.com/aks/fundamental/base-ubuntu:v0.0.11 --namespace="debug"
```

✔️ Adds ``Kubernetes cluster containers should only use allowed AppArmor profiles`` -policy 

- Configuration is compliant with the "runtime/default" out-of-the-box AKS node installed profile. Creates initiative parameter for "runtime/default" appArmor profile, which is the default profile installed in AKS nodes

✔️ Adds  `` Kubernetes cluster containers should run with a read only root file system `` 

✔️ Allows switching state for allow policies from audit to deny in single setting using Initiative parameters

![img](img/init.png)

---
## Disclaimer
[Read license](LICENSE)
## Utilizing the initiative 

- Target the Policy to scope you want to apply it (Recommendation for testing is to target it to single AKS cluster)
- Review audit results produced by the policy
- Implement healthy podSpec / deployment 


### Example of which policy affects which specification in resource 

MS Docs features good example of complete healthy and unhealthy specifications

[Unhealthy specification ](https://docs.microsoft.com/en-us/azure/defender-for-cloud/kubernetes-workload-protections#unhealthy-deployment-example-yaml-file)

[healthy specification ](https://docs.microsoft.com/en-us/azure/defender-for-cloud/kubernetes-workload-protections#healthy-deployment-example-yaml-file)

---

**Mapping to Azure Policy and specification**

The below table highlights healthy, and un-healthy settings examples spec

item|status | YAML 
-|-|-
Pods should run as non-root /non-privileged|✔ | **spec.securityContext** <br> ``privileged: false``  <br>       ``runAsNonRoot:true`` <br> ``allowPrivilegeEscalation: false``
Pod Filesystem access should be read-only or limited only for specifed writes | ✔ | **spec.securityContext** <br> ``readOnlyRootFilesystem: true`` 
Pod actions should be limited with appArmor |✔| **metadata.annotations** <br> ``container.apparmor.security.beta.kubernetes.io/v5: runtime/default`` 
Pod should be disabled for automation of API credentials |✔| **spec** <br> `` automountServiceAccountToken: false `` 
Pod hostPath mounts should only allow predefined mounts, and preferably not used at all (By default Azure Policy audits all hostPath mounts as non compliant, to block hostPath mounts the policy needs to be changed) | ✔ | **spec.Volumes** <br> `` hostPath: <Only allowed values here defined in Azure Policy> `` 
**MS direct reference** *To protect against privilege escalation outside the container, avoid pod access to sensitive host namespaces (host process ID and host IPC) in a Kubernetes cluster*.   |❌| Should not define use of 'true' in following settings **spec.hostPID** <br> `` hostPID:   `` <br> **spec.hostIPC** <br> ``hostIPC: `` 
**MS direct reference**  *Pods created with the hostNetwork attribute enabled will share the node’s network space. To avoid compromised container from sniffing network traffic, we recommend not putting your pods on the host network*|❌|  Should not define use of 'true' in following settings <br> **spec** <br> `` hostNetwork:  `` 


### Example of healthy podspec
- For testing replace the image with any image you want to test the POD deployment against
- The example assumes keyVault integration with secrets store driver. If you don't have such integration configured, remove ``volumes`` and ``volumeMounts``  

```YAML
apiVersion: v1
kind: Pod
metadata:
  name: demorce
  annotations:
    container.apparmor.security.beta.kubernetes.io/v5: runtime/default
spec:
  automountServiceAccountToken: false
  containers:
    - name: nginxHealthy
      image: nginx:1.14.2
      ports:
        - containerPort: 8443
      resources:
          limits:
            cpu: 100m
            memory: 250Mi
      securityContext:
        privileged: false
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
        runAsNonRoot: true
        runAsUser: 1000
      volumeMounts:
        - name: secrets-store01-inline
          mountPath: /mnt/secrets-store
          readOnly: true
  volumes:
      - name: secrets-store01-inline
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: azure-kvname-user-msi 
``` 



## Deployment guide:

### Express installation with password based SPN (recommended only for testing)

Following this method creates SPN with required permissions ``Resource Policy Contributor``  to the subscription defined in cli script

- References https://github.com/marketplace/actions/manage-azure-policy#configure-credentials-for-azure-login-action 
1.  Create SPN

```bash
## Set subscription
SUB="3539c2a2-cd25-48c6-b295-14e59334ef1c"
az account set --subscription $SUB
az ad sp create-for-rbac --name PodSecurityPolicySPN --role 'Resource Policy Contributor' --scopes "/subscriptions/$SUB"

# If you are running Azure Cloud Shell, you can automatically do step 2 here 
# node replacesub.js $SUB
```

Take note of the output, as it is used in step 5.

![img](img/output.png)

2. Replace subscription ``SUBID`` ID in [policyset.json](initiatives/AKSREF_-_Kubernetes_cluster_pod_security_baseline_standards_for_Linux-based_workloads_1f01afd98f33414e995e3ad5/policyset.json)
  ```
    "id": "/subscriptions/SUBID/providers/Microsoft.Authorization/policySetDefinitions/1f01afd98f33414e995e3ad5",
    "type": "Microsoft.Authorization/policySetDefinitions",
    "name": "1f01afd98f33414e995e3ad5"
  ```


3. After this Create **new** Github repo, and force push the contents of this repo to the just created repo

```
 git remote add myTestRepo git@github.com:you/yourTestRepo.git
 git add .; git commit -m "init to myTestRepo"
 git push myTestRepo --force
```

4. replace the credentials in action.yml wit the secret listed in the throwaway repo *(the ID for credential needs to match that of the one listed in secrets)*
```yaml
with:
        creds: ${{secrets.policySecret}}
        allow-no-subscriptions: true
```
5. Add the password to the repo you created. The secret name needs to match "creds" as defined in previous step

- replace as follows AppId = clientID, password = clientSecret

![img](img/secret.png)

6. From actions start the workflow manually

![img](img/wf.png)

7. After update you should see the following output in actions and at azure policy

![img](img/assign.png)

![img](img/deployed.png)

8.  Assign the policy to test cluster

![img](img/assingment.png)

9. Review results and determine which resources need to be fixed, or excluded from policy
![img](img/compl.png)

![img](img/details.png)
10. If you feel that everything works correctly, turn the policy initative to deny

![img](img/init.png)

### Advanced with Workload federation (recommended for enterprise use)

- To be tested!

[documentation](https://docs.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Clinux#set-up-azure-login-with-openid-connect-authentication)

## References
https://kubernetes.io/docs/concepts/policy/pod-security-policy/
