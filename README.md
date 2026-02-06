# Learning Crossplane Composition of Compositions

A comprehensive guide to implementing layered infrastructure compositions using Crossplane XRD v2, demonstrated through an Azure Landing Zone example.

## Overview

This repository demonstrates the **Composition of Compositions** pattern in Crossplane, where higher-level infrastructure abstractions are built by composing lower-level XRs (Composite Resources). This approach enables modular, reusable infrastructure components that can be layered to create complex platforms.

### Key Concepts

- **XRD v2**: Uses `apiVersion: apiextensions.crossplane.io/v2` with no Claims by default
- **Layered Architecture**: Multiple levels of abstraction (e.g., Landing Zone → Network → Subnets → Individual Resources)
- **Modular Design**: Reusable components that can be mixed and matched
- **Azure Landing Zone**: Enterprise-ready foundation following Microsoft’s Cloud Adoption Framework

## Architecture

```
┌─────────────────────────────────────────────────┐
│  Azure Landing Zone XR (Top Level)              │
│  - Networking XR                                │
│  - Security XR                                  │
│  - Monitoring XR                                │
└─────────────────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Network XR   │ │ Security XR  │ │ Monitor XR   │
│ - VNet       │ │ - NSGs       │ │ - Log Analy. │
│ - Subnets    │ │ - Firewall   │ │ - App Insig. │
└──────────────┘ └──────────────┘ └──────────────┘
        │
        ▼
┌──────────────────────────────────┐
│  Managed Resources (Azure)       │
│  - VirtualNetwork                │
│  - Subnet                        │
│  - NetworkSecurityGroup          │
│  - etc.                          │
└──────────────────────────────────┘
```

## Directory Structure

```
.
├── README.md
├── docs/
│   ├── concepts.md                    # Core concepts and patterns
│   ├── azure-landing-zone.md          # Azure Landing Zone architecture
│   └── troubleshooting.md             # Common issues and solutions
├── examples/
│   ├── 01-basic/
│   │   ├── README.md
│   │   ├── simple-xrd.yaml            # Single-level XRD
│   │   └── simple-xr.yaml             # XR instance
│   ├── 02-nested/
│   │   ├── README.md
│   │   ├── parent-xrd.yaml            # Parent XRD
│   │   ├── child-xrd.yaml             # Child XRD
│   │   ├── parent-composition.yaml    # Composes child XR
│   │   └── parent-xr.yaml             # Parent XR instance
│   └── 03-azure-landing-zone/
│       ├── README.md
│       ├── landing-zone-xrd.yaml      # Top-level Landing Zone
│       ├── landing-zone-composition.yaml
│       ├── network-xrd.yaml           # Network layer
│       ├── network-composition.yaml
│       ├── security-xrd.yaml          # Security layer
│       ├── security-composition.yaml
│       ├── monitoring-xrd.yaml        # Monitoring layer
│       ├── monitoring-composition.yaml
│       └── landing-zone-instance.yaml # Full deployment
├── xrds/
│   └── azure/
│       ├── network/
│       │   ├── vnet-xrd.yaml
│       │   ├── vnet-composition.yaml
│       │   ├── subnet-xrd.yaml
│       │   └── subnet-composition.yaml
│       ├── security/
│       │   ├── nsg-xrd.yaml
│       │   ├── nsg-composition.yaml
│       │   ├── firewall-xrd.yaml
│       │   └── firewall-composition.yaml
│       └── monitoring/
│           ├── log-analytics-xrd.yaml
│           ├── log-analytics-composition.yaml
│           ├── app-insights-xrd.yaml
│           └── app-insights-composition.yaml
└── tests/
    └── azure-landing-zone/
        ├── test-basic.yaml
        ├── test-with-firewall.yaml
        └── test-multi-region.yaml
```

## Prerequisites

- Kubernetes cluster (1.25+)
- Crossplane installed (1.14+)
- Azure Provider for Crossplane configured
- Azure credentials with appropriate permissions

## Quick Start

### 1. Install Crossplane and Azure Provider

```bash
# Install Crossplane
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system --create-namespace

# Install Azure Provider
kubectl apply -f - <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure-network
spec:
  package: xpkg.upbound.io/upbound/provider-azure-network:v1.0.0
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure-managedidentity
spec:
  package: xpkg.upbound.io/upbound/provider-azure-managedidentity:v1.0.0
EOF
```

### 2. Configure Azure Provider

