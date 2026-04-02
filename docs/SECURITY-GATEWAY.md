# Ollama Secure Gateway Setup

This guide explains how to secure Ollama backends using nginx as an API gateway with key-based authentication.

## 🎯 Why Secure Ollama?

Ollama has **no built-in authentication**. By default, anyone with network access can:
- Query models (`/api/tags`)
- Generate completions (`/api/generate`)
- Pull new models (`/api/pull`)
- Delete models (`/api/delete`)

This is a security risk in multi-tenant environments.

## 🔐 Solution Architecture

```
Tenant VM                           GPU Node
┌──────────────┐                   ┌────────────────────────────────┐
│ OpenClaw     │─── X-API-Key ───▶│  nginx Gateway (port 11435)   │
│ Container    │                   │  - Validates API key           │
│              │                   │  - Rate limits per tenant      │
│              │◀──── Response ────│  - Adds X-Tenant-ID header     │
└──────────────┘                   └──────────┬───────────────────┘
                                              │
                                              ▼ (localhost only)
                                       ┌──────────────┐
                                       │ Ollama LXC   │
                                       │ (port 11434) │
                                       └──────────────┘
```

**Security Features:**
- ✅ API key authentication (X-API-Key header)
- ✅ Direct Ollama access blocked from tenant network
- ✅ Rate limiting per tenant (100 req/min default)
- ✅ Request logging with tenant identification
- ✅ No GPU passthrough to tenants (ever)

## 🚀 Deployment Order

**IMPORTANT:** Deploy in this exact order:

### Step 1: Deploy Ollama LXCs

```bash
ansible-playbook -i inventory/ playbooks/deploy-ollama-lxc.yml
```

This creates LXC containers with Ollama running on port 11434.

### Step 2: Deploy Secure Gateway

```bash
ansible-playbook -i inventory/ playbooks/deploy-ollama-secure-gateway.yml
```

This deploys nginx on port 11435 with:
- API key validation
- Rate limiting
- Request logging
- Firewall rules

**API keys are generated and saved to:**
```
/opt/ollama-gateway/tenant-keys.json
```

### Step 3: Provision Tenants

```bash
ansible-playbook -i inventory/ playbooks/provision-tenant-secure.yml
```

Tenants automatically receive their API keys and connect to the secure gateway.

## 📋 Generated API Keys

After running the secure gateway playbook, you'll have:

```json
[
  {"tenant": "team-alpha", "key": "a1b2c3d4e5f6"},
  {"tenant": "team-beta", "key": "f6e5d4c3b2a1"},
  {"tenant": "team-gamma", "key": "1a2b3c4d5e6f"}
]
```

**Key Properties:**
- 12 characters, alphanumeric
- One key per tenant
- Keys are stored securely on tenant VMs
- Distribute securely to tenant teams

## 🛠️ Configuration

### Custom Tenant Keys

To add custom tenants, edit the playbook variables:

```yaml
# In deploy-ollama-secure-gateway.yml
tenant_keys:
  - tenant: "custom-tenant"
    key: "your-custom-key-here"  # Or use lookup for random
```

### Rate Limiting

Edit `nginx-ollama-gateway.conf.j2`:

```nginx
# Current: 100 requests per minute
limit_req_zone $tenant zone=tenant_limit:10m rate=100r/m;

# Change to: 500 requests per minute
limit_req_zone $tenant zone=tenant_limit:10m rate=500r/m;
```

### Custom Port

Edit gateway deployment:

```yaml
vars:
  gateway_port: 8080  # Instead of 11435
```

## 🧪 Testing

### Test Without Key (Should Fail)

```bash
curl http://GPU-NODE-IP:11435/api/tags
# Expected: 401 Unauthorized
```

### Test With Key (Should Succeed)

```bash
curl -H "X-API-Key: YOUR-TENANT-KEY" \
  http://GPU-NODE-IP:11435/api/tags
# Expected: JSON list of models
```

### Test Direct Ollama Access (Should Fail)

```bash
# From tenant VM
curl http://GPU-NODE-IP:11434/api/tags
# Expected: Connection refused (firewall blocked)
```

## 📊 Monitoring

### Check Gateway Logs

```bash
# On GPU node
sudo tail -f /var/log/nginx/ollama-access.log

# Example output:
10.0.20.5 - team-alpha [01/Jan/2026:10:00:00 +0000] "POST /api/generate HTTP/1.1" 200 1234 "-" "OpenClaw/1.0"
```

### Check Rate Limits

```bash
# View current rate limit status
sudo cat /var/log/nginx/error.log | grep "limiting"
```

### Check Active Keys

```bash
# On GPU node
cat /etc/nginx/api-keys/keys.json
```

## 🔒 Security Best Practices

1. **Rotate Keys Regularly**
   ```bash
   # Generate new keys and redeploy
   ansible-playbook -i inventory/ playbooks/deploy-ollama-secure-gateway.yml
   ansible-playbook -i inventory/ playbooks/provision-tenant-secure.yml
   ```

2. **Monitor Access Logs**
   - Watch for unusual patterns
   - Set up alerts for rate limit violations
   - Audit tenant usage regularly

3. **Network Segmentation**
   - Use VLANs per tenant
   - Block inter-tenant communication
   - Only allow tenant → gateway traffic

4. **HTTPS (Production)**
   ```nginx
   # Add to nginx config
   listen 11435 ssl;
   ssl_certificate /etc/nginx/ssl/ollama.crt;
   ssl_certificate_key /etc/nginx/ssl/ollama.key;
   ```

## 🚨 Troubleshooting

### Gateway Returns 403

Check that the API key is correct:
```bash
# On tenant VM
cat /opt/openclaw/.tenant-api-key
```

### Gateway Returns 502

Ollama LXC might be down:
```bash
# On GPU node
pct status <lxc-id>
pct exec <lxc-id> -- systemctl status ollama
```

### Rate Limit Hit

Wait or increase limits:
```bash
# Default: 100 req/min with burst of 20
# Wait 1 minute for limit to reset
```

### OpenClaw Can't Connect

Verify config in tenant VM:
```bash
# On tenant VM
cat /opt/openclaw/instance-1/config.yaml | grep api_key

# Test manually
curl -H "X-API-Key: $(cat /opt/openclaw/.tenant-api-key)" \
  http://10.0.10.11:11435/api/tags
```

## 🔄 Migrating Existing Tenants

If you have existing tenants without secure gateway:

1. Deploy secure gateway (Step 2 above)
2. Update tenant configurations:
   ```bash
   ansible-playbook -i inventory/ playbooks/tenant-ops.yml \
     -e "operation=update-secure tenant=team-alpha"
   ```
3. Verify connectivity
4. Block direct Ollama access via firewall

## 📚 References

- [Ollama API Documentation](https://github.com/ollama/ollama/blob/main/docs/api.md)
- [nginx rate limiting](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html)
- [nginx Lua module](https://github.com/openresty/lua-nginx-module)