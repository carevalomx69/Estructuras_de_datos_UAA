# Mantenimiento de seguridad del servidor Oracle (octubre 2026)

Fecha: 24 de octubre de 2026
Servidor: instancia Oracle Cloud Always Free (Ampere A1, ARM), IP 140.84.191.7
Responsable técnico: Dr. Carlos A. Arévalo Mercado

## 1. Motivo

El 21 de octubre de 2026 se publicaron parches para dos fallas críticas de ejecución remota de código en PHP (CVE-2026-109618 y CVE-2026-109619, conocidas como PHP2Shell). Basta una petición sin autenticación para explotarlas, y Apache con mod_php es la configuración de mayor riesgo. Se documentó un intento de explotación el 17 de octubre, antes de la divulgación pública [1]. Las versiones corregidas son 8.2.35, 8.3.36, 8.4.27 y 8.5.12. Para PHP 7.4, 8.0 y 8.1 solo hay retroportes mínimos en código fuente, y no existe corrección para 7.1 [1].

Oracle protege su infraestructura. El PHP que corre dentro de los contenedores es responsabilidad del titular de la instancia.

## 2. Estado de las aplicaciones tras el mantenimiento

| Aplicación | Contenedores | PHP | Estado |
|---|---|---|---|
| Simulador Financiero NÜKO | apache_calculadora, mysql_calculadora | 8.2.35 (corregido) | En producción en http://140.84.191.7/simulador/ |
| Listas ligadas (Laravel 5.8) | listas_app (apagado), listas_db, listas_adminer, listas_nginx | 7.1.33 (sin parche posible) | Fuera de línea temporalmente. La ruta raíz responde 503 |

Nota: listas_nginx es el Nginx del puerto 80 y atiende ambas aplicaciones.

## 3. Cambios realizados

### 3.1 Calculadora (carpeta ~/SimuladorFinanciero)
- Imagen reconstruida con `docker compose build --pull web` y `docker compose up -d web`. Verificado con `docker exec apache_calculadora php -v` (8.2.35) y con la cabecera `X-Powered-By: PHP/8.2.35`.
- Antes de ello se desplegó la versión NÜKO (PR #2) y el PR #3, que dejó en el repositorio las rutas de API relativas y el WebSocket por `/simulador-ws/`. Las reglas quedaron en `CLAUDE.md`.

### 3.2 Listas ligadas (carpeta ~/Estructuras_de_datos_UAA)
1. `docker/nginx.conf`: la ruta `/` devuelve 503 con mensaje de mantenimiento. Se eliminó el bloque `\.php$` con `fastcgi_pass app:9000`, de modo que Nginx ya no depende de listas_app. Los bloques `/simulador/` y `/simulador-ws/` quedaron idénticos.
2. `docker-compose.yml`: el servicio `webserver` ahora declara las redes `default` y `mi_red` (externa, `simuladorfinanciero_mi_red`). Antes la conexión a esa red se había hecho a mano y se perdía al recrear el contenedor.
3. Se reconstruyó la imagen del servicio webserver y se aplicó con `docker compose up -d --no-deps webserver`.
4. Se apagó la aplicación con `docker stop listas_app`. Su política es `unless-stopped`, por lo que sigue apagada tras reiniciar el servidor.
5. Prueba: `docker restart listas_nginx` con listas_app apagada, la calculadora siguió respondiendo 200.

### 3.3 Respaldos creados
- ~/respaldos/respaldo_20261024_0127.sql (base de la calculadora)
- ~/respaldos/nginx.conf.bak (configuración anterior de Nginx)
- ~/respaldos/docker-compose.listas.yml.bak (compose anterior)
- Imagen Docker etiquetada `listas_webserver_antes` (Nginx anterior)
- ~/app.js.servidor.bak (parche manual anterior de js/app.js)

## 4. Cómo revertir o reactivar

Volver a la configuración anterior de Nginx:

```bash
cd ~/Estructuras_de_datos_UAA
cp ~/respaldos/nginx.conf.bak docker/nginx.conf
docker compose build webserver
docker compose up -d --no-deps webserver
docker start listas_app
```

Atención: reactivar la app la deja expuesta a la vulnerabilidad. No hacerlo sin una mitigación adicional (por ejemplo, autenticación básica en Nginx o restricción de acceso).

Si la imagen nueva de Nginx falla, usar la anterior:

```bash
docker tag listas_webserver_antes estructuras_de_datos_uaa-webserver
docker compose up -d --no-deps webserver
```

## 5. Reglas para no romper el servidor compartido
- No ejecutar `docker compose down` en la carpeta de las listas, porque cae el Nginx de ambas aplicaciones. No usar `down -v` en ningún caso.
- Nginx resuelve `apache_calculadora` al arrancar. Si ese contenedor no existe o no está en la red `simuladorfinanciero_mi_red`, Nginx no inicia y caen ambas aplicaciones. Mejora pendiente: usar el resolver de Docker con una variable en `proxy_pass`.
- Dockerfile, docker-compose.yml y docker/ de las listas **no están en git** (aparecen sin seguimiento). Hoy solo existen en este servidor.

## 6. Pendientes de seguridad
1. Respaldar fuera del servidor los archivos sin seguimiento (Dockerfile, docker-compose.yml, docker/).
2. Mover las contraseñas del compose a un archivo `.env` fuera de git, y después versionar los archivos de despliegue.
3. Publicar la base (3306) y Adminer (8080) solo en `127.0.0.1`. Hoy están bloqueados desde fuera por Oracle, lo cual se comprobó el 24 de octubre desde una máquina externa, pero no por la configuración del compose.
4. Rotar las credenciales de las bases de datos.
5. Ascender el usuario de Carlos a administrador en la calculadora. La tabla `usuarios` ya tiene la columna `rol` (gratuito, de_pago, administrador), pero no existe módulo de administración.
6. Dominio con HTTPS e IP reservada.

## 7. Plan para la app de listas ligadas

La aplicación usa Laravel 5.8 sobre PHP 7.1 con frontend en Node 14, todo fuera de soporte. Opciones evaluadas:
- Portar el código a un proyecto Laravel actual (modelos, controladores, vistas).
- Reescribir en un stack soportado.
- Retirarla.

Meta de reactivación: finales de enero o principios de febrero de 2027. La versión nueva deberá incluir un módulo de administración para crear cuentas y monitorear el uso por parte de estudiantes. Antes de estimar el esfuerzo conviene contar rutas, controladores y vistas del repositorio.

## 8. Texto sugerido para informes (SNII, PRODEP, UAA)

"La aplicación de listas ligadas se encuentra temporalmente fuera de línea por una actualización de seguridad, ya que su pila tecnológica (PHP 7.1 y Laravel 5.8) quedó fuera de soporte. Se encuentra en migración a un stack actual con módulo de administración, con reactivación prevista para el semestre de enero a junio de 2027. El código, la documentación y la base de datos se conservan."

## Referencias

[1] Help Net Security. (21 de octubre de 2026). PHP2Shell: Critical PHP vulnerabilities patched, update now! https://www.helpnetsecurity.com/2026/10/21/php2shell-php-vulnerabilities-patched-cve-2026-109618-cve-2026-109619/