```bash
# Create Azure service principal and configure ProviderConfig
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: azure-credentials
  namespace: crossplane-system
type: Opaque
stringData:
  credentials: |
    {
      "clientId": "YOUR_CLIENT_ID",
      "clientSecret": "YOUR_CLIENT_SECRET",
      "subscriptionId": "YOUR_SUBSCRIPTION_ID",
      "tenantId": "YOUR_TENANT_ID"
    }
---
apiVersion: azure.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      name: azure-credentials
      namespace: crossplane-system
      key: credentials
EOF
```

### 3. Deploy Example Compositions

```bash
# Start with basic example
kubectl apply -f examples/01-basic/

# Try nested composition
kubectl apply -f examples/02-nested/

# Deploy full Azure Landing Zone
kubectl apply -f examples/03-azure-landing-zone/
```

## Learning Path

1. **Start with Basic Concepts** (`docs/concepts.md`)
- Understand XRDs vs XRs
- Learn composition basics
- Explore XRD v2 changes
1. **Simple XRD** (`examples/01-basic/`)
- Single-level composition
- Direct managed resource creation
1. **Nested Composition** (`examples/02-nested/`)
- Parent-child XR relationships
- Resource references between layers
1. **Azure Landing Zone** (`examples/03-azure-landing-zone/`)
- Multi-layer architecture
- Enterprise patterns
- Production considerations

## Key Patterns

### Pattern 1: Referencing Child XRs

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: parent-composition
spec:
  compositeTypeRef:
    apiVersion: example.com/v1alpha1
    kind: XParent
  resources:
    - name: child-network
      base:
        apiVersion: example.com/v1alpha1
        kind: XNetwork  # Reference to child XR type
        spec:
          parameters:
            cidr: "10.0.0.0/16"
```

### Pattern 2: Passing Parameters Down

```yaml
spec:
  resources:
    - name: network
      base:
        apiVersion: infrastructure.azure.com/v1alpha1
        kind: XNetwork
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.region
          toFieldPath: spec.parameters.region
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.tags
          toFieldPath: spec.parameters.tags
```

### Pattern 3: Resource References

```yaml
spec:
  resources:
    - name: security-rules
      base:
        apiVersion: infrastructure.azure.com/v1alpha1
        kind: XSecurityRules
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: status.networkId
          toFieldPath: spec.parameters.networkId
```

## Azure Landing Zone Components

The example Azure Landing Zone includes:

- **Hub-Spoke Network Topology**
  - Hub VNet with shared services
  - Spoke VNets for workloads
  - VNet peering configurations
- **Security**
  - Network Security Groups (NSGs)
  - Azure Firewall (optional)
  - DDoS Protection
  - Private DNS Zones
- **Monitoring & Governance**
  - Log Analytics Workspace
  - Application Insights
  - Diagnostic Settings
  - Azure Policy assignments
- **Identity & Access**
  - Managed Identities
  - Role assignments
  - Key Vault integration

## Best Practices

1. **Naming Conventions**
- Use clear, descriptive names for XRDs
- Prefix with `X` for XR kinds (e.g., `XNetwork`)
- Version your APIs properly
1. **Parameter Design**
- Keep parent XRs simple and opinionated
- Allow child XRs to expose more configuration
- Use sensible defaults
1. **Composition Strategy**
- Start with managed resources
- Build reusable components (child XRs)
- Compose into platform abstractions (parent XRs)
1. **Testing**
- Test each layer independently
- Validate parameter propagation
- Check resource dependencies
1. **Documentation**
- Document each XRD’s purpose
- Provide usage examples
- Explain parameter choices

## Troubleshooting

See <docs/troubleshooting.md> for common issues and solutions.

## Resources

- [Crossplane Documentation](https://docs.crossplane.io/)
- [Azure Provider Documentation](https://marketplace.upbound.io/providers/upbound/provider-azure/)
- [Azure Landing Zone Architecture](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/)
- [Crossplane Slack Community](https://slack.crossplane.io/)

## Contributing

This is a learning repository. Feel free to:

- Add more examples
- Improve documentation
- Share your learnings
- Report issues or suggest improvements

## License

MIT License - See LICENSE file for details

## Author

Willem van Heemstra

- GitHub: [@vanHeemstraSystems](https://github.com/vanHeemstraSystems)


-----

**Note**: This repository focuses on XRD v2 patterns. If you need Claims-based examples, see the legacy documentation in Crossplane docs.
