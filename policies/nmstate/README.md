# nmstate AutoShift Policy

## Overview
This policy installs the kubernetes-nmstate-operator operator using AutoShift patterns.

## Status
✅ **Operator Installation**: Ready to deploy  
🔧 **Configuration**: Requires operator-specific setup (see below)

## Quick Deploy

### Test Locally
```bash
# Validate policy renders correctly
helm template policies/nmstate/
```

### Enable on Clusters
Edit AutoShift values files to add the operator labels:

```yaml
# In autoshift/values.hub.yaml (or values.sbx.yaml, etc.)
hubClusterSets:
  hub:
    labels:
      nmstate: 'true'
      nmstate-subscription-name: 'kubernetes-nmstate-operator'
      nmstate-channel: 'stable'
      nmstate-source: 'redhat-operators'
      nmstate-source-namespace: 'openshift-marketplace'
      # nmstate-version: 'kubernetes-nmstate-operator.v1.x.x'  # Optional: pin to specific CSV version

managedClusterSets:
  managed:
    labels:
      nmstate: 'true'
      nmstate-subscription-name: 'kubernetes-nmstate-operator'
      nmstate-channel: 'stable'
      nmstate-source: 'redhat-operators'
      nmstate-source-namespace: 'openshift-marketplace'
      # nmstate-version: 'kubernetes-nmstate-operator.v1.x.x'  # Optional: pin to specific CSV version

# For specific clusters (optional override)
clusters:
  my-cluster:
    labels:
      nmstate: 'true'
      nmstate-channel: 'fast'  # Override channel for this cluster
```

Labels are automatically propagated to clusters via the cluster-labels policy.

### Add to AutoShift ApplicationSet
Edit `autoshift/templates/applicationset.yaml` and add:
```yaml
- name: nmstate
  path: policies/nmstate
  helm:
    valueFiles:
    - values.yaml
```

## Configuration

### Namespace Scope
This operator is configured as:
- **Cluster-scoped**: Manages resources across all namespaces (default)
- **Namespace-scoped**: Limited to specific target namespaces (if `targetNamespaces` enabled in values.yaml)

To change scope, edit `values.yaml` and uncomment/configure the `targetNamespaces` field.

### Version Control
This policy supports AutoShift's operator version control system:

- **Automatic Upgrades**: By default, the operator follows automatic upgrade paths within its channel
- **Version Pinning**: Add `nmstate-version` label to pin to a specific CSV version
- **Manual Control**: Pinned versions require manual updates to upgrade

To pin to a specific version, add the version label to your cluster or clusterset:
```yaml
nmstate-version: 'kubernetes-nmstate-operator.v1.x.x'
```

Find available CSV versions:
```bash
# List available versions for this operator
oc get packagemanifests kubernetes-nmstate-operator -o jsonpath='{.status.channels[*].currentCSV}'
```

## Next Steps: Configuration

### 1. Explore Installed CRDs
After operator installation, check what Custom Resources are available:
```bash
# Wait for operator to install
oc get pods -n openshift-nmstate

# Check available CRDs
oc get crds | grep nmstate

# Explore CRD specifications
oc explain <CustomResourceName>
```

### 2. Create Configuration Policies
Add operator-specific configuration policies to `templates/` directory.

#### Common Patterns:
- `policy-nmstate-config.yaml` - Main configuration
- `policy-nmstate-<feature>.yaml` - Feature-specific configs

