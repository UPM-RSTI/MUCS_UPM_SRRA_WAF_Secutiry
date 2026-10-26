# =============================================================================
#  Contenedor "puesto del analista": JupyterLab con kernel BASH
# =============================================================================
#  Este contenedor es el que ejecuta el notebook del laboratorio. Lleva:
#
#    - JupyterLab + el kernel de bash (bash_kernel): cada celda del notebook es
#      una terminal bash real, con estado persistente entre celdas.
#    - El cliente de Docker (solo el binario): junto con el socket del host que
#      se monta en docker-compose.yml, permite "docker exec" contra waf-proxy.
#    - curl, jq, iproute2: para lanzar peticiones y analizar el log JSON.
#
#  No se instala Python-para-analisis (pandas, etc.) a proposito: el objetivo es
#  trabajar "a mano", con herramientas de consola, no con librerias.
# =============================================================================

FROM python:3.11-slim

# --- Herramientas de sistema -------------------------------------------------
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
        bash \
        curl \
        jq \
        iproute2 \
        ca-certificates \
 && rm -rf /var/lib/apt/lists/*

# --- JupyterLab + kernel de bash ---------------------------------------------
# bash_kernel convierte cada celda en una sesion bash persistente: lo que
# defines en una celda (variables, funciones) sigue disponible en la siguiente.
RUN pip install --no-cache-dir jupyterlab bash_kernel \
 && python -m bash_kernel.install

# --- Cliente de Docker -------------------------------------------------------
# Se copia solo el binario del cliente desde la imagen oficial "docker:cli"
# (multi-arquitectura), sin instalar el demonio. Hablara con el Docker del host
# a traves del socket montado en docker-compose.yml.
COPY --from=docker:cli /usr/local/bin/docker /usr/local/bin/docker

WORKDIR /workspace

EXPOSE 8888

# Jupyter sin token ni contrasena: es un laboratorio local de un solo usuario.
CMD ["jupyter", "lab", "--ip=0.0.0.0", "--port=8888", "--allow-root", \
     "--IdentityProvider.token=", "--ServerApp.password="]
