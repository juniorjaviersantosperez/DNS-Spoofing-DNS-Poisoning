DNS Spoofing / DNS Poisoning

DNS Spoofing/DNS Poisoning: Haga Spoofing del registro DNS itla.edu.do para que apunte a un servicio web local

Institución: Instituto Tecnológico de Las Américas (ITLA)
Asignatura: Seguridad de Redes
Estudiante: Junior Javier Santos Perez
Matrícula: 2024-1599
Link video: https://www.youtube.com/watch?v=I7Q85TNgc_A 
Enlace GitHub: https://github.com/juniorjaviersantosperez/DNS-Spoofing-DNS-Poisoning 

1. Objetivo del Laboratorio
Demostrar, en un entorno controlado con GNS3 y VMware Workstation, la técnica de ataque conocida como DNS Spoofing (DNS Cache Poisoning). El objetivo central es falsificar la resolución del nombre de dominio itla.edu.do de modo que el tráfico legítimo de la víctima sea redirigido hacia un servidor web falso controlado por el atacante (10.15.99.150).
Al finalizar el laboratorio, el estudiante es capaz de:
•	Comprender el funcionamiento del protocolo DNS y sus vulnerabilidades inherentes.
•	Ejecutar un ataque de DNS Spoofing combinando ARP Poisoning e intercepción de consultas DNS con Scapy.
•	Verificar el éxito del ataque observando cómo la víctima carga el servidor web falso.
•	Implementar y verificar contramedidas efectivas (DoH y DAI) para mitigar el ataque.




2. Topología de Red
 

GNS3 integrado con VMware Workstation · Servidor: GNS3 VM (juniorsantos-20241599) Host: LenovoV15
Tabla de Direccionamiento IP
Dispositivo	Interfaz	Dirección IP	Rol
kali-linux-2025.3-vmware-amd64-1	eth0 → e1 (Swich-1)	10.15.99.100/24	🔴 Atacante
Clonekali-1	eth0 → e2 (Swich-1)	10.15.99.50/24	🟡 Víctima
SERVIDOR-KALI-1	eth0 → e0 (Swich-1)	10.15.99.150/24	🟠 Servidor Web Falso
R2 (Cisco)	f0/0 → e0 (Swich-1)	10.15.99.1/24	🟢 Gateway / Router

Parámetro	Valor
Red	10.15.99.0/24
Máscara	255.255.255.0
Gateway	10.15.99.1
VLAN	1 (nativa)
Simulador	GNS3 + VMware Workstation

3. Objetivo y Funcionamiento de los Scripts
El laboratorio usa dos scripts Python que trabajan en conjunto:
dns_spoofing_lab.py — Script del Atacante
Objetivo: Posicionarse como MITM mediante ARP Poisoning e interceptar las consultas DNS de la víctima hacia itla.edu.do, respondiendo con la IP del servidor falso (10.15.99.150) antes de que llegue la respuesta legítima.
Parámetros
Parámetro	Valor en el Lab	Descripción
VICTIM_IP	10.15.99.50	IP de la máquina víctima
ROUTER_IP	10.15.99.1	IP del gateway (R2)
DNS_TARGET	itla.edu.do	Dominio a falsificar
FAKE_IP	10.15.99.150	IP del servidor web falso
IFACE	eth0	Interfaz de red del atacante
ARP_INTERVAL	2s	Frecuencia de envío de ARPs falsos
Requisitos
# Sistema: Kali Linux 2025.3 — ejecutar como root
pip3 install scapy

Cómo ejecutar
sudo python3 dns_spoofing_lab.py

fake_server.py — Servidor Web Señuelo
Objetivo: Levantar un servidor HTTP en 10.15.99.150:80 que sirva una página que simula el sitio de ITLA, confirmando visualmente que el ataque fue exitoso cuando la víctima lo visita.
Cómo ejecutar
# Ejecutar en SERVIDOR-KALI-1 ANTES de iniciar el ataque
sudo python3 fake_server.py

4. Demostración del Ataque
Paso 1 — Servidor Web Falso en Ejecución
El script fake_server.py se inicia en SERVIDOR-KALI-1 (10.15.99.150). El servidor queda escuchando en el puerto 80 y registra la petición de la víctima (10.15.99.50 → /) una vez que el ataque es exitoso.
 
