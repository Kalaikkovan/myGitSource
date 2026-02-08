# Kubernetes Certificate Extractor & Confluence Publisher

A Python script to automatically extract certificate information from Kubernetes secrets and publish a formatted report to Confluence.

## Features

- ✅ Scans all Kubernetes namespaces (or specific ones)
- ✅ Extracts certificates from TLS secrets and other secret types
- ✅ Parses certificate details: CN, SAN, Serial, Issuer, Expiry dates
- ✅ Color-coded status (Expired, Expiring Soon, Warning, Valid)
- ✅ Publishes formatted HTML table to Confluence
- ✅ Updates existing page or creates new one
- ✅ Automatic expiry calculation

## Installation

### 1. Install Dependencies

```bash
pip install -r requirements.txt
```

### 2. Configure Kubernetes Access

The script can run either:

**Inside a Kubernetes cluster** (uses service account):
- No additional setup needed

**Outside the cluster** (uses kubeconfig):
```bash
# Make sure kubectl is configured
kubectl get nodes

# The script will use your current kubeconfig context
```

### 3. Generate Confluence API Token

1. Go to: https://id.atlassian.com/manage-profile/security/api-tokens
2. Click "Create API token"
3. Give it a name (e.g., "K8s Cert Extractor")
4. Copy the token (you won't see it again!)

### 4. Configure the Script

Edit `k8s_cert_extractor.py` and update the configuration section:

```python
# Kubernetes Configuration
NAMESPACES = None  # None = all namespaces, or ['prod', 'staging']

# Confluence Configuration
CONFLUENCE_URL = "https://yourcompany.atlassian.net"
CONFLUENCE_USERNAME = "your-email@company.com"
CONFLUENCE_API_TOKEN = "your-api-token-here"
CONFLUENCE_SPACE_KEY = "SRE"  # Your Confluence space key
CONFLUENCE_PAGE_TITLE = "Kubernetes Certificate Inventory"
CONFLUENCE_PARENT_PAGE_ID = None  # Optional
```

**Finding your Space Key:**
- Open any page in your Confluence space
- Look at the URL: `https://yourcompany.atlassian.net/wiki/spaces/SRE/pages/...`
- The space key is `SRE` in this example

## Usage

### Basic Run

```bash
python k8s_cert_extractor.py
```

### Run for Specific Namespaces

Edit the script to set:
```python
NAMESPACES = ['default', 'kube-system', 'prod', 'staging']
```

### Schedule with Cron (Daily at 6 AM)

```bash
crontab -e
```

Add:
```
0 6 * * * /usr/bin/python3 /path/to/k8s_cert_extractor.py >> /var/log/cert-extractor.log 2>&1
```

### Run as Kubernetes CronJob

Create `cronjob.yaml`:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cert-extractor
  namespace: default
spec:
  schedule: "0 6 * * *"  # Daily at 6 AM
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: cert-extractor-sa
          containers:
          - name: cert-extractor
            image: python:3.11-slim
            command:
            - /bin/sh
            - -c
            - |
              pip install kubernetes cryptography atlassian-python-api python-dateutil
              python /scripts/k8s_cert_extractor.py
            env:
            - name: CONFLUENCE_URL
              value: "https://yourcompany.atlassian.net"
            - name: CONFLUENCE_USERNAME
              valueFrom:
                secretKeyRef:
                  name: confluence-creds
                  key: username
            - name: CONFLUENCE_API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: confluence-creds
                  key: api-token
            volumeMounts:
            - name: script
              mountPath: /scripts
          volumes:
          - name: script
            configMap:
              name: cert-extractor-script
          restartPolicy: OnFailure
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cert-extractor-sa
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cert-extractor-role
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cert-extractor-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cert-extractor-role
subjects:
- kind: ServiceAccount
  name: cert-extractor-sa
  namespace: default
```

Create secrets:
```bash
kubectl create secret generic confluence-creds \
  --from-literal=username='your-email@company.com' \
  --from-literal=api-token='your-api-token'

kubectl create configmap cert-extractor-script \
  --from-file=k8s_cert_extractor.py
```

## Output

The script generates a Confluence page with:

📊 **Summary Section:**
- Total certificates count
- Expired certificates (red)
- Expiring in 30 days (orange)
- Last updated timestamp

📋 **Certificate Table:**
| Namespace | Secret Name | Key | CN | SAN | Serial | Issuer | Not Before | Not After | Days Until Expiry | Status |
|-----------|-------------|-----|----|----|--------|--------|------------|-----------|-------------------|--------|

**Color Coding:**
- 🔴 Red background: Expired
- 🟠 Orange background: Expiring in ≤30 days
- 🟡 Orange text: Expiring in ≤90 days
- 🟢 Green text: Valid

## Troubleshooting

### "No certificates found"
- Check if you have TLS secrets: `kubectl get secrets --all-namespaces -o json | jq '.items[] | select(.type=="kubernetes.io/tls")'`
- Verify namespace access: `kubectl get secrets -n <namespace>`

### "Failed to connect to Confluence"
- Verify your Confluence URL (should NOT end with `/wiki`)
- Check API token is correct and not expired
- Ensure your account has permission to create/edit pages in the space

### "403 Forbidden" from Confluence
- Verify you have edit permissions in the target space
- Check if the space key is correct
- Try creating a page manually first to verify permissions

### "Unable to connect to Kubernetes"
- Verify kubectl works: `kubectl get nodes`
- Check kubeconfig: `echo $KUBECONFIG`
- Ensure you have RBAC permissions to list secrets

## Security Considerations

⚠️ **Important:**
- Store Confluence credentials securely (use environment variables or Kubernetes secrets)
- Grant minimal Kubernetes RBAC permissions (only `get` and `list` on secrets)
- Review certificate data before publishing to Confluence (ensure no sensitive data exposure)
- Consider using namespaced scanning instead of cluster-wide for better security

## Example Output Structure

```
Kubernetes Certificate Inventory
Total Certificates: 47
Expired: 2
Expiring in 30 days: 5
Last Updated: 2026-02-08 14:30:00 UTC

[Color-coded table with all certificates sorted by expiry date]
```

## Customization

### Add More Certificate Fields

In `extract_certificate_info()`, add:
```python
'key_size': cert.public_key().key_size,
'version': cert.version.value,
'fingerprint_sha256': cert.fingerprint(hashes.SHA256()).hex()
```

### Custom Alert Thresholds

Modify in `generate_html_table()`:
```python
elif cert['days_until_expiry'] <= 60:  # Change from 90 to 60
    status = '<span style="color: #ff9800;">Warning</span>'
```

### Export to CSV Instead

Add this method to `ConfluencePublisher`:
```python
def export_to_csv(self, certificates: List[Dict], filename: str):
    import csv
    with open(filename, 'w', newline='') as f:
        writer = csv.DictWriter(f, fieldnames=certificates[0].keys())
        writer.writeheader()
        writer.writerows(certificates)
```

## License

MIT

## Support

For issues related to:
- **Kubernetes access**: Check RBAC and kubeconfig
- **Certificate parsing**: Verify certificate format (PEM)
- **Confluence API**: Check Atlassian API documentation

---

**Pro Tip for SREs:** Run this daily and set up Confluence page watchers for automatic email notifications when certificates are expiring! 🚀
