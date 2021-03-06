# EgressIP IPAM Operator

This operator automates the assignment of egressIPs to namespaces.
Namespaces can opt in to receiving one or more egressIPs with the following annotation `egressip-ipam-operator.redhat-cop.io/egressipam:<egressIPAM>`, where `egressIPAM` is the CRD that controls how egressIPs are assigned.
IPs assigned to the namespace can be looked up in the following annotation: `egressip-ipam-operator.redhat-cop.io/egressips`.

EgressIP assignments is managed by the EgressIPAM CRD. Here is an example of it:

```yaml
apiVersion: redhatcop.redhat.io/v1alpha1
kind: EgressIPAM
metadata:
  name: example-egressipam
spec:
  cidrAssignment:
    - labelValue: "true"
      CIDR: 192.169.0.0/24
  nodeLabel: egressGateway
  nodeSelector:
    node-role.kubernetes.io/worker: ""
```

This EgressIPAM specifies that all nodes that comply with the specified node selector and that also have labels `egressGateway: "true"` will be assigned egressIP from the specified CIDR.

Note that the `cidrAssigment` field is an array and therefore, multiple groups of nodes ca be identified with the labelValue and different CIDRs can be assigned to them. This is usually not necessary on a bare metal deployment.

When this egressCRD is created all the `hostsubnet` relative to the nodes selected by this EgressIPAM will be update to have the EgressCIDRs field equal to the specified CIDR.

When a namespace is created with the opt-in annotation, a free egressIP is selected from the CIDR and assigned to the namespace.The `netnamespace` associated with this namespace is update to use that egressIP.

## Support for AWS

In AWS as well as other cloud providers, one cannot freely assign IPs to machines. Additional steps need to be performed in this case. Considering this EgressIPAM

```yaml
apiVersion: redhatcop.redhat.io/v1alpha1
kind: EgressIPAM
metadata:
  name: example-egressipam
spec:
  cidrAssignment:
    - labelValue: us-east-1
      CIDR: 192.169.0.0/24
    - labelValue: us-east-2
      CIDR: 192.170.0.0/24
  nodeLabel: topology.kubernetes.io/zone
  nodeSelector:
    node-role.kubernetes.io/worker: ""
```

When a namespace with the opt-in annotation is created, the following happens:

1. for each of the CIDRs, an available IP is selected and assigned to the namespace
2. the relative `netnamespace` is update to reflect the assignment (multiple IPs will be assigned in this case).
3. one node per zone is selected to carry the egressIP
4. the relative aws machines are assigned the additional IP on the main interface (support for secondary interfaces in not available)
5. the relative `hostsubnets` are updated to reflect the assigned IP, the `egressIP` field is updated.

## Local Development

Execute the following steps to develop the functionality locally. It is recommended that development be done using a cluster with `cluster-admin` permissions.

```shell
go mod download
```

optionally:

```shell
go mod vendor
```

Using the [operator-sdk](https://github.com/operator-framework/operator-sdk), run the operator locally:

```shell
oc apply -f deploy/crds/redhatcop.redhat.io_egressipams_crd.yaml
oc new-project egressip-ipam-operator
oc apply -f deploy/service_account.yaml -n egressip-ipam-operator
oc apply -f deploy/role.yaml -n egressip-ipam-operator
oc apply -f deploy/role_binding.yaml -n egressip-ipam-operator
export token=$(oc serviceaccounts get-token 'egressip-ipam-operator')
oc login --token=${token}
OPERATOR_NAME='egressip-ipam-operator' operator-sdk --verbose up local --namespace ""
```

## Testing


## Release Process

To release execute the following:

```shell
git tag -a "<version>" -m "release <version>"
git push upstream <version>
```

use this version format: vM.m.z