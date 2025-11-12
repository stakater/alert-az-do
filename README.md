# alert-az-do

[![Build Status](https://github.com/jm-stakater/alert-az-do/actions/workflows/test.yaml/badge.svg?branch=master)](https://github.com/jm-stakater/alert-az-do/actions?query=workflow%3Atest)
[![Go Report Card](https://goreportcard.com/badge/github.com/jm-stakater/alert-az-do)](https://goreportcard.com/report/github.com/jm-stakater/alert-az-do) 
[![GoDoc](https://godoc.org/github.com/jm-stakater/alert-az-do?status.svg)](https://godoc.org/github.com/jm-stakater/alert-az-do)
[![Slack](https://img.shields.io/badge/join%20slack-%23alert-az-do-brightgreen.svg)](https://slack.cncf.io/)

[Prometheus Alertmanager](https://github.com/prometheus/alertmanager) webhook receiver for [Azure DevOps](https://azure.microsoft.com/en-us/products/devops/).

## Overview

alert-az-do implements Alertmanager's webhook HTTP API and connects to one or more Azure DevOps organizations to create highly configurable Azure DevOps work items. One work item is created per distinct group key — as defined by the [`group_by`](https://prometheus.io/docs/alerting/configuration/#<route>) parameter of Alertmanager's `route` configuration section — but not closed when the alert is resolved. The expectation is that a human will look at the work item, take any necessary action, then close it. If no human interaction is necessary then it should probably not alert in the first place. This behavior however can be modified by setting the `auto_resolve` section, which will resolve the Azure DevOps work item with the required state.

If a corresponding Azure DevOps work item already exists but is resolved, it is reopened. An Azure DevOps state transition must exist between the resolved state and the reopened state — as defined by `reopen_state` — or reopening will fail. Optionally a "skip reopen state" — defined by `skip_reopen_state` — may be defined: an Azure DevOps work item in this state will not be reopened by alert-az-do (e.g., work items marked as "Removed" or "Cut").

## Features

- **Multiple Authentication Methods**: Support for Service Principal, Managed Identity, and Personal Access Token authentication
- **Flexible Work Item Creation**: Create different types of work items (Bug, Task, Issue, etc.) based on alert content
- **Template-based Content**: Use Go templates to generate dynamic work item titles, descriptions, and field values
- **Auto-resolution**: Automatically resolve work items when alerts are resolved
- **Custom Fields**: Set standard and custom Azure DevOps fields using templates
- **Multi-project Support**: Search across multiple projects for existing work items
- **Label Management**: Copy Prometheus labels as Azure DevOps tags
- **Update Modes**: Choose between updating work items directly or adding comments
- **Environment Variable Support**: Environment variables take precedence over config file settings

## Usage

Get alert-az-do, either as a [packaged release](https://github.com/jm-stakater/alert-az-do/releases) or build it yourself:

```bash
go get github.com/jm-stakater/alert-az-do/cmd/alert-az-do
```

then run it from the command line:

```bash
alert-az-do
```

Use the `-help` flag to get help information.

```bash
alert-az-do -help
Usage of alert-az-do:
  -config string
      The alert-az-do configuration file (default "config/alert-az-do.yml")
  -listen-address string
      The address to listen on for HTTP requests. (default ":9097")
  -log-level string
      Only log messages with the given severity or above (debug, info, warn, error) (default "info")
  -log-format string
      Output format of log messages (logfmt, json) (default "logfmt")
```

## Testing

alert-az-do expects a JSON object from Alertmanager. The format of this JSON is described in the [Alertmanager documentation](https://prometheus.io/docs/alerting/configuration/#<webhook_config>) or, alternatively, in the [Alertmanager GoDoc](https://godoc.org/github.com/prometheus/alertmanager/template#Data).

To quickly test if alert-az-do is working you can run:

```bash
curl -H "Content-type: application/json" -X POST \
  -d '{"receiver": "contoso-ab", "status": "firing", "alerts": [{"status": "firing", "labels": {"alertname": "TestAlert", "severity": "critical"} }], "groupLabels": {"alertname": "TestAlert"}}' \
  http://localhost:9097/alert
```

## Configuration

The configuration file is essentially a list of receivers matching 1-to-1 all Alertmanager receivers using alert-az-do; plus defaults (in the form of a partially defined receiver); and a pointer to the template file.

Each receiver must have a unique name (matching the Alertmanager receiver name), Azure DevOps API access fields (organization, authentication credentials), a handful of required work item fields (such as the Azure DevOps project and work item summary), some optional work item fields (e.g. priority, area path, iteration path) and a `fields` map for other (standard or custom) Azure DevOps fields. Most of these may use [Go templating](https://golang.org/pkg/text/template/) to generate the actual field values based on the contents of the Alertmanager notification. The exact same data structures and functions as those defined in the [Alertmanager template reference](https://prometheus.io/docs/alerting/notifications/) are available in alert-az-do.

Similar to Alertmanager, alert-az-do supports environment variable substitution with the `$(...)` syntax.

### Authentication

alert-az-do supports three authentication methods with automatic precedence handling:

#### 1. Service Principal (recommended for applications)
Use when running alert-az-do as an application with Azure AD registration:

```yaml
organization: contoso
tenant_id: "12345678-1234-1234-1234-123456789012"
client_id: "87654321-4321-4321-4321-210987654321" 
client_secret: $(CLIENT_SECRET)
```

#### 2. Managed Identity (recommended for Azure resources)
Use when running alert-az-do on Azure compute resources (VMs, Container Instances, AKS, etc.):

```yaml
organization: contoso
client_id: "87654321-4321-4321-4321-210987654321"
subscription_id: "11111111-2222-3333-4444-555555555555"
```

#### 3. Personal Access Token (simplest for development)
Use for development or when other methods are not feasible:

```yaml
organization: contoso
personal_access_token: $(PAT_TOKEN)
```

#### Authentication Precedence

alert-az-do uses the following precedence order (highest to lowest):

1. **Environment Variables** (any complete authentication method)
   - Service Principal: `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`
   - Managed Identity: `AZURE_CLIENT_ID`, `AZURE_SUBSCRIPTION_ID`  
   - PAT: `AZURE_PAT`

2. **Configuration File** (receiver-specific settings)

3. **Configuration File** (defaults section)

**Important**: Authentication methods are mutually exclusive. You cannot mix Service Principal fields with Managed Identity fields or PAT tokens.

#### Environment Variable Examples

```bash
# Service Principal via environment
export AZURE_TENANT_ID="12345678-1234-1234-1234-123456789012"
export AZURE_CLIENT_ID="87654321-4321-4321-4321-210987654321"
export AZURE_CLIENT_SECRET="your-secret-here"

# OR Managed Identity via environment  
export AZURE_CLIENT_ID="87654321-4321-4321-4321-210987654321"
export AZURE_SUBSCRIPTION_ID="11111111-2222-3333-4444-555555555555"

# OR PAT via environment
export AZURE_PAT="your-pat-token-here"
```

### Example Configuration

```yaml
# Global defaults - Service Principal authentication
defaults:
  organization: contoso
  tenant_id: $(TENANT_ID)
  client_id: $(CLIENT_ID)  
  client_secret: $(CLIENT_SECRET)
  
  issue_type: Bug
  priority: '{{ template "azdo.priority" . }}'
  summary: '{{ template "azdo.summary" . }}'
  description: '{{ template "azdo.description" . }}'
  reopen_state: "To Do"
  skip_reopen_state: "Removed"
  
receivers:
  # Inherits Service Principal from defaults
  - name: 'team-alpha'
    project: TeamAlpha
    add_group_labels: true
    
  # Uses Managed Identity authentication  
  - name: 'team-beta-managed'
    project: TeamBeta
    client_id: $(MI_CLIENT_ID)
    subscription_id: $(SUBSCRIPTION_ID)
    issue_type: Task
    fields:
      System.AssignedTo: '{{ .CommonLabels.owner }}'
      
  # Uses PAT authentication
  - name: 'team-gamma-pat'
    project: TeamGamma  
    personal_access_token: $(GAMMA_PAT)
    issue_type: Issue
    priority: Critical

template: alert-az-do.tmpl
```

#### Alternative Example - Environment Variable Override

When environment variables are set, they take precedence over config file settings:

```yaml
# Minimal config - authentication comes from environment variables
defaults:
  organization: contoso  # AZURE_TENANT_ID, AZURE_CLIENT_ID, AZURE_CLIENT_SECRET set via env
  issue_type: Bug
  summary: '{{ template "azdo.summary" . }}'
  reopen_state: "To Do"
  
receivers:
  - name: 'alerts'
    project: Operations

template: alert-az-do.tmpl
```

```bash
# Environment variables provide authentication
export AZURE_TENANT_ID="12345678-1234-1234-1234-123456789012"
export AZURE_CLIENT_ID="87654321-4321-4321-4321-210987654321"  
export AZURE_CLIENT_SECRET="your-secret-here"
```

### Template Functions

alert-az-do provides additional template functions beyond the standard Alertmanager functions:

- `toUpper`: Convert string to uppercase
- `toLower`: Convert string to lowercase  
- `join`: Join string slice with separator
- `match`: Test if string matches regex pattern
- `reReplaceAll`: Replace all regex matches in string
- `stringSlice`: Create a string slice from arguments
- `getEnv`: Get environment variable value

## Alertmanager Configuration

To enable Alertmanager to talk to alert-az-do you need to configure a webhook in Alertmanager. You can do that by adding a webhook receiver to your Alertmanager configuration.

```yaml
receivers:
- name: 'team-alpha'
  webhook_configs:
  - url: 'http://localhost:9097/alert'
    # Send resolved alerts if you want auto-resolution
    send_resolved: true
```

## Azure DevOps Setup

### Permissions Required

The authentication method you choose needs the following Azure DevOps permissions:

- **Work Items**: Read & write
- **Project and team**: Read (to access project information)

### Method 1: Personal Access Token (PAT)

**Best for**: Development, testing, or simple setups

1. Go to Azure DevOps → User Settings → Personal Access Tokens
2. Create a new token with "Work Items (Read & write)" scope  
3. Use the token in your configuration:
   ```yaml
   personal_access_token: $(PAT_TOKEN)
   ```

### Method 2: Azure AD Service Principal

**Best for**: Production applications, CI/CD pipelines

1. Go to Azure Portal → Azure Active Directory → App registrations
2. Create a new application
3. Note the **Application (client) ID** and **Directory (tenant) ID**
4. Create a client secret in "Certificates & secrets"
5. Add the application to your Azure DevOps organization:
   - Go to Azure DevOps → Organization Settings → Users
   - Add the service principal with appropriate permissions
6. Use in configuration:
   ```yaml
   tenant_id: "your-tenant-id"
   client_id: "your-client-id" 
   client_secret: $(CLIENT_SECRET)
   ```

### Method 3: Managed Identity

**Best for**: Azure compute resources (VMs, Container Instances, AKS)

1. Enable managed identity on your Azure resource
2. Note the **Client ID** of the managed identity
3. Add the managed identity to your Azure DevOps organization:
   - Go to Azure DevOps → Organization Settings → Users  
   - Add the managed identity with appropriate permissions
4. Use in configuration:
   ```yaml
   client_id: "your-managed-identity-client-id"
   subscription_id: "your-azure-subscription-id"
   ```

### Container/Kubernetes Deployment

For containerized deployments, use environment variables:

```yaml
# Kubernetes Secret example
apiVersion: v1
kind: Secret
metadata:
  name: alert-az-do-auth
type: Opaque
stringData:
  AZURE_TENANT_ID: "12345678-1234-1234-1234-123456789012"
  AZURE_CLIENT_ID: "87654321-4321-4321-4321-210987654321"
  AZURE_CLIENT_SECRET: "your-secret-here"
```

```yaml
# Deployment example
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alert-az-do
spec:
  template:
    spec:
      containers:
      - name: alert-az-do
        image: ghcr.io/jm-stakater/alert-az-do:latest
        envFrom:
        - secretRef:
            name: alert-az-do-auth
```

## Profiling

alert-az-do imports [`net/http/pprof`](https://golang.org/pkg/net/http/pprof/) to expose runtime profiling data on the `/debug/pprof` endpoint. For example, to use the pprof tool to look at a 30-second CPU profile:

```bash
go tool pprof http://localhost:9097/debug/pprof/profile
```

To enable mutex and block profiling (i.e. `/debug/pprof/mutex` and `/debug/pprof/block`) run alert-az-do with the `DEBUG` environment variable set:

```bash
env DEBUG=1 ./alert-az-do
```

## Docker

alert-az-do is available as a Docker image with multiple authentication options:

### Service Principal via Environment Variables
```bash
docker run \
  -v $(pwd)/config:/config \
  -p 9097:9097 \
  -e AZURE_TENANT_ID="12345678-1234-1234-1234-123456789012" \
  -e AZURE_CLIENT_ID="87654321-4321-4321-4321-210987654321" \
  -e AZURE_CLIENT_SECRET="your-secret-here" \
  ghcr.io/jm-stakater/alert-az-do:latest \
  -config /config/alert-az-do.yml
```

### PAT via Environment Variables  
```bash
docker run \
  -v $(pwd)/config:/config \
  -p 9097:9097 \
  -e AZURE_PAT="your-pat-token-here" \
  ghcr.io/jm-stakater/alert-az-do:latest \
  -config /config/alert-az-do.yml
```

### Using Config File Only
```bash
docker run -v $(pwd)/config:/config \
  -p 9097:9097 \
  ghcr.io/jm-stakater/alert-az-do:latest \
  -config /config/alert-az-do.yml
```

## Troubleshooting

### Authentication Issues

#### "missing authentication in receiver" Error
This error occurs when no complete authentication method is configured. Ensure you have one of:

- **Service Principal**: `tenant_id`, `client_id`, and `client_secret`
- **Managed Identity**: `client_id` and `subscription_id`  
- **PAT**: `personal_access_token`

#### "mutually exclusive" Authentication Error
This error occurs when you mix authentication methods. Examples of invalid configurations:

```yaml
# ❌ Invalid - mixing Service Principal with PAT
tenant_id: "..."
client_id: "..."
client_secret: "..."
personal_access_token: "..."  # Cannot mix with Service Principal

# ❌ Invalid - mixing Service Principal with Managed Identity  
tenant_id: "..."
client_id: "..."
client_secret: "..."
subscription_id: "..."  # Creates ambiguity between methods
```

#### Debug Authentication Method Used

Enable debug logging to see which authentication method is selected:

```bash
alert-az-do -log-level debug -config config/alert-az-do.yml
```

Look for log entries showing credential selection logic.

#### Environment Variable Troubleshooting

Check if environment variables are set correctly:

```bash
# Check Service Principal environment variables
echo "Tenant: $AZURE_TENANT_ID"
echo "Client: $AZURE_CLIENT_ID"  
echo "Secret: ${AZURE_CLIENT_SECRET:0:4}..."  # Only show first 4 chars

# Check Managed Identity environment variables
echo "Client: $AZURE_CLIENT_ID"
echo "Subscription: $AZURE_SUBSCRIPTION_ID"

# Check PAT environment variable
echo "PAT: ${AZURE_PAT:0:4}..."  # Only show first 4 chars
```

## Contributing

We welcome contributions! Please see our [contributing guidelines](CONTRIBUTING.md) for details.

## Community

alert-az-do is an open source project and we welcome new contributors and members of the community. Here are ways to get in touch with the community:

* Issue Tracker: [GitHub Issues](https://github.com/jm-stakater/alert-az-do/issues)

## License

alert-az-do is licensed under the [Apache License 2.0](https://github.com/jm-stakater/alert-az-do/blob/master/LICENSE).

Copyright (c) 2025 Stakater AB
