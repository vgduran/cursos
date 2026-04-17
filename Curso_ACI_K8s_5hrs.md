# CURSO: Kubernetes Infrastructure Connectivity para ACI
## Network Designs para Data Centers Modernos
**Duración:** 5 horas | **Nivel:** Expertos IT | **Formato:** Híbrido Teórico-Práctico

---

## ESTRUCTURA GENERAL

| MÓDULO | DURACIÓN | FORMATO |
|--------|----------|---------|
| **1. Fundamentos K8s y Desafíos de Red** | 45 min | Teoría + Demo |
| **2. ACI-CNI: Arquitectura y Características** | 60 min | Teoría + Lab Simulado |
| **3. Arquitecturas BGP: Full vs Hybrid** | 50 min | Análisis Comparativo + Casos |
| **4. Cilium: El Futuro con eBPF** | 40 min | Técnico + Demostración |
| **5. Implementación, Troubleshooting y Decisiones** | 45 min | Workshop Práctico |
| **DESCANSO** | 20 min | |
| **Q&A + Certificación** | 20 min | Cierre |

---

# MÓDULO 1: FUNDAMENTOS KUBERNETES Y DESAFÍOS DE RED
## (45 minutos)

### OBJETIVO
Comprender la arquitectura de red en Kubernetes y los desafíos específicos que enfrenta las empresas modernas.

### 1.1 Conceptos Clave de Kubernetes

#### POD (Unidad de Scheduling)
```
┌─────────────────────────────────┐
│  POD (Namespace compartido IP)   │
├─────────────────────────────────┤
│ ┌─────────────┐ ┌─────────────┐ │
│ │ Container 1 │ │ Container N │ │
│ │   App 1     │ │   App N     │ │
│ └─────────────┘ └─────────────┘ │
│     192.168.1.5 (IP compartida)  │
└─────────────────────────────────┘
```

**Características críticas:**
- Múltiples contenedores comparten IP
- Todos los contenedores en un pod acceden a localhost
- La más pequeña unidad deployable en K8s

#### DEPLOYMENT (Controlador de Replicación)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.14
        ports:
        - containerPort: 80
```

**Control deseado:**
- El Deployment Controller asegura que siempre haya 3 réplicas
- Auto-healing: si una cae, se crea otra automáticamente

#### SERVICE (Abstracción de Red)
```
┌──────────────────────────────────────┐
│  Service: web-service (ClusterIP)    │
│  IP: 10.37.0.124:80                  │
├──────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐   │
│  │  Pod 1      │  │  Pod 2      │   │
│  │  192.168... │  │  192.168... │   │
│  └─────────────┘  └─────────────┘   │
│         ↓              ↓             │
│      Load Balanced (Round-Robin)    │
└──────────────────────────────────────┘
```

**Funciones del Service:**
- **DNS Discovery:** web-service.default.svc.cluster.local
- **Load Balancing:** Automático entre endpoints
- **Estabilidad de IP:** IP constante aunque los pods cambien

### 1.2 El Modelo de Networking en Kubernetes

#### Requisitos Fundamentales (RFC del CNI)

1. **Pod a Pod (mismo nodo)**
   - Comunicación directa a través de docker bridge
   - NO requiere NAT

2. **Pod a Pod (diferente nodo)**
   ```
   Node A                    Node B
   ┌──────────────┐        ┌──────────────┐
   │ Pod A        │        │ Pod B        │
   │ 10.0.1.2     │ ====→  │ 10.0.2.3     │
   │ (Cluster IP) │        │ (Cluster IP) │
   └──────────────┘        └──────────────┘
   
   Req: Enrutamiento L3 sin encapsulación
   ```

3. **Pod a Nodo**
   - El pod puede alcanzar cualquier dirección IP del nodo

### 1.3 Desafíos de Networking en K8s Empresarial

#### 🔴 DESAFÍO #1: Segmentación y Aislamiento

**Problema:**
- Mismo cluster ejecuta múltiples aplicaciones
- Necesidad de aislamiento de seguridad entre teams
- Firewall externo no puede operar en nivel de POD (IPs efímeras)

**Ejemplo:**
```
Frontend (Namespace: app-frontend)   ← Debe hablar con...
    ↓                                  ↗
API Gateway (Namespace: app-api)   ← Solo con...
    ↓                                  ↙
Backend DB (Namespace: infra-db)   ← NO cualquier pod
```

#### 🔴 DESAFÍO #2: Conectividad Externa y Servicios

**Problema:**
```
┌─────────────────────────────────────────┐
│        KUBERNETES CLUSTER               │
│  ┌──────────────────────────────────┐  │
│  │  Pod Networking (10.0.0.0/16)    │  │
│  │  ¿Cómo exponer servicios?        │  │
│  └──────────────────────────────────┘  │
└─────────────────────────────────────────┘
         ↓
¿NodePort? ¿LoadBalancer? ¿Ingress?
```

**Opciones actuales y limitaciones:**
| Tipo | Puerto Fijo | External IP | Balanceo |
|------|------------|-------------|----------|
| NodePort | Sí (30000-32767) | Ninguno | Manual |
| LoadBalancer | No | SI | Hardware |
| Ingress | No | SI | Control |

#### 🔴 DESAFÍO #3: Integración con Infraestructura Existente

**Problema:** Las empresas tienen:
- Firewalls que solo entienden IPs estables
- Subnets corp predefinidas
- Políticas de routing BGP existentes
- Sistemas legacy que necesitan conectar con K8s

**Solución requerida:**
```
┌─────────────────────────────────┐
│   Enterprise Network (ACI)      │
│   - Subnets conocidas           │
│   - BGP routing                 │
│   - Firewalls L3/L4            │
└─────────────────────────────────┘
         ↑↓ (Integración?)
┌─────────────────────────────────┐
│   Kubernetes Cluster            │
│   - Subnets dinámicas           │
│   - IPs efímeras               │
│   - Namespaces aislados         │
└─────────────────────────────────┘
```

### 1.4 Comparativa: Soluciones Tradicionales vs. Modernas

#### Enfoque Tradicional (Overlay Networks)
```
Pod 1 (10.0.1.2)          Pod 2 (10.0.2.3)
     ↓                          ↓
   VXLAN Encapsulation      VXLAN Encapsulation
     ↓                          ↓
Node A (192.168.1.10) → → → Node B (192.168.1.11)
  (VXLAN Tunnel)
```

**Desventajas:**
- ⚠️ Overhead de encapsulación (16 bytes extra)
- ⚠️ Latencia aumentada (~0.4ms)
- ⚠️ No integra con infraestructura corp

#### Enfoque Moderno con ACI/BGP
```
Pod 1 (10.0.1.2)          Pod 2 (10.0.2.3)
     ↓                          ↓
  Direct Routing (BGP)    Direct Routing (BGP)
     ↓                          ↓
Node A (192.168.1.10) → → → Node B (192.168.1.11)
          ↓
   ACI Fabric (APIC)
       ↓
  Integration con corp network
```

**Ventajas:**
- ✅ Sin overhead de encapsulación
- ✅ Menor latencia (~0.25ms)
- ✅ Integración nativa con ACI/BGP
- ✅ Aislamiento seguro a nivel de política

### 📊 DEMOSTRACIÓN INTERACTIVA: Network Flow Tracing

```
ESCENARIO: Frontend Pod intenta conectar con Database Pod

