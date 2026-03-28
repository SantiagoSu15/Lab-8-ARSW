# Lab #8 — Infraestructura como Código con Terraform (Azure)
**Curso:** BluePrints / ARSW  
**Duración estimada:** 2–3 horas (base) + 1–2 horas (retos)  
**Última actualización:** 2025-11-09

## Propósito
Modernizar el laboratorio de balanceo de carga en Azure usando **Terraform** para definir, aprovisionar y versionar la infraestructura. El objetivo es que los estudiantes diseñen y desplieguen una arquitectura reproducible, segura y con buenas prácticas de _IaC_.

## Objetivos de aprendizaje
1. Modelar infraestructura de Azure con Terraform (providers, state, módulos y variables).
2. Desplegar una arquitectura de **alta disponibilidad** con **Load Balancer** (L4) y 2+ VMs Linux.
3. Endurecer mínimamente la seguridad: **NSG**, **SSH por clave**, **tags**, _naming conventions_.
4. Integrar **backend remoto** para el _state_ en Azure Storage con _state locking_.
5. Automatizar _plan_/**apply** desde **GitHub Actions** con autenticación OIDC (sin secretos largos).
6. Validar operación (health probe, página de prueba), observar costos y destruir con seguridad.

> **Nota:** Este lab reemplaza la versión clásica basada en acciones manuales. Enfócate en _IaC_ y _pipelines_.

---

## Arquitectura objetivo
- **Resource Group** (p. ej. `rg-lab8-<alias>`)
- **Virtual Network** con 2 subredes:
  - `subnet-web`: VMs detrás de **Azure Load Balancer (público)**
  - `subnet-mgmt`: Bastion o salto (opcional)
- **Network Security Group**: solo permite **80/TCP** (HTTP) desde Internet al LB y **22/TCP** (SSH) solo desde tu IP pública.
- **Load Balancer** público:
  - Frontend IP pública
  - Backend pool con 2+ VMs
  - **Health probe** (TCP/80 o HTTP)
  - **Load balancing rule** (80 → 80)
- **2+ VMs Linux** (Ubuntu LTS) con cloud-init/Custom Script Extension para instalar **nginx** y servir una página con el **hostname**.
- **Azure Storage Account + Container** para Terraform **remote state** (con bloqueo).
- **Etiquetas (tags)**: `owner`, `course`, `env`, `expires`.

> **Opcional** (retos): usar **VM Scale Set**, o reemplazar LB por **Application Gateway** (L7).

---

## Requisitos previos
- Cuenta/Subscription en Azure (Azure for Students o equivalente).
- **Azure CLI** (`az`) y **Terraform >= 1.6** instalados en tu equipo.
- **SSH key** generada (ej. `ssh-keygen -t ed25519`).
- Cuenta en **GitHub** para ejecutar el pipeline de Actions.

---

## Estructura del repositorio (sugerida)
```
.
├─ infra/
│  ├─ main.tf
│  ├─ providers.tf
│  ├─ variables.tf
│  ├─ outputs.tf
│  ├─ backend.hcl.example
│  ├─ cloud-init.yaml
│  └─ env/
│     ├─ dev.tfvars
│     └─ prod.tfvars (opcional)
├─ modules/
│  ├─ vnet/
│  │  ├─ main.tf
│  │  ├─ variables.tf
│  │  └─ outputs.tf
│  ├─ compute/
│  │  ├─ main.tf
│  │  ├─ variables.tf
│  │  └─ outputs.tf
│  └─ lb/
│     ├─ main.tf
│     ├─ variables.tf
│     └─ outputs.tf
└─ .github/workflows/terraform.yml
```

---

## Bootstrap del backend remoto
Primero crea el **Resource Group**, **Storage Account** y **Container** para el _state_:

```bash
# Nombres únicos
SUFFIX=$RANDOM
LOCATION=eastus
RG=rg-tfstate-lab8
STO=sttfstate${SUFFIX}
CONTAINER=tfstate

az group create -n $RG -l $LOCATION
az storage account create -g $RG -n $STO -l $LOCATION --sku Standard_LRS --encryption-services blob
az storage container create --name $CONTAINER --account-name $STO
```

Completa `infra/backend.hcl.example` con los valores creados y renómbralo a `backend.hcl`.

---

## Variables principales (ejemplo)
En `infra/variables.tf` define:
- `prefix`, `location`, `vm_count`, `admin_username`, `ssh_public_key`
- `allow_ssh_from_cidr` (tu IPv4 en /32)
- `tags` (map)

En `infra/env/dev.tfvars`:
```hcl
prefix        = "lab8"
location      = "eastus"
vm_count      = 2
admin_username= "student"
ssh_public_key= "~/.ssh/id_ed25519.pub"
allow_ssh_from_cidr = "X.X.X.X/32" # TU IP
tags = { owner = "tu-alias", course = "ARSW/BluePrints", env = "dev", expires = "2025-12-31" }
```

---

## cloud-init de las VMs
Archivo `infra/cloud-init.yaml` (instala nginx y muestra el hostname):
```yaml
#cloud-config
package_update: true
packages:
  - nginx
runcmd:
  - echo "Hola desde $(hostname)" > /var/www/html/index.nginx-debian.html
  - systemctl enable nginx
  - systemctl restart nginx
```

---

## Flujo de trabajo local
```bash
cd infra

# Autenticación en Azure
az login
az account show # verifica la suscripción activa

# Inicializa Terraform con backend remoto
terraform init -backend-config=backend.hcl

# Revisión rápida
terraform fmt -recursive
terraform validate

# Plan con variables de dev
terraform plan -var-file=env/dev.tfvars -out plan.tfplan

# Apply
terraform apply "plan.tfplan"

# Verifica el LB público (cambia por tu IP)
curl http://$(terraform output -raw lb_public_ip)
```

**Outputs esperados** (ejemplo):
- `lb_public_ip`
- `resource_group_name`
- `vm_names`

---

## GitHub Actions (CI/CD con OIDC)
El _workflow_ `.github/workflows/terraform.yml`:
- Ejecuta `fmt`, `validate` y `plan` en cada PR.
- Publica el plan como artefacto/comentario.
- Job manual `apply` con _workflow_dispatch_ y aprobación.

**Configura OIDC** en Azure (federación con tu repositorio) y asigna el rol **Contributor** al _principal_ del _workflow_ sobre el RG del lab.

---

## Entregables en TEAMS
1. **Repositorio GitHub** del equipo con:
   - Código Terraform (módulos) y `cloud-init.yaml`.
   - `backend.hcl` **(sin secretos)** y `env/dev.tfvars` (sin llaves privadas).
   - Workflow de GitHub Actions y evidencias del `plan`.
2. **Diagrama** (componente y secuencia) del caso de estudio propuesto.
3. **URL/IP pública** del Load Balancer + **captura** mostrando respuesta de **2 VMs** (p. ej. refrescar y ver hostnames cambiar).
4. **Reflexión técnica** (1 página máx.): decisiones, trade‑offs, costos aproximados y cómo destruir seguro.
5. **Limpieza**: confirmar `terraform destroy` al finalizar.

---

## Rúbrica (100 pts)
- **Infra desplegada y funcional (40 pts):** LB, 2+ VMs, health probe, NSG correcto.
- **Buenas prácticas Terraform (20 pts):** módulos, variables, `fmt/validate`, _remote state_.
- **Seguridad y costos (15 pts):** SSH por clave, NSG mínimo, tags y _naming_; estimación de costos.
- **CI/CD (15 pts):** pipeline con `plan` automático y `apply` manual (OIDC).
- **Documentación y diagramas (10 pts):** README del equipo, diagramas claros y reflexión.

---

## Retos (elige 2+)
- Migrar a **VM Scale Set** con _Custom Script Extension_ o **cloud-init**.
- Reemplazar LB por **Application Gateway** con _probe_ HTTP y _path-based routing_ (si exponen múltiples apps).
- **Azure Bastion** para acceso SSH sin IP pública en VMs.
- **Alertas** de Azure Monitor (p. ej. estado del probe) y **Budget alert**.
- **Módulos privados** versionados con _semantic versioning_.

---

## Limpieza
```bash
terraform destroy -var-file=env/dev.tfvars
```

> **Tip:** Mantén los recursos etiquetados con `expires` y **elimina** todo al terminar.

---

## Preguntas de reflexión
- ¿Por qué L4 LB vs Application Gateway (L7) en tu caso? ¿Qué cambiaría?
- ¿Qué implicaciones de seguridad tiene exponer 22/TCP? ¿Cómo mitigarlas?
- ¿Qué mejoras harías si esto fuera **producción**? (resiliencia, autoscaling, observabilidad).

---
## desarrollo del laboratorio

<img width="1600" height="575" alt="image" src="https://github.com/user-attachments/assets/3d3ee046-e9ff-47b5-8475-efacec3f2b84" />

se realiza la instalacion del azure CLI

<img width="636" height="296" alt="image" src="https://github.com/user-attachments/assets/1204bdfd-87ad-4179-8d5f-d4f3a3e8ee49" />

verificacion de las versiones de azure y terrraform

<img width="676" height="450" alt="image" src="https://github.com/user-attachments/assets/463b61aa-5d75-491b-aaad-85592de119ae" />

se genera el par de claves publicas y privadas ssh

<img width="914" height="505" alt="image" src="https://github.com/user-attachments/assets/ec8028b4-a097-4fd8-82e3-a9c60b0966c7" />

<img width="1600" height="843" alt="image" src="https://github.com/user-attachments/assets/ee613835-0d99-411a-b840-833906c0c129" />

<img width="1600" height="281" alt="image" src="https://github.com/user-attachments/assets/c39fbc61-c0a9-4384-ab98-79fa1c5b8414" />

se definen variables de entorno (SUFFIX, LOCATION, RG, STO, CONTAINER) que permiten generar nombres únicos para los recursos. Luego se ejecuta el comando az group create para crear un Resource Group llamado rg-tfstate-lab8 en la región eastus.

Después, se crea una cuenta de almacenamiento (az storage account create) con cifrado habilitado para guardar archivos de forma segura. por ultimo se crea un contenedor (az storage container create) llamado tfstate dentro de esa cuenta, que será el lugar donde Terraform almacenará el archivo de estado del proyecto.

```bash
resource_group_name  = "rg-tfstate-lab8"
storage_account_name = "sttfstate1445"
container_name       = "tfstate"
key                  = "lab8/terraform.tfstate"



prefix   = "lab8"
location = "eastus"
vm_count = 2
admin_username = "student"
ssh_public_key = "~/.ssh/id_ed25519.pub"
allow_ssh_from_cidr = "181.63.150.68/32"
tags = { owner = "alias", course = "ARSW", env = "dev", expires = "2025-12-31" }
```
se realiza la configuracion de terraform para definir dónde se guardará el estado de la infraestructura y algunos parámetros de despliegue de máquinas virtuales en Azure

<img width="738" height="591" alt="image" src="https://github.com/user-attachments/assets/fc993a7c-d7fa-4e78-9013-c62527680cdf" />
<img width="714" height="194" alt="image" src="https://github.com/user-attachments/assets/6a16692f-4ace-4cfa-b1cb-2c72264f342a" />

se inicializa y se valida terraform

<img width="960" height="968" alt="image" src="https://github.com/user-attachments/assets/059be2b9-6724-47ed-bd7f-f91a130a1ea0" />
<img width="797" height="894" alt="image" src="https://github.com/user-attachments/assets/fefe776c-e1c6-4f47-a8d9-4df1894a0900" />
<img width="1600" height="329" alt="image" src="https://github.com/user-attachments/assets/fdbb6db7-7f20-4a11-9ced-6c8df29fb77e" />

se realiza y ejecuta el plan de terraform para la creacion de recursos

<img width="1600" height="326" alt="image" src="https://github.com/user-attachments/assets/083a1afb-7783-4403-b938-d6352506c640" />
<img width="928" height="248" alt="image" src="https://github.com/user-attachments/assets/ef9acfbf-1e43-4bf4-ad25-6fb34639930c" />

se finaliza la creacion de las 2 maquinas virtuales y todos los demas recursos, tambien se realizan pruebas para verificar el funcionamiento del balanceador de carga

<img width="901" height="713" alt="image" src="https://github.com/user-attachments/assets/fd97ce0c-244a-4885-966b-e2fbdc40b0f8" />
<img width="1558" height="399" alt="image" src="https://github.com/user-attachments/assets/89f97249-8027-4483-80b0-1c75760d894b" />

se crea un registro de aplicación en Azure Active Directory y se le dan permisos a la aplicación sobre la suscripción.

<img width="730" height="167" alt="image" src="https://github.com/user-attachments/assets/ac890a3e-ee8b-4de9-ae7f-00505894436c" />

se crea una credencial federada entre la aplicación de Azure AD y el repositorio de GitHub, lo que establece una relación de confianza entre GitHub Actions y Azure AD y Permite que los workflows de GitHub en el repositorio se autentiquen en Azure sin usar secretos

<img width="991" height="338" alt="image" src="https://github.com/user-attachments/assets/9ec063b1-d9d7-4af1-b91a-a46fb12f06be" />

se verifican los secretos configurados en el repositorio de GitHub necesarios para que GitHub Actions pueda autenticarse en Azure mediante OIDC.





