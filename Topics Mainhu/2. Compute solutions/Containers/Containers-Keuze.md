# 🎯 Wanneer Welke Container Service Gebruiken?

## 📊 Quick Decision Matrix

| Vraag | ACR | ACI | Container Apps |
|-------|-----|-----|----------------|
| Image storage? | ✅ | ❌ | ❌ |
| Quick test/demo? | ❌ | ✅ | ❌ |
| Production workload? | ❌ | ⚠️ | ✅ |
| Auto-scaling nodig? | ❌ | ❌ | ✅ |
| Microservices? | ❌ | ❌ | ✅ |
| Event-driven? | ❌ | ❌ | ✅ |
| Dapr support? | ❌ | ❌ | ✅ |
| Load balancing? | ❌ | ❌ | ✅ |
| Blue/green deployment? | ❌ | ❌ | ✅ |

---

## 🗄️ Azure Container Registry (ACR)

### Gebruik wanneer...

✅ Je container images wilt **opslaan en beheren**  
✅ Je een **private registry** nodig hebt  
✅ Je images wilt **scannen op vulnerabilities**  
✅ Je **automated builds** wilt vanaf Git commits  
✅ Je images **dichtbij je workloads** wilt hebben (per region)

### Gebruik NIET wanneer...

❌ Je alleen images wilt **runnen** (gebruik ACI/Container Apps)  
❌ Je publieke images van Docker Hub voldoende zijn  
❌ Je geen CI/CD pipeline hebt

### Ideale Scenario's

**Development Team met CI/CD:**
```
Git Push → ACR Task → Build Image → Store in ACR → Deploy to ACI/Container Apps
```

**Multi-region Deployment:**
```
ACR Premium + Geo-replication
├── West Europe (primary)
├── North Europe (replica)
└── East US (replica)
```

**Security Scanning:**
```
ACR + Microsoft Defender for Cloud
→ Automatic vulnerability scanning
→ Security recommendations
```

---

## 📦 Azure Container Instances (ACI)

### Gebruik wanneer...

✅ Je **simpele containers** zonder complexiteit wilt runnen  
✅ Je **snel** een container nodig hebt (seconds)  
✅ Je **batch jobs** of **scheduled tasks** moet draaien  
✅ Je **geen load balancer** of auto-scaling nodig hebt  
✅ Je container slechts **korte tijd** draait  
✅ Je **development/testing** environment wilt

### Gebruik NIET wanneer...

❌ Je **auto-scaling** nodig hebt  
❌ Je **load balancing** nodig hebt  
❌ Je **zero-downtime deployments** wilt  
❌ Je **microservices** architectuur hebt  
❌ Je **production workloads** met high availability nodig hebt

### Ideale Scenario's

**Batch Processing:**
```yaml
# Run overnight report generation
az container create \
  --name report-generator \
  --image myacr.azurecr.io/reports:latest \
  --restart-policy Never \
  --environment-variables REPORT_DATE=2024-01-15
```

**Quick Demo:**
```bash
# Show client proof of concept in 30 seconds
az container create \
  --name demo-app \
  --image nginx \
  --dns-name-label my-demo \
  --ports 80
```

**Scheduled Tasks:**
```bash
# Azure Logic App triggers ACI for data processing
Logic App (schedule) → ACI container → Process data → Stop
```

**Development Testing:**
```bash
# Test container before production deployment
az container create \
  --name test-api \
  --image myapp:dev \
  --environment-variables ConnectionString="..." \
  --ports 80
```

---

## 🚀 Azure Container Apps

### Gebruik wanneer...

✅ Je **production applications** draait  
✅ Je **auto-scaling** nodig hebt (0 tot honderden instances)  
✅ Je **microservices** architectuur hebt  
✅ Je **load balancing** en **ingress** nodig hebt  
✅ Je **event-driven** workloads hebt  
✅ Je **Dapr** wilt gebruiken  
✅ Je **blue/green deployments** wilt  
✅ Je **revision management** nodig hebt  
✅ Je **internal networking** tussen containers nodig hebt