┌────────────────────────────────────────────────────────┐
│ Cliente (10.0.1.5) → Service (10.37.0.124:80)         │
└────────────────────────────────────────────────────────┘
                    ↓
          ¿Quién resuelve DNS?
          → kube-dns (10.96.0.10)
                    ↓
        Responde: IP de un Pod del servicio
                    ↓
┌────────────────────────────────────────────────────────┐
│ Tráfico real: 10.0.1.5 → 10.0.2.10 (Pod real)        │
│ (La Service IP es solo para DNS/Abstracción)          │
└────────────────────────────────────────────────────────┘
                    ↓
    ¿Por dónde va? Depende del CNI:
    - Overlay: VXLAN tunnel
    - BGP: Ruta directa en APIC fabric
    - ACI-CNI: Cálculo en APIC + OVS en nodo
```

### ✅ RESUMEN MÓDULO 1

| Concepto | Clave Técnica |
|----------|---------------|
| **Pod** | Unidad mínima con IP compartida |
| **Service** | Abstracción L4 con DNS + LB |
| **Desafío Principal** | IPs efímeras + Segmentación + Integración |
| **CNI** | Plugin que implementa estas abstracciones |

---

# MÓDULO 2: ACI-CNI - ARQUITECTURA Y CARACTERÍSTICAS
## (60 minutos)

### OBJETIVO
Entender cómo ACI-CNI implementa networking seguro, escalable e integrado con la infraestructura ACI.

### 2.1 Componentes de ACI-CNI

#### Arquitectura General
```
┌──────────────────────────────────────────────────────────┐
│                      Kubernetes API                       │
│              (kubectl, Deployments, Pods, Services)       │
└────────────────────┬─────────────────────────────────────┘
                     ↓
        ┌────────────────────────────┐
        │   ACI-CNI Plugin Binary     │
        │   (/opt/cni/bin/)           │
        └────────────────────────────┘
                     ↓
   ┌─────────────────────────────────────────┐
   │  ACI Containers Host Agent (DaemonSet) │
   │  - Ejecuta en cada nodo                 │
   │  - Maneja interfaces de red             │
   │  - Interactúa con OpFlex                │
   └─────────────────────────────────────────┘
                     ↓
   ┌─────────────────────────────────────────┐
   │     Open vSwitch (OVS)                  │
   │  - Data plane del cluster               │
   │  - FlowRules programadas por OpFlex     │
   │  - Maneja VXLAN/Routing                 │
   └─────────────────────────────────────────┘
                     ↓
   ┌─────────────────────────────────────────┐
   │    ACI Fabric (APIC Controlled)         │
   │  - EPGs (Endpoint Groups)               │
   │  - Contratos de seguridad               │
   │  - Integración completa                 │
   └─────────────────────────────────────────┘
```

#### Protocolo OpFlex (Open Policy Exchange)
```
OpFlex: Protocolo entre APIC y agentes locales

APIC (Controller)
    ↓↑ OpFlex (JSON-RPC)
Host Agent ← → OVS

Información distribuida:
- Endpoints (Pods, VMs, Servidores)
- Políticas de seguridad
- Rutas y topología
- Estados de conectividad
```

### 2.2 Ciclo de Vida de un Pod con ACI-CNI

#### Paso 1: Creación del Pod
```yaml
kubectl apply -f pod.yaml
  ↓
API Server recibe la solicitud
  ↓
Scheduler asigna a un nodo
  ↓
kubelet en ese nodo ejecuta:
  - Crea container runtime
  - Llama al plugin CNI
```

#### Paso 2: Invocación del Plugin CNI
```bash
# El kubelet ejecuta:
/opt/cni/bin/aci-containers-cni \
  add \
  {
    "cniVersion": "0.3.1",
    "name": "aci-containers",
    "type": "aci-containers",
    "args": {
      "K8S_POD_NAME": "web-app-xyz",
      "K8S_POD_NAMESPACE": "default",
      "K8S_POD_INFRA_CONTAINER_ID": "abc123..."
    }
  }
```

#### Paso 3: Host Agent Procesa
```
Host Agent recibe comando CNI
  ↓
1. Consulta APIC vía OpFlex
   - ¿A qué EPG asignar este pod?
   - ¿Qué políticas aplican?
   ↓
2. Asigna IP de POD subnet
   - IP: 10.0.1.42
   - Gateway: 10.0.1.1 (en OVS)
   ↓
3. Crea interfaz en OVS
   - veth pair (virtual ethernet)
   - Conecta al bridge OVS
   ↓
4. Programa FlowRules en OVS
   - Destino: cualquier IP → reachable
   - Origen: Pod → permitido desde EPG
```

#### Paso 4: Configuración de Red
```
┌──────────────────────────────────────┐
│         Container Namespace          │
│                                      │
│  eth0 (10.0.1.42)  ←─ veth:pod     │
│         ↓                            │
│      10.0.1.1 (gateway)              │
└──────────────────────────────────────┘
         ↓
┌──────────────────────────────────────┐
│         Host Namespace               │
│                                      │
│  OVS Bridge (br-int)                 │
│  ├─ veth:host (paired)              │
│  ├─ vxlan0 (VXLAN tunnel)           │
│  └─ Flow Rules (seguridad)          │
└──────────────────────────────────────┘
         ↓
┌──────────────────────────────────────┐
│   Kubernetes/ACI Network             │
└──────────────────────────────────────┘
```

### 2.3 Características Avanzadas de ACI-CNI

#### A) ASIGNACIÓN DE POD A EPG

**Concepto:** EPG = Grupo de Endpoints (puede contener Pods, VMs, etc.)

```yaml
# Pod con anotación ACI
apiVersion: v1
kind: Pod
metadata:
  name: web-server
  namespace: production
  annotations:
    opflex.cisco.com/eepg-name: "EPG-WebServers"
spec:
  containers:
  - name: nginx
    image: nginx:latest
```

**En ACI APIC:**
```
Tenant: mycompany
  → Application Profile: webapp-ap
    → EPG: EPG-WebServers
      - Contiene este pod
      - Hereda políticas del EPG
      - Aislado de otros EPGs
```

#### B) SNAT (Source NAT) - Solución al Problema de IPs Efímeras

**Problema Original:**
```
Pod quiere conectar a BD corporativa (192.168.100.x)
Pero su IP es 10.0.1.42 (efímera, puede cambiar)
→ BD no puede crear regla firewall
```

**Solución ACI-CNI SNAT:**
```
Pod (10.0.1.42) quiere salir del cluster
  ↓
OVS intercepta tráfico egreso
  ↓
Source IP reescrita a: 172.30.0.10 (IP predecible)
  ↓
BD corporativa ve: 172.30.0.10 (estable)
  ↓
Pueden crear reglas firewall confiables
```

**Niveles de SNAT:**
```
1. Cluster Level (todos los pods)
   Egreso: Pod cualquiera → IP corporativa (172.30.0.x)

2. Namespace Level (pods específicos)
   Namespace: data-processing
   Egreso: → IP 172.30.0.50

3. Deployment Level (pods aún más específicos)
   Deployment: batch-jobs
   Egreso: → IP 172.30.0.51

