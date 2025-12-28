# ติดตั้ง Actions Runner Controller (ARC) บน Kubernetes 

คำแนะนำฉบับย่อสำหรับติดตั้ง Actions Runner Controller (ARC) บน cluster ของคุณ พร้อมคำสั่งพร้อมใช้งาน (Helm), ตัวอย่างการสร้าง secret สำหรับ **GitHub App**, ตัวอย่าง `การใช้การ runner` และคำสั่งตรวจสอบ

---

## ข้อกำหนดล่วงหน้า 
- ติดตั้ง `kubectl` และ `helm` (Helm 3)
- มีสิทธิ์สร้าง namespace / deployments / RBAC ใน cluster

- สร้าง **GitHub App** (แนะนำสำหรับ production)

---

## 1) ติดตั้ง Actions Runner Controller ด้วย Helm  

```bash
NAMESPACE="arc-systems"
helm install arc \
    --namespace "${NAMESPACE}" \
    --create-namespace \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

---

## 2) ใช้ GitHub App — แนะนำสำหรับ production

1. สร้าง GitHub App ใน GitHub (จด App ID และ INSTALLATION ID, ติดตั้งไปยัง org/repo และดาวน์โหลด private key)
2. สร้าง secret ใน Kubernetes:

```bash
kubectl create secret generic controller-manager \
  --from-literal=github_app_id='YOUR_APP_ID' \
  --from-file=github_app_private_key=/path/to/private-key.pem \
  --from-literal=github_app_installation_id='INSTALLATION_ID' \
  --namespace arc-runners --create-namespace
```

3. ติดตั้ง Runner Scale Set ด้วย Helm (chart จะอ่าน secret `controller-manager`)  

สร้างไฟล์ runner-values.yaml
```yaml
# runner-values.yaml
githubConfigUrl: "Your github url"
githubConfigSecret: controller-manager

containerMode:
  type: "dind"

template:
  spec:
    containers:
      - name: runner
        image: ghcr.io/actions/actions-runner:latest
        command: ["/home/runner/run.sh"]
        env:
          - name: DOCKER_HOST
            value: unix:///var/run/docker.sock
        volumeMounts:
          - name: work
            mountPath: /home/runner/_work
          - name: dind-sock
            mountPath: /var/run
```
ติดตั้ง Runner Scale Set ด้วย helm value
```bash
INSTALLATION_NAME="arc-runner-set" 
NAMESPACE="arc-runners"
helm install "${INSTALLATION_NAME}" \
    --namespace "${NAMESPACE}" \
    --create-namespace \
    -f runner-values.yaml \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

## 3) ตรวจสอบการทำงาน 

```bash
helm list -A
kubectl get pods -n arc-systems
```

และตรวจสอบใน GitHub → Settings ของ Repo/Org → Actions → Runners ว่ามี runner ถูกลงทะเบียน

## 4) วิธีใช้งาน runner scale sets
```yaml
name: Actions Runner Controller Demo
on:
  workflow_dispatch:

jobs:
  Explore-GitHub-Actions:
    # You need to use the INSTALLATION_NAME from the previous step
    runs-on: arc-runner-set
    steps:
    - run: echo "🎉 This job uses runner scale set runners!"
``` 