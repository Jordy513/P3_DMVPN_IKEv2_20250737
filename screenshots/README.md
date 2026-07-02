# Capturas de pantalla — VPN Hub-and-Spoke DMVPN Fase 3 con IKEv2

Capturas del laboratorio en orden de demostración.

| # | Archivo de Evidencia | Descripción Técnica Detallada |
|---|---|---|
| 1 | [`01_topologia.png`](/screenshots/01_topologia.png) | Topología funcional en PNETLab con nombre completo y matrícula (`20250737`) visibles, los tres routers encendidos. |
| 2 | [`02_config_hub_fase3.png`](/screenshots/02_config_hub_fase3.png) | Consola de R1 mostrando `ip nhrp redirect`, `ip ospf network broadcast`, `ip ospf priority 2` y `set ikev2-profile` en el IPSec Profile. |
| 3 | [`03_config_spoke_fase3.png`](/screenshots/03_config_spoke_fase3.png) | Consola de R2 mostrando `ip nhrp shortcut`, `ip ospf network broadcast` e `ip ospf priority 0`. |
| 4 | [`04_show_dmvpn.png`](/screenshots/04_show_dmvpn.png) | Salida de `show dmvpn` en R1 mostrando ambos Spokes con estado `UP` y `Attrb: D`. |
| 5 | [`05_ikev2_sa_ready.png`](/screenshots/05_ikev2_sa_ready.png) | Salida de `show crypto ikev2 sa` en R1 mostrando dos SAs en estado `READY`. |
| 6 | [`06_ospf_neighbors.png`](/screenshots/06_ospf_neighbors.png) | Salida de `show ip ospf neighbor` en R1 mostrando los Spokes como `FULL/DROTHER`. |
| 7 | [`07_ping_spoke_spoke.png`](/screenshots/07_ping_spoke_spoke.png) | Ping exitoso desde R2 (`source 20.25.37.65`) hacia PC3 (`20.25.37.130`) con `repeat 10`. |