4. Service LoadBalancer (servicios externos)
   Service Type: LoadBalancer
   Egreso: → IP 172.25.0.10 (Ext Service IP)
```

#### C) LOADBALANCING CON PBR (Policy-Based Routing)

**Escenario:** Servicio web con 3 replicas en diferentes nodos

```
Cliente Externo
      ↓
  172.25.0.5 (External Service IP - asignada por ACI)
      ↓
   ┌─────────────────────────────┐
   │  ACI Service Graph + PBR    │
   │  Destino: 172.25.0.5:80     │
   │  Acción: Balancear entre:   │
   │  - Node1:30001 (Pod1)       │
   │  - Node2:30001 (Pod2)       │
   │  - Node3:30001 (Pod3)       │
   └─────────────────────────────┘
      ↓
ECMP (Equal Cost Multipath)
Distribuye en 64 caminos simultáneamente
```

**Ventaja vs. K8s Nativo:**
```
K8s Nativo (iptables):
- Centralizado en Node
- Cuello de botella
- CPU intensivo

ACI-CNI (PBR en fabric):
- Distribuido en switches
- Hardware acceleration
- Menor latencia (<0.25ms)
```

### 2.4 Proceso de Provisioning con acc-provision

#### Herramienta: acc-provision (Python-based)

```bash
# Instalación
pip install acc-provision

# Generar archivo de configuración
acc-provision --sample -f kubernetes-1.31 > aci-config.yaml
```

#### Archivo de Configuración (Simplificado)
```yaml
apiVersion: aci.cci.cisco.com/v1
kind: AciContainersConfig
metadata:
  name: aci-containers-config
  namespace: kube-system
spec:
  system_id: k8s-cluster-prod          # Identifier único
  tenant_name: k8s-tenant              # Tenant en ACI
  aci_cluster_name: k8s-fabric         # Nombre interno
  
  # VLAN Configuration
  node_subnet: 172.31.0.0/24           # Network de nodos
  pod_subnet: 10.0.0.0/16              # Subnet para PODs
  service_subnet: 10.96.0.0/12         # Subnet para Services
  
  # ACI Fabric Details
  apic_hosts:
  - 192.168.1.30                       # APIC primaria
  - 192.168.1.31                       # APIC secundaria
  apic_username: admin
  apic_password: password123
  
  # VLAN IDs
  vlan_config:
    node_vlan: 3900                    # VLAN para nodos
    service_vlan: 3901                 # VLAN para services
    infra_vlan: 4094                   # VLAN infra ACI
  
  # Feature Flags
  enable_ovs_hardware_offload: true    # Hardware accel.
  enable_pbr: true                     # Policy-Based Routing
  enable_snat: true                    # Source NAT
```

#### Ejecución del Provisioning
```bash
acc-provision --sample \
  -f kubernetes-1.31 \
  -c aci-config.yaml \
  -o aci-containers-cni.yaml
```

**Lo que hace:**
1. ✅ Crea Tenant en APIC
2. ✅ Configura EPGs (kube-system, default pods, nodes)
3. ✅ Establece contratos de seguridad
4. ✅ Genera YAML para desplegar en K8s
5. ✅ Configura VMM domain (si aplica)

#### Instalación en Cluster
```bash
# 1. Crear namespace
kubectl create namespace aci-containers

# 2. Aplicar configuración
kubectl apply -f aci-containers-cni.yaml

# 3. Verificar deployment
kubectl get pods -n aci-containers
# Debe mostrar:
# - aci-containers-controller (1 replica)
# - aci-containers-host (1 por nodo)
# - aci-containers-openvswitch (1 por nodo)
```

### 2.5 Seguridad y Políticas en ACI-CNI

#### Modelo de Políticas de Dos Capas

```
┌─────────────────────────────────────────────────┐
│  CAPA 1: Kubernetes Network Policy (eBPF/OVS)  │
│  - Pod a Pod (ingress/egress)                  │
│  - Granularidad: Labels de pods                │
│  - Implementado en: Linux kernel + OVS         │
└─────────────────────────────────────────────────┘
              ↓ AND ↓
┌─────────────────────────────────────────────────┐
│  CAPA 2: ACI Contracts (Fabric Level)          │
│  - EPG a EPG (grupos completos)                │
│  - Granularidad: Subnets, servicios            │
│  - Implementado en: Switches ACI               │
└─────────────────────────────────────────────────┘

Resultado: Seguridad en capas redundantes
```

#### Ejemplo: Pod que viola política

```
Pod Frontend (app=frontend) → quiere hablar → Backend Database

K8s NetworkPolicy:
- Permite frontend → api (OK)
- NO permite frontend → database (REJECT)

ACI Contract:
- EPG-Frontend puede enviar a EPG-API (permitido)
- EPG-Frontend NO puede enviar a EPG-Database (denegado)

Resultado del tráfico:
1. Kernel rechaza (K8s policy)
2. Si esquiva kernel, switch rechaza (ACI contract)
Defensa en profundidad ✅
```

### 📊 DEMOSTRACIÓN: Anotaciones y Mapeo de EPG

```yaml
# En K8s: Definir Pod con anotación
apiVersion: v1
kind: Pod
metadata:
  name: payment-processor
  namespace: production
  labels:
    app: payment
    tier: backend
  annotations:
    opflex.cisco.com/pod-network: "secure-backend"
spec:
  containers:
  - name: payment-app
    image: payment-app:v2.1
    ports:
    - containerPort: 5432

# En APIC: Resultado
Tenant: mycompany
  Application Profile: secure-backend
    EPG: secure-backend
      - Contract IN: allows-from-frontend
      - Contract OUT: allows-to-database
      - Este pod es miembro automático
```

### ✅ RESUMEN MÓDULO 2

| Componente | Función |
|-----------|---------|
| **Host Agent** | Gestiona interfaz local y OpFlex |
| **OVS** | Data plane, Flow rules |
| **APIC** | Control plane, Políticas |
| **SNAT** | IPs corporativas estables |
| **PBR** | Balanceo de carga en fabric |
| **acc-provision** | Herramienta de provisioning |

---

# MÓDULO 3: ARQUITECTURAS BGP - FULL vs HYBRID
## (50 minutos)

### OBJETIVO
Comparar dos modelos arquitectónicos de integración K8s-ACI y ayudar a elegir el correcto.

### 3.1 Arquitectura FULL BGP

#### Concepto General
```
Cada Nodo K8s (AS 65003) peering BGP ↔ Leaves ACI (AS 65002)
     ↓
Todos los prefijos se anuncian vía BGP
     ↓
Pod subnet, Node subnet, Service subnet → Routables en fabric
```

#### Topología Full BGP
```
                    ┌─────────────────────────┐
                    │  ACI Fabric (AS 65002)  │
                    │  Border Leafs           │
                    │  (L101, L201)           │
                    └────┬────────────────┬───┘
                         │ eBGP Peering  │
                    ┌────┴────────────┬──┴────┐
                    │                 │       │
              ┌─────┴─────┐     ┌────┴──┐    │
              │  Node 1    │     │Node 2 │    │
              │ AS 65003   │     │AS 65003   │
              │ 10.0.1.0/24│     │10.0.2.0/24│
              │ BGP Peer   │     │BGP Peer   │
              └────────────┘     └───────────┘

