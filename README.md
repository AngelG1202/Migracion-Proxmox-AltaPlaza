# Proyecto: Migración Multiplataforma, Consolidación y Alta Disponibilidad en Proxmox VE

## Contexto y Objetivo del Proyecto
El cliente operaba con una infraestructura tecnológica severamente fragmentada y carente de un esquema centralizado de Recuperación ante Desastres. Los servicios críticos se encontraban dispersos en múltiples entornos: máquinas virtuales respaldadas en discos externos, servidores en VMware ESXi, cargas de trabajo en Microsoft Azure, datos en SharePoint y aplicaciones físicas locales. 

**Objetivo:** Diseñar, implementar y migrar la totalidad de la infraestructura hacia un clúster de alta disponibilidad on-premise basado en **Proxmox VE**, utilizando almacenamiento redundante **ZFS** y replicación asíncrona entre nodos para garantizar la continuidad operativa.

---

## Arquitectura de Hardware y Almacenamiento
La infraestructura base se construyó sobre **dos (2) servidores HPE ProLiant DL360 Gen11** (Producción y Contingencia), cada uno configurado con:

* **Cómputo:** Procesador Intel Xeon Silver 4514Y con 128 GB de memoria RAM DDR5 ECC.
* **Gestión Out-of-Band:** HPE iLO 6 (v1.77).
* **Almacenamiento del Hipervisor:** Arreglo RAID 1 por hardware compuesto por 2 unidades SSD SATA HPE de 480 GB, dedicado exclusivamente a Proxmox VE para proteger el sistema operativo ante fallos.
* **Almacenamiento de Producción:** Pool ZFS (`zfs-data`) configurado en RAIDZ1 utilizando 4 discos SAS HPE de 2.4 TB a 10,000 RPM, garantizando integridad de datos y tolerancia a fallos para los discos de las máquinas virtuales.

## Topología de Red: Aislamiento y Segmentación
Se implementó una separación estricta del tráfico físico para maximizar la seguridad y facilitar la gestión:
* **Puerto Físico 1 (Management):** Dedicado exclusivamente a la administración de Proxmox, la comunicación de clúster (Corosync) y accesos SSH. Aislado completamente sobre la **VLAN 8**.
* **Puerto Físico 2 (Producción):** Configurado como Trunk a través del bridge `vmbr1`. Esto permite que el tráfico de las VMs sea transportado y segmentado mediante etiquetas VLAN dinámicas asignadas directamente desde la consola de Proxmox.

---

## Ejecución de Migración y Troubleshooting Técnico
Se migraron exitosamente 12 servidores hacia el nuevo clúster, superando importantes desafíos técnicos según la plataforma de origen:

### 1. Entorno Microsoft Azure (Windows Server 2025)
* **Máquinas:** `SRV-ALTA-DC` (Active Directory, DNS, SYSVOL, FSMO) y `SRV-ALTA-SAGE50`. Migradas vía Veeam Backup & Replication v13.
* **Incidencia Técnica:** El Controlador de Dominio restaurado inició en Modo Seguro, impidiendo la carga de servicios AD DS y DNS. Además, existía un DC secundario obsoleto registrado en Azure.
* **Resolución:** Se corrigió el arranque (BCD) y se ejecutó una limpieza profunda de metadatos de Active Directory (Metadata Cleanup) para eliminar registros DNS y referencias de replicación del servidor fantasma en la nube, recuperando el quórum del dominio y los roles FSMO.

### 2. Entorno VMware ESXi
* **Máquinas:** `SRV-ALTA-FTP-SERVER` y `SRV-ALTA-EQUS`.
* **Incidencia Técnica 1 (I/O Errors):** Intentos de importar discos VMDK directamente fallaron por errores de lectura en el datastore VMFS. Se resolvió orquestando un respaldo a nivel de bloque y restauración V12 utilizando Veeam v11.
* **Incidencia Técnica 2 (Blue Screen of Death):** El servidor FTP presentó BSOD por incompatibilidad de controladores de almacenamiento IDE/SATA en Proxmox.
* **Resolución:** Se inyectaron controladores VirtIO, se reconstruyó el Boot Configuration Data (BCD) y se ejecutaron comprobaciones de integridad lógica (`chkdsk` / `sfc`), recuperando el 100% de la operatividad.

### 3. Entorno Cloud SharePoint
* **Máquina:** `SRV-ALTA-FILE-SERVER`.
* **Ejecución:** Al no existir una máquina exportable, se provisionó un nuevo servidor Windows Server en Proxmox con un datastore ZFS de 2 TB y se descargó y reestructuró el árbol documental corporativo directamente desde SharePoint.

### 4. Backups en Discos Externos y Bare-Metal
* Importación de discos y configuración de hardware virtual para sistemas como `SRV-ALTA-NAGIOS`, `SRV-ALTA-UNIFI` y `SRV-ALTA-DOCKER`.
* Migración P2V (Physical-to-Virtual) del servidor físico `SRV-ALTA-BMS` usando Veeam v13.

---

## Clúster y Recuperación ante Desastres (DRP)
Una vez consolidadas las 12 cargas de trabajo en el nodo `srv-proxmox-01`, se unió el nodo de contingencia `srv-proxmox-02` para formar el Clúster "ALTA" (Quorate: Yes). 
Se eliminó la dependencia de trabajos de respaldo locales y se implementó la **Replicación ZFS a nivel de bloques**. Todos los datasets de las máquinas virtuales productivas se sincronizan de forma asíncrona hacia el nodo secundario **cada 15 minutos (`*/15`)**, garantizando un RPO estricto y la capacidad de levantar los servicios en minutos ante un fallo del nodo principal.

---

## Evidencias Fotográficas

> **Nota de Confidencialidad:** Por políticas de seguridad de la información y acuerdos de confidencialidad (NDA) con el cliente, las capturas de pantalla de la infraestructura en producción (consolas de administración HPE iLO 6, direcciones IP internas, inventario de máquinas virtuales y estado del almacenamiento ZFS) no se exponen públicamente en este repositorio. La ejecución técnica y la validación de los servicios migrados fueron certificadas y aprobadas en el documento de cierre formal del proyecto.
