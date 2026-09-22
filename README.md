# Despliegue-de-Wazuh-single-node-en-Docker-Solucion-al-agente-no-visible-en-Dashboard
Guía para desplegar Wazuh Manager (Indexer y Dashboard) usando Docker Compose, y enrolar un agente Linux cuando el agente no aparece en el Dashboard por falta de key.

---

## Índice

1. [Pre-requisitos](#pre-requisitos)
2. [Puertos requeridos](#puertos-requeridos)
3. [Despliegue del Wazuh Server (Manager)](#despliegue-del-wazuh-server-manager)
4. [Creación y enrolamiento de agente](#creación-y-enrolamiento-de-agente)
5. [Verificación final](#verificación-final)
6. [Troubleshooting](#troubleshooting)
7. [Referencias](#referencias)

---

## Pre-requisitos

La versión del Server debe ser superior o igual al Agente, pero no menor, por temas de compatibilidad.

| Componente | RAM | Disco | SO probado |
| --- | --- | --- | --- |
| Wazuh Server | 4GB | 50GB | Ubuntu 22.04.3 LTS |
| Wazuh Agent | 2GB | 20GB | Linux Mint 22.3 |

Software:

- Docker Engine 29.7.2
- Docker Compose v5.5.0
- netcat (nc) en el agente para pruebas de conectividad
- gnupg y apt-transport-https en el agente

En entornos de producción no se recomienda usar este procedimiento, pero en este caso se ejecutará un script automatizado de Docker oficial para detectar el sistema, agregar el repositorio e instalar los paquetes necesarios Engine, CLI, Containerd.

---

## Puertos requeridos

| Puerto | Protocolo | Uso |
| --- | --- | --- |
| 443 | TCP | Dashboard (HTTPS) |
| 1514 | TCP | Comunicación de agentes |
| 1515 | TCP | Alistamiento de agentes (enrollment) |
| 55000 | TCP | API de Wazuh |

---

## Despliegue del Wazuh Server (Manager)

### 1. Instalar Docker

Comenzamos aplicando los siguientes comandos en la VMBox del Server que será nuestro Wazuh Manager:

```
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker "$USER"
```

### 2. Clonar el repositorio oficial

```
git clone https://github.com/wazuh/wazuh-docker.git
cd wazuh-docker/single-node
```

### 3. Generar los certificados SSL

```
docker compose -f generate-indexer-certs.yml run --rm generator
```

### 4. Configurar vm.max_map_count (requerido por el indexer)

```
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

### 5. Levantar los contenedores

```
docker compose up -d
docker compose ps
docker compose logs --tail=50
```

### 6. Acceder al Dashboard

```
https://localhost
https://<IP_DEL_SERVER>
```

Credenciales iniciales: admin / admin

### 7. Cambiar password

Antes de continuar se recomienda el cambio usando --change-all.

```
docker exec -it single-node-wazuh.indexer-1 bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh --change-all
```

Verificamos el nombre real del contenedor con docker compose ps, puede variar según la versión.

### 8. Verificar puertos en escucha

```
sudo ss -lntp | grep -E ':1514|:1515'
```

---

## Creación y enrolamiento de agente

Antes de ir hacia el agente nos podría surgir un problema a la hora de crearlo: podría haberse creado el agente pero no verse en el Dashboard. Para eso es necesario identificar el ID del agente para poder darle permiso a través de un hash que se encuentra en el Wazuh Server.

Para eso colocamos:

```
sudo docker exec -it single-node-wazuh.manager-1 bash
bash# /var/ossec/bin/manage_agents
```

Con la opción (A) creamos ID, Name, IP.

(Q) y exit para salir.

Para luego ir otra vez a bash# /var/ossec/bin/manage_agents, esta vez eligiendo (E) para poder buscar el ID creado y así aplicar el key. Una vez que vemos el key lo copiamos para después copiarlo en el agente si es necesario.

### Machine desde el nuevo agente

Instalamos la GPG Key para tener autenticidad de los paquetes de Wazuh y así no tener problemas de paquetes corruptos o falsificados.

```
sudo apt-get install -y gnupg apt-transport-https
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee -a /etc/apt/sources.list.d/wazuh.list
sudo apt-get update
```

Desde el Wazuh Server para crear el agente copiamos el enlace o el endpoint en el terminal del agente, pero es preferible, según la documentación del GitHub de Wazuh, que por buenas prácticas se debería usar el siguiente ejemplo de código:

```
sudo WAZUH_MANAGER="IP.MANAGER" \
WAZUH_AGENT_NAME="NAME_AGENT" \
WAZUH_AGENT_GROUP="GROUP1,GROUP2" \
apt-get install wazuh-agent=4.13.1-1
```

Después iniciamos el agente:

```
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Conectividad y logs:

```
sudo grep ^status /var/ossec/var/run/wazuh-agentd.state
nc -vz <IP.MANAGER> 1514
sudo tail -f /var/ossec/logs/ossec.log
```

Verificamos la conexión hacia el Server de Wazuh Manager, y en el address MANAGER_IP se le colocará la IP del Server Wazuh Manager:

```
sudo grep -A 15 -B 2 '<client>' /var/ossec/etc/ossec.conf

nc -vz ip.del.server.manager 1514
sudo systemctl restart wazuh-agent
sudo tail -f /var/ossec/logs/ossec.log
```

En esta parte, si no se visualiza en el Dashboard del Wazuh Server, acá es donde agregamos el key anteriormente copiado desde el Wazuh Server, pero esta vez en la opción (I):

```
sudo /var/ossec/bin/manage_agents
```

(Q) y exit para salir.

```
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

---

## Verificación final

En el Dashboard: Agents → Agents management. El agente debe figurar como Active con un Last keep alive reciente.

En el agente:

```
sudo systemctl status wazuh-agent
sudo grep ^status /var/ossec/var/run/wazuh-agentd.state
```

---

## Troubleshooting

### El agente no aparece en el Dashboard

1. Verificar que en ossec.conf el address apunte a la IP correcta del Manager.
2. Verificar conectividad con nc -vz IP_MANAGER 1514.
3. Importar la key con manage_agents opción I.
4. Reiniciar wazuh-agent.

### Los puertos 1514/1515 no escuchan en el Server

```
docker compose ps
docker compose logs --tail=100
sudo ss -lntp | grep -E ':1514|:1515'
```

---

## Referencias

- [Wazuh Docker (repo oficial)](https://github.com/wazuh/wazuh-docker)
- [Documentación Wazuh](https://documentation.wazuh.com/)
- [Instalación de agente por APT](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-linux.html)
- [manage_agents](https://documentation.wazuh.com/current/user-manual/reference/tools/manage-agents.html)