Anuncios BGP:
Node 1 → Leaf: 10.0.1.0/24 (Pod subnet en Node 1)
               172.30.0.0/24 (Node subnet)
               10.96.0.0/12 (Service subnet)

Node 2 → Leaf: 10.0.2.0/24 (Pod subnet en Node 2)
               172.30.0.0/24 (Node subnet)
               10.96.0.0/12 (Service subnet)

Fabric fabric puede alcanzar todos los prefijos directamente
```

#### Sesión BGP (Configuración)
```bash
# En Node 1 (running cilium con BGP)
# Configurar IsoVolent BGP Cluster Config (CRD)

apiVersion: cilium.io/v2alpha1
kind: CiliumBGPClusterConfig
metadata:
  name: bgp-cluster-config
spec:
  nodeSelector:
    bgp-cluster: "true"
  bgpInstances:
  - name: "default"
    localASN: 65003
    peers:
    - peerASN: 65002
      peerAddress: 192.168.1.101  # Leaf 101
      families:
      - afi: ipv4
        safi: unicast
```

#### Ventajas FULL BGP
✅ **Escalabilidad:** Múltiples sesiones BGP distribuidas
✅ **Sencillez:** Un único protocolo (BGP) para todo
✅ **Rendimiento:** Sin overhead de encapsulación
✅ **Visibilidad:** Fabric ve todos los prefijos

#### Desventajas FULL BGP
❌ **Complejidad:** Más sesiones BGP = más estados
❌ **Escala BGP:** Si tienes 100 nodos = 100 sesiones BGP
❌ **Consistencia:** Debe sincronizar estado en todos los nodos

### 3.2 Arquitectura HYBRID BGP

#### Concepto General
```
SOLO algunos nodos (Ingress Nodes) hacen BGP peering
    ↓
Resto de nodos usa local networking
    ↓
Services se anuncian via BGP desde Ingress Nodes
```

#### Topología Hybrid BGP
```
                    ┌─────────────────────────┐
                    │  ACI Fabric (AS 65002)  │
                    │  Border Leafs (L101)    │
                    └────┬────────────────────┘
                         │ eBGP Peering (Solo Ingress)
                    ┌────┴───────────────┐
                    │                    │
              ┌─────┴─────┐       ┌─────┴──────┐
              │  Node 1    │       │  Node 2    │
              │(Ingress)   │       │(Worker)    │
              │ AS 65003   │       │AS 65003    │
              │ BGP Peer ✓ │       │BGP Peer ✗  │
              │10.0.1.0/24 │       │10.0.2.0/24 │
              └────────────┘       └────────────┘
                    ↓                   ↓
                    └─── ESG (Endpoint Security Group) ───┘
                          Clasificación de tráfico

Anuncios BGP (Solo Node 1):
- Service subnet: 10.96.0.0/12
- Service routes (host routes /32)
- Egress IPs de namespaces

Nodo 2 no hace BGP:
- Sus pods usan networking local
- No anunciables en fabric
```

#### Configuración Hybrid BGP (Cilium)
```bash
# IsoValent BGP Cluster Config para Hybrid

apiVersion: cilium.io/v2alpha1
kind: CiliumBGPClusterConfig
metadata:
  name: hybrid-bgp-config
spec:
  nodeSelector:
    bgp-node: "ingress"  # Solo nodos con este label
  bgpInstances:
  - name: "default"
    localASN: 65003
    peers:
    - peerASN: 65002
      peerAddress: 192.168.1.101
      families:
      - afi: ipv4
        safi: unicast

---
# Endpoint Security Groups (ESG) para Egress
apiVersion: cilium.io/v2alpha1
kind: CiliumEndpointSliceStateBackend
metadata:
  name: egress-esg
spec:
  egressGatewayPolicy:
  - namespace: production
    egressIP: 192.168.100.50  # IP estable para salidas
```

#### Ventajas HYBRID BGP
✅ **Escalabilidad simplificada:** Menos sesiones BGP
✅ **Menor complejidad:** Menos estados a sincronizar
✅ **Control de Ingress:** Solo routers designados
✅ **ESG Integration:** Mejor integración con ACI ESGs

#### Desventajas HYBRID BGP
❌ **Punto único de fallo (Ingress):** Si Ingress Node cae
❌ **Restricción de anuncios:** Solo Services (no Pods)
❌ **Latencia:** Tráfico puede ir a Ingress Node primero

### 3.3 Cuadro Comparativo Detallado

| Aspecto | FULL BGP | HYBRID BGP |
|--------|----------|-----------|
| **Sesiones BGP** | 1 por nodo | Configurables |
| **Anuncios** | Pod, Node, Service | Solo Service |
| **Escalabilidad K8s** | Hasta 100+ nodos | Mejor >100 nodos |
| **Latencia** | Más baja | Ligeramente mayor |
| **Complejidad Config** | Media | Baja |
| **Hardware requerido** | Estándar | Estándar |
| **Integración ACI** | Completa | Recomendada |
| **ESG Support** | Parcial | Completo |

### 3.4 Caso de Uso: Decisión Arquitectónica

#### 📋 ESCENARIO A: Startup Tech (100 nodos)
```
Requisitos:
- Crecimiento rápido
- Latencia crítica
- Presupuesto limitado

RECOMENDACIÓN: FULL BGP
Razón: Escalabilidad desde inicio, latencia mínima
```

#### 📋 ESCENARIO B: Empresa Corporativa (300 nodos)
```
Requisitos:
- Integración con ACI existente
- Security groups predefinidas (ESG)
- Múltiples clusters

RECOMENDACIÓN: HYBRID BGP
Razón: Menos complejidad, mejor para ESGs, mantenimiento
```

#### 📋 ESCENARIO C: Datos Sensibles (50 nodos)
```
Requisitos:
- Segmentación máxima
- Compliance regulatorio
- Tráfico predecible

RECOMENDACIÓN: HYBRID BGP
Razón: Control granular, integración ESG, auditoría
```

### 📊 DEMOSTRACIÓN: BGP Peering Status

```bash
# En Node Ingress (Cilium con BGP)
kubectl exec -it cilium-xyz -n kube-system -- \
  cilium bgp peers

Output:
Peer             ASN    IPv4    IPv6    State
192.168.1.101    65002  Up      Down    Established
192.168.1.102    65002  Up      Down    Established

# Ver rutas anunciadas
kubectl exec -it cilium-xyz -n kube-system -- \
  cilium bgp routes ipv4 advertised

Output:
Route                Via         Attributes
10.96.0.0/12         local       origin igp, localpref 100
172.30.0.0/24        local       origin igp, localpref 100
```

### ✅ RESUMEN MÓDULO 3

| Arquitectura | Mejor Para | Escalabilidad |
|-------------|-----------|---------------|
| **FULL BGP** | Baja latencia, simple | <100 nodos |
| **HYBRID BGP** | Escalabilidad, ESG | >100 nodos |

---

# MÓDULO 4: CILIUM - EL FUTURO CON eBPF
## (40 minutos)

### OBJETIVO
Entender la revolución de Cilium con eBPF y cómo complementa (o reemplaza) ACI-CNI.

### 4.1 ¿Qué es eBPF?

#### Definición Técnica
```
eBPF (Extended Berkeley Packet Filter)
= Máquina virtual en el kernel Linux
= Ejecutar código sin modificar kernel
= Permite networking, seguridad, observabilidad sin syscalls
```

#### Comparación: Tradicional vs eBPF

**Enfoque Tradicional (iptables):**
```
Paquete entra en Node
    ↓
