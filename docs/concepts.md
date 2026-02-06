# Crossplane Composition Concepts

## What is Crossplane?

Crossplane is a Kubernetes extension that transforms your cluster into a universal control plane. It allows you to manage cloud infrastructure using Kubernetes-style declarative APIs.

## Core Components

### Managed Resources (MRs)

The lowest-level resources that directly represent cloud provider resources:

- Azure VirtualNetwork
- AWS VPC
- GCP Network

Example:

```yaml
apiVersion: network.azure.upbound.io/v1beta1
kind: VirtualNetwork
metadata:
  name: my-vnet
spec:
  forProvider:
    location: westeurope
    resourceGroupName: my-rg
    addressSpace:
      - 10.0.0.0/16
```

### Composite Resource Definitions (XRDs)

XRDs define custom API types that abstract cloud resources:

```yaml
apiVersion: apiextensions.crossplane.io/v2
kind: CompositeResourceDefinition
metadata:
  name: xnetworks.infrastructure.azure.com
spec:
  group: infrastructure.azure.com
  names:
    kind: XNetwork
    plural: xnetworks
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  properties:
                    region:
                      type: string
                    cidr:
                      type: string
                  required:
                    - region
                    - cidr
```

### Composite Resources (XRs)

Instances of XRDs that users create:

```yaml
apiVersion: infrastructure.azure.com/v1alpha1
kind: XNetwork
metadata:
  name: production-network
spec:
  parameters:
    region: westeurope
    cidr: 10.0.0.0/16
```

### Compositions

Define how XRs are translated into Managed Resources or other XRs:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: azure-network
spec:
  compositeTypeRef:
    apiVersion: infrastructure.azure.com/v1alpha1
    kind: XNetwork
  resources:
    - name: vnet
      base:
        apiVersion: network.azure.upbound.io/v1beta1
        kind: VirtualNetwork
        spec:
          forProvider:
            addressSpace:
              - 10.0.0.0/16
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.location
```

## XRD v2 Changes

### API Version Change

**XRD v2 uses `apiVersion: apiextensions.crossplane.io/v2`** (not v1!)

### Scope Options

XRD v2 simplifies the resource model with three scope options:

**Namespaced (Default in v2)**:

```yaml
apiVersion: apiextensions.crossplane.io/v2
kind: CompositeResourceDefinition
spec:
  scope: Namespaced  # Default - resources exist in namespaces
  names:
    kind: XNetwork
    plural: xnetworks
```

**Cluster**:

```yaml
spec:
  scope: Cluster  # Cluster-scoped, no claims
```

**LegacyCluster** (v1 compatibility):

```yaml
spec:
  scope: LegacyCluster  # Cluster-scoped with claims support
  claimNames:
    kind: Network
    plural: networks
```

### Migration from v1

Old (v1):

```yaml
apiVersion: apiextensions.crossplane.io/v1  # ← Old API version
kind: CompositeResourceDefinition
spec:
  claimNames:           # ← Claims defined here
    kind: Network
    plural: networks
  names:
    kind: XNetwork
    plural: xnetworks
```

New (v2):

```yaml
apiVersion: apiextensions.crossplane.io/v2  # ← New API version
kind: CompositeResourceDefinition
spec:
  scope: Namespaced    # ← No claims by default
  names:
    kind: XNetwork
    plural: xnetworks
```

## Composition of Compositions Pattern

### Why Compose XRs?

1. **Modularity**: Build reusable infrastructure components
1. **Abstraction Layers**: Hide complexity at each level
1. **Separation of Concerns**: Different teams own different layers
1. **Reusability**: Combine components in different ways

### Three-Layer Architecture

**Layer 1: Managed Resources**
Direct cloud provider resources (VNet, Subnet, NSG)

**Layer 2: Component XRs**
Reusable building blocks (Network, Security, Monitoring)

**Layer 3: Platform XRs**
Complete solutions (Landing Zone, Application Platform)

### Example Hierarchy

```
XAzureLandingZone (Platform Layer)
├── XNetwork (Component Layer)
│   ├── VirtualNetwork (Managed Resource)
│   ├── Subnet (Managed Resource)
│   └── RouteTable (Managed Resource)
├── XSecurity (Component Layer)
│   ├── NetworkSecurityGroup (Managed Resource)
│   └── Firewall (Managed Resource)
└── XMonitoring (Component Layer)
    ├── LogAnalyticsWorkspace (Managed Resource)
    └── ApplicationInsights (Managed Resource)
