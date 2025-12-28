# ติดตั้ง SonarQube Server

คำแนะนำฉบับย่อสำหรับติดตั้ง SonarQube Server บน cluster ของคุณ พร้อมคำสั่งพร้อมใช้งาน (Helm)

---

## ข้อกำหนดล่วงหน้า 
- ติดตั้ง `kubectl` และ `helm` (Helm 3)
- มีสิทธิ์สร้าง namespace / deployments / RBAC ใน cluster

---

## 1. ติดตั้ง SonarQube Server ด้วย helm chart

```bash
helm upgrade --install -n sonarqube 
sonarqube sonarqube/sonarqube --set ingress.enabled=true --set community.enabled=true
```

สามารถดูค่า helm value ต่างๆ ได้ [ที่นี้](https://artifacthub.io/packages/helm/sonarqube/sonarqube)

---

## 2. สร้าง Project ใน SonarQube Server

- Login SonarQube Server  
ถ้าเข้าครั้งแรกจะ login ด้วย username/password ดังนี้  
```
Username: admin  
Paswword: admin
```
- สร้าง Project 
```markdown
คลิกไปที่ Project → Create Project → Loacal Project  
กรอก Project display name, Project key และ Main branch name ให้เรียบร้อย  
เมื่อทำทุกอย่างครบให้คลิก Create Project
```
---

## 3. สร้าง Tokens เพื่อนำไปใช้ทำ CI
```markdown
คลิกไปที่ Profile (มุมขวาบน) → My Account → Security → Generate Tokens  
กรอก Name, Type และ Expires in ให้เรียบร้อย  
คลิก Generate  
Copy Tokens เก็บไว้ให้เรียบร้อย
```

## 4. วิธีการนำไปใช้การใน github action
นำ API Keys ที่ได้ไปใส่ใน Secret ใน github
- ไปที่ setting → Secrets and variables → Action 
- สร้าง Environment secrets แล้วตั้งว่า Dev
- คลิก Add environment secrets ตั้งชื่อว่า SONAR_TOKEN แล้วใส่ Tokens
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
    steps:
      # เช็คเอาท์โค้ด
      - name: Checkout repository
        uses: actions/checkout@v4

      # รันการสแกน SonarQube
      - name: SonarQube Scan
        uses: SonarSource/sonarqube-scan-action@v6
        with:
          projectBaseDir: . # โฟลเดอร์โปรเจค
          args: >
            -Dsonar.projectKey=Nodegoat 
            "-Dsonar.projectName=Nodegoat"
            -Dsonar.projectVersion=1.0
            -Dsonar.sources=.
            -Dsonar.exclusions=**/node_modules/**,**/test/**,**/helm/**,**/*.yml,**/*.yaml,**/*.md,Dockerfile,,**/k8s/**
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
```
อ่านเพิ่มเติมเกัยวกับ sonarqube-scan-action ได้[ที่นี้](https://github.com/SonarSource/sonarqube-scan-action)

---