Kernel → Userspace (context switch)
    ↓
iptables (procesa en CPU)
    ↓
Kernel → Userspace (context switch)
    ↓
Paquete sale del Node

Costo: Context switches frecuentes = CPU intensivo
Latencia: ~0.4ms (Flannel/Calico tradicional)
```

**Enfoque eBPF (Cilium):**
```
Paquete entra en Node
    ↓
eBPF program (ejecuta directamente en kernel)
    ↓
Sin context switches
    ↓
Paquete sale del Node

Costo: Mínimo (kernel-space nativo)
Latencia: ~0.2ms (eBPF optimal)
```

### 4.2 Cilium Architecture

#### Componentes Principales
```
┌──────────────────────────────────────────────┐
│          Cilium Components                   │
├──────────────────────────────────────────────┤
│                                              │
│  ┌────────────────────────────────────────┐ │
│  │  Cilium Agent (UserSpace Daemon)       │ │
│  │  - Carga eBPF programs al kernel       │ │
│  │  - Sinc con APIC/BGP peers            │ │
│  │  - Maneja actualización de políticas   │ │
│  └────────────────────────────────────────┘ │
│           ↓↑                                 │
│  ┌────────────────────────────────────────┐ │
│  │  eBPF Programs (Kernel)                │ │
│  │  - tc_ingress: entrada a interfaz      │ │
│  │  - xdp_program: muy temprano en NIC    │ │
│  │  - socket_connect: syscall hooks       │ │
│  │  - kprobe: tracing de eventos          │ │
│  └────────────────────────────────────────┘ │
│           ↓↑                                 │
│  ┌────────────────────────────────────────┐ │
│  │  BPF Maps (Kernel In-Memory Storage)   │ │
│  │  - Endpoint info (Pods, Services)      │ │
│  │  - Policy rules (Allow/Deny)           │ │
│  │  - Routing tables                      │ │
│  └────────────────────────────────────────┘ │
│                                              │
└──────────────────────────────────────────────┘
```

#### Flujo de Decisión eBPF para Tráfico de Pod

```
Paquete 192.168.1.5:40123 → 10.0.1.2:80

eBPF program carga (ns-map):
├─ ¿Quién soy? (origen)
│  → EPG: Frontend
├─ ¿A dónde voy? (destino)
│  → EPG: Backend
├─ ¿Política permite?
│  → Lookup en policy_map[Frontend→Backend]
│  → Resultado: ALLOW
├─ ¿Es necesario SNAT?
│  → Lookup en snat_rules
│  → Reescribe 192.168.1.5 → 172.30.0.10
└─ Forward paquete
   → OVS/kernel routing
   → Sale del nodo

Total latency: <0.2ms
```

### 4.3 Cilium + ACI Integration

#### Modelo Híbrido Moderno
```
┌──────────────────────────────────┐
│   CILIUM (eBPF + K8s native)     │
│  - Networking de pods            │
│  - Network policies             │
│  - Observabilidad con Hubble    │
└──────────────────┬───────────────┘
                   ↓
        ┌──────────────────────┐
        │   BGP Integration    │
        │   (Cilium BGP Agent) │
        └──────────┬───────────┘
                   ↓
     ┌──────────────────────────┐
     │   ACI FABRIC             │
     │  - Service routing       │
     │  - ESG segmentation      │
     │  - Contratos             │
     └──────────────────────────┘
```

#### Ventaja: Separación de Responsabilidades
```
Cilium:
- ¿Pod A puede hablar con Pod B?
- Validación rápida en eBPF kernel

ACI:
- ¿Service X puede salir del cluster?
- ¿ESG y firewall aprueban?
- Validación en fabric
```

### 4.4 Hubble: Observabilidad

#### ¿Qué es Hubble?
```
Observador de flujos de red en tiempo real
Usando eBPF tracing
```

#### Capabilities de Hubble
```bash
# Ver todos los flows en real-time
hubble observe --protocol-filter http

# Output:
Pod-Frontend:8080    → Pod-Backend:5432   ALLOWED
Pod-Frontend:8080    → Pod-Backend:5433   DENIED (policy)
Service-LB:80       → Pod-Frontend:8080   ALLOWED
External:80        ← Service-LB:80        ALLOWED

# Métricas
hubble metrics --export-interval=10s

# Tracing completo de packet
hubble trace --follow \
  -f cilium_monitors \
  --type all
```

### 4.5 BGP Routing en Cilium

#### Cilium BGP Control Plane
```
Cilium Agent detecta:
├─ Pod creado con IP 10.0.1.42
├─ Pod tiene label: bgp-advertise=true
└─ Anuncia 10.0.1.42/32 a BGP peers

BGP Peers (configurados en CRD):
├─ Leaf ACI #1 (AS 65002)
├─ Leaf ACI #2 (AS 65002)
└─ External Router (AS 65000)
```

#### Configuración BGP en Cilium
```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeeringPolicy
metadata:
  name: aci-peering
spec:
  virtualRouters:
  - localASN: 65003
    exportPodCIDR: true
    neighbors:
    - peerASN: 65002
      peerAddress: 192.168.1.101
      gracefulRestart:
        restartTimeSeconds: 15
        helperCheckIntervalSeconds: 3
```

### 4.6 Performance Comparativa

#### Benchmark: Throughput y Latencia
```
Test: 1000 pods, iperf3 connections

┌─────────────────────┬──────────┬─────────┐
│ CNI                 │ Throughput│ Latency │
├─────────────────────┼──────────┼─────────┤
│ Flannel (VXLAN)     │ 6.5 Gbps │ 0.40 ms │
│ Calico (BGP)        │ 8.5 Gbps │ 0.25 ms │
│ Cilium (eBPF native)│ 9.2 Gbps │ 0.20 ms │
│ ACI-CNI (OVS)       │ 8.8 Gbps │ 0.24 ms │
└─────────────────────┴──────────┴─────────┘

Resultado: Cilium eBPF es superior en latencia
```

### 4.7 Casos de Uso de Cilium

#### ✅ Mejor para Cilium
- Clusters puros Kubernetes (sin ACI existente)
- Máxima latencia crítica (<0.2ms)
- Observabilidad en tiempo real (Hubble)
- Escalas muy grandes (>1000 nodos)

#### ✅ Mejor para ACI-CNI
- Integración con infra ACI existente
- Migración desde políticas ACI
- Multitenancy con EPGs
- Segmentación a nivel de fabric

#### 🤝 HÍBRIDO: Cilium + ACI
- Cilium para pod-to-pod (rápido)
- ACI para service-to-external (seguro)
- Hubble para observabilidad
- BGP de Cilium hacia ACI fabric

### 📊 DEMO: Cilium Network Policy

```yaml
# Network Policy en K8s que entiende Cilium

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: production
      podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 5432

