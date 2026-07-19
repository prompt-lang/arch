# PML Environments Extension

## Environment Declaration

PML supports declaring execution environments for workflows and nodes:

```
environments:
  - id: "production"
    type: "cloud"
    region: "us-east-1"
    resources:
      cpu: "4"
      memory: "16Gi"
    scaling:
      min_instances: 2
      max_instances: 10

  - id: "staging"
    type: "cloud"
    region: "us-west-2"
    resources:
      cpu: "2"
      memory: "8Gi"

  - id: "edge"
    type: "edge"
    device: "jetson-nano"
    resources:
      cpu: "4"
      memory: "4Gi"
      gpu: "128-core-maxwell"
```

## Environment Binding

Workflows and PNodes can be bound to specific environments:

```
workflow:
  environment: "production"
  
pnodes:
  - id: inference_node
    type: predictor
    environment: "edge"
    model: "yolov8n.onnx"
```

## Environment Variables

```
environments:
  - id: "production"
    env:
      LOG_LEVEL: "info"
      API_ENDPOINT: "https://api.example.com"
      MAX_BATCH_SIZE: "32"
```

## Environment Propagation

Child workflows and sub-PNodes inherit the parent's environment unless explicitly overridden:

```
pnodes:
  - id: parent_node
    type: subworkflow
    workflow: "child_workflow"
    environment: "inherit"    # Default behavior
```

## Resource Constraints

```
environments:
  - id: "limited"
    resources:
      cpu: "1"
      memory: "2Gi"
    constraints:
      max_execution_time_sec: 300
      max_network_egress_mb: 100
      allowed_regions: ["us-east-1", "eu-west-1"]
```