### Gebruik NIET wanneer...

❌ Je alleen **simpele batch jobs** hebt (gebruik ACI)  
❌ Je **geen auto-scaling** nodig hebt (ACI is goedkoper)  
❌ Je **Kubernetes features** nodig hebt die niet ondersteund worden

### Ideale Scenario's

**Microservices Architectuur:**
```
Container Apps Environment
├── Frontend App (React)
│   └── Ingress: External, HTTPS
├── API Service (Node.js)
│   └── Ingress: Internal only
├── Auth Service (C#)
│   └── Dapr: Service-to-service
└── Background Worker
    └── Scale: 0 to 10 based on queue
```

**Event-driven Application:**
```yaml
# Scale based on message queue
scale:
  minReplicas: 0
  maxReplicas: 30
  rules:
  - name: queue-rule
    type: azure-queue
    metadata:
      queueName: orders
      messageCount: "100"
```

**Production Web App:**
```yaml
# High availability web application
ingress:
  external: true
  targetPort: 80
  traffic:
  - latestRevision: false
    revisionName: myapp--v1
    weight: 90
  - latestRevision: true
    weight: 10  # Canary deployment
```

**Dapr Microservices:**
```yaml
# Service-to-service communication with Dapr
dapr:
  enabled: true
  appId: order-service
  appPort: 3000
  
# Call from other service:
# http://order-service/api/orders
```

---

## 🔄 Container Apps vs ACI - Detailed Comparison

### Feature Matrix

| Feature | ACI | Container Apps |
|---------|-----|----------------|
| **Startup time** | Seconds | Seconds |
| **Pricing model** | Per second | Per vCPU/memory |
| **Min replicas** | 1 | 0 |
| **Max replicas** | 1 | 300 |
| **Auto-scaling** | ❌ | ✅ (HTTP, CPU, Memory, Custom) |
| **Load balancer** | ❌ | ✅ Built-in |
| **HTTPS ingress** | Manual setup | ✅ Automatic |
| **Custom domains** | ⚠️ Limited | ✅ Full support |
| **SSL certificates** | Manual | ✅ Managed |
| **Blue/green** | ❌ | ✅ Traffic splitting |
| **Dapr** | ❌ | ✅ Built-in |
| **Revisions** | ❌ | ✅ Versioning |
| **Internal networking** | ⚠️ VNET | ✅ Native |
| **Secrets** | Env vars | ✅ Managed secrets |
| **Monitoring** | Basic | ✅ Advanced (Log Analytics) |

### Cost Comparison

**ACI Pricing (per second):**
```
1 vCPU + 1 GB = ~$0.0000012/second = ~$3.14/month (24/7)
2 vCPU + 4 GB = ~$0.0000044/second = ~$11.52/month (24/7)
```

**Container Apps Pricing:**
```
Base: $0.000012/vCPU-second + $0.000002/GB-second
+ $0.40/million requests

Example (1 vCPU, 2GB, 1M requests):
= ~$31.42/month + $0.40 = ~$31.82/month (24/7)

With scale to zero:
= Only pay during active time
```

**Breakeven Analysis:**
- **ACI goedkoper:** Voor workloads die constant draaien, simpele setups
- **Container Apps goedkoper:** Voor variabele workloads met scale-to-zero
- **Container Apps betere value:** Voor production apps die features nodig hebben

---

## 🎯 Decision Tree

```
Heb je images nodig?
├── Ja → ACR
└── Nee → Verder...

Wil je images runnen?
├── Ja → Verder...
└── Nee → Stop

Is het een productie workload?
├── Nee → ACI (development/testing)
└── Ja → Verder...

Heb je auto-scaling nodig?
├── Nee → ACI (goedkoper)
└── Ja → Container Apps

Heb je microservices?
├── Ja → Container Apps
└── Nee → Verder...

Heb je load balancing nodig?
├── Ja → Container Apps
└── Nee → ACI (als geen scaling)

Is het een batch job?
├── Ja → ACI
└── Nee → Container Apps
```

