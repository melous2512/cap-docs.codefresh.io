---
title: "Add external clusters to runtimes"
description: ""
group: runtime
toc: true
---

Register external clusters to provisioned hybrid or hosted runtimes in Codefresh. Once you add an external cluster, you can deploy applications to that cluster without having to install Argo CD in order to do so. External clusters allow you to manage multiple clusters through a single runtime.  

When you add an external cluster to a provisioned runtime, the cluster is registered as a managed cluster. A managed cluster is treated as any other managed K8s resource, meaning that you can monitor its health and sync status, deploy applications on the cluster and view information in the Applications dashboard, and remove the cluster from the runtime's managed list.  

Add managed clusters through:
* Codefresh CLI
* Kustomize

Adding a managed cluster via Codefresh ensures that Codefresh applies the required RBAC resources (`ServiceAccount`, `ClusterRole` and `ClusterRoleBinding`) to the target cluster, creates a `Job` that updates the selected runtime with the information, registers the cluster in Argo CD as a managed cluster, and updates the platform with the new cluster information.
 

### Add a managed cluster with Codefresh CLI
Add an external cluster to a provisioned runtime through the Codefresh CLI. When adding the cluster, you can also add labels and annotations to the cluster, which are added to the cluster secret created by Argo CD.
Optionally, to first generate the YAML manifests, and then manually apply them, use the `dry-run` flag in the CLI. 

**Before you begin**  

  
* For _hosted_ runtimes: [Configure access to these IP addresses]({{site.baseurl}}/docs/administration/platform-ip-addresses/)
* Verify that:
  * Your Git personal access token is valid and has the correct permissions
  * You have installed the latest version of the Codefresh CLI

**How to**  