```

## Key Patterns

### Pattern 1: Direct Composition

Parent XR directly creates child XRs:

```yaml
spec:
  resources:
    - name: network
      base:
        apiVersion: infrastructure.azure.com/v1alpha1
        kind: XNetwork
```

### Pattern 2: Parameter Propagation

Pass configuration from parent to child:

```yaml
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.parameters.region
    toFieldPath: spec.parameters.region
```

### Pattern 3: Status Publishing

Child XR publishes information back to parent:

```yaml
patches:
  - type: ToCompositeFieldPath
    fromFieldPath: status.networkId
    toFieldPath: status.components.networkId
```

### Pattern 4: Resource References

One component references another:

```yaml
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: status.networkId
    toFieldPath: spec.parameters.networkId
```

## Best Practices

### Naming Conventions

- **XRD Kind**: Prefix with `X` (e.g., `XNetwork`)
- **API Group**: Use reverse domain (e.g., `infrastructure.azure.com`)
- **Versions**: Start with `v1alpha1`, progress to `v1beta1`, then `v1`

### Parameter Design

**Good**: Simple, focused parameters

```yaml
spec:
  parameters:
    region: westeurope
    environment: production
```

**Avoid**: Exposing too many low-level details

```yaml
spec:
  parameters:
    vnetAddressSpace: ["10.0.0.0/16"]
    subnet1Name: "frontend"
    subnet1AddressPrefix: "10.0.1.0/24"
    # Too granular!
```

### Composition Strategy

1. Start with managed resources
1. Group related resources into component XRs
1. Build platform XRs from components
1. Keep each layer focused and cohesive

### Testing

Test each layer independently:

```bash
# Test component XR
kubectl apply -f network-xrd.yaml
kubectl apply -f network-composition.yaml
kubectl apply -f test-network.yaml

# Verify it works before composing
kubectl get xnetwork
kubectl describe xnetwork test-network

# Then test platform XR
kubectl apply -f landing-zone-xrd.yaml
kubectl apply -f landing-zone-composition.yaml
kubectl apply -f test-landing-zone.yaml
```

## Common Pitfalls

### 1. Circular Dependencies

❌ **Wrong**: XRD A references XRD B, which references XRD A
✅ **Right**: Clear hierarchy with no cycles

### 2. Too Many Layers

❌ **Wrong**: 5+ layers of XRDs
✅ **Right**: 2-3 layers maximum

### 3. Tight Coupling

❌ **Wrong**: Child XR requires specific status fields from parent
✅ **Right**: Child XR works independently

### 4. Missing Defaults

❌ **Wrong**: Requiring users to specify everything
✅ **Right**: Sensible defaults for common scenarios

## Debugging Tips

### Check XRD Installation

```bash
kubectl get xrd
kubectl describe xrd xnetworks.infrastructure.azure.com
```

### Verify Composition

```bash
kubectl get composition
kubectl describe composition azure-network
```

### Inspect XR Status

```bash
kubectl get xnetwork production-network -o yaml
```

### Check Events

```bash
kubectl describe xnetwork production-network
kubectl get events --sort-by='.lastTimestamp'
```

### View Composed Resources

```bash
kubectl get managed
kubectl get virtualnetwork
```

## Additional Resources

- [Crossplane Documentation](https://docs.crossplane.io/)
- [Composition Reference](https://docs.crossplane.io/latest/concepts/compositions/)
- [XRD Specification](https://docs.crossplane.io/latest/concepts/composite-resource-definitions/)