# Cilium lo compila a:
# 1. eBPF programs para permitir tráfico
# 2. BPF maps con endpoint info
# 3. Policy enforcement en tc_ingress
# 4. Hubble logging de violaciones
```

### ✅ RESUMEN MÓDULO 4

| Característica | Cilium | ACI-CNI |
|---|---|---|
| **eBPF Native** | ✅ | ❌ |
| **Latencia** | 0.2ms | 0.24ms |
| **Observabilidad** | ✅✅ (Hubble) | ✅ |
| **BGP** | ✅ (Cilium agent) | ✅ (OVS) |
| **ACI Integration** | ✅ (vía BGP) | ✅✅ (nativa) |

---

# MÓDULO 5: IMPLEMENTACIÓN, TROUBLESHOOTING Y DECISIONES
## (45 minutos) 

### OBJETIVO
Guía práctica para implementar, diagnosticar problemas y elegir la solución correcta.

### 5.1 Checklist de Implementación

#### FASE 1: Planning (2-3 semanas)

```yaml
[ ] Audit de infraestructura actual
    [ ] ¿Tenemos ACI fabric ya?
    [ ] ¿Cuántos nodos K8s planeados?
    [ ] ¿Latencia requerida?
    [ ] ¿Presupuesto para training?

[ ] Seleccionar CNI
    [ ] Completar matriz decisoria
    [ ] PoC (Proof of Concept)
    [ ] Benchmarks internos

[ ] Arquitectura de red
    [ ] Subnets para Pod/Node/Service
    [ ] VLANs
    [ ] Ruta para tráfico egreso
    [ ] Diseño de seguridad

[ ] Recursos
    [ ] Asignar equipo técnico
    [ ] Training ACI/Cilium/K8s
    [ ] Documentación
```

#### FASE 2: Preparación de Fabric (1-2 semanas)

```bash
# Para ACI-CNI:

[ ] APIC Checklist
  [ ] Versión APIC compatible (mínimo 4.2.x)
  [ ] Licenses for VMM domain
  [ ] Credentials para acc-provision
  
[ ] Fabric Switches
  [ ] Verificar modelo (9300-EX/FX required para PBR)
  [ ] Actualizar firmware si es necesario
  [ ] Verificar conectividad Leaf-Spine

[ ] Configuración VLANs
  [ ] Pool de VLANs disponible
  [ ] Asignar VLANs para K8s
  [ ] Trunking configurado en puertos Leaf

# Para Cilium + BGP:

[ ] Verificar BGP
  [ ] Leaf switches pueden hacer BGP eBGP
  [ ] AS numbers asignados
  [ ] BGP timers (1s/3s recomendado)
  
[ ] L3Out
  [ ] Crear L3Out si conecta a external routers
  [ ] BGP enabled en L3Out
  [ ] Verificar convergencia
```

#### FASE 3: Instalación K8s (1 semana)

```bash
# 1. Vanilla K8s sin CNI
kubeadm init --pod-network-cidr=10.0.0.0/16 \
  --apiserver-advertise-address=172.31.0.10

# 2. Opción A: ACI-CNI
pip install acc-provision
acc-provision --sample -f kubernetes-1.31 \
  -c aci-config.yaml \
  -o aci-cni.yaml
kubectl apply -f aci-cni.yaml

# 3. Opción B: Cilium
helm repo add cilium https://helm.cilium.io
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set k8sServiceHost=172.31.0.10 \
  --set k8sServicePort=6443
```

#### FASE 4: Validación (3-5 días)

```bash
[ ] Connectivity Tests
  [ ] Pod a Pod mismo nodo
  [ ] Pod a Pod diferente nodo
  [ ] Pod a Service
  [ ] Service a External

[ ] Security Tests
  [ ] Network Policy enforcement
  [ ] EPG isolation (ACI-CNI)
  [ ] SNAT validation
  [ ] Firewall rules

[ ] Performance Tests
  [ ] iperf3 throughput
  [ ] Latency measurements
  [ ] Load test (100+ concurrent)
  [ ] Memory/CPU usage

[ ] Disaster Recovery
  [ ] Node failure scenarios
  [ ] Service recovery time
  [ ] DNS failover
  [ ] BGP reconvergence
```

### 5.2 Troubleshooting Común

#### ❌ PROBLEMA: Pod no obtiene IP

```bash
# Síntomas
kubectl get pods
# Pod está en estado "Pending"

# Diagnóstico
kubectl describe pod web-app
# Eventos:
#   Warning  FailedScheduling  ... no nodes available

# Solución

# 1. Verificar CNI plugin está instalado
kubectl get pods -n kube-system | grep cni
# Debe mostrar: aci-containers-host, openvswitch, controller

# 2. Ver logs del CNI
kubectl logs -n kube-system aci-containers-controller
# Buscar errores de OpFlex connectivity

# 3. Verificar APIC connectivity
ping 192.168.1.30  # APIC IP
# Debe responder

# 4. Ver si hay IPs disponibles
# En APIC: Tenant → Network → Bridge Domains → Pod-BD
#   Verificar subnet de pods tiene IPs libres

# 5. Reiniciar CNI (último recurso)
kubectl delete pod -n kube-system -l app=aci-containers-host
# Auto-respawn de DaemonSet
```

#### ❌ PROBLEMA: Pods pueden hablar pero no deben

```bash
# Síntomas
Pod Frontend → Pod Backend (debería estar bloqueado)
Tráfico permitido pero NetworkPolicy lo deniega

# Diagnóstico (Cilium)
hubble observe --protocol-filter all | grep DENIED
# Output:
#   frontend-pod:random → backend-pod:5432  DENIED  (Policy)

# Verificar policy
kubectl get networkpolicies -n production
kubectl describe networkpolicy allow-frontend-to-backend

# Diagnóstico (ACI-CNI)
# Verificar EPG contracts en APIC
# Tenant → Contracts → Debería haber "deny-all" default

# Solución

# Si es falsa alarma (policy correcta pero tráfico pasa):
# 1. Verificar labels del pod
kubectl get pod -n production -o json | jq '.items[].metadata.labels'

# 2. Reaplica la policy
kubectl apply -f network-policy.yaml

# 3. Force resync (solo último recurso)
kubectl delete pod -n production -l app=backend  # Se recrea
```

#### ❌ PROBLEMA: External connectivity no funciona

```bash
# Síntomas
Pod no puede alcanzar 192.168.100.x (red corporativa)

# Diagnóstico

# 1. ¿Pod resuelve DNS?
kubectl exec -it <pod> -- nslookup google.com
# Debe responder

# 2. ¿Hay ruta hacia destino?
kubectl exec -it <pod> -- ip route
# Debería mostrar: default via 10.0.x.1

# 3. ¿SNAT está activo?
# En ACI APIC: verificar si SNAT está enabled
# Leaf →OpenStack → Port-Snat-IP

# 4. ¿Firewall corporativo permite?
# Probar desde jumpbox hacia SNAT IP
ping 172.30.0.10  # El SNAT IP
# Si no responde, firewall está bloqueando

# Solución

# Si es problema de SNAT:
# Editar ACI SNAT config
kubectl edit configmap aci-containers-config -n kube-system

# Agregar:
# snat_ips:
# - 172.30.0.0/24

# Reiniciar pod
kubectl delete pod -n kube-system -l app=aci-containers-controller

# Esperar reconvergencia (2-5 min)
```

#### ❌ PROBLEMA: BGP no converge

```bash
# Síntomas (Cilium/Full BGP)
kubectl get ciliumbgppeering
# Estado: "Failed"