#### Template Structure:
```yaml
{{- $policyName := "policy-nmstate-config" }}
{{- $placementName := "placement-policy-nmstate-config" }}

apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: {{ $policyName }}
  namespace: {{ .Values.policy_namespace }}
  annotations:
    policy.open-cluster-management.io/standards: NIST SP 800-53
    policy.open-cluster-management.io/categories: CM Configuration Management
    policy.open-cluster-management.io/controls: CM-2 Baseline Configuration
spec:
  disabled: false
  dependencies:
    - name: policy-nmstate-operator-install
      namespace: {{ .Values.policy_namespace }}
      apiVersion: policy.open-cluster-management.io/v1
      compliance: Compliant
      kind: Policy
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: nmstate-config
        spec:
          remediationAction: enforce
          severity: high
          object-templates:
            - complianceType: musthave
              objectDefinition:
                apiVersion: # Your operator's API version
                kind: # Your operator's Custom Resource
                metadata:
                  name: nmstate-config
                  namespace: {{ .Values.nmstate.namespace }}
                spec:
                  # Your operator-specific configuration
                  # Use dynamic labels when needed:
                  # setting: '{{ "{{hub" }} index .ManagedClusterLabels "autoshift.io/nmstate-setting" | default "default-value" {{ "hub}}" }}'
          pruneObjectBehavior: None
---
# Use same placement as operator install or create specific targeting
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: {{ $placementName }}
  namespace: {{ .Values.policy_namespace }}
spec:
  clusterSets:
  {{- range $clusterSet, $value := $.Values.hubClusterSets }}
    - {{ $clusterSet }}
  {{- end }}
  {{- range $clusterSet, $value := $.Values.managedClusterSets }}
    - {{ $clusterSet }}
  {{- end }}
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: 'autoshift.io/nmstate'
              operator: In
              values:
              - 'true'
  tolerations:
    - key: cluster.open-cluster-management.io/unreachable
      operator: Exists
    - key: cluster.open-cluster-management.io/unavailable
      operator: Exists
---
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: {{ $placementName }}
  namespace: {{ .Values.policy_namespace }}
placementRef:
  name: {{ $placementName }}
  apiGroup: cluster.open-cluster-management.io
  kind: Placement
subjects:
  - name: {{ $policyName }}
    apiGroup: policy.open-cluster-management.io
    kind: Policy
```

### 3. Reference Examples
**Study similar complexity policies:**
- **Simple**: `policies/openshift-gitops/` - Basic operator + ArgoCD config
- **Medium**: `policies/advanced-cluster-security/` - Multiple related policies
- **Complex**: `policies/metallb/` - Multiple configuration types (L2, BGP, etc.)
- **Advanced**: `policies/openshift-data-foundation/` - Storage cluster configuration

### 4. AutoShift Labels
Add configuration labels to `values.yaml` and use in templates:

```yaml
# Add to values.yaml AutoShift Labels Documentation:
# nmstate-setting<string>: Configuration option (default: 'value')
# nmstate-feature-enabled<bool>: Enable optional feature (default: 'false')
# nmstate-provider<string>: Provider-specific config (default: 'generic')

# Use in templates:
setting: '{{ "{{hub" }} index .ManagedClusterLabels "autoshift.io/nmstate-setting" | default "default-value" {{ "hub}}" }}'
```

## Common Patterns

### CSV Status Checking (Optional)
For operators that need installation verification:
```yaml
- objectDefinition:
    apiVersion: policy.open-cluster-management.io/v1
    kind: ConfigurationPolicy
    metadata:
      name: nmstate-csv-status
    spec:
      remediationAction: inform
      severity: high
      object-templates:
        - complianceType: musthave
          objectDefinition:
            apiVersion: operators.coreos.com/v1alpha1
            kind: ClusterServiceVersion
            metadata:
              namespace: {{ .Values.nmstate.namespace }}
            status:
              phase: Succeeded
```

### ArgoCD Sync Annotations (If Needed)
For policies requiring special sync behavior:
```yaml
annotations:
  argocd.argoproj.io/sync-options: Prune=false,SkipDryRunOnMissingResource=true
  argocd.argoproj.io/compare-options: IgnoreExtraneous
  argocd.argoproj.io/sync-wave: "1"
```

## Troubleshooting

### Policy Not Applied
1. Check cluster labels: `oc get managedcluster <cluster> --show-labels`
2. Verify placement: `oc get placement -n open-cluster-policies`
3. Check policy status: `oc describe policy policy-nmstate-operator-install`

### Operator Installation Issues
1. Check subscription: `oc get subscription -n openshift-nmstate`
2. Check install plan: `oc get installplan -n openshift-nmstate`
3. Verify operator source exists: `oc get catalogsource -n openshift-marketplace`

### Template Rendering Issues
1. Test locally: `helm template policies/nmstate/`
2. Check hub escaping: Look for `{{ "{{hub" }} ... {{ "hub}}" }}` patterns
3. Validate YAML: `helm lint policies/nmstate/`

## Resources
- [Operator Documentation](https://operatorhub.io/operator/kubernetes-nmstate-operator) - Find your operator details
- [AutoShift Policy Patterns](../../README-DEVELOPER.md) - Comprehensive policy development guide  
- [ACM Policy Documentation](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes) - Policy syntax reference in Governence Section
- [Similar Policies](../) - Browse other policies for patterns and examples