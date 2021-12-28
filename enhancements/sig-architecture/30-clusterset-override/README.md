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

- As cluster admin, I want to create a managedClusterSet which includes all clusters in one region. Then I can apply certain configurations to all clusters in the region.
- As a SRE team member, I want to create a managedClusterSet for Disaster Recovery purposes. The managedClusterSet should select two or more clusters in different regions.
- As a DEV team member, I want to create a managedClusterSet to group clusters based on the middleware enabled on these clusters, so I could run special applications in these clusters.

## Proposal
1. Changes on managedClusterSet APIs.

Currently, managedClusterSets use the label `cluster.open-cluster-management.io/clusterset=<ManagedClusterSet Name>` to select managedClusters. 

In this proposal, managedClusterSets could use any label to select managedClusters. Meanwhile, managedClusterSets should be exclusive within a purpose, but for different purposes it can overlap.

So, we add a field `ClusterLabel` into managedClusterSet spec

```go
type ManagedClusterSetSpec struct {
 // ClusterLabel defines the label which the managedClusterSet will use to select managedClusters.
 // If the field is not set, we will use "cluster.open-cluster-management.io/clusterset=<ManagedClusterSet Name>" as default value.
 // This field can not be changed once the managedClusterSet is created.
 // +optional
 // +immutable
 ClusterLabel string `json:"clusterLabel"`
}
```


2. RBAC change

a. Create a managedClusterSet
Currently, When a user wants to create a managedClusterSet, he/she should only have `create` permission to the managedClusterSet. 
After the proposal is applied, the new managedClusterSet may include any managedClusters which have specific labels. 
So we should use a permission to control the managedClusterSet creation with a specific label.

In order to control the managedClusterSet select managedClusters with a specific label, we use a subresource `managedclustersets/clusterlabel` permission. 
The subresource `managedclustersets/clusterlabel` permission means users could use this label to select managedClusters in a managedClusterSet.

So if I want to create a managedClusterSet `apacSet` with `ClusterLabel` `region:apac`, I must have the following permission.
- `create` permission to managedClusterSet `apacSet`
- `create` permission to subresource `managedclustersets/clusterlabel` with value `region:apac`

