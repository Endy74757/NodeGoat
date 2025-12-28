# สิ่งที่ต้องเตรียม
1. Kubernate Cluster (Kubernate, kind, minikube, etc.)  
2. Helm  
3. [Actions Runner Controller (ARC) บน Kubernetes ](Doc/ARC.md)
4. [SonarQube Server](Doc/SQ.md)
5. [Dependency-Track Server](Doc/DD.md)
---
# เตรียม Kudeconfig
เตรียม Kudeconfig เพื่อให้ Actions Runner Controller (ARC) สามารถเข้าถึง Kubernate Cluster ได้โดยสามารถสร้าง Kudeconfig ได้ด้วยคำสั่งนี้:

```bash
# Kubernate
kubectl config view --minify --raw > kubeconfig.yaml

# kind
kind get kubeconfig --internal --name {your cluster name} > kubeconfig.yaml
```

ถ้าต้องการนำไปใส่ github secret ใส่เข้ารหัส base64 ก่อน
```bash
base64 -w 0 path/to/kubeconfig.yaml
```
- จากนั้นไปที่ setting → Secrets and variables → Action 
- สร้าง Environment secrets แล้วตั้งว่า Dev
- คลิก Add environment secrets ตั้งชื่อแล้วใส่ kubeconfig
---