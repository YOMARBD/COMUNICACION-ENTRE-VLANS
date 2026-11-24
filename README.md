<img width="1065" height="540" alt="image" src="https://github.com/user-attachments/assets/dddabddb-fcb1-4011-9ae6-cdc71b55889f" />

# 🌐 Laboratorio 01: Enrutamiento Inter-VLAN con VyOS y Subnetting `/25`

Implementación de una red segmentada mediante **VLANs (802.1Q)** utilizando un router **VyOS** en esquema *Router-on-a-Stick* dentro de **GNS3**, aplicando subredes con máscara `/25` (`255.255.255.128`).

---

## 1. Topología de Red

![Topología de Red en GNS3](<img width="1050" height="547" alt="image" src="https://github.com/user-attachments/assets/0caa5d89-8740-47a4-8069-9521e2f61455" />
)


## 2. Esquema de Direccionamiento IP y Subnetting

Partiendo de la red base **`192.168.100.0/24`**, se prestó **1 bit** (\(2^1 = 2\text{ subredes}\)) obteniendo la máscara **`/25` (`255.255.255.128`)** con 126 hosts útiles por subred:

* **Subred 1 (VLAN 10):** `192.168.100.0/25` | Rango útil: `.1` – `.126` | Broadcast: `.127`
* **Subred 2 (VLAN 30):** `192.168.100.128/25` | Rango útil: `.129` – `.254` | Broadcast: `.255`

### Tabla de Asignación de Dispositivos

| Dispositivo | Interfaz | VLAN | Dirección IP / Máscara | Puerta de Enlace (Gateway) |
| :--- | :--- | :--- | :--- | :--- |
| **ROUTER-1 (VyOS)** | `eth0.10` (`vif 10`) | 10 | `192.168.100.1/25` | N/A |
| **ROUTER-1 (VyOS)** | `eth0.30` (`vif 30`) | 30 | `192.168.100.129/25` | N/A |
| **Switch1** | Puerto hacia Router | 1 (Troncal) | N/A (Capa 2 - `dot1q`) | N/A |
| **Switch1** | Puerto hacia PC1 | 10 (Acceso) | N/A (Capa 2 - `access`) | N/A |
| **Switch1** | Puerto hacia PC2 | 30 (Acceso) | N/A (Capa 2 - `access`) | N/A |
| **PC1 (VPCS)** | `eth0` | 10 | `192.168.100.10/25` | `192.168.100.1` |
| **PC2 (VPCS)** | `eth0` | 30 | `192.168.100.130/25` | `192.168.100.129` |

---

## 3. Configuración de los Equipos

### A. Router (`ROUTER-1` - VyOS)
Creación de subinterfaces virtuales (`vif`) sobre `eth0` para habilitar el enrutamiento entre VLANs:

```bash
configure

# Configuración VLAN 10
set interfaces ethernet eth0 vif 10 address 192.168.100.1/25
set interfaces ethernet eth0 vif 10 description "VLAN_10"

# Configuración VLAN 30
set interfaces ethernet eth0 vif 30 address 192.168.100.129/25
set interfaces ethernet eth0 vif 30 description "VLAN_30"

# Aplicar y guardar cambios en disco
commit
save
exit
```

### B. Switch (`Switch1` - Ethernet Switch GNS3)
Configuración de puertos en el menú *Configure* del switch:
* **Puerto conectado a `ROUTER-1`:** Type `dot1q` | VLAN `1` (Enlace troncal).
* **Puerto conectado a `PC1`:** Type `access` | VLAN `10`.
* **Puerto conectado a `PC2`:** Type `access` | VLAN `30`.

### C. Hosts Finales (`VPCS`)

**En `PC1`:**
```bash
ip 192.168.100.10/25 192.168.100.1
save
```

**En `PC2`:**
```bash
ip 192.168.100.130/25 192.168.100.129
save
```

---

## 4. Verificación y Pruebas de Conectividad

### Comandos de diagnóstico en VyOS
```bash
show interfaces
show ip route
```

### Prueba de Ping Inter-VLAN (`PC1` ↔ `PC2`)
Se verifica conectividad bidireccional entre distintas subredes pasando por el router:

```bash
PC1> ping 192.168.100.130
84 bytes from 192.168.100.130 icmp_seq=1 ttl=63 time=30.067 ms
```

![Prueba de conectividad exitosa](conexion pc1 a pc2 : <img width="608" height="127" alt="image" src="https://github.com/user-attachments/assets/be6b77e7-48ec-42d3-b687-c3521808feb5" />
conexión de pc2 a pc1: <img width="785" height="125" alt="image" src="https://github.com/user-attachments/assets/ae9b213e-0893-4355-9bf4-07c10ebe2814" />
 )

> **Nota técnica (Análisis de TTL):** El valor **`ttl=63`** confirma que los paquetes salen de la PC con TTL=64, suben al router VyOS (donde se decrementa 1 salto) y son enrutados correctamente hacia la otra VLAN. Si el TTL fuera `64`, indicaría conmutación directa en Capa 2 sin pasar por el router.

---

## 5. Lecciones Aprendidas y Troubleshooting
* **Uso obligatorio de `commit` y `save` en VyOS:** Los comandos `set` o `delete` solo preparan la configuración en memoria; sin `commit` no entran en vigor y sin `save` se pierden al reiniciar el nodo.
* **Limpieza de ARP en VPCS:** Si se realiza un `ping` antes de terminar de configurar el router o el switch, VPCS puede guardar una entrada inválida y mostrar `host not reachable`; se soluciona ejecutando `clear arp`.
