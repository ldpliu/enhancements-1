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
Today, managedClusterSet is designed to be exclusive. A managedCluster can only be included in only one managedClusterSet, by setting a label `cluster.open-cluster-management.io/clusterset` on the managedCluster. A user must have explicit RBAC rules, `managedclustersets/join`, to add such labels.
 
There are also raising requirements on using managedClusterSet for the following purposes. Within a purpose, the managedClusterSet must be exclusive, but for different purposes it can overlap.
- I want to create a managedClusterSet which includes all clusters in one region. Then I can apply certain configurations to all clusters in the region.
- I want to create a managedClusterSet for Disaster Recovery purposes, the managedClusterSet should select two or more clusters in different regions.
- I want to create a managedClusterSet to group clusters based on the middleware enabled on these clusters, so I could run special applications in these clusters.
 
## Proposal
1. Changes on ManagedClusterSet APIs.
 
Add a field into ManagedClusterSet spec
 
```go
type ManagedClusterSetSpec struct {
 // ExclusiveKey defines the key of the label which the managedClusterSet will use to select managedCluster.
 // The managedClusterSet will select ManagedClusters based on the managedCluster's label "<ExclusiveKey>:<ManagedClusterSetName>".
 // If the exclusiveKey is not set, we will use `cluster.open-cluster-management.io/clusterset` as exclusiveKey.
 // The value can not change once the managedClusterSet is created.
 // +optional
 ExclusiveKey string `json:"exclusiveKey"`
}
```
 
Notes:
As managedClusterSet uses the label `<ExclusiveKey>:<ManagedClusterSetName>` to filter managedclusters. So if I want to use one label key as exclusiveKey, the label value must match the k8s name rule.
 
2. RBAC change
 
Given the flexibility introduced by exclusive key, instead of control whether a cluster can be added into a managedClusterSet with sub resource of `managedclustersets/join`, it might make more sense to control whether a user can create a managedClusterSet with a certain exclusive key. We can define a subresource of `managedclustersets/exclusiveKey`, to control the creation of certain managedClusterSet.
 
I want to create the managedClusterSet `apac`, I must have the following permission.
 
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterSet
metadata:
name: apac
spec:
exclusiveKey: region
```
 
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
 
 
## Examples
### I want to create a managedClusterSet which include all clusters in apac region. Then I can apply certain configurations to all clusters in apac.
1. For each cluster, I need to set a label `region=apac` for all clusters in the apac region.
2. I need to get the following permission to create a managedClusterSet.
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
 
3. Create a managedClusterSet to select all cluster in apac
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterSet
metadata:
name: apac
spec:
exclusiveKey: region
```
 
This managedClusterSet includes all clusters with label `region: apac`, so I could use this managedClusterSet to select all clusters in apac, then apply a certain configuration to these clusters.
 
### I want to create a managedClusterSet for Disaster Recovery purposes, the managedClusterSet should select two or more clusters in different regions.
1. I need to manually select one cluster in each region: `apac`,`us`,`europe`, and set label: `clusterAttribute=backup` for these clusters
2. Get the following permission to create a managedClusterSet.
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
   verbs: ["*"]
```
 
3. Create a managedClusterSet to select
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterSet
metadata:
name: backup
spec:
exclusiveKey: clusterAttribute
```
 
### I want to create a managedClusterSet to group clusters based on the middleware enabled on these clusters, so I could run special applications in these clusters.
1. For all clusters which enabled middleware, I need to set label: `middlewareAttribute=middlewareenabled`
2. Get the following permission to create a managedClusterSet.
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/exclusiveKey"]
   resourceNames: ["middlewareAttribute"]
   verbs: ["create"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets"]
   resourceNames: ["middlewareenabled"]
   verbs: ["*"]
```
 
3. Create a managedClusterSet to select all clusters which enabled middleware.
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterSet
metadata:
name: middlewareenabled
spec:
exclusiveKey: middlewareAttribute
```
 
### Risks and Mitigation
Currently, if I want to create a managedClusterSet, I must get permissions about two kinds of resources, `managedclustersets/exclusiveKey` and `managedclustersets`. When there are multiple resources for both `managedclustersets/exclusiveKey` and `managedclustersets`, it may cause permission leak.
 
For example:
1. I want to have permission to create a managedClusterSet which name is `apac` and `exclusiveKey` is `region`, I need to get the following permissions.
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
 
2. I want to has permission to create another managedClusterSet which name is `aws` and `exclusiveKey` is `vendor`, I need to get the following permissions
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
 
3. If I get permissions in both #1 and #2, Actually, I also get the permissions to create another two managedClusterSets. This is what I should not have.
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
 
 
 
### Test Plan
- Unit tests will cover the functionality of the controllers.
- Unit tests will cover the new APIs.
- And e2e tests will cover the following cases.
 - Create/update/delete managedClusterSet with permission `managedclustersets/exclusiveKey`
 - Create/update/delete managedClusterSet without permission `managedclustersets/exclusiveKey`
 - Update managedClusterSet spec exclusiveKey
 
### Upgrade / Downgrade Strategy
N/A
 
### Graduation Criteria
#### Alpha
1. The new APIs is reviewed and accepted;
2. Implementation is completed to support the RBAC;
 
#### Beta
1. Need to revisit the API shape before upgrading to beta based on user’s feedback.
