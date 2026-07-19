# PML Role Extension & Permission System

## Permission System

For the agent's own security system, permissions for different prompt inputs are designed and implemented using a permission system.

### RBAC Prompts

Security mechanisms implemented through role-based access control systems.

#### Master Prompts

The agent's own control commands from the owner are designated as Master Prompts.


## PML Role Extension

### Role Declaration

```
roles:
  - id: "admin"
    permissions: ["read", "write", "execute", "delete"]
    scope: "global"
  - id: "developer"
    permissions: ["read", "write", "execute"]
    scope: "project"
  - id: "viewer"
    permissions: ["read"]
    scope: "project"
```

### Role Binding

Roles can be bound to agents, workflows, or specific PNodes:

```
pnodes:
  - id: sensitive_operation
    type: tool
    required_role: "admin"
    action: "delete_records"
```

### Permission Inheritance

Roles can inherit from parent roles:

```
roles:
  - id: "super_admin"
    inherits: ["admin"]
    additional_permissions: ["system_config"]
```

### Dynamic Permission Checks

PMLScript can perform runtime permission checks:

```javascript
// Check if current agent has permission
if (await ctx.auth.hasPermission("write")) {
    await ctx.memory.write("critical_data", value);
}
```

### Permission Scopes

| Scope | Description |
|-------|-------------|
| `global` | Access to all resources across all projects |
| `project` | Limited to current project resources |
| `workflow` | Limited to current workflow execution |
| `node` | Limited to specific node execution |
