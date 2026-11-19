# Confinamiento de Nginx con AppArmor en Debian

## Objetivo
Configurar Nginx y crear un perfil de AppArmor mediante aprendizaje en complain, ajustar sus permisos y comprobar el funcionamiento en enforce.

## Entorno
- Debian en una máquina virtual de VirtualBox.
- Nginx, AppArmor y herramientas apparmor-utils.
- Versiones exactas: pendientes de incorporar.

## Estructura
- `configuracion/`: configuración del sitio Nginx.
- `perfiles/`: perfil final de AppArmor.
- `evidencias/`: registros y salidas de las comprobaciones.
- `capturas/`: capturas del proceso, pendientes de subir y enlazar.

## Configuración inicial
Comandos utilizados durante la práctica:

```bash
sudo apt update
sudo apt install -y nginx curl apparmor apparmor-utils auditd
sudo nano /etc/nginx/sites-available/default
sudo nginx -t
sudo systemctl enable --now nginx
curl -I http://localhost
```

Se corrigió un error de contexto de la directiva server_name antes de continuar.

## Creación y ajuste del perfil

```bash
sudo aa-genprof /usr/sbin/nginx
```

Desde otra terminal se reinició Nginx, se realizaron peticiones con curl y se probó su recarga. Se analizaron los eventos con Scan, se revisaron las reglas y se guardaron los cambios. Finish completó el asistente y activó enforce.

Se identificaron inclusiones de Dovecot, NIS y Postfix. Se indicó sustituirlas por las capacidades concretas net_bind_service, setgid y setuid. El archivo final y las evidencias posteriores a este ajuste están pendientes de incorporar al repositorio.

## Registros del aprendizaje
Los eventos se localizaron en `/var/log/audit/audit.log`; la consulta inicial al journal del kernel no devolvió resultados.

```bash
sudo grep 'profile="/usr/sbin/nginx"' /var/log/audit/audit.log
```

Los registros aportados durante la práctica incluyen eventos ALLOWED de lectura de la configuración y de `/var/www/html/index.html`, escritura de logs, gestión de `/run/nginx.pid`, operaciones de red en el puerto 80 y solicitudes de capacidades. El registro original está pendiente de subir a `evidencias/`.

## Validación

```bash
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx
sudo aa-enforce /usr/sbin/nginx
sudo systemctl restart nginx
sudo systemctl status nginx --no-pager -l
curl -I http://localhost
sudo aa-status
```

La salida compartida antes de la simplificación final mostró el perfil y los procesos de Nginx en enforce, el servicio activo y HTTP/1.1 200 OK. Deben adjuntarse las comprobaciones realizadas tras el último ajuste del perfil.

## Complain y enforce
En complain se registran los accesos que el perfil no autoriza sin bloquearlos, salvo las denegaciones explícitas. En enforce se aplica la política y se bloquean los accesos no autorizados.

## Reflexión
Nginx necesitó leer su configuración y el contenido web, escribir registros, gestionar su PID y utilizar la red. Las capacidades setuid y setgid permiten ajustar las identidades de los trabajadores; net_bind_service permite utilizar puertos privilegiados. Se propuso reemplazar abstracciones ajenas al servicio por capacidades concretas. La capacidad dac_override se conservó provisionalmente y requiere revisar los permisos tradicionales antes de justificar su necesidad definitiva.

## Documentación pendiente
- Incorporar versiones exactas.
- Subir la configuración y el perfil final reales.
- Subir los registros y las comprobaciones finales.
- Subir las capturas y enlazarlas en cada apartado.
