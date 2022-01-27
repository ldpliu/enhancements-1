# ManagedClusterSet Override

## Release Signoff Checklist
 
- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)
 
## Summary
The proposed work enhances the managedClusterSet API to support managedClusterSet Override.

## Motivation
Today, managedClusterSet is designed to group the managedClusters, a managedCluster can only be included in only one managedClusterSet, by setting a label `cluster.open-cluster-management.io/clusterset` on the managedCluster.
A user must have explicit RBAC rules, `managedclustersets/join`, to add such labels.
There are also raising requirements on using managedClusterSet for the following purposes. Within a purpose, the managedClusterSet must be exclusive, but for different purposes it can overlap.
- As cluster admin, I want to create a managedClusterSet which includes all clusters in the apac region. Then I can apply certain configurations to all clusters in apac.
- As a SRE team member, I want to use a managedClusterSet for Disaster Recovery purposes. The managedClusterSet should select two or more clusters in different regions. 
- As a DEV team member, I want to use a managedClusterSet to select clusters based on the middleware enabled on these clusters, so I could run special applications in these clusters.

## Proposal

### Changes on managedClusterSet APIs.

Currently, managedClusterSets use the label `cluster.open-cluster-management.io/clusterset:<ManagedClusterSet Name>` to select managedClusters. 
But for some use cases, I want to use other labels to select managedClusters in a managedClusterSets.

So, In this proposal, we change the managedClusterSets spec and want to provid a flexible way to select managedClusters. 

