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
But for some use cases, I want to use other labels to select managedClusters in a managedClusterSets. Meanwhile, managedClusterSets should be exclusive within a purpose, but for different purposes it can overlap.
So, In this proposal, managedClusterSets could use any labels to select managedClusters. 

So, we add a field `ClusterLabel` into managedClusterSet spec

```go
 type ManagedClusterSetSpec struct {
 // ClusterLabel defines the label which the managedClusterSet will use to select managedClusters.
 // If the field is not set, we will use "cluster.open-cluster-management.io/clusterset:<ManagedClusterSet Name>" as default value.
 // +optional
 ClusterLabel ClusterLabel  `json:"clusterLabel"`
}

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
If I want to add/update other labels, I just need to have `update` permission to the managedCluster.

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

#### Who can join a managedCluster to a managedClusterSet
In this proposal, managedClusterSet could use any labels to select managedClusters. If one user adds a label to a managedClusters, it may lead to this managedClusters added to a managedClusterSet. So this question also means "who can add/update a label for managedClusters".

1. Cluster admin
Cluster admin could add any labels to any managedClusters.

2. Controller/agent which run in managedClusters
The controller/agent may add labels like: `region: apac`, `cloud: Amazon`, `middlewareEnabled: true`
These labels should not be changed by the user. And these labels mean these clusters have some attributes.
- `region: apac` means the cluster is provisioned in apac region
- `cloud: Amazon` means the cluster is provisioned in Amazon cloud provider
- `middlewareEnabled: true` menas this cluster enables the middleware.

3. Non Cluster admin users
Currently, non cluster admin users may `add/update` some managedClusters labels. And these labels may lead to the managedClusters added to a managedClusterSet.

But for some managedClusterSet(like: Disaster Recovery ManagedClusterSet[a], Resource Group ManagedClusterSet[d]), we do not want any users to add their managedClusters to these managedClusterSets. 

For these managedClusterSets, we need a way to identify them. 
We popose use the prefix `cluster.open-cluster-management.io/` in managedClusterSet `spec.clusterLabel.key` to identify the managedClusterSet which need "special" permission to join. (This prefix may be configured. But in this proposal, it is const)

So if someone wants to add the label `cluster.open-cluster-management.io/*:*` to a managedCluster. He/She should have the "special" permission.

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

So we group the managedClusterSet into two categories based on the managedClusterSet purposes, 
- ManagedClusterSets do not need permission to join, like: "Region/Cloud Provider ManagedClusterSet[b]" and "Middleware ManagedClusterSet[c]".
   These managedClusterSets `spec.clusterLabel.key` do not have the prefix `cluster.open-cluster-management.io/`. Anyone could join to these managedClusterSets by setting a label in their managedClusters when the new label matchs managedClusterSet `spec.clusterLabel`.

- ManagedClusterSets only allow someone to join, like: "Disaster Recovery ManagedClusterSet[a]" and "Resource Group ManagedClusterSet[d]".
  These managedClusterSets `spec.clusterLabel.key` should have the prefix `cluster.open-cluster-management.io/`. If someone want to add a label which key has prefix `cluster.open-cluster-management.io/`, he/she should have `create` permission to subresource `managedclusters/label` which resourceName is the `<LabelKey: LabelValue>`.


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
  clusterLabel: 
    key: region
    value: apac
```

This managedClusterSet includes all managedClusters with label `region:apac`, so the cluster admin could use this managedClusterSet to select all managedClusters in apac region, then apply a certain configuration to these clusters.

### Example2:  As a SRE team member, I want to use a managedClusterSet for Disaster Recovery purposes. The managedClusterSet should select two or more clusters in different regions. 
1. Cluster admin creates a managedClusterSet for Disaster Recovery purposes.
```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClusterSet
metadata:
  name: backupset
spec:
  clusterLabel: 
    key: cluster.open-cluster-management.io/clusterAttribute
    value: backup
```

2. Cluster admin should give the following permissions to SRE team.
- `create`, `update`, `delete` permission to three managedClusters `backupCluster1`, `backupCluster2`, `backupCluster3`
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
   verbs: ["create"]
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

3. As a SRE team member, I could create three clusters: `backupCluster1`, `backupCluster2` and `backupCluster3` in the region: `apac`,`us`,`europe`. And set a label `cluster.open-cluster-management.io/clusterAttribute:backup` for these managedClusters. 

4. As a SRE team member, I could bind the managedClusterSet to my namespace and do disaster and recovery work in these clusters.

This managedClusterSet includes all managedClusters with label `cluster.open-cluster-management.io/clusterAttribute:backup`, So I could use this managedClusterSet to select all backup clusters.

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
  clusterLabel: 
    key: middlewareEnabled
    value: true
```
4. Cluster admin should give following permissions to DEV team, so these team members could run applications in the managedClusterSet's clusters.
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

This managedClusterSet `middlewareenabledset` includes all managedClusters with label `middlewareenabledset: true`, So I could use this managedClusterSet to select all clusters which enabled the middleware, then run application in these clusters.

## Test Plan
- Unit tests will cover the functionality of the controllers.
- Unit tests will cover the new APIs.
- And e2e tests will cover the following cases.
 - Create/update/delete managedClusterSet 
 - Create/update/delete managedClusters labels
 - Update managedClusterSet spec `ClusterLabel` field

## Impact to other components
1. [Placement](https://github.com/open-cluster-management-io/placement) 

Currently, `placement` use label `cluster.open-cluster-management.io/clusterset:<clusterset Name>` to select managedClusters, after the proposal applied, `placement` should use the `clusterLabel` in managedClusterSet spec to select target clusters.

## Option1 Upgrade / Downgrade Strategy 
The new api is compatible with the previous version. So there is no external work needed when upgrading

## Graduation Criteria
### Alpha
1. The new APIs is reviewed and accepted;
2. Implementation is completed to support the RBAC;

### Beta
1. Need to revisit the API shape before upgrading to beta based on user’s feedback.
