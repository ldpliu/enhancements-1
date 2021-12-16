# Adding Exclusive Key in ManagedClusterSet
 
## Release Signoff Checklist
 
- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)
 
## Summary
The proposed work enhances the managedClusterSet API to support managedClusterSet Override.

## Motivation
Today, managedClusterSet is designed to group the managedClusters, then we can control a group of managedClusters' permissions. 
So a managedCluster can only be included in only one managedClusterSet, by setting a label `cluster.open-cluster-management.io/clusterset` on the managedCluster.
A user must have explicit RBAC rules, `managedclustersets/join`, to add such labels.

There are also raising requirements on using managedClusterSet for the following purposes. Within a purpose, the managedClusterSet must be exclusive, but for different purposes it can overlap.
- As cluster admin, I want to create a managedClusterSet which includes all clusters in one region. Then I can apply certain configurations to all clusters in the region.
- As a SRE team member, I want to create a managedClusterSet for Disaster Recovery purposes. The managedClusterSet should select two or more clusters in different regions.
- As a DEV team member, I want to use a managedClusterSet to group clusters based on the middleware enabled on these clusters, so I could run special applications in these clusters.

## Proposal
1. Changes on ManagedClusterSet APIs.

Add a field into ManagedClusterSet spec

```go
type ManagedClusterSetSpec struct {
 // ExclusiveKey defines the key of the label which the managedClusterSet will use to select managedCluster.
 // The managedClusterSet will select ManagedClusters based on the managedCluster's label "<ExclusiveKey>:<ManagedClusterSetName>".
 // If the exclusiveKey is not set, we will use `cluster.open-cluster-management.io/clusterset` as exclusiveKey.
 // This field can not be changed once the managedClusterSet is created.
 // +optional
 // +immutable
 ExclusiveKey string `json:"exclusiveKey"`
 ClusterLabel ClusterLabel `json:"clusterLabel"`
}

type ClusterLabel struct {
  Key string `json:"key"`
  Value string `json:"value"`
}

```

The managedClusterSet will select ManagedClusters based on the managedCluster's label "<ExclusiveKey>:<ManagedClusterSetName>", instead of using general label selector, the current way is compatible with older apis by setting exclusiveKey as `cluster.open-cluster-management.io/clusterset`. And it can make sure the managedClusterSet is exclusive within a purpose, but for different purposes it can overlap.

Notes:
As managedClusterSet uses the label `<ExclusiveKey>:<ManagedClusterSetName>` to filter managedClusters. So if I want to use one label key as exclusiveKey, the label value must match the k8s name rule.

2. RBAC change

a. Permission change for managedClusters
Currently, Anyone who has `update` permission to a managedCluster could update the managedCluster label. 

After the proposal is applied, the managedCluster's label change may lead to the managedCluster added to /removed from a managedClusterSet.
So we need to control the permission of managedCluster's label change.

We still use subresource `managedclustersets/join` to check if the user has permission to join the managedClusterSet.

So if there is a managedCluster named `mc1`, which has a label `region: apac`. and there is a managedClusterSet which name is `apac` and exclusiveKey is `region`. 
If I try to update the `mc1`'s label: `region: apac`, I should have the following permissions
  - `update` permission to the managedCluster `mc1`
  - `create` permission of subresource `managedclustersets/join` to managedClusterSets `apac`
