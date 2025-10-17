# 🔐 Secrets Management Guide

## 📋 สถานะปัจจุบัน

ตอนนี้ใช้ **Kubernetes Secret ธรรมดา** (base64 encoded) ซึ่ง:
- ✅ ใช้งานได้ทันที ไม่ต้องติดตั้งอะไรเพิ่ม
- ⚠️ **ไม่ปลอดภัย** - base64 decode ได้ง่าย
- ⚠️ **Secret ยังอยู่ใน Git** - ใครก็อ่านได้

---

## 🚀 วิธีใช้งาน Secret ปัจจุบัน

### 1. Apply Secret ไปยัง Kubernetes

```bash
# วิธีที่ 1: Apply ด้วย Kustomize (แนะนำ)
kubectl apply -k overlays/production

# วิธีที่ 2: Apply เฉพาะ Secret
kubectl apply -f overlays/production/secret.yaml
```

### 2. ตรวจสอบ Secret ที่สร้าง

```bash
# ดู Secret ที่มี
kubectl get secrets -n default

# ดูรายละเอียด
kubectl describe secret it-request-secret -n default

# ดูค่าที่ถูก decode (ระวัง!)
kubectl get secret it-request-secret -n default -o jsonpath='{.data.PGPASSWORD}' | base64 -d
```

### 3. Pod จะใช้ Secret อัตโนมัติ

Deployment ได้กำหนด `envFrom` ไว้แล้ว:
```yaml
envFrom:
- secretRef:
    name: it-request-secret
```

Pod จะได้ environment variables:
- `PGUSER`
- `PGPASSWORD`
- `DB_USER`
- `DB_PASSWORD`

---

## 🔄 วิธีอัพเดท Secret

### ถ้าต้องการเปลี่ยน password:

```bash
# 1. Encode ค่าใหม่
echo -n "new-password" | base64
# Output: bmV3LXBhc3N3b3Jk

# 2. แก้ไข secret.yaml
# เปลี่ยน PGPASSWORD: bmV3LXBhc3N3b3Jk

# 3. Apply ใหม่
kubectl apply -f overlays/production/secret.yaml

# 4. Restart pod ให้ใช้ค่าใหม่
kubectl rollout restart deployment/it-request -n default
```

---

## 🛡️ วิธีปรับปรุงความปลอดภัย (แนะนำสำหรับ Production)

### ตัวเลือกที่ 1: Sealed Secrets (ง่ายที่สุด)

**ข้อดี:**
- Encrypt secret ด้วย public key ของ cluster
- เก็บใน Git ได้ปลอดภัย
- ไม่ต้องพึ่ง external service

**ติดตั้ง:**

```bash
# 1. ติดตั้ง Sealed Secrets Controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# 2. ติดตั้ง kubeseal CLI
brew install kubeseal  # macOS
# หรือ
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-$(uname -s | tr '[:upper:]' '[:lower:]')-amd64.tar.gz
tar xfz kubeseal-*.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

# 3. Encrypt secret ปัจจุบัน
kubectl create secret generic it-request-secret \
  --from-literal=PGUSER=itadmin \
  --from-literal=PGPASSWORD='P@ssw0rd@samco' \
  --from-literal=DB_USER=itadmin \
  --from-literal=DB_PASSWORD='P@ssw0rd@samco' \
  --dry-run=client -o yaml | \
  kubeseal --format=yaml > overlays/production/sealed-secret.yaml

# 4. ลบ secret.yaml เดิม และใช้ sealed-secret.yaml แทน
rm overlays/production/secret.yaml
# แก้ kustomization.yaml ให้ชี้ไปที่ sealed-secret.yaml
```

### ตัวเลือกที่ 2: External Secrets Operator (ยืดหยุ่นที่สุด)

**ข้อดี:**
- ดึง secrets จาก AWS Secrets Manager, Vault, GCP Secret Manager
- Sync อัตโนมัติ
- Centralized secret management

**ติดตั้ง:**

```bash
# 1. ติดตั้ง External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets-system --create-namespace

# 2. สร้าง SecretStore (ตัวอย่าง AWS)
kubectl apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: default
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-credentials
            key: access-key-id
          secretAccessKeySecretRef:
            name: aws-credentials
            key: secret-access-key
EOF

# 3. ใช้ ExternalSecret (ดู secret.example.yaml)
kubectl apply -f overlays/production/secret.example.yaml
```

### ตัวเลือกที่ 3: ใช้ .env local (สำหรับ dev/testing)

```bash
# 1. สร้าง .env ใน local (ไม่ commit ใน Git)
cat > .env <<EOF
PGUSER=itadmin
PGPASSWORD=P@ssw0rd@samco
DB_USER=itadmin
DB_PASSWORD=P@ssw0rd@samco
EOF

# 2. สร้าง Secret จาก .env
kubectl create secret generic it-request-secret \
  --from-env-file=.env \
  -n default

# 3. ไม่ต้องเก็บ secret.yaml ใน Git เลย
```

---

## 📝 Best Practices

1. **ห้าม commit plain text secrets** ใน Git
2. **ใช้ base64 อย่างน้อย** สำหรับ development
3. **Production ต้องใช้** Sealed Secrets หรือ External Secrets
4. **Rotate secrets** เป็นประจำ
5. **ใช้ RBAC** จำกัดการเข้าถึง secrets
6. **Enable audit logging** สำหรับ secret access

---

## 🔗 ลิงก์เพิ่มเติม

- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [External Secrets Operator](https://external-secrets.io/)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