---

## 💡 Best Practices per Scenario

### Scenario 1: Development Team

**Setup:**
```
ACR (development)
├── Store dev images
├── ACR Tasks voor builds
└── Deploy naar ACI voor testing

ACR (production)
├── Store production images
├── Geo-replication
└── Deploy naar Container Apps
```

### Scenario 2: Microservices

**Gebruik Container Apps:**
```yaml
# Frontend
- External ingress
- Scale 1-10 based on HTTP

# API Services
- Internal ingress only
- Dapr service-to-service
- Scale based on CPU

# Background Workers
- No ingress
- Scale based on queue length
- Can scale to 0
```

### Scenario 3: Batch Processing

**Gebruik ACI:**
```bash
# Nightly batch job
az container create \
  --name batch-job \
  --restart-policy Never \
  --cpu 4 \
  --memory 16
  
# Container runs → Processes → Exits
# Only pay for actual runtime
```

### Scenario 4: Web Application

**Small website → ACI:**
```yaml
# Simple website
- No scaling needed
- Fixed traffic
- Cost-effective
- Quick setup
```

**Production website → Container Apps:**
```yaml
# Production website
- Auto-scaling voor traffic spikes
- Zero downtime deployments
- Blue/green testing
- Custom domain + SSL
- Advanced monitoring
```

---

## 🔑 Key Takeaways

### ACR
- **Role:** Image storage en management
- **Primary use:** Private registry voor al je containers
- **Pair with:** ACI of Container Apps voor deployment

### ACI
- **Role:** Simple container execution
- **Primary use:** Quick tasks, testing, batch jobs
- **Best for:** Non-production of eenvoudige workloads

### Container Apps
- **Role:** Production container platform
- **Primary use:** Productie applications met scaling en HA
- **Best for:** Microservices, production web apps, event-driven workloads

---

## 📈 Migration Path

### Van ACI naar Container Apps

**Wanneer migreren:**
- Traffic groeit en scaling nodig is
- Je load balancing nodig hebt
- Je zero-downtime deployments wilt
- Je monitoring en observability wilt

**Migration steps:**
```bash
# 1. Test in Container Apps
az containerapp create ...

# 2. Verify functionality
# Test auto-scaling, ingress, networking

# 3. Switch traffic
# Update DNS naar Container Apps

# 4. Decommission ACI
az container delete ...
```

### Van Container Apps naar AKS

**Wanneer migreren:**
- Je full Kubernetes control nodig hebt
- Je custom CRDs of Operators nodig hebt
- Je specifieke Kubernetes features nodig hebt
- Team heeft Kubernetes expertise

**Alternative:** Blijf bij Container Apps voor managed experience

---

## ✅ Decision Checklist

Gebruik deze checklist om te beslissen:

**ACR:**
- [ ] Heb je een private registry nodig?
- [ ] Wil je automated builds?
- [ ] Heb je vulnerability scanning nodig?
- [ ] Wil je geo-replication?

**ACI:**
- [ ] Is het een simpele container?
- [ ] Draait het kort (batch/job)?
- [ ] Is het development/testing?
- [ ] Heb je geen scaling nodig?
- [ ] Wil je minimale kosten?

**Container Apps:**
- [ ] Is het een productie workload?
- [ ] Heb je auto-scaling nodig?
- [ ] Is het een microservice architectuur?
- [ ] Heb je load balancing nodig?
- [ ] Wil je zero-downtime deployments?
- [ ] Heb je Dapr nodig?
- [ ] Wil je scale-to-zero?

**Score:**
- Meeste ✅ bij ACI → Gebruik ACI
- Meeste ✅ bij Container Apps → Gebruik Container Apps
- ACR is bijna altijd nodig voor beide