Paso 2 — Script de Ataque en Ejecución (ARP Poisoning + DNS Spoof)
Desde la máquina atacante se ejecuta dns_spoofing_lab.py. El script resuelve las MACs, habilita IP forwarding, inicia el ARP Poisoning cada 2 segundos y queda escuchando las consultas DNS de la víctima. Los mensajes WARNING de Scapy sobre la dirección MAC Ethernet son normales y no afectan el funcionamiento.
[*] Resolviendo MAC de la víctima (10.15.99.50)...
    → 00:0c:29:a0:0a:51
[*] Resolviendo MAC del router (10.15.99.1)...
    → 00:50:56:c0:00:02
[*] IP forwarding habilitado.
[*] Iniciando ARP poisoning cada 2s...
[*] Objetivo DNS: itla.edu.do → 10.15.99.150
[*] Presiona Ctrl+C para detener y restaurar la red.
[*] Escuchando DNS queries de 10.15.99.50 (filtro: udp port 53 and src host 10.15.99.50)
  

Paso 3 — Ataque Exitoso (Navegador de la Víctima)
La víctima (Clonekali-1) accede a http://itla.edu.do en Firefox. En lugar del sitio legítimo, carga el servidor falso en 10.15.99.150:80. La barra de direcciones muestra Not Secure ya que el servidor falso no tiene certificado SSL. El timestamp confirma la hora del ataque.
 

5. Contramedidas
Contramedida 1 — DNS over HTTPS (DoH) en Firefox
Capa: Cliente/Navegador
Mecanismo: Cifra las consultas DNS dentro de tráfico HTTPS (puerto 443). El atacante MITM no puede ver ni modificar las consultas DNS en tránsito ya que viajan cifradas hacia el proveedor DoH (Cloudflare).
Configuración aplicada:
Firefox → about:preferences#privacy
→ Enable DNS over HTTPS using: Max Protection
→ Provider: Cloudflare (Default) — 1.1.1.1
Con Max Protection activo:
•	Firefox siempre usa DNS seguro cifrado.
•	Muestra advertencia de riesgo antes de usar el DNS del sistema.
•	Si el DNS seguro no está disponible, los sitios no cargan (no fallback inseguro).
 

Contramedida 2 — Dynamic ARP Inspection (DAI) en Switch Cisco
Capa: Infraestructura / Switch
Mecanismo: El switch valida cada paquete ARP contra la tabla de DHCP Snooping. Los paquetes ARP del atacante no coinciden con la tabla y son descartados automáticamente, bloqueando el ARP Poisoning desde la raíz y eliminando la posición MITM.
Comandos de configuración:
Switch(config)#ip dhcp snooping
Switch(config)#ip dhcp snooping vlan 1
Switch(config)#interface GigabitEthernet0/0
Switch(config-if)#ip dhcp snooping trust
Switch(config-if)#ip arp inspection vlan 1
Switch(config)#interface GigabitEthernet0/0
Switch(config-if)#ip arp inspection trust
Los logs del switch confirman el bloqueo de los ARPs del atacante:
%SW_DAI-4-DHCP_SNOOPING_DENY: 1 Invalid ARPs (Req) on Gi0/1, vlan 1.
([000c.29b0.f61c/10.15.99.100/0000.0000.0000/10.15.99.1/04:29:54 UTC Fri Jun 5 2026])

%SW_DAI-4-DHCP_SNOOPING_DENY: 1 Invalid ARPs (Req) on Gi0/2, vlan 1.
([000c.29a0.0a51/10.15.99.50/00:00:00:00:00:00/10.15.99.1/04:30:07 UTC Fri Jun 5 2026])
 
Tabla Resumen de Contramedidas
Contramedida	Capa	Mecanismo	Estado
DoH (DNS over HTTPS)	Cliente	Cifra consultas DNS sobre TLS/HTTPS	 Implementada
DAI + DHCP Snooping	Switch	Descarta ARP Spoofing en la infraestructura	 Implementada
DNSSEC	Servidor DNS	Firma criptográfica de registros DNS	 Recomendada
HSTS + HTTPS	Servidor Web	Fuerza TLS; rechaza certificados inválidos	Recomendada
ARP Estático	Endpoint	Entradas ARP fijas para el gateway	Recomendada



⚠️ Aviso Legal
Este repositorio es de uso exclusivamente educativo y fue desarrollado en un entorno de laboratorio controlado (GNS3 + VMware Workstation) como parte de la asignatura de Seguridad de Redes del ITLA.
Ninguna de las técnicas documentadas aquí debe utilizarse en redes reales sin autorización expresa.



