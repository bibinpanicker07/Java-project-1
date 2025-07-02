
# 🚀 Blue-Green Deployment with Jenkins CI/CD on Amazon EKS

This project demonstrates a **production-grade DevOps pipeline** that deploys a Spring Boot + MySQL application to **Amazon EKS** using **Blue-Green Deployment**, fully automated with **Jenkins CI/CD** and integrated with **DevSecOps tools**.

![Architecture Diagram](./A_flowchart_diagram_in_this_2D_digital_illustratio.png)

---

## 📌 Key Features

- ✅ End-to-end **CI/CD pipeline using Jenkins**
- ☸️ **Blue-Green deployment strategy** with zero downtime using Kubernetes on EKS
- 📦 Docker image tagging and versioning (`blue` / `green`)
- 🔐 **Secure infrastructure** with Kubernetes Secrets, RBAC, and isolated namespaces
- 💾 MySQL database backed by **dynamic AWS EBS PVC provisioning**
- 📊 Quality and security checks:
  - 🔍 Static code analysis using **SonarQube**
  - 🛡️ Image scanning with **Trivy**
  - 📥 Artifact storage in **Nexus Repository**
- ⚙️ **Parameter-driven Jenkins pipeline** to control environment and traffic routing
- 🧱 Infrastructure as Code using **Terraform**

---

## 🛠️ Technologies Used

- Java (Spring Boot)
- Docker
- Kubernetes (Amazon EKS)
- Jenkins (CI/CD)
- Trivy (Security scanning)
- SonarQube (Code analysis)
- Nexus Repository (Artifact management)
- AWS EBS (Persistent storage)
- Terraform (IaC)

---

## 📂 Project Structure

```
.
├── app-deployment-blue.yml
├── app-deployment-green.yml
├── bankapp-service.yml
├── mysql-ds.yml
├── Jenkinsfile
├── Dockerfile
├── terraform/
│   ├── eks-cluster.tf
│   └── vpc.tf
└── README.md
```

---

## ⚙️ How It Works

1. **Code push to GitHub** triggers Jenkins pipeline
2. Pipeline stages:
   - `Compile` and `Test` with Maven
   - `SonarQube` analysis and `Trivy` scan
   - `Package` and push Docker image to Nexus
   - Deploy MySQL to EKS with PVC backed by EBS
   - Deploy application to **Blue** or **Green** environment
3. **Traffic switching** handled via `kubectl patch` to update Service selector
4. Option to verify or roll back easily using Jenkins parameters

---

## 🚦 Traffic Switch Logic (Blue/Green)

- Jenkins pipeline includes:
  - `DEPLOY_ENV`: Choose `blue` or `green` environment
  - `SWITCH_TRAFFIC`: Toggle to reroute live traffic
- Traffic routing is achieved by patching the Kubernetes service selector.

---

## 📸 Sample Output

Run:
```bash
kubectl get svc bankapp-service -n webapps
kubectl get pods -l version=blue -n webapps
```

---

## 🙌 Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

---

## 📣 Connect With Me

> **🔗 LinkedIn**: [linkedin.com/in/bibinpanicker](https://www.linkedin.com/in/bibinpanicker)  
> **📁 Project Repo**: [Java-project-1](https://github.com/bibinpanicker07/Java-project-1)