The permissions should be like:
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets"]
   resourceNames: ["apacSet"]
   verbs: ["create"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/clusterlabel"]
   resourceNames: ["region:apac"]
   verbs: ["create"]
```

Note: If I want to create a managedClusterSet which `ClusterLabel` is not set, the permission `managedclustersets/clusterlabel` is not needed.
If I create a managedClusterSet with `ClusterLabel` `cluster.open-cluster-management.io/clusterset=<Value>`. The `<Value>` must be managedClusterSet name, if not the request will be forbidden. (Forward Compatibility)


b. Add a label for managedCluster
Currently, anyone who has `update` permission to a managedCluster could update the managedCluster label(except label `cluster.open-cluster-management.io/clusterset=<xxx>`).
In this proposal, the managedClusterSet could use any label to select managedclusters. So when the managedCluster add a label, it may lead to the managedCluster added to a managedClusterSet if the managedClusterSet match the label. So we also need an permission to control the managedCluster's label change.

we use a subresource `managedclusters/label` to control the permission of managedCluster's label change.

So if I want to create a managedCluster named `mc1`, which has a label `region: apac`. I must have the following permission.
- `create` permission to managedCluster `mc1`
- `create` permission to subresource `managedclusters/label` with value `region:apac`

The permissions should be like:
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters"]
   resourceNames: ["mc1"]
   verbs: ["create"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters/label"]
   resourceNames: ["region:apac"]
   verbs: ["create"]
```

Note: If I want to add a label `cluster.open-cluster-management.io/clusterset=<xxx>` to a managedCluster, I should have either the following permissions:
- `create` permission for subresource `managedclustersets/join` with value `<xxx>`.(Forward Compatibility)
- `create` permission for subresource `managedclustersets/clusterlabel` with value `cluster.open-cluster-management.io/clusterset=<xxx>`

## Examples
### As cluster admin, I want to create a managedClusterSet which includes all clusters in the apac region. Then I can apply certain configurations to all clusters in apac.
1. As cluster admin, I should give the following permissions to agent(which run in managedclusters.)
- `update` permission for related cluster
- `create`, `update`, `delete` permissions for subresource `managedclusters/label` with value `region=apac`

The permissions should be like:
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters"]
   resourceNames: ["mc1"]
   verbs: ["update"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters/label"]
   resourceNames: ["region=apac"]
   verbs: ["create", "update", "delete"]
```

2. For each cluster in the apac region, related agent shoud add a label: `region:apac` to these managedCluster.
3. As cluster admin, I could create a managedClusterSet to select all clusters in `apac` region
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterSet
metadata:
  name: apac
spec:
  clusterLabel: region=apac
```

This managedClusterSet includes all clusters with label `region=apac`, so the cluster admin could use this managedClusterSet to select all clusters in apac region, then apply a certain configuration to these clusters.

### As a SRE team member, I want to create a managedClusterSet for Disaster Recovery purposes. The managedClusterSet should select two or more clusters in different regions. 
1. Cluster admin should give SRE team members the following permissions
- `create`, `update`, `delete` permissions for managedClusters
- `create`, `update`, `delete` permissions for subresource `managedclusters/label` with value `clusterAttribute=backup`
- `create`, `update`, `delete` permissions for managedClusterSets
- `create` permission for subresource `managedclusterset/clusterLabel` with value `clusterAttribute=backup`

The permissions should be like:
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
   resources: ["managedclusters/label"]
   resourceNames: ["clusterAttribute=backup"]
   verbs: ["*"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets"]
   resourceNames: ["backupSet"]
   verbs: ["*"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusterset/clusterLabel"]
   resourceNames: ["clusterAttribute=backup"]
   verbs: ["create"]
```
2. As a SRE team member, I could create three clusters `backupCluster1`, `backupCluster2`, `backupCluster3` in the region: `apac`,`us`,`europe`
3. As a SRE team member, I need to set a label `clusterAttribute=backup` for these managedClusters
4. As a SRE team member, I could create a managedClusterSet to select all the backup clusters
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterSet
metadata:
  name: backupSet
spec:
  clusterLabel: clusterAttribute=backup
```

This managedClusterSet includes all clusters with label `clusterAttribute=backup`, So I could use this managedClusterSet to select all backup clusters.


### As a DEV team member, I want to create a managedClusterSet to group clusters based on the middleware enabled on these clusters, so I could run special applications in these clusters.
1. Cluster admin should give SRE team members the following permissions to allow them create clusters and deploy applications in these clusters
- `create`/`update`/`delete` permissions for managedClusters
- `create`/`update`/`delete` permissions for subresource `managedclusters/label` with value `middlewareEnabled=true`

The permissions should be like:
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters"]
   resourceNames: ["*"]
   verbs: ["*"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters/label"]
   resourceNames: ["middlewareEnabled=true"]
   verbs: ["*"]
```

2. Cluster admin give DEV team the following permission to allow them create managedClusterSet using label `middlewareEnabled=true`
- `create`/`update`/`delete` permission for managedClusterSets
- `create` permission for subresource `managedclusterset/clusterLabel` with value `middlewareEnabled=true`

The permission should be like:
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets"]
   resourceNames: ["middlewareEnabledSet"]
   verbs: ["*"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/clusterLabel"]
   resourceNames: ["middlewareEnabled=true"]
   verbs: ["create"]
```
3. SRE team members create some clusters and enabled the middleware in these clusters.
4. SRE team members set label `middlewareEnabled=true` to these clusters.
5. As DEV team admin, I could create a managedClusterSet to select all clusters which enabled middleware.
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterSet
metadata:
  name: middlewareenabledSet
spec:
  clusterLabel: middlewareEnabled=true
```
This managedClusterSet includes all clusters with label `middlewareEnabled=true`, So I could use this managedClusterSet to run special applications which must enable middleware.

## Test Plan
- Unit tests will cover the functionality of the controllers.
- Unit tests will cover the new APIs.
- And e2e tests will cover the following cases.
 - Create/update/delete managedClusterSet with permission `managedclustersets/clusterlabel`
 - Create/update/delete managedClusterSet without permission `managedclustersets/clusterlabel`
 - Create/update/delete managedClusters labels with permission `managedclusters/label`
 - Create/update/delete managedClusters labels without permission `managedclusters/label` 
 - Update managedClusterSet spec `ClusterLabel` field

## Impact to other components
1. [Placement](https://github.com/open-cluster-management-io/placement) 

Currently, `placement` use label `cluster.open-cluster-management.io/clusterset=<clusterset Name>` to select managedclusters, after the proposal applied, `placement` should use the `clusterlabel` in managedClusterSet spec to select target clusters.

2. Any components which create managedCluster with labels or update managedCluster's labels should have permissions `managedclusters/label` to related labels.

## Upgrade / Downgrade Strategy
As the update managedCluster labels permission changed. So any role which need to update managedclusters or create managedclusters with labels need to add the external permissions like: 
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters/label"]
   resourceNames: ["region=apac"]
   verbs: ["*"]
```

## Graduation Criteria
### Alpha
1. The new APIs is reviewed and accepted;
2. Implementation is completed to support the RBAC;

### Beta
1. Need to revisit the API shape before upgrading to beta based on user’s feedback.
