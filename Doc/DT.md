# ติดตั้ง Dependency-Track Server

คำแนะนำฉบับย่อสำหรับติดตั้ง Dependency-Track Server บน cluster ของคุณ พร้อมคำสั่งพร้อมใช้งาน (Helm)

---

## ข้อกำหนดล่วงหน้า 
- ติดตั้ง `kubectl` และ `helm` (Helm 3)
- มีสิทธิ์สร้าง namespace / deployments / RBAC ใน cluster

---

## 1. ติดตั้ง Dependency-Track Server ด้วย helm chart

```bash
helm udgrade --install dependency-track dependencytrack/dependency-track --version 0.40.0 -n dependency-track --create-namespace --set ingress.enabled=true
```

สามารถดูค่า helm value ต่างๆหรืออ่านเพิ่มเติมได้[ที่นี้](https://artifacthub.io/packages/helm/dependencytrack/dependency-track)

---

## 2. สร้าง Project ใน Dependency-Track Server

- Login Dependency-Track Server 
ถ้าเข้าครั้งแรกจะ login ด้วย username/password ดังนี้  
```
Username: admin  
Paswword: admin
```
- สร้าง Project 
```markdown
คลิกไปที่ มุมซ้ายบน(☰) → Project → Create Project
กรอก Project name ให้เรียบร้อย  
เมื่อทำทุกอย่างครบให้คลิก Create
```

## 3. สร้าง API Keys เพื่อนำไปใช้ทำ CI
```markdown
คลิกไปที่ มุมซ้ายบน(☰) → Administration → Access Management → Teams → Automation  
หา API Keys คลิกที่รูปบวก  
เพิ่ม permision เพื่อให้สามารถสร้าง Project ผ่าน API Keys ได้โดยคลิกที่รูปบวก แล้วติ๊ก PROJECT_CREATION_UPLOAD และ VIEW_VULNERABILITY 
Copy API Keys เก็บไว้ให้เรียบร้อย
```
---

## 4. วิธีการนำไปใช้การใน github action
นำ API Keys ที่ได้ไปใส่ใน Secret ใน github
- ไปที่ setting → Secrets and variables → Action 
- สร้าง Environment secrets แล้วตั้งว่า Dev
- คลิก Add environment secrets ตั้งชื่อว่า DEPENDENCYTRACK_APIKEY แล้วใส่ API Keys

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