# Diagnóstico

# 1. Ver estado BGP peers
cilium bgp peers
# Output: State = Idle (debe ser Established)

# 2. Ver logs del Cilium Agent
kubectl logs -n kube-system -l k8s-app=cilium | grep BGP

# 3. Verificar reachabilidad al peer
ping 192.168.1.101  # Leaf IP
# Debe responder

# 4. Verificar AS numbers coinciden
# En Cilium CRD: localASN debe diferir de peerASN

# Solución

# 1. Verificar conectividad capa 3
mtr -r 192.168.1.101
# Debe ser <50ms latency

# 2. Reconfigurar BGP si cambió AS
kubectl apply -f cilium-bgp-policy.yaml

# 3. Verificar timers BGP (recomendado: 1s/3s)
# En Cilium CRD:
# keepaliveInterval: 1s
# holdTimeInterval: 3s

# 4. Debugear en Leaf
# SSH a Leaf ACI
show bgp summary
# Ver si peering muestra "Established"

show bgp neighbors 172.31.0.10
# Ver detalles del peer (Node K8s)
```

### 5.3 MATRIZ DE DECISIÓN

#### Cuestionario de 10 preguntas

```
1. ¿Ya tienes ACI fabric operacional?
   SI → Puntaje ACI +3
   NO → Puntaje Cilium +3

2. ¿Necesitas latencia < 0.25ms?
   SI → Cilium +2
   NO → Neutral

3. ¿Tienes >500 nodos planeados?
   SI → Cilium +2 (mejor escalabilidad)
   NO → Neutral

4. ¿Necesitas integración con EPGs existentes?
   SI → ACI-CNI +3
   NO → Neutral

5. ¿Observabilidad en tiempo real es crítica?
   SI → Cilium +2 (Hubble)
   NO → Neutral

6. ¿Necesitas SNAT predecible?
   SI → ACI-CNI +2
   NO → Neutral

7. ¿Tienes budget limitado?
   SI → Cilium +1 (OSS)
   NO → Neutral

8. ¿Necesitas BGP desde inicio?
   SI → Cilium +2
   NO → Neutral

9. ¿Equipo tiene experiencia en ACI?
   SI → ACI-CNI +2
   NO → Cilium +1

10. ¿Compliance requiere audit trail?
    SI → ACI-CNI +2 (APIC logs)
    NO → Neutral

PUNTUACIÓN:
- ACI-CNI >10: Recomendado ACI-CNI
- Cilium >10: Recomendado Cilium
- Empate: Considerar HYBRID (Cilium + BGP a ACI)
```

### 5.4 Casos de Estudio de Decisión

#### CASO 1: Startup Fintech (100 nodos, crecimiento rápido)

```
Respuestas:
1. ¿ACI? NO (nueva infra)
2. ¿<0.25ms? SI (transacciones criticas)
3. ¿>500 nodos? SI (planeado)
4. ¿EPGs? NO
5. ¿Observabilidad? SI
6. ¿SNAT? NO (usa load balancer)
7. ¿Budget? SI (startup)
8. ¿BGP? SI (conecta a ISP)
9. ¿Experiencia ACI? NO
10. ¿Compliance? SI (KYC)

PUNTUACIÓN:
- ACI-CNI: 2
- Cilium: 2+2+2+2+1 = 11

RECOMENDACIÓN: ✅ CILIUM (puro)

ARQUITECTURA:
Cilium (eBPF) → BGP a ISP
Hubble para compliance logs
SNAT con egress IPs de Cilium
```

#### CASO 2: Empresa Fortune 500 (1000 nodos, ACI existente)

```
Respuestas:
1. ¿ACI? SI (ya tenemos fabric)
2. ¿<0.25ms? NO (suficiente 0.4ms)
3. ¿>500 nodos? SI
4. ¿EPGs? SI (políticas ACI existentes)
5. ¿Observabilidad? SI
6. ¿SNAT? SI (firewall corporativo)
7. ¿Budget? NO (indiferente)
8. ¿BGP? SI (integración con fabric)
9. ¿Experiencia ACI? SI
10. ¿Compliance? SI (regulación financiera)

PUNTUACIÓN:
- ACI-CNI: 3+3+2+2+2 = 12
- Cilium: 2+1 = 3

RECOMENDACIÓN: ✅ ACI-CNI + HYBRID BGP

ARQUITECTURA:
ACI-CNI para pod networking
Hybrid BGP (solo ingress nodes)
ESG integration con compliance policies
SNAT a IPs corporativas estables
```

#### CASO 3: Empresa en Transición (500 nodos, ACI + K8s nuevos)

```
Respuestas:
1. ¿ACI? SI (estamos instalando)
2. ¿<0.25ms? NO
3. ¿>500 nodos? SI
4. ¿EPGs? SI (la idea es usar)
5. ¿Observabilidad? SI (auditoría)
6. ¿SNAT? SI
7. ¿Budget? NO
8. ¿BGP? SI
9. ¿Experiencia ACI? NO (aprendiendo)
10. ¿Compliance? SI

PUNTUACIÓN:
- ACI-CNI: 3+2+2+2 = 9
- Cilium: 2+2+1 = 5

RECOMENDACIÓN: ✅ HYBRID (Cilium + BGP a ACI)

ARQUITECTURA:
Cilium para performance / observabilidad
BGP de Cilium se conecta a APIC
Servicios expuestos via ACI
SNAT de Cilium hacia ACI ESGs
Beneficio: Aprenden Cilium + ACI en paralelo
```

### 5.5 Post-Implementation Monitoring

#### Métricas Clave

```bash
# Latencia P99 de pod-to-pod
kubectl exec -it netcat-pod -- \
  ping -i 0.1 -c 100 another-pod | \
  tail -1
# Max should be <1ms

# Throughput de conexión
kubectl exec -it iperf-server -- \
  iperf3 -s &
kubectl exec -it iperf-client -- \
  iperf3 -c iperf-server
# Should match cluster network capacity

# Policy evaluation time
# Para Cilium Hubble:
hubble metrics --export-interval=10s
# Ver policy_verdict_total

# BGP convergence
# Para Full/Hybrid BGP:
# Time from node join to announcing routes
# Target: <30 seconds
```

### ✅ RESUMEN MÓDULO 5

| Fase | Duración | Entregable |
|------|----------|------------|
| Planning | 2-3 sem | Matriz de decisión |
| Fabric Prep | 1-2 sem | APIC configured |
| K8s Install | 1 sem | CNI running |
| Validation | 3-5 días | Benchmarks OK |
| Production | Ongoing | Monitoring |

---

# CONCLUSIÓN Y CERTIFICACIÓN
## (20 minutos)

### Matriz Final de Decisión Completa

```
┌─────────────────────────────────────────────────────────┐
│              DECISION MATRIX                            │
│                                                         │
│  SI ACI existente y EPGs importantes:                   │
│  → ACI-CNI (FULL o HYBRID BGP según escala)            │
│                                                         │
│  SI nueva infra, latencia crítica, >500 nodos:         │
│  → Cilium (eBPF puro)                                  │
│                                                         │
│  SI transición gradual, aprender ambas:                │
│  → Cilium + BGP hacia ACI fabric                       │
│                                                         │
│  SI máxima seguridad y compliance:                     │
│  → ACI-CNI (auditoría APIC) + Cilium Hubble           │
│                                                         │
│  SI performance > everything:                          │
│  → Cilium eBPF (0.2ms latency)                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Key Takeaways

