# Azure AD OIDC Setup Guide for Kagent Enterprise

This guide documents the complete Azure AD configuration required for Kagent Enterprise authentication.

## Architecture Overview

Kagent Enterprise uses **two separate Azure AD app registrations**:

1. **Backend API** (`kagent-backend`) - Resource server that validates tokens
2. **Frontend SPA** (`kagent-frontend`) - Single-Page Application that authenticates users

The frontend requests access tokens for the backend API scope, and the backend validates those tokens.

## Prerequisites

- Azure CLI installed and authenticated (`az login`)
- Access to Azure AD tenant with permissions to:
  - Manage app registrations
  - Create and manage Azure AD groups
  - Add users to groups
- Kubernetes cluster (kind) running with kagent namespace

---

## Part 1: Backend API App Registration

### Step 1.1: Create Backend API App

If it doesn't exist, create the backend API app registration:

```bash
TENANT_ID="<YOUR_TENANT_ID>"

az ad app create \
  --display-name "kagent-backend" \
  --sign-in-audience "AzureADMyOrg"
```

Save the `appId` (Client ID) and `id` (Object ID):
```bash
BACKEND_CLIENT_ID="<appId from output>"
BACKEND_OBJECT_ID="<id from output>"
```

**Example values**:
```bash
BACKEND_CLIENT_ID="aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
BACKEND_OBJECT_ID="<object-id>"
```

### Step 1.2: Configure Backend API Identifier URI

Set the Application ID URI to expose the API:

```bash
az rest --method PATCH \
  --uri "https://graph.microsoft.com/v1.0/applications/$BACKEND_OBJECT_ID" \
  --headers "Content-Type=application/json" \
  --body "{\"identifierUris\":[\"api://$BACKEND_CLIENT_ID\"]}"
```

### Step 1.3: Configure Backend to Accept v2.0 Tokens

**CRITICAL**: Set the backend API to accept v2.0 tokens:

```bash
az rest --method PATCH \
  --uri "https://graph.microsoft.com/v1.0/applications/$BACKEND_OBJECT_ID" \
  --headers "Content-Type=application/json" \
  --body '{"api":{"requestedAccessTokenVersion":2}}'
```

**Why this is critical**: Without this, tokens will have v1.0 issuer (`https://sts.windows.net/...`) instead of v2.0 issuer (`https://login.microsoftonline.com/.../v2.0`), causing issuer mismatch errors.

### Step 1.4: Enable Group Claims for Backend

```bash
az rest --method PATCH \
  --uri "https://graph.microsoft.com/v1.0/applications/$BACKEND_OBJECT_ID" \
  --headers "Content-Type=application/json" \
  --body '{"groupMembershipClaims":"SecurityGroup"}'
```

### Step 1.5: Create Backend API Scope

Create a `user_impersonation` scope that the frontend will request:

```bash
az rest --method PATCH \
  --uri "https://graph.microsoft.com/v1.0/applications/$BACKEND_OBJECT_ID" \
  --headers "Content-Type=application/json" \
  --body '{
    "api": {
      "requestedAccessTokenVersion": 2,
      "oauth2PermissionScopes": [
        {
          "id": "'$(uuidgen)'",
          "adminConsentDescription": "Allow the application to access kagent-backend on behalf of the signed-in user.",
          "adminConsentDisplayName": "Access kagent-backend",
          "isEnabled": true,
          "type": "User",
          "userConsentDescription": "Allow the application to access kagent-backend on your behalf.",
          "userConsentDisplayName": "Access kagent-backend",
          "value": "user_impersonation"
        }
      ]
    }
  }'
```

### Step 1.6: Create Client Secret for Backend

```bash
az ad app credential reset --id $BACKEND_CLIENT_ID \
  --append \
  --display-name "kagent-backend-secret"
```

Save the `password` value - this is your backend client secret:
```bash
BACKEND_CLIENT_SECRET="<password from output>"
```

### Step 1.7: Verify Backend Configuration

