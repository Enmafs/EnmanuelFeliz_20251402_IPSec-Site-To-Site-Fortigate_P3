# 🔒 Lab 10 — IPSec Site-to-Site VPN (FortiGate)

> **Práctica 3 — Semana 6 | Seguridad en Redes**
> Instituto Tecnológico de Las Américas (ITLA)
> Estudiante: Enmanuel Feliz Soto | Matrícula: 2025-1402
> Docente: Jonathan Esteban Rondón Corniel | Sección: 2-1C

---

## 📋 Descripción

Implementación de una VPN **IPSec Site-to-Site** entre dos dispositivos FortiGate usando **IKEv2** con autenticación por clave precompartida (PSK). El túnel cifrado permite la comunicación entre la LAN de la sede principal y la LAN de la sucursal a través de un ISP simulado en PNetLab.

---

## 🗺️ Topología de Red

<img width="1479" height="326" alt="image" src="https://github.com/user-attachments/assets/be195309-92e2-4a40-a302-709be70f2c9b" />


---

## 📊 Tabla de Direccionamiento

| Dispositivo | Interfaz | Dirección IP       | Máscara         | Descripción         |
|-------------|----------|--------------------|-----------------|---------------------|
| FGT-A       | port1    | 20.25.1.2          | /30             | WAN → ISP           |
| FGT-A       | port2    | 14.2.10.254        | /24             | LAN Sede Principal  |
| FGT-B       | port1    | 20.25.2.2          | /30             | WAN → ISP           |
| FGT-B       | port2    | 14.2.20.254        | /24             | LAN Sucursal        |
| ISP Router  | e0/0     | 20.25.1.1          | /30             | Enlace → FGT-A      |
| ISP Router  | e0/1     | 20.25.2.1          | /30             | Enlace → FGT-B      |

---

## 🔐 Parámetros IKE / IPSec

### Phase 1 (IKE)

| Parámetro        | Valor                    |
|------------------|--------------------------|
| IKE Version      | **2**                    |
| Modo             | Main                     |
| Autenticación    | Pre-Shared Key (PSK)     |
| Clave PSK        | `TuClaveSegura123!`      |
| Cifrado          | DES                      |
| Hash             | MD5                      |
| Peer Type        | Any                      |
| Net Device       | Disable                  |

### Phase 2 (IPSec / Quick Mode)

| Parámetro          | FGT-A                        | FGT-B                        |
|--------------------|------------------------------|------------------------------|
| Phase1 Name        | VPN-To-FGT-B                 | VPN-To-FGT-A                 |
| Cifrado            | DES / MD5                    | DES / MD5                    |
| Source Subnet      | 14.2.10.0/255.255.255.0      | 14.2.20.0/255.255.255.0      |
| Destination Subnet | 14.2.20.0/255.255.255.0      | 14.2.10.0/255.255.255.0      |

---

## 🖥️ Evidencias de Configuración

### Figura 1 — FGT-A: IPSec Phase 1 (VPN-To-FGT-B)
![Phase 1 FGT-A](screenshots/lab10_phase1_fgta.png)

### Figura 2 — FGT-A: IPSec Phase 2 (Selector de Tráfico)
![Phase 2 FGT-A](screenshots/lab10_phase2_fgta.png)

### Figura 3 — FGT-A: Firewall Policies (LAN ↔ VPN)
![Policies FGT-A](screenshots/lab10_policy_fgta.png)

### Figura 4 — Rutas Estáticas + Topología
![Routes y Topología](screenshots/lab10_routes_topology.png)

---

## ⚙️ Procedimiento de Configuración

### 1. ISP Router (Cisco IOS)

```cisco
enable
configure terminal

interface Ethernet0/0
 description Link_to_FGT-A
 ip address 20.25.1.1 255.255.255.252
 no shutdown
exit

interface Ethernet0/1
 description Link_to_FGT-B
 ip address 20.25.2.1 255.255.255.252
 no shutdown
exit

end
write memory
```

### 2. FGT-A — Sede Principal (Rojo)

#### 2.1 Interfaces

```fortios
config system interface
    edit "port1"
        set mode static
        set ip 20.25.1.2 255.255.255.252
        set allowaccess ping
    next
    edit "port2"
        set mode static
        set ip 14.2.10.254 255.255.255.0
        set allowaccess ping
    next
end
```

#### 2.2 Rutas Estáticas

```fortios
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 20.25.1.1
        set device "port1"
    next
    edit 2
        set dst 14.2.20.0 255.255.255.0
        set device "VPN-To-FGT-B"
    next
end
```

#### 2.3 VPN IPSec Phase 1

```fortios
config vpn ipsec phase1-interface
    edit "VPN-To-FGT-B"
        set interface "port1"
        set ike-version 2
        set peertype any
        set net-device disable
        set proposal des-md5
        set remote-gw 20.25.2.2
        set psksecret "TuClaveSegura123!"
    next
end
```

#### 2.4 VPN IPSec Phase 2

```fortios
config vpn ipsec phase2-interface
    edit "VPN-To-FGT-B-P2"
        set phase1name "VPN-To-FGT-B"
        set proposal des-md5
        set src-subnet 14.2.10.0 255.255.255.0
        set dst-subnet 14.2.20.0 255.255.255.0
    next
end
```

#### 2.5 Políticas de Firewall

```fortios
config firewall policy
    edit 1
        set name "LAN_to_VPN"
        set srcintf "port2"
        set dstintf "VPN-To-FGT-B"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
    edit 2
        set name "VPN_to_LAN"
        set srcintf "VPN-To-FGT-B"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end
```

### 3. FGT-B — Sucursal (Azul)

> Configuración simétrica a FGT-A con IPs invertidas. Remote GW: `20.25.1.2`. Subredes src/dst intercambiadas.

```fortios
config vpn ipsec phase1-interface
    edit "VPN-To-FGT-A"
        set interface "port1"
        set ike-version 2
        set peertype any
        set net-device disable
        set proposal des-md5
        set remote-gw 20.25.1.2
        set psksecret "TuClaveSegura123!"
    next
end

config vpn ipsec phase2-interface
    edit "VPN-To-FGT-A-P2"
        set phase1name "VPN-To-FGT-A"
        set proposal des-md5
        set src-subnet 14.2.20.0 255.255.255.0
        set dst-subnet 14.2.10.0 255.255.255.0
    next
end
```

---

## ✅ Verificación

```fortios
# Verificar estado del túnel Phase 1
get vpn ipsec tunnel summary

# Diagnóstico IKE
diagnose vpn ike gateway list

# Verificar Phase 2
diagnose vpn tunnel list

# Ping entre redes (desde FGT-A hacia red B)
execute ping-options source 14.2.10.254
execute ping 14.2.20.254
```

---

## 📁 Recursos

| Recurso | Enlace |
|---------|--------|
| Repositorio | [EnmanuelFelizSoto_20251402_VPN-Site-To-Site-Fortigate](https://github.com/Enmafs/EnmanuelFelizSoto_20251402_VPN-Site-To-Site-Fortigate) |
| Repositorio padre (NetSec) | [Enmafs/NetSec](https://github.com/Enmafs/NetSec) |
| Video demostrativo | 🎬 [Aquí](https://youtu.be/RCI8Qt6YqPU) |

---

> ⚠️ *Laboratorio realizado en entorno controlado (PNetLab). Fines exclusivamente académicos.*
