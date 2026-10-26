# =============================================================================
#  WAF del laboratorio: nginx + ModSecurity v3 + OWASP Core Rule Set
# =============================================================================
#
#  Este Dockerfile NO usa la imagen ya preparada "owasp/modsecurity-crs:nginx".
#  Se instala todo a mano, paso a paso, porque el objetivo de la practica es
#  DIDACTICO: cada linea de aqui es exactamente el comando que escribirias en
#  una consola si estuvieses montando este WAF sobre un servidor Debian real.
#
#  Tampoco se usa NINGUNA variable de entorno para configurar el WAF: toda la
#  configuracion vive en ficheros .conf explicitos (ver waf/*.conf), igual que
#  en una instalacion tradicional de sistema.
#
#  Mapa de lo que queda instalado dentro del contenedor:
#
#    /etc/nginx/nginx.conf                         <- config principal (del paquete)
#    /etc/nginx/modules-enabled/..modsecurity..    <- carga del modulo (del paquete)
#    /etc/nginx/sites-available/default            <- NUESTRO virtual host (proxy + WAF)
#    /etc/nginx/modsec/main.conf                   <- NUESTRO orden de carga
#    /etc/nginx/modsec/modsecurity.conf            <- NUESTRO motor ModSecurity
#    /etc/nginx/modsec/tuning-local.conf           <- NUESTROS ajustes locales
#    /etc/nginx/modsec/coreruleset/                <- OWASP CRS descargado
#    /var/log/modsec/audit.log                     <- log de auditoria (JSON)
#
# =============================================================================

FROM debian:bookworm-slim

# -----------------------------------------------------------------------------
# PASO 1. Paquetes del sistema
# -----------------------------------------------------------------------------
# "libnginx-mod-http-modsecurity" es el conector de ModSecurity v3 para nginx,
# compilado por Debian contra EL MISMO nginx del repositorio: por eso instalamos
# el nginx de Debian y no el de nginx.org (mezclarlos rompe la ABI del modulo).
#
# El paquete deja el fichero que carga el modulo en /etc/nginx/modules-enabled/,
# asi que nginx ya arranca con ModSecurity disponible sin tocar nada mas.
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
        nginx \
        libnginx-mod-http-modsecurity \
        ca-certificates \
        curl \
        procps \
        iproute2 \
        less \
 && rm -rf /var/lib/apt/lists/*

# -----------------------------------------------------------------------------
# PASO 2. OWASP Core Rule Set (CRS)
# -----------------------------------------------------------------------------
# CRS no es un WAF: es el CONJUNTO DE REGLAS que el motor (ModSecurity) ejecuta.
# Lo descargamos con una version fijada, para que la practica sea reproducible y
# para que se vea que las reglas son simples ficheros de texto en el disco.
#
# Despues de este paso tendras, dentro del contenedor:
#    /etc/nginx/modsec/coreruleset/crs-setup.conf     (configuracion de CRS)
#    /etc/nginx/modsec/coreruleset/rules/*.conf       (las reglas: 9xx, 93x, 94x...)
RUN mkdir -p /etc/nginx/modsec \
 && curl -fsSL \
      https://github.com/coreruleset/coreruleset/archive/refs/tags/v4.0.0.tar.gz \
      -o /tmp/crs.tar.gz \
 && tar -xzf /tmp/crs.tar.gz -C /etc/nginx/modsec \
 && mv /etc/nginx/modsec/coreruleset-4.0.0 /etc/nginx/modsec/coreruleset \
 && cp /etc/nginx/modsec/coreruleset/crs-setup.conf.example \
       /etc/nginx/modsec/coreruleset/crs-setup.conf \
 && rm -f /tmp/crs.tar.gz

# -----------------------------------------------------------------------------
# PASO 3. Nuestra configuracion de ModSecurity (explicita, sin variables)
# -----------------------------------------------------------------------------
COPY modsecurity.conf   /etc/nginx/modsec/modsecurity.conf
COPY main.conf          /etc/nginx/modsec/main.conf
COPY tuning-local.conf  /etc/nginx/modsec/tuning-local.conf

# -----------------------------------------------------------------------------
# PASO 4. Virtual host de nginx: proxy inverso hacia Juice Shop + WAF activado
# -----------------------------------------------------------------------------
COPY nginx-waf.conf     /etc/nginx/sites-available/default

# -----------------------------------------------------------------------------
# PASO 5. Directorio del log de auditoria
# -----------------------------------------------------------------------------
# En el arranque montaremos aqui una carpeta del host (bind mount) para poder
# leer el log desde fuera con cat/grep/jq, sin entrar al contenedor.
RUN mkdir -p /var/log/modsec

EXPOSE 80

# nginx en primer plano: el PID 1 del contenedor es el propio servidor.
CMD ["nginx", "-g", "daemon off;"]
