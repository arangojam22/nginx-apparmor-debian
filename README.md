# Confinamiento de Nginx con AppArmor en Debian

## Objetivo
Aplicar control de acceso obligatorio a Nginx con AppArmor: aprendizaje en complain, revisión de accesos y validación del servicio en enforce.

## Entorno
- Debian Trixie en VirtualBox.
- Nginx: instalación mostrada de la versión 1.26.3-3+deb13u1.
- AppArmor y apparmor-utils: 4.1.0-1.
- Sitio estático en /var/www/html, puerto 80.

## 1. Instalación
```bash
sudo apt update
sudo apt install -y nginx curl apparmor apparmor-utils auditd
```
Se actualizó el índice de paquetes y se instalaron el servidor web, el cliente HTTP y las herramientas de confinamiento y auditoría.

![Instalación](<evidencias/Captura de pantalla 2026-11-19 162259.png>)

## 2. Comprobación de AppArmor
```bash
sudo systemctl status apparmor
sudo aa-status
```
La salida muestra el servicio active (exited), ejecución correcta y el módulo cargado con 124 perfiles en ese momento. No es todavía una evidencia de confinamiento de Nginx.

![Estado inicial de AppArmor](<evidencias/Captura de pantalla 2026-11-19 162921.png>)

## 3. Servicio Nginx
```bash
sudo systemctl daemon-reload
sudo systemctl enable nginx
sudo systemctl status nginx --no-pager
```
Nginx aparece habilitado y active (running), con proceso principal y trabajador.

![Servicio Nginx](<evidencias/Captura de pantalla 2026-11-19 163338.png>)

## 4. Página de prueba
```bash
curl -I http://localhost
echo '<h1>Prueba AppArmor + Nginx</h1>' | sudo tee /var/www/html/index.html
curl http://localhost
```
Se obtuvo HTTP 200 OK y se comprobó la devolución del contenido de prueba. Esta es la comprobación inicial, previa al confinamiento.

![Página de prueba](<evidencias/Captura de pantalla 2026-11-19 163524.png>)

## 5. Configuración del sitio
Archivo /etc/nginx/sites-available/default:
```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    root /var/www/html;
    index index.html index.htm;
    server_name _;
    location / {
        try_files $uri $uri/ =404;
    }
}
```
El sitio escucha por IPv4 e IPv6 y publica archivos de /var/www/html. Se corrigió un error previo de contexto de server_name.

![Configuración del sitio](<evidencias/Captura de pantalla 2026-11-19 164013.png>)

```bash
sudo nginx -t
sudo systemctl reload nginx
curl -I http://localhost
```
La validación terminó con syntax is ok y test is successful. Después de recargar se obtuvo HTTP 200 OK.

![Validación de configuración](<evidencias/Captura de pantalla 2026-11-19 164136.png>)

## 6. Creación y aprendizaje del perfil
```bash
sudo aa-genprof /usr/sbin/nginx
```
Mientras el asistente mantenía el perfil en aprendizaje se probaron, desde otra terminal, el reinicio, las peticiones y la recarga:
```bash
sudo systemctl restart nginx
curl http://localhost
curl -I http://localhost
sudo systemctl reload nginx
```
Se utilizó Scan para revisar eventos y Save Changes para guardar reglas. Finish finalizó el asistente y activó enforce.

### Selección inicial de capacidades
La captura muestra la aceptación de dac_override y de las abstracciones nis, dovecot-common y postfix-common. No muestra todavía su sustitución por capacidades individuales.

![Selección inicial de permisos](<evidencias/Captura de pantalla 2026-11-19 165122.png>)

### Accesos a archivos
Se añadieron lecturas de boot_id y /run/systemd/userdb/. Para access.log se pasó de una regla de escritura con owner a otra sin esa restricción. Esta ampliación debe valorarse según los usuarios que escriben el registro.

![Ajuste de reglas de archivos](<evidencias/Captura de pantalla 2026-11-19 165129.png>)