```go
type ManagedClusterSetSpec struct {
    // Selector represents a selector of ManagedClusters by labels and names.
    ClusterSelector ManagedClusterSelector  `json:"clusterSelector"`
}

type ManagedClusterSelector struct{
    // "" means to use the current mechanism of matching label <cluster.open-cluster-management.io/clusterset:<ManagedClusterSet Name>.
    // "LabelSelector" means to use the LabelSelector to select target managedClusters
    // "ClusterLabel" means to use a particular cluster label. It is guaranteed that clustersets with same label key are exclusive with each others
    // +optional
    SelectorType SelectorType `json:"selectorType"`

    // For managedCluster labels which has prefix `cluster.open-cluster-management.io/*`, we will has secondary Access control check. And other labels do not have any secondary Access control protection.
    // ClusterLabel defines one label which clusterset could use to select target managedClusters. In this way, we could:
    // 1. Guarantee clustersets with same label key are exclusive
    // 2. Enable additional permission check when cluster joining/leaving a clusterset (the label key should start with the reserved prefix "cluster.open-cluster-management.io/" and "info.open-cluster-management.io/");
    ClusterLabel *ClusterLabel `json:"clusterLabel"`

    // LabelSelector define the general labelSelector which clusterset will use to select target managedClusters
    LabelSelector *metav1.LabelSelector `json:"labelSelector"`
}

type SelectorType string

const (
	  LabelSelector       SelectorType = "LabelSelector"
    ClusterLabel        SelectorType = "ClusterLabel"
)

//ClusterLabel defines one cluster label
type ClusterLabel struct {
	  Key string `json:"key"`
    Value string `json:"value"`
}
```

### RBAC
Currently, When I want to create a managedClusterSet, I only should have `create` permission to the managedClusterSet. 
In the managedCluster part, label `cluster.open-cluster-management.io/clusterset:<ManagedClusterSet Name>` means the cluster is in a managedClusterSet.
If I want to add the label `cluster.open-cluster-management.io/clusterset:<ManagedClusterSet Name>` to a managedCluster, I must have `create` permission for subresource `managedclustersets/join`. 
If I want to add/update other labels, I just need to have `update` permission to the managedCluster resource.

In this proposal, managedClusterSet could use any labels to select managedClusters. And if I add a label for a managedCluster, it may lead to the managedCluster added to a managedClusterSet. So we need to rethink about the rbac control of managedClusterSet

We will use the following questions to consider the RBAC control.

#### What purposes will managedClusterSet may be used for
a. Create a managedClusterSet for disaster recovery purposes, which should include specific clusters.
b. Create a managedClusterSet which should include all clusters in one region or cloud provider.
c. Create a managedClusterSet which has the same capacity(like: all clusters enabled middleware).
d. Create a managedClusterSet for each squad for resource group purposes.

#### Who can create a managedClusterSet
Same as the current way, Anyone who has `create` permission to `managedclustersets` could create managedClusterSet. 
But generally, Cluster admin should not give this permission to others. So only cluster admin could create the managedClusterSet.

#### What kinds of roles exist related to managedClusterSets and managedClusters
In General, there are three kinds of roles related to managedClusterSet.
1. Cluster admin
   - Cluster admin could `create`, `get`, `list`, `update`, `delete` all managedClusterSets.
   - Cluster admin could `bind` all managedClusterSets to all namespaces.
   - Cluster admin could `create`, `get`, `list`, `update`, `delete` all managedClusters.
   - Only Cluster admin could `accept` managedClusters.

2. Controller/agent which run in managedClusters
   - The Controller/agent should not `create`, `get`, `list`, `update`, `delete` managedClusterSets.
   - The Controller/agent should not `bind` managedClusterSet.
   - The Controller/agent only should `get` related managedCluster and `update` the managedCluster status and labels.

3. Non Cluster admin users
   - These users should `get` some managedClusterSets, and could not `create`, `update`, `delete`, `list` managedClusterSets.
   - These users should `bind` some managedClusterSets to their namespaces.
   - These users may need to `create` managedClusters(For Resource Group Purpose managedClusterSet[d], in each group, the group members may need to create the managedClusters). They could also `delete` these managedClusters, and `update` these managedClusters labels.

#### Who can add labels for managedClusters
1. Cluster admin
Cluster admin could add any labels to any managedClusters.

2. Controller/agent which run in managedClusters
Currentlly, the controller/agent may add labels like: `region: apac`, `cloud: Amazon`, `middlewareEnabled: true`. And these labels mean these clusters have some attributes.
- `region: apac` means the cluster is provisioned in apac region
- `cloud: Amazon` means the cluster is provisioned in Amazon cloud provider
- `middlewareEnabled: true` means this cluster enables the middleware.

3. Non Cluster admin users
Currently, non cluster admin users may also `add/update` some managedClusters labels. 

#### Add restrictions for adding/updating/deleting labels to managedClusters
In this proposal, managedClusterSet could use any labels to select managedClusters. And the new added labels may lead to this managedClusters added to a managedClusterSet.

But for some managedClusterSet(like: Disaster Recovery ManagedClusterSet[a], Resource Group ManagedClusterSet[d]), we do not want any users could add their managedClusters to these managedClusterSets. 

So we propose adding restrictions for some labels, and if user want to add these kinds of labels, he/she must have a "special" permission. Then the managedClusterSet which do not want anyone to join could use these labels to select managedClusters.

We propose adding restrictions for labels which key have prefix `cluster.open-cluster-management.io/` and `info.open-cluster-management.io/`
- `cluster.open-cluster-management.io/` are encouraged for non cluster admin users. Like for "Resource Group managedClusterSet", non cluster admin user could add a label like: `cluster.open-cluster-management.io/clusterset: dev`
- The spoke agents are encouraged to use labels which has prefix `info.open-cluster-management.io/` and the agents should have permissions on their own managedcluster to modify these kind of labels.
  The label key should be: `info.open-cluster-management.io/region`, `info.open-cluster-management.io/cloud`, `info.open-cluster-management.io/vendor`

Then we need a way to define the "special" permission.
Investigate the existing rbac rule in current k8s, we use a rule which is similar to [csr-approver](https://github.com/kubernetes/website/blob/3ca34b9a3be454055ba234f1d2ff7d55809b5040/content/zh/examples/access/certificate-signing-request/clusterrole-approve.yaml#L25)

We define a subresource `managedclusters/label`, which resource name is the label of managedCluster. This subresource means user could add/remove the label to managedClusters.

For example: 
- This permission means the users could add/remove the label `cluster.open-cluster-management.io/clusterset:dev` to their managedClusters.
```yaml
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters/label"]
   resourceNames: ["cluster.open-cluster-management.io/clusterset:dev"]
   verbs: ["create"]
```

- This permission means the users could add/remove any labels to their managedClusters.
```yaml
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters/label"]
   resourceNames: ["*"]
   verbs: ["create"]
```

- This permission means the users could add/remove labels with `cluster.open-cluster-management.io/clusterset` key and arbitrary value to their managedClusters.
```yaml
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters/label"]
   resourceNames: ["cluster.open-cluster-management.io/clusterset:*"]
   verbs: ["create"]
```

Notes: The `managedclustersets/join` permission will be deprecated, and the existing `managedclustersets/join` permission could be replaced with the following permission.
```yaml
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters/label"]
   resourceNames: ["cluster.open-cluster-management.io/clusterset:<ManagedClusterSet Name>"]
   verbs: ["create"]
```

#### Who can bind the managedClusterSet to a namespace
Same as the current way, anyone who has `managedclustersets/bind` permission could bind a managedClusterSet to a namespace.

### Examples
### Example1: As cluster admin, I want to create a managedClusterSet which includes all clusters in the apac region. Then I can apply certain configurations to all clusters in apac.
1. For each cluster, there should be an agent running on the managedCluster. The agent should have the `update` permission to related managedCluster
2. For each cluster in the apac region, related agents should add a label: `region:apac` to these managedClusters.
3. As cluster admin, I could create a managedClusterSet to select all managedClusters in `apac` region

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClusterSet
metadata:
  name: apac
spec:
  clusterSelector:
    selectorType: LabelSelector
    labelSelector:
      matchLabels:
        region: apac
```

This managedClusterSet includes all managedClusters with label `region:apac`, so the cluster admin could use this managedClusterSet to select all managedClusters in apac region, then apply a certain configuration to these clusters.

### Example2:  As a SRE team member, I want to use a managedClusterSet for Disaster Recovery purposes. The managedClusterSet should select two or more clusters in different regions. 
1. Cluster admin create three clusters: `backupCluster1`, `backupCluster2` and `backupCluster3` in the region: `apac`,`us`,`europe`. 
2. Cluster admin creates a managedClusterSet to specified these three clusters for Disaster Recovery purposes.
```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClusterSet
metadata:
  name: backupset
spec:
  clusterSelector:
    selectorType: ClusterNames
    clusterNames: "backupCluster1", "backupCluster2", "backupCluster3"
```

3. Cluster admin should give the following permissions to SRE team.
- `get` permission to three managedClusters `backupCluster1`, `backupCluster2`, `backupCluster3`
- `get` permission to managedClusterSet `backupset`
- `create` permission to subresource `managedclustersets/bind` to managedClusterSet `backupset`
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters"]
   resourceNames: ["backupCluster1", "backupCluster2", "backupCluster3"]
   verbs: ["*"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets"]
   resourceNames: ["backupset"]
   verbs: ["get"]
- apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/bind"]
   resourceNames: ["backupset"]
   verbs: ["create"]
```

4. As a SRE team member, I could bind the managedClusterSet to my namespace and do disaster and recovery work in these clusters.

### Example3: As a DEV team member, I want to use a managedClusterSet to select clusters based on the middleware enabled on these clusters, so I could run special applications in these clusters.
1. For each cluster, there should be an agent running on the managedCluster. The agent should have the `update` permission to related managedCluster
2. For each cluster which enables the middleware, related agents should add a label: `middlewareEnabled:true` to these managedClusters.
3. Cluster admin create a managedClusterSet `middlewareenabledset`
```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClusterSet
metadata:
  name: middlewareenabledset
spec:
  clusterSelector:
    selectorType: LabelSelector
    labelSelector:
      matchLabels:
        middlewareEnabled: "true"
```

5. Cluster admin should give following permissions to DEV team, so these team members could run applications in the managedClusterSet's clusters.
- `get` permission to managedClusterSet `middlewareenabledset`
- `create` permission to subresource `managedclustersets/bind` to managedClusterSet `middlewareenabledset`
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets"]
   resourceNames: ["middlewareenabledset"]
   verbs: ["get"]
- apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/bind"]
   resourceNames: ["middlewareenabledset"]
   verbs: ["create"]
```

6. As DEV team member, I could `bind` the managedClusterSet `middlewareenabledset` to my namespace and run application in the managedClusterSet.

### Example4: As cluster admin, I want to create two managedClusterSets for DEV and QA squad. and the DEV/QA could deploy applications in their own managedClusterSet.
1. Cluster admin create two managedClusterSets for Dev team and QA team
```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClusterSet
metadata:
  name: devset
spec:
  clusterSelector:
    selectorType: LabelSelector
    labelSelector:
      matchLabels:
        cluster.open-cluster-management.io/clusterset: dev
```

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClusterSet
metadata:
  name: qaset
spec:
  clusterSelector:
    selectorType: LabelSelector
    labelSelector:
      matchLabels:
        cluster.open-cluster-management.io/clusterset: qa
```

2. Cluster admin give the following permission to DEV team and QA team
Permission for DEV team
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters"]
   verbs: ["create"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets"]
   resourceNames: ["devset"]
   verbs: ["get"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters/label"]
   resourceNames: ["cluster.open-cluster-management.io/clusterset:devset"]
   verbs: ["create"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/bind"]
   resourceNames: ["devset"]
   verbs: ["create"]
```

Permission for QA team
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters"]
   verbs: ["create"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets"]
   resourceNames: ["qaset"]
   verbs: ["get"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters/label"]
   resourceNames: ["cluster.open-cluster-management.io/clusterset:qaset"]
   verbs: ["create"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/bind"]
   resourceNames: ["qaset"]
   verbs: ["create"]
```
3. As a DEV/QA team member, I can `create` a managedCluster, and add label `cluster.open-cluster-management.io/clusterset:devset/qaset` to the managedCluster.
4. As a DEV/QA team member, I could `bind` the managedClusterSet `devset/qaset` to my namespace and run application in the managedClusterSet.

## Test Plan
- Unit tests will cover the functionality of the controllers.
- Unit tests will cover the new APIs.
- And e2e tests will cover the following cases.
 - Create/update/delete managedClusterSet 
 - Create/update/delete managedClusters labels
 - Update managedClusterSet spec `clusterSelector` field

## Impact to other components
1. Impact to components which get managedClusters from a managedClusterSet.
[Placement](https://github.com/open-cluster-management-io/placement) 
Currently, `placement` use label `cluster.open-cluster-management.io/clusterset:<clusterset Name>` to select managedClusters, after the proposal applied, `placement` should use the `ClusterSelector` in managedClusterSet spec to select target clusters.

2. Impact to components which only use managedClusterSet for exclusive purpose.
a. [multicloud-operators-foundation](https://github.com/stolostron/multicloud-operators-foundation)
multicloud-operators-foundation use managedClusterSet for resource group purpose. So it should only watch the following managedClusterSet:
- `spec.ClusterSelector.SelectorType` is `ClusterLabel`, the `ClusterLabel.key` must be `cluster.open-cluster-management.io/clusterset`
- `spec.ClusterSelector.SelectorType` is ""

b. [submariner-addon](https://github.com/stolostron/submariner-addon)
Currentlly, `submariner-addon` try to use managedClusterSet group clusters based on network. And in different managedClusterSet, the clusters should be exclusive. So it should only watch the following managedClusterSet:
- `spec.ClusterSelector.SelectorType` is `ClusterLabel` and the `ClusterLabel.key` must be `cluster.open-cluster-management.io/clusterset`, the `ClusterLabel.value` must be managedClusterSet name.
- `spec.ClusterSelector.SelectorType` is ""

3. Impact to components which have RBAC check for adding managedClusters to a managedClusterSet.
[multicloud-operators-foundation](https://github.com/stolostron/multicloud-operators-foundation)
multicloud-operators-foundation use managedClusterSet for resource group purpose, and it gives the users `join` permission to a managedClusterSet if the user has admin permission to the managedClusterSet. 
So after the proposal, it should change the permission the following rule:
```yaml
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters/label"]
   resourceNames: ["cluster.open-cluster-management.io/clusterset:<ManagedClusterSet Name>"]
   verbs: ["create"]
```

## Option1 Upgrade / Downgrade Strategy 
The new api is compatible with the previous version. So there is no external work needed when upgrading

## Graduation Criteria
### Alpha
1. The new APIs is reviewed and accepted;
2. Implementation is completed to support the RBAC;

### Beta
1. Need to revisit the API shape before upgrading to beta based on user’s feedback.