```bash
az ad app show --id $BACKEND_CLIENT_ID --query '{
  displayName:displayName,
  appId:appId,
  identifierUris:identifierUris,
  tokenVersion:api.requestedAccessTokenVersion,
  groupClaims:groupMembershipClaims,
  scopes:api.oauth2PermissionScopes[].value
}' -o json
```

Expected output:
```json
{
  "displayName": "kagent-backend",
  "appId": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
  "identifierUris": ["api://aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"],
  "tokenVersion": 2,
  "groupClaims": "SecurityGroup",
  "scopes": ["user_impersonation"]
}
```

---

## Part 2: Frontend SPA App Registration

### Step 2.1: Create Frontend App

```bash
az ad app create \
  --display-name "kagent-frontend" \
  --sign-in-audience "AzureADandPersonalMicrosoftAccount"
```

Save the `appId` and `id`:
```bash
FRONTEND_CLIENT_ID="<appId from output>"
FRONTEND_OBJECT_ID="<id from output>"
```

**Example values**:
```bash
FRONTEND_CLIENT_ID="11111111-2222-3333-4444-555555555555"
```

### Step 2.2: Configure Frontend as SPA

**CRITICAL**: Configure redirect URIs under the SPA platform (NOT Web):

```bash
az rest --method PATCH \
  --uri "https://graph.microsoft.com/v1.0/applications/$FRONTEND_OBJECT_ID" \
  --headers "Content-Type=application/json" \
  --body '{
    "spa": {
      "redirectUris": [
        "http://localhost:4000/callback",
        "http://localhost:4000"
      ]
    },
    "web": {
      "redirectUris": []
    }
  }'
```

**Why SPA platform is required**: The UI uses PKCE flow. Cross-origin token redemption requires SPA client type. Using Web platform will cause error: `AADSTS9002326: Cross-origin token redemption is permitted only for the 'Single-Page Application' client-type.`

### Step 2.3: Configure Frontend for v2.0 Tokens

```bash
az rest --method PATCH \
  --uri "https://graph.microsoft.com/v1.0/applications/$FRONTEND_OBJECT_ID" \
  --headers "Content-Type=application/json" \
  --body '{"api":{"requestedAccessTokenVersion":2}}'
```

### Step 2.4: Enable Group Claims for Frontend

```bash
az rest --method PATCH \
  --uri "https://graph.microsoft.com/v1.0/applications/$FRONTEND_OBJECT_ID" \
  --headers "Content-Type=application/json" \
  --body '{"groupMembershipClaims":"SecurityGroup"}'
```

### Step 2.5: Grant Frontend Permission to Call Backend API

Authorize the frontend to request the backend API's `user_impersonation` scope:

```bash
# Get the backend API's OAuth2 permission scope ID
SCOPE_ID=$(az ad app show --id $BACKEND_CLIENT_ID --query "api.oauth2PermissionScopes[?value=='user_impersonation'].id" -o tsv)

# Create service principal for backend if it doesn't exist
az ad sp create --id $BACKEND_CLIENT_ID 2>/dev/null || true

# Get the backend service principal's Object ID
BACKEND_SP_ID=$(az ad sp show --id $BACKEND_CLIENT_ID --query id -o tsv)

# Grant the frontend required permissions
az rest --method PATCH \
  --uri "https://graph.microsoft.com/v1.0/applications/$FRONTEND_OBJECT_ID" \
  --headers "Content-Type=application/json" \
  --body "{
    \"requiredResourceAccess\": [
      {
        \"resourceAppId\": \"$BACKEND_CLIENT_ID\",
        \"resourceAccess\": [
          {
            \"id\": \"$SCOPE_ID\",
            \"type\": \"Scope\"
          }
        ]
      }
    ]
  }"
```

### Step 2.6: Grant Admin Consent

```bash
# Create service principal for frontend if it doesn't exist
az ad sp create --id $FRONTEND_CLIENT_ID 2>/dev/null || true

# Grant admin consent (requires admin privileges)
az ad app permission admin-consent --id $FRONTEND_CLIENT_ID
```