## 7. Registros en complain
Los registros se localizaron en /var/log/audit/audit.log:
```bash
sudo grep 'profile="/usr/sbin/nginx"' /var/log/audit/audit.log
```
Extractos de campos reales obtenidos durante la práctica; se omiten los restantes campos de auditoría:
```text
apparmor="ALLOWED" operation="open" profile="/usr/sbin/nginx" name="/etc/nginx/nginx.conf" requested_mask="r" denied_mask="r"
apparmor="ALLOWED" operation="open" profile="/usr/sbin/nginx" name="/var/www/html/index.html" requested_mask="r" denied_mask="r"
apparmor="ALLOWED" operation="file_perm" profile="/usr/sbin/nginx" name="/var/log/nginx/access.log" requested_mask="w" denied_mask="w"
apparmor="ALLOWED" operation="listen" profile="/usr/sbin/nginx" lport=80 family="inet" sock_type="stream" requested_mask="listen" denied_mask="listen"
```
Son accesos permitidos durante el aprendizaje aunque el perfil aún no los autorizaba. denied_mask no significa que se bloquearan en estos eventos ALLOWED. Se identificaron lectura de configuración y contenido, escritura de logs y escucha de red. El archivo original completo queda pendiente de adjuntar.

## 8. Perfil generado y ajuste
```bash
sudo cat /etc/apparmor.d/usr.sbin.nginx
```
Esta captura documenta el perfil generado antes de la simplificación: aún incluye abstracciones de Dovecot, NIS y Postfix. No se presenta como evidencia del contenido final ajustado.

![Perfil antes de la simplificación](<evidencias/Captura de pantalla 2026-11-19 165445.png>)

Tras revisar esas abstracciones se propuso sustituir sus inclusiones por:
```apparmor
capability net_bind_service,
capability setgid,
capability setuid,
```
Se conservó provisionalmente dac_override. El contenido exacto del perfil final debe adjuntarse en perfiles/usr.sbin.nginx para documentar las reglas efectivamente utilizadas.

### Incidencia: copia de seguridad duplicada
Una copia usr.sbin.nginx.bak dentro de /etc/apparmor.d provocó un conflicto por definir dos veces el perfil. Se indicó moverla fuera del directorio:
```bash
sudo mkdir -p /root/apparmor-backups
sudo mv /etc/apparmor.d/usr.sbin.nginx.bak /root/apparmor-backups/usr.sbin.nginx.bak
```
La activación posterior de enforce terminó correctamente, como muestra la siguiente evidencia.

## 9. Enforce y comprobación final
```bash
sudo aa-enforce /usr/sbin/nginx
sudo systemctl restart nginx
sudo systemctl status nginx --no-pager -l
curl -I http://localhost
```
La salida confirma Setting /usr/sbin/nginx to enforce mode, reinicio correcto, estado active (running) y HTTP/1.1 200 OK. Así se comprueba que el servicio continúa respondiendo después de activar el confinamiento.

![Activación enforce y respuesta HTTP](<evidencias/Captura de pantalla 2026-11-19 174249.png>)

```bash
sudo aa-status
```
La segunda captura muestra el módulo cargado, 125 perfiles y 24 en enforce, pero está recortada antes de la línea de Nginx. Complementa la captura anterior; no demuestra por sí sola el modo específico de Nginx.

![Resumen de AppArmor](<evidencias/Captura de pantalla 2026-11-19 174303.png>)

## 10. Diferencias entre modos
| Modo | Comportamiento | Uso |
|---|---|---|
| complain | Registra infracciones sin bloquearlas; las reglas deny explícitas sí se aplican. | Aprendizaje y ajuste. |
| enforce | Aplica la política y bloquea accesos no autorizados. | Confinamiento efectivo. |

## 11. Reflexión
Fue necesario autorizar la lectura de la configuración y del contenido web, la escritura de registros y la gestión del archivo PID. También se observaron operaciones de red para atender HTTP y capacidades setuid/setgid relacionadas con la identidad de los trabajadores. net_bind_service permite utilizar puertos privilegiados como el 80.

Las abstracciones facilitan generar el perfil, pero pueden aportar permisos ajenos al servicio; por eso se revisaron las de Dovecot, NIS y Postfix. dac_override es una capacidad amplia y su aparición en los registros no demuestra que sea imprescindible: queda pendiente comprobar si puede retirarse corrigiendo propietarios o permisos tradicionales.

La respuesta HTTP 200 acredita el funcionamiento probado, no una auditoría completa ni que la política sea mínima.


