\# Segundo Parcial: Servicios Telemáticos



\*\*Estudiante:\*\* José Alejandro Castillo Arce · \*\*Código:\*\* 2230181

\*\*Universidad Autónoma de Occidente · Facultad de Ingeniería\*\*



Implementación de transferencia segura de archivos (FTPS y SFTP) protegida por firewall UFW con reenvío de puertos (NAT), y DNS sobre TLS (DoT) con verificación en Wireshark.



\## Topología



| VM | Hostname | Red interna (`srv-link`, intnet) | Red "pública" (host-only) |

|---|---|---|---|

| Servidor 1 (UFW/NAT) | `srv1-2230181` | `192.168.50.3` | `192.168.60.3` |

| Servidor 2 (vsftpd/SFTP) | `srv2-2230181` | `192.168.50.2` | sin acceso directo |

| Cliente Linux | `cli-2230181` | no | `192.168.60.10` |

| Host Windows (WinSCP, Wireshark) | no | no | `192.168.60.1` |



\*\*Nota sobre `public\_network`:\*\* el bridging no fue viable sobre Wi-Fi (el punto de acceso descarta tramas con una MAC distinta a la del adaptador) ni sobre un adaptador host-only. Se usó `private\_network` para la red pública y una red interna `intnet` para el enlace srv1↔srv2, que aísla realmente a srv2 del cliente y del host.



\## Estructura del repositorio



| Ruta | Contenido |

|---|---|

| `Vagrantfile` | Definición de las 3 VMs |

| `configs/srv1/` | `before.rules` (NAT/DNAT), `default-ufw` (`/etc/default/ufw`), `sysctl.conf` |

| `configs/srv2/` | `vsftpd.conf`, `sshd\_config`, ruta estática (netplan), `ca.crt`, `servidor.crt` |

| `configs/cli/` | `resolved.conf` (DoT) |

| `evidencias/texto/` | Salidas de `ufw status`, `iptables`, `resolvectl`, `openssl s\_client` y huellas |

| `evidencias/capturas-pcap/` | FTP sin cifrar, DoT, DNS plano, SFTP |



Las claves privadas (`\*.key`) no se incluyen en el repositorio por seguridad.



\## Primera parte: FTPS + UFW



| # | Requisito | Evidencia |

|---|---|---|

| 1 | UFW deny incoming/routed, ip\_forward, DNAT, srv2 inalcanzable | `configs/srv1/before.rules`, `srv1\_ufw\_status\_verbose.txt` |

| 2 | Solo 22/tcp, route 21/tcp y 50000:50010/tcp | `srv1\_ufw\_status\_numbered.txt` |

| 3 | Fallo sin regla de reenvío, éxito con ella | Pruebas con `Test-NetConnection` |

| 4 | `ufw status verbose` y reglas NAT | `srv1\_iptables\_nat.txt` |

| 5 | vsftpd con TLS explícito, cifrado obligatorio, SSLv2/3 off | `configs/srv2/vsftpd.conf` |

| 6 | `pasv\_min/max\_port` y `pasv\_address=192.168.60.3` | `configs/srv2/vsftpd.conf` |

| 7 | WinSCP, huella SHA-256 coincide con `openssl` | `srv2\_huella\_sha256.txt`, `winscp\_huella\_sha256.txt` |

| 8 | `openssl s\_client`: `Verify return code: 0 (ok)`, TLS 1.3 | `cli\_openssl\_s\_client.txt` |

| 9 | Wireshark: FTP en claro (USER/PASS) vs FTPS (`AUTH TLS`, Client Hello, Application Data) | `ftp\_sin\_cifrar.pcapng` y capturas de pantalla de Wireshark |



\*\*`ufw allow` vs `ufw route allow`:\*\* `allow` controla el tráfico destinado a la propia máquina (cadena INPUT). `route allow` controla el tráfico que la atraviesa hacia otro destino (cadena FORWARD). Con `default deny routed`, nada se reenvía sin una regla `route` explícita.



\*\*`pasv\_address` detrás de NAT:\*\* el servidor debe anunciar la IP pública de srv1, no la interna de srv2. Si no se configura, el cliente recibe una IP inalcanzable para el canal de datos y la transferencia falla aunque el control funcione.



\## Segunda parte: DNS sobre TLS



| # | Requisito | Evidencia |

|---|---|---|

| 10 | `DNS=1.1.1.1#cloudflare-dns.com 8.8.8.8#dns.google`, `DNSOverTLS=yes`, `FallbackDNS`, stub 127.0.0.53 | `configs/cli/resolved.conf` |

| 11 | `+DNSOverTLS` en `resolvectl status` | `cli\_resolvectl\_status.txt` |

| 12 | 3 dominios con `resolvectl query` (transporte cifrado) | Salida de `resolvectl query` |

| 13 | TCP/853 cifrado vs UDP/53 en claro | `dot\_capture.pcap`, `plain\_dns\_capture\_new.pcap` |

| 14 | Visibilidad con DoT, bloqueo del 853, DoT vs DoH | Sección siguiente |



\- \*\*`yes` vs `opportunistic`:\*\* con `yes` DoT es obligatorio y, si falla, no hay resolución. Con `opportunistic` se degrada a DNS plano sin avisar, lo que abre la puerta a un ataque de downgrade.

\- \*\*Visible con DoT activo:\*\* IP del resolver, puerto 853, SNI (`cloudflare-dns.com`), tamaños y tiempos. No es visible el dominio consultado ni la respuesta.

\- \*\*Si un firewall bloquea el 853:\*\* con `yes` la resolución falla por completo; con `opportunistic` cae a DNS plano por el puerto 53.

\- \*\*DoT vs DoH:\*\* DoT usa un puerto dedicado (853), fácil de identificar y bloquear. DoH viaja por 443, mezclado con el tráfico web.

\- \*\*`dig @8.8.8.8`:\*\* omite `systemd-resolved`, que es quien implementa DoT, por eso la consulta viaja en texto plano.



\## Tercera parte: SFTP + UFW



| # | Requisito | Evidencia |

|---|---|---|

| 15 | Usuario `sftp\_2230181` con `Match User`, `ChrootDirectory`, `ForceCommand internal-sftp`; SSH rechazado | `configs/srv2/sshd\_config` |

| 16 | DNAT 2222→192.168.50.2:22 y `ufw route allow`; fallo sin la regla | `configs/srv1/before.rules` |

| 17 | `sftp -P 2222`, huella de host ED25519, `ls`/`put`/`get` | Salida de la sesión

