 

![imag](https://github.com/CR0NYM3X/GCP-Labs/blob/main/img/CBL389%20.png)

 

### 🚨 El Escenario del Mundo Real: La Problemática

Imagina que eres el Arquitecto Cloud de **"EuroBank S.A."**, una institución financiera que está adoptando un modelo de nube híbrida.

**El Problema:**
El banco acaba de desplegar su nueva y moderna aplicación de análisis de transacciones en Google Cloud, pero el sistema central (Core Bancario) que almacena los saldos de los usuarios sigue viviendo físicamente en el centro de datos local de la empresa (On-Premise) en Europa.

Actualmente, la aplicación en GCP no puede comunicarse con el Core Bancario. El Director de TI (CTO) te ha dado dos requisitos innegociables:

1. **Seguridad:** El tráfico entre Google Cloud y el centro de datos físico no puede viajar expuesto por el internet público; debe estar fuertemente encriptado.
2. **Resiliencia Extrema:** Como es una aplicación bancaria, no pueden permitirse caídas si Google realiza mantenimiento en un servidor. El negocio exige por contrato un **Acuerdo de Nivel de Servicio (SLA) del 99.99% de disponibilidad**.

### 🎯 La Solución Arquitectónica: El Objetivo del Laboratorio

Para resolver esto, vas a implementar una **HA VPN (High Availability VPN)** con **Cloud Router**.

En este laboratorio simularás ambos entornos (GCP y tu Data Center On-Premise) usando dos redes VPC separadas. Vas a construir "puertas de enlace" con dos interfaces independientes y conectarás **dos túneles IPsec simultáneos**. Sobre estos túneles, usarás un protocolo llamado **BGP (Border Gateway Protocol)** para que las redes descubran sus rutas dinámicamente.

**¿El resultado esperado?** Si un túnel falla o entra en mantenimiento (lo cual simularemos destruyendo uno a propósito), el Cloud Router desviará todo el tráfico bancario por el túnel sobreviviente en milisegundos. Problema resuelto, negocio protegido.

---

### ⚖️ Diferenciación Crítica: ¿Por qué elegimos HA VPN?

Como consultor, debes saber defender por qué elegiste HA VPN y no otra tecnología. Aquí tienes tu arsenal para convencer al CTO (o al examinador de Google):

| Escenario / Requisito | Solución Recomendada | Justificación Arquitectónica |
| --- | --- | --- |
| Encriptación por Internet con **99.99%** de SLA | **HA VPN** | Requiere 2 túneles activos y un Cloud Router (BGP). Es tolerante a fallos de hardware en una sola interfaz. |
| Encriptación por Internet, router On-Prem antiguo (no soporta BGP) | **Classic VPN** | Solo ofrece **99.9%** de SLA. Permite enrutamiento estático basado en políticas. *(GCP ya la considera legacy)*. |
| Conexión privada sin usar el internet público (>10 Gbps) | **Cloud Interconnect** | Conexión física directa. Es mucho más costosa, tarda semanas en instalarse físicamente y no usa internet. |

 




### 🏛️ Fundamentos de Jerarquía y Permisos

Antes de teclear, recuerda esto: Las redes **VPC**, las **Instancias de Compute Engine**, los **Gateways VPN** y los **Cloud Routers** son recursos que viven a nivel de **Proyecto** (`Project`).

* Para ejecutar este laboratorio, necesitas que IAM te herede (desde la Organización, Carpeta o directamente en el Proyecto) los roles de `roles/compute.networkAdmin` y `roles/compute.instanceAdmin.v1`.

---

### 🛠️ Tarea 1: Configura un entorno de VPC global

Vamos a construir la red principal de tu empresa en Google Cloud.

**Paso 1: Crear la VPC**
Creamos una red en modo personalizado para tener control total sobre nuestras direcciones IP.

```bash
gcloud compute networks create vpc-demo --subnet-mode custom

```

*Nota:* El flag `--subnet-mode custom` evita que GCP cree automáticamente 30+ subredes (una por cada región del mundo), ahorrando cuota y manteniendo la seguridad.

**Paso 2: Crear las subredes regionales**
La VPC es global, pero las subredes son **regionales**. Crearemos una en Bélgica y otra en Países Bajos.

```bash
gcloud compute networks subnets create vpc-demo-subnet1 \
--network vpc-demo --range 10.1.1.0/24 --region "europe-west1"

gcloud compute networks subnets create vpc-demo-subnet2 \
--network vpc-demo --range 10.2.1.0/24 --region europe-west4

```

**Paso 3: Configurar Reglas de Firewall**
El firewall en GCP se aplica a nivel de red (VPC), pero se hace efectivo en las interfaces de red (NICs) de las instancias.

```bash
# Permite todo el tráfico interno dentro de la red (10.0.0.0/8)
gcloud compute firewall-rules create vpc-demo-allow-custom \
  --network vpc-demo \
  --allow tcp:0-65535,udp:0-65535,icmp \
  --source-ranges 10.0.0.0/8

# Permite acceso externo por SSH (puerto 22) y Ping (ICMP)
gcloud compute firewall-rules create vpc-demo-allow-ssh-icmp \
    --network vpc-demo \
    --allow tcp:22,icmp

```

**Paso 4: Desplegar Instancias de Compute Engine**
Estas serán nuestras máquinas de prueba.

```bash
gcloud compute instances create vpc-demo-instance1 --machine-type=e2-medium --zone europe-west1-c --subnet vpc-demo-subnet1

gcloud compute instances create vpc-demo-instance2 --machine-type=e2-medium --zone europe-west4-a --subnet vpc-demo-subnet2

```

---

### 🏢 Tarea 2: Configura un entorno local simulado (On-Prem)

Para probar una VPN, necesitamos un centro de datos externo. Lo simularemos con otra VPC en GCP.

**Paso 1: Crear la red y subred On-Premise**

```bash
gcloud compute networks create on-prem --subnet-mode custom

gcloud compute networks subnets create on-prem-subnet1 \
--network on-prem --range 192.168.1.0/24 --region europe-west1

```

**Paso 2: Reglas de Firewall On-Premise**

```bash
gcloud compute firewall-rules create on-prem-allow-custom \
  --network on-prem \
  --allow tcp:0-65535,udp:0-65535,icmp \
  --source-ranges 192.168.0.0/16

gcloud compute firewall-rules create on-prem-allow-ssh-icmp \
    --network on-prem \
    --allow tcp:22,icmp

```

**Paso 3: Crear la instancia local**
*¡Atención!* Reemplaza `[ZONE_NAME]` con una zona real de `europe-west1` (ej. `europe-west1-b`).

```bash
gcloud compute instances create on-prem-instance1 --machine-type=e2-medium --zone [ZONE_NAME] --subnet on-prem-subnet1

```

---

### 🛡️ Tarea 3: Configura una puerta de enlace de VPN con HA

Aquí nace la magia. Un Gateway de VPN HA provee automáticamente **dos direcciones IP públicas** (Interface 0 y 1).

**Paso 1: Crear los Gateways VPN**

```bash
gcloud compute vpn-gateways create vpc-demo-vpn-gw1 --network vpc-demo --region europe-west1

gcloud compute vpn-gateways create on-prem-vpn-gw1 --network on-prem --region europe-west1

```

**Paso 2: Verificar la configuración (Opcional pero recomendado)**
Puedes usar el flag `--format=json` si necesitas procesar esto en un script, pero por defecto te mostrará un YAML con las IPs públicas asignadas.

```bash
gcloud compute vpn-gateways describe vpc-demo-vpn-gw1 --region europe-west1

gcloud compute vpn-gateways describe on-prem-vpn-gw1 --region europe-west1

```

**Paso 3: Crear los Cloud Routers**
El Cloud Router no enruta datos; es un plano de control que gestiona BGP. Requiere un ASN (Autonomous System Number) privado (64512 - 65534).

```bash
gcloud compute routers create vpc-demo-router1 \
    --region europe-west1 --network vpc-demo --asn 65001

gcloud compute routers create on-prem-router1 \
    --region europe-west1 --network on-prem --asn 65002

```

---

### 🌉 Tarea 4: Crea dos túneles VPN

Para el SLA del 99.99%, **debes** conectar las interfaces simétricamente (0 con 0, 1 con 1). Reemplaza `[SHARED_SECRET]` por una contraseña fuerte (ej. `GoogleCloud123!`).

**Paso 1: Túneles desde GCP hacia On-Prem**

```bash
# Túnel para la Interfaz 0
gcloud compute vpn-tunnels create vpc-demo-tunnel0 \
    --peer-gcp-gateway on-prem-vpn-gw1 --region europe-west1 \
    --ike-version 2 --shared-secret [SHARED_SECRET] \
    --router vpc-demo-router1 --vpn-gateway vpc-demo-vpn-gw1 --interface 0

# Túnel para la Interfaz 1
gcloud compute vpn-tunnels create vpc-demo-tunnel1 \
    --peer-gcp-gateway on-prem-vpn-gw1 --region europe-west1 \
    --ike-version 2 --shared-secret [SHARED_SECRET] \
    --router vpc-demo-router1 --vpn-gateway vpc-demo-vpn-gw1 --interface 1

```

**Paso 2: Túneles desde On-Prem hacia GCP**

```bash
gcloud compute vpn-tunnels create on-prem-tunnel0 \
    --peer-gcp-gateway vpc-demo-vpn-gw1 --region europe-west1 \
    --ike-version 2 --shared-secret [SHARED_SECRET] \
    --router on-prem-router1 --vpn-gateway on-prem-vpn-gw1 --interface 0

gcloud compute vpn-tunnels create on-prem-tunnel1 \
    --peer-gcp-gateway vpc-demo-vpn-gw1 --region europe-west1 \
    --ike-version 2 --shared-secret [SHARED_SECRET] \
    --router on-prem-router1 --vpn-gateway on-prem-vpn-gw1 --interface 1

```

---

### 🧠 Tarea 5: Intercambio de Tráfico BGP

Aquí configuramos las IPs de "enlace local" (`169.254.x.x`) para que los Cloud Routers se hablen entre sí.

**Paso 1: BGP en vpc-demo**

```bash
# Configuración Interfaz/Peer para el Túnel 0
gcloud compute routers add-interface vpc-demo-router1 --interface-name if-tunnel0-to-on-prem --ip-address 169.254.0.1 --mask-length 30 --vpn-tunnel vpc-demo-tunnel0 --region europe-west1

gcloud compute routers add-bgp-peer vpc-demo-router1 --peer-name bgp-on-prem-tunnel0 --interface if-tunnel0-to-on-prem --peer-ip-address 169.254.0.2 --peer-asn 65002 --region europe-west1

# Configuración Interfaz/Peer para el Túnel 1
gcloud compute routers add-interface vpc-demo-router1 --interface-name if-tunnel1-to-on-prem --ip-address 169.254.1.1 --mask-length 30 --vpn-tunnel vpc-demo-tunnel1 --region europe-west1

gcloud compute routers add-bgp-peer vpc-demo-router1 --peer-name bgp-on-prem-tunnel1 --interface if-tunnel1-to-on-prem --peer-ip-address 169.254.1.2 --peer-asn 65002 --region europe-west1

```

**Paso 2: BGP en on-prem**
Nota cómo las IPs se invierten respecto al paso anterior.

```bash
# Configuración Interfaz/Peer para el Túnel 0
gcloud compute routers add-interface on-prem-router1 --interface-name if-tunnel0-to-vpc-demo --ip-address 169.254.0.2 --mask-length 30 --vpn-tunnel on-prem-tunnel0 --region europe-west1

gcloud compute routers add-bgp-peer on-prem-router1 --peer-name bgp-vpc-demo-tunnel0 --interface if-tunnel0-to-vpc-demo --peer-ip-address 169.254.0.1 --peer-asn 65001 --region europe-west1

# Configuración Interfaz/Peer para el Túnel 1
gcloud compute routers add-interface on-prem-router1 --interface-name if-tunnel1-to-vpc-demo --ip-address 169.254.1.2 --mask-length 30 --vpn-tunnel on-prem-tunnel1 --region europe-west1

gcloud compute routers add-bgp-peer on-prem-router1 --peer-name bgp-vpc-demo-tunnel1 --interface if-tunnel1-to-vpc-demo --peer-ip-address 169.254.1.1 --peer-asn 65001 --region europe-west1

```

---

### 🌐 Tarea 6: Verificación y Enrutamiento Global

Antes de probar, debemos abrir el firewall para que las subredes se hablen a través de la VPN.

**Paso 1: Abrir Firewalls a través del túnel**

```bash
gcloud compute firewall-rules create vpc-demo-allow-subnets-from-on-prem \
    --network vpc-demo --allow tcp,udp,icmp --source-ranges 192.168.1.0/24

gcloud compute firewall-rules create on-prem-allow-subnets-from-vpc-demo \
    --network on-prem --allow tcp,udp,icmp --source-ranges 10.1.1.0/24,10.2.1.0/24

```

**Paso 2: Verificar Túneles**

```bash
gcloud compute vpn-tunnels list
# Revisa el estado individual (debe decir "ESTABLISHED"):
gcloud compute vpn-tunnels describe vpc-demo-tunnel0 --region europe-west1
# (Repite para vpc-demo-tunnel1, on-prem-tunnel0, on-prem-tunnel1)

```

**Paso 3: ¡Prueba de Fuego! (Ping Regional)**
Entra por SSH a tu instancia on-prem y haz ping a la instancia de GCP en la misma región:

```bash
gcloud compute ssh on-prem-instance1 --zone [ZONE_NAME]
ping -c 4 10.1.1.2

```

**Paso 4: Activar Enrutamiento Global**
*Diferenciación Crítica:* Por defecto, el Cloud Router es **Regional**. No puede enviar tráfico a `europe-west4`. Al volverlo **Global**, anuncia las subredes de TODAS las regiones.

| Modo de Enrutamiento BGP | Comportamiento del Cloud Router | Uso |
| --- | --- | --- |
| **Regional** (Por defecto) | Solo conoce y anuncia subredes de su propia región. | Infraestructuras pequeñas o estrictamente localizadas. |
| **Global** | Conoce y anuncia subredes de **todas** las regiones de la VPC. | Redes multinacionales. *Requisito casi seguro en el examen*. |

Actualiza la red (abre una nueva pestaña en Cloud Shell para no perder el SSH):

```bash
gcloud compute networks update vpc-demo --bgp-routing-mode GLOBAL
gcloud compute networks describe vpc-demo # Para verificar

```

Regresa a tu sesión SSH e intenta alcanzar la instancia en `europe-west4`:

```bash
ping -c 2 10.2.1.2 # ¡Ahora funcionará!

```

---

### 🔥 Tarea 7: Prueba de Falla (Alta Disponibilidad real)

Un arquitecto no solo construye, también destruye para probar la resiliencia.

**Paso 1: Tira un túnel**

```bash
gcloud compute vpn-tunnels delete vpc-demo-tunnel0 --region europe-west1

```

*Puedes verificar que el lado opuesto colapsó con:* `gcloud compute vpn-tunnels describe on-prem-tunnel0 --region europe-west1`

**Paso 2: Verifica que tu red sigue viva**
Desde tu SSH, vuelve a hacer ping:

```bash
ping -c 3 10.1.1.2

```

¡Sigue respondiendo! BGP re-enrutó mágicamente los paquetes por el túnel sobreviviente (`tunnel1`).

---

### 🗑️ Tarea 8: Limpieza del Laboratorio (Destrucción Ordenada)

Si haces esto en tu propia cuenta, **debes** borrar los recursos para no pagar de más. El orden es inverso a la creación (Jerarquía de dependencias).

```bash
# 1. Túneles
gcloud compute vpn-tunnels delete on-prem-tunnel0 --region europe-west1 -q
gcloud compute vpn-tunnels delete vpc-demo-tunnel1 --region europe-west1 -q
gcloud compute vpn-tunnels delete on-prem-tunnel1 --region europe-west1 -q

# 2. BGP Peers
gcloud compute routers remove-bgp-peer vpc-demo-router1 --peer-name bgp-on-prem-tunnel0 --region europe-west1 -q
gcloud compute routers remove-bgp-peer vpc-demo-router1 --peer-name bgp-on-prem-tunnel1 --region europe-west1 -q
gcloud compute routers remove-bgp-peer on-prem-router1 --peer-name bgp-vpc-demo-tunnel0 --region europe-west1 -q
gcloud compute routers remove-bgp-peer on-prem-router1 --peer-name bgp-vpc-demo-tunnel1 --region europe-west1 -q

# 3. Routers
gcloud compute routers delete on-prem-router1 --region europe-west1 -q
gcloud compute routers delete vpc-demo-router1 --region europe-west1 -q

# 4. Gateways VPN
gcloud compute vpn-gateways delete vpc-demo-vpn-gw1 --region europe-west1 -q
gcloud compute vpn-gateways delete on-prem-vpn-gw1 --region europe-west1 -q

# 5. Instancias (Reemplaza la zona on-prem)
gcloud compute instances delete vpc-demo-instance1 --zone europe-west1-c -q
gcloud compute instances delete vpc-demo-instance2 --zone europe-west4-a -q
gcloud compute instances delete on-prem-instance1 --zone [ZONE_NAME] -q

# 6. Firewalls
gcloud compute firewall-rules delete vpc-demo-allow-custom on-prem-allow-subnets-from-vpc-demo on-prem-allow-ssh-icmp on-prem-allow-custom vpc-demo-allow-subnets-from-on-prem vpc-demo-allow-ssh-icmp -q

# 7. Subredes
gcloud compute networks subnets delete vpc-demo-subnet1 --region europe-west1 -q
gcloud compute networks subnets delete vpc-demo-subnet2 --region europe-west4 -q
gcloud compute networks subnets delete on-prem-subnet1 --region europe-west1 -q

# 8. VPCs
gcloud compute networks delete vpc-demo -q
gcloud compute networks delete on-prem -q

```

*(Nota: Añadí `-q` o `--quiet` a los comandos de limpieza para automatizar el "Sí" que te pide la consola en cada paso y ahorrarte tiempo).*

> **Tip para el Examen ACE 🏆**
> Presta muchísima atención al **Enrutamiento Dinámico Global vs Regional**. En el examen te plantearán un escenario donde "Una VM en una región X no puede acceder al Data Center local a través de la VPN que está en la región Y". La solución directa y arquitectónicamente correcta es ejecutar `gcloud compute networks update [NOMBRE_VPC] --bgp-routing-mode GLOBAL`. ¡Punto seguro para tu certificación!