The cluster role should be like:
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/join"]
   resourceNames: ["apac"]
   verbs: ["create"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters"]
   resourceNames: ["mc1"]
   verbs: ["*"]
```

b. Permission change for managedClusterSet
Currently, When a user wants to create a managedClusterSet, he/she only should have `create` permission to the managedClusterSet. 
After the proposal is applied, the created managedClusterSet will include all managedClusters which have the label `<ManagedClusterSet ExclusiveKey: ManagedClusterSet Name>`. 

So the managedClusterSet creator should have the `update` permission to these managedClusters which will be added to the managedClusterSet. 
Additionally, We can define a subresource of `managedclustersets/exclusiveKey`, to control the creation of certain managedClusterSet.

So if I want to create the managedClusterSet `apac` with exclusiveKey `region`, I must have the following permission.
- `update` permission to all managedClusters which have label: `region: apac`. 
  Note: 
    In order to check `update` permission to all managedClusters which have label: `region: apac`, we need to check the `update` permissions for these managedClusters one by one, it will have performance issue, So we can simplify the check with the following:
    a. `update` permission to all managedClusters
    b. Or there is no managedClusters have label `region: apac`
- `create` permission to managedClusterSet `apac`
- `create` permission to subresource `managedclustersets/exclusiveKey` with value `region`

A possible cluster role should be like:
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/exclusiveKey"]
   resourceNames: ["region"]
   verbs: ["create"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets"]
   resourceNames: ["apac"]
   verbs: ["create"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters"]
   verbs: ["update"]
```

3. Impact to clients
Currentlly, the new change may have impact to existing components.
[placement](https://github.com/open-cluster-management-io/placement)
Currentlly, the placement use `cluster.open-cluster-management.io/clusterset` as label key to select target cluster in one clusterset.
after the proposal applied, placement should handle it's code and using managedclusterset's `exclusiveKey` as label key to select target cluster. 


## Example
### As cluster admin, I want to create a managedClusterSet which includes all clusters in the apac region. Then I can apply certain configurations to all clusters in apac.
1. cluster-admin create a managedClusterSet `apac`
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterSet
metadata:
  name: apac
spec:
  exclusiveKey: region
```
2. cluster-admin add managedClusterSets `join` permission to agent which should allow the agent to join a cluster to managedClusterSet `apac`.
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/join"]
   resourceNames: ["apac"]
   verbs: ["create"] 
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets"]
   resourceNames: ["mc1"]
   verbs: ["update"]
```
3. user `dev1` create a managedCluster `mc1` in region `apac`
4. agent in `mc1` add the label `region: apac`, then the managedCluster `mc1` will be added to managedClusterSet `apac`


### As a SRE team member, I want to create a managedClusterSet for Disaster Recovery purposes. The managedClusterSet should select two or more clusters in different regions. 
1. cluster-admin add managedClusterSet `create` and `join` permission to `sre1`
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/exclusiveKey"]
   resourceNames: ["clusterAttribute"]
   verbs: ["create"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets"]
   resourceNames: ["backup"]
   verbs: ["create"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/join"]
   resourceNames: ["backup"]
   verbs: ["create"] 
```
2. user `sre1` create a managedClusterSet `backup` with exclusiveKey `clusterAttribute`.
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterSet
metadata:
  name: backup
spec:
  exclusiveKey: clusterAttribute
```
3. user `sre1` create three managedClusters in region `apac`, `us` and `europe`
4. user `sre1` add label `clusterAttribute: backup` to these managedClusters, then these three managedClusters will be added to managedClusterSet `backup`


### As a DEV team member, I want to use a managedClusterSet to group clusters based on the middleware enabled on these clusters, so I could run special applications in these clusters.
1. user `dev1` create a cluster `mc1`
2. user `dev1` enable the middleware on the cluster `mc1`, and set label `middlewareAttribute: middlewareenabled` to `mc1`.
3. cluster-admin create a managedClusterSet `middlewareenabled` with exclusiveKey `middlewareAttribute`.
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterSet
metadata:
  name: middlewareenabled
spec:
  exclusiveKey: middlewareAttribute
```
4. cluster-admin add managedClusterSets `join` permission to user `dev2`
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/join"]
   resourceNames: ["apac"]
   verbs: ["create"] 
```
5. user `dev2` create a managedCluster `mc2`
6. user `dev2` enable the middleware on `mc2` and add the label `middlewareAttribute: middlewareenabled` to `mc2`


### Risks and Mitigation
1. If I want to create a managedClusterSet, I must have permissions about two kinds of resources, `managedclustersets/exclusiveKey` and `managedclustersets`. When there are multiple resources for both `managedclustersets/exclusiveKey` and `managedclustersets`, it may cause permission leak.

For example:
a. I want to have permission to create a managedClusterSet which name is `apac` and `exclusiveKey` is `region`, I need to get the following permissions.
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/exclusiveKey"]
   resourceNames: ["region"]
   verbs: ["create"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets"]
   resourceNames: ["apac"]
   verbs: ["*"]
```

b. I want to has permission to create another managedClusterSet which name is `aws` and `exclusiveKey` is `vendor`, I need to get the following permissions
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/exclusiveKey"]
   resourceNames: ["vendor"]
   verbs: ["create"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets"]
   resourceNames: ["aws"]
   verbs: ["*"]
```

c. If I get permissions in both #a and #b, Actually, I also get the permissions to create another two managedClusterSets. This is unexpected.
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterSet
metadata:
  name: apac
spec:
  exclusiveKey: vendor
```

and

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterSet
metadata:
  name: aws
spec:
  exclusiveKey: region
```

#### Mitigation
As the managedClusterSet name should be unique in one hub cluster, and if the clusterset name is defined, the meaningful exclusiveKey should also be unique.

So the cluster admin should know the meaningful pairs for all managedClusterSet. He/She could store them in a configmap.

```yaml
apiVersion: v1
kind: ConfigMap
metadata: 
  name: clusterset-validation
  namespace: open-cluster-management-hub
data:
  region: apac, us
  vendor: aws, gcp
```

If there is a managedClusterSet is creating, we need to check if the managedClusterSet's `name` matches the value of one pair  
or the managedClusterSet's `exclusiveKey` match the key of some pairs in configmap data.
If yes, the `<ManagedClusterSet ExclusiveKey: ManagedClusterSet Name>` should matches the whole pair. Or the creating request should be forbidden.

### Test Plan
- Unit tests will cover the functionality of the controllers.
- Unit tests will cover the new APIs.
- And e2e tests will cover the following cases.
 - Create/update/delete managedClusterSet with permission `managedclustersets/exclusiveKey`
 - Create/update/delete managedClusterSet without permission `managedclustersets/exclusiveKey`
 - Update managedClusterSet spec exclusiveKey

### Upgrade / Downgrade Strategy
No changes required for existing clusters to use the enhancement.

### Graduation Criteria
#### Alpha
1. The new APIs is reviewed and accepted;
2. Implementation is completed to support the RBAC;
3. The Mitigation section will not be implemented

#### Beta
1. Need to revisit the API shape before upgrading to beta based on user’s feedback.