### Step 2.7: Verify Frontend Configuration

```bash
az ad app show --id $FRONTEND_CLIENT_ID --query '{
  displayName:displayName,
  appId:appId,
  spaRedirectUris:spa.redirectUris,
  webRedirectUris:web.redirectUris,
  tokenVersion:api.requestedAccessTokenVersion,
  groupClaims:groupMembershipClaims,
  signInAudience:signInAudience
}' -o json
```

Expected output:
```json
{
  "displayName": "kagent-frontend",
  "appId": "11111111-2222-3333-4444-555555555555",
  "spaRedirectUris": [
    "http://localhost:4000/callback"
  ],
  "webRedirectUris": [],
  "tokenVersion": 2,
  "groupClaims": "SecurityGroup",
  "signInAudience": "AzureADandPersonalMicrosoftAccount"
}
```

**IMPORTANT**: `webRedirectUris` must be empty, and all URIs should be under `spaRedirectUris`.

---

## Part 3: Azure AD Groups and RBAC

### Step 3.1: Create Azure AD Groups

Create three groups for RBAC:

```bash
# Create kagent-admins group
az ad group create \
  --display-name "kagent-admins" \
  --mail-nickname "kagent-admins"

# Create kagent-readers group
az ad group create \
  --display-name "kagent-readers" \
  --mail-nickname "kagent-readers"

# Create kagent-writers group
az ad group create \
  --display-name "kagent-writers" \
  --mail-nickname "kagent-writers"
```

### Step 3.2: Get Group Object IDs

```bash
ADMINS_GROUP_ID=$(az ad group show --group "kagent-admins" --query id -o tsv)
READERS_GROUP_ID=$(az ad group show --group "kagent-readers" --query id -o tsv)
WRITERS_GROUP_ID=$(az ad group show --group "kagent-writers" --query id -o tsv)

echo "Admins Group ID: $ADMINS_GROUP_ID"
echo "Readers Group ID: $READERS_GROUP_ID"
echo "Writers Group ID: $WRITERS_GROUP_ID"
```

**Example values**:
```bash
ADMINS_GROUP_ID="ffffffff-1111-2222-3333-444444444444"
READERS_GROUP_ID="gggggggg-5555-6666-7777-888888888888"
WRITERS_GROUP_ID="hhhhhhhh-9999-aaaa-bbbb-cccccccccccc"
```

### Step 3.3: Add Users to Groups

Add yourself or other users to the appropriate groups:

```bash
# Get current user ID
USER_ID=$(az ad signed-in-user show --query id -o tsv)

# Add to all groups (for full access)
az ad group member add --group $ADMINS_GROUP_ID --member-id $USER_ID
az ad group member add --group $READERS_GROUP_ID --member-id $USER_ID
az ad group member add --group $WRITERS_GROUP_ID --member-id $USER_ID
```

Or add specific users by their User Principal Name:
```bash
az ad group member add --group $WRITERS_GROUP_ID --member-id $(az ad user show --id user@domain.com --query id -o tsv)
```

### Step 3.4: Verify Group Membership

```bash
echo "=== Checking group memberships for current user ==="
echo "Admins:" && az ad group member check --group $ADMINS_GROUP_ID --member-id $USER_ID --query value -o tsv
echo "Readers:" && az ad group member check --group $READERS_GROUP_ID --member-id $USER_ID --query value -o tsv
echo "Writers:" && az ad group member check --group $WRITERS_GROUP_ID --member-id $USER_ID --query value -o tsv
```

---

## Part 4: Helm Configuration

### Step 4.1: Create mgmt-values.yaml

Create or update your `mgmt-values.yaml` file:

```yaml
cluster: kagent-ent-kind-entra-id

# Image pull secrets for private registries
imagePullSecrets: []

global:
  imagePullPolicy: IfNotPresent

# OIDC configuration
oidc:
  issuer: https://login.microsoftonline.com/<TENANT_ID>/v2.0
  additionalScopes:
    - api://<BACKEND_CLIENT_ID>/user_impersonation

rbac:
  roleMapping:
    # CEL expression to map OIDC claims to roles
    roleMapper: "has(claims.Groups) ? claims.Groups.transformList(i, v, v in rolesMap, rolesMap[v]) : (has(claims.groups) ? claims.groups.transformList(i, v, v in rolesMap, rolesMap[v]) : [])"
    # Map Azure AD group Object IDs to internal roles
    roleMappings:
      <ADMINS_GROUP_ID>: "global.Admin"     # kagent-admins
      <READERS_GROUP_ID>: "global.Reader"   # kagent-readers
      <WRITERS_GROUP_ID>: "global.Writer"   # kagent-writers

tracing:
  verbose: true

service:
  type: LoadBalancer
  clusterIP: ""

ui:
  backend:
    repository: kagent-enterprise-public-nonprod
    oidc:
      clientId: "<BACKEND_CLIENT_ID>"
      secret: "<BACKEND_CLIENT_SECRET>"
  frontend:
    oidc:
      clientId: "<FRONTEND_CLIENT_ID>"

tunnelserver:
  repository: kagent-enterprise-public-nonprod

clickhouse:
  enabled: true
```

**Example with placeholder values**:
```yaml
oidc:
  issuer: https://login.microsoftonline.com/YOUR_TENANT_ID/v2.0
  additionalScopes:
    - api://YOUR_BACKEND_CLIENT_ID/user_impersonation

rbac:
  roleMapping:
    roleMapper: "has(claims.Groups) ? claims.Groups.transformList(i, v, v in rolesMap, rolesMap[v]) : (has(claims.groups) ? claims.groups.transformList(i, v, v in rolesMap, rolesMap[v]) : [])"
    roleMappings:
      YOUR_ADMINS_GROUP_ID: "global.Admin"    # kagent-admins
      YOUR_READERS_GROUP_ID: "global.Reader"   # kagent-readers
      YOUR_WRITERS_GROUP_ID: "global.Writer"   # kagent-writers

ui:
  backend:
    oidc:
      clientId: "YOUR_BACKEND_CLIENT_ID"
      secret: "YOUR_BACKEND_CLIENT_SECRET"
  frontend:
    oidc:
      clientId: "YOUR_FRONTEND_CLIENT_ID"
```

### Step 4.2: Deploy with Helm

```bash
cd /path/to/your/kagent-ent-kind

helm upgrade -i kagent-mgmt \
  oci://us-docker.pkg.dev/developers-369321/kagent-enterprise-public-nonprod/charts/management \
  -n kagent \
  --create-namespace \
  --version 0.1.10-2025-12-12-nikolasmatt-in-memory-jwt-introspection-96ef8d8d \
  --values "mgmt-values.yaml"
```

---

## Part 5: Verification and Access

### Step 5.1: Verify Deployment

```bash
# Check pods
kubectl get pods -n kagent

# Check UI backend logs
kubectl logs -n kagent -l app.kubernetes.io/name=management -c ui-backend --tail=50

# Check UI frontend logs
kubectl logs -n kagent -l app.kubernetes.io/name=management -c ui-frontend --tail=50
```

### Step 5.2: Access the UI

For local kind cluster with LoadBalancer:

```bash
# The service should be exposed on localhost:4000
open http://localhost:4000
```

You'll be redirected to Azure AD for authentication.

### Step 5.3: Important - Token Refresh

**After adding users to groups or making Azure AD changes**, users MUST get a new token:

1. Log out of the application (if there's a logout button)
2. Clear browser cookies for `localhost:4000` OR use incognito/private mode
3. Log in again - the new token will include updated group memberships

---

## Quick Reference

### Environment Variables

```bash
TENANT_ID="YOUR_TENANT_ID"
BACKEND_CLIENT_ID="YOUR_BACKEND_CLIENT_ID"
FRONTEND_CLIENT_ID="YOUR_FRONTEND_CLIENT_ID"
ADMINS_GROUP_ID="YOUR_ADMINS_GROUP_ID"
READERS_GROUP_ID="YOUR_READERS_GROUP_ID"
WRITERS_GROUP_ID="YOUR_WRITERS_GROUP_ID"
```

### Verify Complete Configuration

```bash
# Backend API
az ad app show --id $BACKEND_CLIENT_ID --query '{
  name:displayName,
  appId:appId,
  identifierUris:identifierUris,
  tokenVersion:api.requestedAccessTokenVersion,
  groupClaims:groupMembershipClaims
}' -o json

# Frontend SPA
az ad app show --id $FRONTEND_CLIENT_ID --query '{
  name:displayName,
  appId:appId,
  spa:spa.redirectUris,
  web:web.redirectUris,
  tokenVersion:api.requestedAccessTokenVersion,
  groupClaims:groupMembershipClaims
}' -o json

# Groups
az ad group list --filter "startsWith(displayName, 'kagent')" --query "[].{name:displayName, id:id}" -o table
```

---

## Common Errors and Solutions

See [AZURE-AD-TROUBLESHOOTING.md](./AZURE-AD-TROUBLESHOOTING.md) for detailed troubleshooting guide.

### Quick Fixes

| Error | Solution |
|-------|----------|
| `AADSTS9002326: Cross-origin token redemption...` | Move redirect URIs from Web to SPA platform (Part 2, Step 2.2) |
| `issuer does not match: Expected v2.0, got sts.windows.net` | Set backend API to accept v2.0 tokens (Part 1, Step 1.3) |
| `PERMISSION_DENIED` | Add user to appropriate Azure AD group (Part 3, Step 3.3) and refresh token |

---

## Security Best Practices

1. **Client Secret Protection**: Never commit client secrets to version control. Use environment variables or secret management systems.
2. **Redirect URI Validation**: Only add trusted redirect URIs. Remove unused URIs in production.
3. **Group-Based Access**: Use Azure AD groups for RBAC. Avoid hardcoding user emails.
4. **Token Expiration**: Monitor client secret expiration and rotate before expiry.
5. **Production Setup**: For production, use:
   - Proper DNS names (not localhost)
   - HTTPS redirect URIs
   - Single-tenant audience (`AzureADMyOrg`) for frontend
   - Restricted sign-in audience

---

## Summary Checklist

- [ ] Backend API app created and configured
  - [ ] v2.0 token version set
  - [ ] Application ID URI set
  - [ ] `user_impersonation` scope created
  - [ ] Group membership claims enabled
  - [ ] Client secret created
- [ ] Frontend SPA app created and configured
  - [ ] SPA platform with redirect URIs (not Web)
  - [ ] v2.0 token version set
  - [ ] Group membership claims enabled
  - [ ] Permission granted to call backend API
  - [ ] Admin consent granted
- [ ] Azure AD groups created
  - [ ] kagent-admins group
  - [ ] kagent-readers group
  - [ ] kagent-writers group
  - [ ] Users added to appropriate groups
- [ ] Helm values configured
  - [ ] Both client IDs specified
  - [ ] Backend client secret specified
  - [ ] Group Object IDs in RBAC mappings
  - [ ] v2.0 issuer URL
  - [ ] Backend API scope in additionalScopes
- [ ] Deployment successful
  - [ ] Pods running
  - [ ] UI accessible
  - [ ] Authentication working
  - [ ] RBAC permissions working

---

## Additional Resources

- [Azure AD v1.0 vs v2.0 tokens](https://learn.microsoft.com/en-us/azure/active-directory/develop/access-tokens)
- [SPA application registration](https://learn.microsoft.com/en-us/azure/active-directory/develop/scenario-spa-app-registration)
- [Configure group claims](https://learn.microsoft.com/en-us/azure/active-directory/develop/optional-claims#configure-group-optional-claims)
- [PKCE flow for SPAs](https://learn.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-auth-code-flow)
- [Troubleshooting Guide](./AZURE-AD-TROUBLESHOOTING.md)
