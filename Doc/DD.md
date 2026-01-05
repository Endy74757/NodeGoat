# ติดตั้ง DefectDojo Server

คำแนะนำฉบับย่อสำหรับติดตั้ง DefectDojo Server บน cluster ของคุณ พร้อมคำสั่งพร้อมใช้งาน (Helm)

---

## ข้อกำหนดล่วงหน้า 
- ติดตั้ง `kubectl` และ `helm` (Helm 3)
- มีสิทธิ์สร้าง namespace / deployments / RBAC ใน cluster

---

## 1. ติดตั้ง DefectDojo Server ด้วย helm chart

```bash
DJANGO_INGRESS_ENABLED=true
DJANGO_INGRESS_ACTIVATE_TLS=true
helm upgrade --instal \
  defectdojo \
  defectdojo \
  --set django.ingress.enabled=${DJANGO_INGRESS_ENABLED} \
  --set django.ingress.activateTLS=${DJANGO_INGRESS_ACTIVATE_TLS} \
  --set createSecret=true \
  --set createValkeySecret=true \
  --set createPostgresqlSecret=true \
  -namespace defectdojo --create-namespace
```

ถ้าต้องการให้เรียก API ผ่าน Internal service ได้ให้ config value นี้:
```yaml
django:
  # -- Autoscaling configuration for the Django deployment.
  uwsgi:
    extraEnv: 
      - name: DD_ALLOWED_HOSTS
        value: "defectdojo-django.defectdojo.svc.cluster.local"
```

สามารถดูค่า helm value ต่างๆหรืออ่านเพิ่มเติมได้[ที่นี้](https://artifacthub.io/packages/helm/defectdojo/defectdojo)

---

## 2. สร้าง Project ใน DefectDojo Server

- Login DefectDojo Server 
ถ้าเข้าครั้งแรกจะ login ด้วย username/password ดังนี้  

Username:
```
admin  
```
Password:
```bash
echo "DefectDojo admin password: $(kubectl \
      get secret defectdojo \
      --namespace=default \
      --output jsonpath='{.data.DD_ADMIN_PASSWORD}' \
      | base64 --decode)"
```
- สร้าง Product
```markdown
คลิกไปที่ มุมซ้ายบน(☰) → Add Product
กรอก Product name ให้เรียบร้อย  
เมื่อทำทุกอย่างครบให้คลิก Summit
```
- สร้าง Engagements
```markdown
คลิกไปที่ Engagements → Add New CI/CD Engagement
กรอก Engagement name ให้เรียบร้อย  
เมื่อทำทุกอย่างครบให้คลิก Done
```

## 3. สร้าง API Keys เพื่อนำไปใช้ทำ CI
```markdown
คลิกไปที่ มุมขวาบน → API v2 Key
หา API Keys    
Copy API Keys เก็บไว้ให้เรียบร้อย
```
---

## 4. วิธีการนำไปใช้การใน github action
นำ API Keys ที่ได้ไปใส่ใน Secret ใน github
- ไปที่ setting → Secrets and variables → Action 
- สร้าง Environment secrets แล้วตั้งว่า Dev
- คลิก Add environment secrets ตั้งชื่อว่า DEFECTDOJO_API_KEY แล้วใส่ API Keys

```yaml
name: CI/CD

on:
  push:
    branches: [ master ]

permissions:
  actions: read
  contents: write
  packages: write
  id-token: write

jobs:
  build:
    runs-on: arc-scale-set
    environment: Dev
    steps:
      # เช็คเอาท์โค้ด
      - name: Checkout repository
        uses: actions/checkout@v4

      # รัน Syft เพื่อสร้าง SBOM
      - name: Generate SBOM using Syft
        uses: anchore/sbom-action@v0
        with:
          path: . # scan โฟลเดอร์ปัจจุบัน
          format: cyclonedx-json # รูปแบบไฟล์ (spdx-json, cyclonedx-json, syft-json)
          output-file: nodegoat-code-sbom.json

      # อัพโหลด SBOM ขึ้น Dependency Track
      - name: Dependeency Track
        uses: DependencyTrack/gh-upload-sbom@v3
        with:
          protocol: 'http'
          serverHostname: 'dependency-track-api-server.dependency-track.svc.cluster.local'
          port: "8080"
          apiKey: ${{ secrets.DEPENDENCYTRACK_APIKEY }}
          projectName: 'Nodegoat-Code'
          projectVersion: 'master'
          bomFilename: "nodegoat-code-sbom.json"
          autoCreate: true
```
อ่านเพิ่มเติมเกัยวกับ sbom-action ได้[ที่นี้](https://github.com/anchore/sbom-action) และ gh-upload-sbom ได้[ที่นี้](https://github.com/DependencyTrack/gh-upload-sbom)



helm install defectdojo ./helm/defectdojo --set django.ingress.enabled=true --set django.ingress.activateTLS=false --set createSecret=true --set createValkeySecret=true --set createPostgresqlSecret=true