1. **Kubernetes Networking es complejo** pero soluble
2. **ACI-CNI es la solución empresarial** si tienes ACI
3. **Cilium es el futuro** con eBPF y observabilidad
4. **BGP es el puente** entre K8s y infraestructura corp
5. **No hay solución única** - depende de contexto

### Certificado de Completitud

```
╔═══════════════════════════════════════════════════════╗
║                                                       ║
║    CURSO COMPLETADO                                  ║
║                                                       ║
║    Kubernetes Infrastructure Connectivity para ACI   ║
║                                                       ║
║    Participante: _____________________________        ║
║                                                       ║
║    Módulos Cubiertos:                                ║
║    ✅ Fundamentos K8s y Networking                  ║
║    ✅ ACI-CNI Arquitectura                          ║
║    ✅ BGP (Full vs Hybrid)                          ║
║    ✅ Cilium y eBPF                                 ║
║    ✅ Implementación Práctica                       ║
║                                                       ║
║    Competencias Adquiridas:                          ║
║    ✓ Diseñar K8s networks empresariales             ║
║    ✓ Integrar con ACI fabric                        ║
║    ✓ Troubleshoot connectivity issues               ║
║    ✓ Elegir CNI apropiado                           ║
║                                                       ║
║    Fecha: _____________                             ║
║    Instructor: Networking Avanzado 2025             ║
║                                                       ║
╚═══════════════════════════════════════════════════════╝
```

### Recursos Recomendados

```
📚 DOCUMENTACIÓN
- https://github.com/noironetworks/aci-containers (ACI-CNI)
- https://docs.cilium.io (Cilium oficial)
- https://www.cisco.com/c/dam/.../aci-k8s-whitepaper.pdf

🎥 VIDEOS
- KubeAcademy: "Introduction to CNI"
- Cisco Learning Network: ACI courses
- IsovalentWebinars: Cilium BGP routing

🧪 LABS PRÁCTICOS
- Docker Labs: Kubernetes Deep Dive
- Cisco Dcloud: ACI + K8s demos
- Cilium Tutorials en minikube

👥 COMUNIDADES
- CNCF Slack: #cni
- Kubernetes community: sig-network
- Cisco Learning Network: ACI forums

📋 HERRAMIENTAS
- kubectl (K8s)
- hubble (Cilium observability)
- acc-provision (ACI provisioning)
- Prometheus + Grafana (monitoring)
```

### Call to Action

```
PRÓXIMOS PASOS RECOMENDADOS:

1. INMEDIATO (Hoy)
   [ ] Revisar matriz de decisión
   [ ] Asignar equipo de evaluación
   [ ] Descargar whitepapers ACI/Cilium

2. ESTA SEMANA
   [ ] Setup lab con Cilium en minikube
   [ ] Revisar acc-provision tool
   [ ] Contactar a Cisco TAM

3. ESTE MES
   [ ] PoC con CNI seleccionado
   [ ] Benchmark interno
   [ ] Design review con team

4. Q2 2025
   [ ] Implementación piloto (10-20 nodos)
   [ ] Training de operaciones
   [ ] Rollout gradual a producción

SOPORTE:
- Documentación: https://...
- TAM Cisco: cisco_tam@company.com
- Lab Environment: dcloud.cisco.com
- Slack Support: #k8s-networking-support
```

---

## FORMATO DE EVALUACIÓN (OPCIONAL)

```
POST-COURSE ASSESSMENT (10 preguntas)

1. ¿Cuál es la diferencia técnica entre Full BGP e Hybrid BGP?
   A) Número de sesiones BGP
   B) AAncho de banda
   C) Encapsulación VXLAN
   D) A es correcta

2. En ACI-CNI, ¿qué protocolo usa Host Agent para programar OVS?
   A) OpenFlow
   B) OpFlex
   C) gRPC
   D) REST API

3. eBPF en Cilium permite latencia de aproximadamente:
   A) 0.4ms
   B) 0.2ms
   C) 1ms
   D) 2ms

4. ¿Para cuál caso de uso es MEJOR ACI-CNI?
   A) Startup sin infra
   B) Empresa con ACI existente
   C) Máxima observabilidad
   D) Menor presupuesto

5. En SNAT de ACI-CNI, la IP corporativa es:
   A) Dinámica (por pod)
   B) Estable (configurable)
   C) Auto-asignada
   D) Del pool de servicios

6. El componente que ejecuta eBPF programs en Cilium es:
   A) Cilium Agent
   B) Kernel Linux
   C) BPF Maps
   D) Hubble

7. ¿Cuál es la principal desventaja de Hybrid BGP?
   A) Complejidad alta
   B) Punto único de fallo
   C) Menos seguridad
   D) Mayor latencia

8. En acc-provision, el archivo de salida contiene:
   A) Configuración APIC
   B) YAML de Kubernetes
   C) Scripts de provisioning
   D) B y C

9. ¿Qué herramienta se usa para observabilidad en Cilium?
   A) Prometheus
   B) Hubble
   C) Jaeger
   D) ELK

10. El pod-to-pod flow en ACI-CNI sin SNAT va de:
    A) IP efímera a IP efímera
    B) SNAT IP a SNAT IP
    C) Pod subnet a Pod subnet
    D) Servicio IP a Endpoint IP

SCORING:
8-10: ✅ EXCELENTE
6-7:  ✅ BUENO
4-5:  ⚠️ NECESITA REPASO
<4:   ❌ REVISAR MÓDULOS
```

---

## APÉNDICES

### A. Comandos Útiles de Troubleshooting

```bash
# ACI-CNI
kubectl get pods -n kube-system -l app=aci-containers
kubectl logs -n kube-system -l app=aci-containers-host -f
kubectl exec -it <aci-pod> -n kube-system -- opflex-agent-inspect

# Cilium
cilium status
cilium endpoint list
hubble observe --follow
hubble metrics --export-interval 10s

# Networking general
kubectl exec -it <pod> -- ip route
kubectl exec -it <pod> -- iptables -L -n -v
kubectl exec -it <pod> -- ss -tlnp
```

### B. Links de Referencia

- Cisco ACI: https://www.cisco.com/c/en/us/solutions/data-center/application-centric-infrastructure/index.html
- Cilium: https://cilium.io
- Kubernetes networking: https://kubernetes.io/docs/concepts/services-networking/
- CNCF CNI: https://github.com/containernetworking

### C. Glosario

- **EPG**: Endpoint Group (grupo de endpoints)
- **OpFlex**: Open Policy Exchange (protocolo de control)
- **eBPF**: Extended Berkeley Packet Filter (VM en kernel)
- **SNAT**: Source Network Address Translation
- **PBR**: Policy-Based Routing
- **VXLAN**: Virtual Extensible LAN
- **BGP**: Border Gateway Protocol
- **APIC**: Application Policy Infrastructure Controller

---

**FIN DEL CURSO**

*Duración Total: 5 horas*
*Nivel: Expertos IT*
*Versión: 1.0 (2025)*