1. In the Codefresh UI, go to [Runtimes](https://g.codefresh.io/2.0/account-settings/runtimes){:target="\_blank"}.
1. From either the **Topology** or **List** views, select the runtime to which to add the cluster. 
1. Topology View: Select {::nomarkdown}<img src="../../../images/icons/add-cluster.png" display=inline-block/>{:/}.  
  List View: Select the **Managed Clusters** tab, and then select **+ Add Cluster**.  
1. In the Add Managed Cluster panel, copy and run the command:  
  `cf cluster add <runtime-name> [--labels label-key=label-value] [--annotations annotation-key=annotation-value][--dry-run]`  
  where:   
  * `--labels` is optional, and required to add labels to the cluster. When defined, add a label in the format `label-key=label-value`. Separate multiple labels with `commas`.   
  * `--annotations` is optional, and required to add annotations to the cluster. When defined, add an annotation in the format `annotation-key=annotation-value`. Separate multiple annotations with `commas`.   
  * `--dry-run` is optional, and required if you want to generate a list of YAML manifests that you can redirect and apply manually with `kubectl`.   


   {% include 
	image.html 
	lightbox="true" 
	file="/images/runtime/managed-cluster-add-panel.png" 
	url="/images/runtime/managed-cluster-add-panel.png" 
	alt="Add Managed Cluster panel" 
	caption="Add Managed Cluster panel"
  max-width="40%" 
%}

{:start="5"}
1. If you used `dry-run`, apply the generated manifests to the same target cluster on which you ran the command.  
  Here is an example of the YAML manifest generated with the `--dry-run` flag. Note that there are placeholders in the example, which are replaced with the actual values with `--dry-run`.  
  

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd-manager
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argocd-manager-role
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs:
  - '*'
  verbs:
  - '*'
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd-manager-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: argocd-manager-role
subjects:
- kind: ServiceAccount
  name: argocd-manager
  namespace: kube-system
---
apiVersion: v1
data:
  contextName: <context-name>
  ingressUrl: <ingressUrl>
  server: <server>
kind: ConfigMap
metadata:
  name: csdp-add-cluster-cm
  namespace: kube-system
---
apiVersion: v1
data:
  annotations: |
    <annotation-key1>:<annotation-value1>
    <annotation-key2>:<annotation-value2>
  contextName: <context-name>
  ingressUrl: ingressurl.com
  labels: |
    <label-key1>:<label-value1>
    <label-key2>:<label-value2>
  server: https://<hash>.gr7.us-east-1.eks.amazonaws.com/
  csdpToken: <csdpToken>
kind: Secret
metadata:
  name: csdp-add-cluster-secret
  namespace: kube-system
type: Opaque
---
apiVersion: batch/v1
kind: Job
metadata:
  name: csdp-add-cluster-job
  namespace: kube-system
spec:
  template:
    metadata:
      name: csdp-add-cluster-pod
    spec:
      containers:
      - args:
        - ./add-cluster.sh
        command:
        - bash
        env:
        - name: SERVICE_ACCOUNT_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.serviceAccountName
        - name: INGRESS_URL
          valueFrom:
            configMapKeyRef:
              key: ingressUrl
              name: csdp-add-cluster-cm
        - name: CSDP_TOKEN
          valueFrom:
            secretKeyRef:
              key: csdpToken
              name: csdp-add-cluster-secret
        - name: CONTEXT_NAME
          valueFrom:
            configMapKeyRef:
              key: contextName
              name: csdp-add-cluster-cm
        - name: SERVER
          valueFrom:
            configMapKeyRef:
              key: server
              name: csdp-add-cluster-cm
        image: quay.io/codefresh/csdp-add-cluster:0.1.0
        imagePullPolicy: Always
        name: main
        resources:
          limits:
            cpu: "1"
            memory: 512Mi
          requests:
            cpu: "0.2"
            memory: 256Mi
      restartPolicy: Never
      serviceAccount: argocd-manager
  ttlSecondsAfterFinished: 600

```

The new cluster is registered to the runtime as a managed cluster.  

### Add a managed cluster with Kustomize
Create a `kustomization.yaml` file with the information shown in the example below, and run `kustomize build` on it.  

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: kube-system

configMapGenerator:
  - name: csdp-add-cluster-cm
    namespace: kube-system
    behavior: merge
    literals:
        # contextName is the name of the kube context (in the local kubeconfig file) that connects to the target cluster
      - "contextName=<contextName>"
        # ingressUrl is the url used to access the Codefresh runtime
        # example https://some.domain.name
      - "ingressUrl=<ingressUrl>"
        # server is the k8s cluster API endpoint url
        # can be obtained by
        #   CONTEXT_NAME=<TARGET_CONTEXT_NAME>
        #   CLUSTER_NAME=$(kubectl config view --raw --flatten -o jsonpath='{.contexts[?(@.name == "'"${CONTEXT_NAME}"'")].context.cluster}')
        #   kubectl config view --raw --flatten -o jsonpath='{.clusters[?(@.name == "'"${CLUSTER_NAME}"'")].cluster.server}'
      - "server=https://<hash>.gr7.us-east-1.eks.amazonaws.com/"
      - |
        annotations=<key1: value1>
        <key2.with.dots/and-backslash: value2 with: as:pace>
      - |
        labels=<and.another-one/field: value>
        <label.key.with.long.name/field: some_long_value>

secretGenerator:
- behavior: merge
  literals:
  - csdpToken=<your-personal-token>
  name: csdp-add-cluster-secret
  namespace: kube-system
 
resources:
  - https://github.com/codefresh-io/cli-v2/manifests/add-cluster/kustomize?ref=<runtimeVersion>
```


### Work with managed clusters 
Work with managed clusters in hybrid or hosted runtimes in either the Topology or List runtime views. For information on runtime views, see [Runtime views]({{site.baseurl}}/docs/runtime/runtime-views).  
As the cluster is managed through the runtime, updates to the runtime automatically updates the components on all the managed clusters that include it. 
     
View connection status for the managed cluster, and health and sync errors. Health and sync errors are flagged by the error notification in the toolbar, and visually flagged in the List and Topology views.  
  
#### Install Argo Rollouts 
Install Argo Rollouts directly from Codefresh with a single click to visualize rollout progress in the [Applications dashboard]({{site.baseurl}}/docs/deployment/applications-dashboard/). If Argo Rollouts has not been installed, an **Install Argo Rollouts** button is displayed on selecting the managed cluster. 

1. In the Codefresh UI, go to [Runtimes](https://g.codefresh.io/2.0/account-settings/runtimes){:target="\_blank"}.
1. Select **Topology View**.
1. Select the target cluster, and then select **+ Install Argo Rollouts**.
 
{% include 
	image.html 
	lightbox="true" 
	file="/images/runtime/cluster-install-rollout.png" 
	url="/images/runtime/cluster-install-rollout.png" 
	alt="Install Argo Rollouts" 
	caption="Install Argo Rollouts"
  max-width="40%" 
%}


#### Remove a managed cluster from the Codefresh UI 
Remove a cluster from the runtime's list of managed clusters from the Codefresh UI.

> You can also remove it through the CLI.

1. In the Codefresh UI, go to [Runtimes](https://g.codefresh.io/2.0/account-settings/runtimes){:target="\_blank"}.
1. Select either the **Topology View** or the **List View** tabs.
1. Do one of the following:
    * In the Topology View, select the cluster node from the runtime it is registered to. 
    * In the List View, select the runtime, and then select the **Managed Clusters** tab.
1. Select the three dots next to the cluster name, and then select **Uninstall** (Topology View) or **Remove** (List View). 

{% include 
	image.html 
	lightbox="true" 
	file="/images/runtime/managed-cluster-remove-single.png" 
	url="/images/runtime/managed-cluster-remove-single.png" 
	alt="Remove a managed cluster from runtime (List View)" 
	caption="Remove a managed cluster from runtime (List View)"
  max-width="50%" 
%}


#### Remove a managed cluster through the Codefresh CLI 
Remove a  cluster from the list managed by the runtime, through the CLI.  

* Run:  
  `cf cluster remove <runtime-name> --server-url <server-url>`  
  where:  
  `<runtime-name>` is the name of the runtime that the managed cluster is registered to.  
  `<server-url>` is the URL of the server on which the managed cluster is installed. 


### Related articles
[Add Git Sources to runtimes]({{site.baseurl}}/docs/runtime/git-sources/)  
[Manage provisioned hybrid runtimes]({{site.baseurl}}/docs/runtime/monitor-manage-runtimes/)  
[(Hybrid) Monitor provisioned runtimes]({{site.baseurl}}/docs/runtime/monitoring-troubleshooting/)  