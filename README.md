# WAF con OWASP Juice Shop — Las reglas del OWASP CRS y cómo escribir las tuyas

Laboratorio práctico (SERRC · UPM). Despliega **OWASP Juice Shop** detrás de un
**WAF (nginx + ModSecurity v3 + OWASP Core Rule Set 4.0.0)** y lo analiza desde un
**notebook Jupyter con celdas bash** que corre **dentro de la propia red Docker**.

En la **Parte 1** lanzas ataques reales (SQLi, XSS, Command Injection, Path
Traversal, CSRF) y **lees en el log** qué reglas del CRS saltan, con qué severidad
y cómo suman el *anomaly score*. En la **Parte 2** **escribes tus propias reglas**
en `waf/tuning-local.conf`, las cargas en caliente y las ves actuar.

> **Solo para uso local y formativo.** Juice Shop es una aplicación
> deliberadamente vulnerable: no la expongas a internet ni ataques sistemas que no
> sean los de este laboratorio.

---

## Arquitectura

```
   Tu ordenador (navegador) ──▶ localhost:3000  (Juice Shop SIN WAF)
                                localhost:8080  (Juice Shop CON WAF)
                                localhost:8888  (JupyterLab)

   ┌───────────────────────── red Docker: waf-net ─────────────────────────┐
   │                                                                        │
   │   ┌───────────────┐   http://waf:80      ┌──────────────────────┐      │
   │   │  waf-jupyter  │ ───────────────────▶ │      waf-proxy       │      │
   │   │  (notebook,   │   docker exec        │ nginx + ModSecurity  │      │
   │   │  celdas bash) │ ───────────────────▶ │    + OWASP CRS       │      │
   │   └──────┬────────┘                      └──────────┬───────────┘      │
   │          │ http://juice-shop:3000                   │ proxy inverso    │
   │          │                                          ▼                  │
   │          │                               ┌──────────────────────┐      │
   │          └─────────────────────────────▶ │      juice-shop      │      │
   │                                          │ (backend vulnerable) │      │
   │                                          └──────────────────────┘      │
   └────────────────────────────────────────────────────────────────────────┘
```

---

## Requisitos

- **Docker** con **Docker Compose v2** (Docker Desktop en macOS/Windows, o Docker
  Engine en Linux), en marcha.
- Unos **4 GB de RAM** libres para Docker y ~2 GB de disco.
- **Internet la primera vez**: se descarga la imagen de Juice Shop y, al construir
  el WAF, las reglas de OWASP CRS desde GitHub.
- Puertos libres en tu ordenador: **3000**, **8080** y **8888**.

---

## Puesta en marcha

1. Descarga el repositorio: botón verde **Code → Download ZIP** (y descomprímelo)
   o, si usas git:

   ```bash
   git clone https://github.com/<usuario>/<repositorio>.git
   cd <repositorio>
   ```

2. Desde la carpeta del repositorio, en una terminal:

   ```bash
   docker compose up -d --build
   ```

   La primera vez tarda unos minutos (descarga Juice Shop, construye el WAF,
   instala Jupyter). Comprueba que los tres contenedores están `Up`:

   ```bash
   docker compose ps
   ```

3. Abre **http://localhost:8888** en el navegador (JupyterLab, sin contraseña).
4. Abre el notebook **`WAF_JuiceShop_Reglas.ipynb`** y ejecuta las celdas **en
   orden**, leyendo cada explicación antes de ejecutar.

Para comprobar que la aplicación responde: http://localhost:3000 (sin WAF) y
http://localhost:8080 (con WAF) deben mostrar la tienda de Juice Shop.

---

## Contenido del notebook

| Sección | Qué haces |
|---|---|
| Preparación | Compruebas los contenedores, la red y el log de auditoría. Opcional: vaciar el log. |
| **1.1** SQL Injection (`942xxx`) | Rompes la búsqueda, extraes datos con `UNION` y lees el log del WAF. |
| **1.2** XSS (`941xxx`) | XSS reflejado bloqueado por el WAF y un XSS de DOM ejecutándose en el navegador. |
| **1.3** Command Injection (`932xxx`) | Qué quiere robar el atacante (`/etc/passwd`) y cómo lo corta el WAF. |
| **1.4** Path Traversal / LFI (`930xxx`) | Travesía de directorios: quién responde 400 y quién 403. |
| **1.5** Anomaly scoring | Cómo suman los puntos por severidad y quién decide el 403 (`949110`). |
| **1.6** Paranoia Level | Subir el PL activa más reglas… y más falsos positivos. |
| **1.7** CSRF | Por qué el CRS no trae reglas para esto. |
| **2.A** Regla A | Un marcador propio atado a un Paranoia Level (`chain`). |
| **2.B** Regla B | Límite de peticiones por IP → **429**. |
| **2.C** Regla C | Bloquear la subida de ficheros con extensión peligrosa (`FILES`, fase 2). |
| **2.Z** Cierre | Restaurar `tuning-local.conf` y dejar el laboratorio limpio. |

Cada URL del notebook aparece en sus dos formas: la que usa la celda dentro de
Docker (`http://waf:80/rest/products/search?q=limon`) y la que puedes pegar en tu
navegador (`http://localhost:8080/rest/products/search?q=limon`).

---

## Estructura del repositorio

| Ruta | Qué es |
|---|---|
| `WAF_JuiceShop_Reglas.ipynb` | **El laboratorio.** Notebook de celdas bash. |
| `docker-compose.yml` | Levanta los 3 contenedores en la red `waf-net`. |
| `jupyter/Dockerfile` | Puesto del analista: JupyterLab + kernel bash + cliente docker + curl/jq. |
| `waf/Dockerfile` | Construye el WAF a mano: nginx + ModSecurity + descarga de OWASP CRS 4.0.0. |
| `waf/modsecurity.conf` | Configuración del motor ModSecurity (log de auditoría en JSON). |
| `waf/main.conf` | Orden de carga: motor → `crs-setup.conf` → reglas CRS → `tuning-local.conf`. |
| `waf/tuning-local.conf` | **Tus reglas y ajustes**, cargados al final. Se edita desde el notebook. |
| `waf/nginx-waf.conf` | Virtual host de nginx: activa el WAF y hace de proxy hacia Juice Shop. |
| `waf-audit-log/` | Aquí escribe el WAF su log de auditoría (`audit.log`) al arrancar. |

---

## Comandos útiles (en tu ordenador, desde la carpeta del repositorio)

```bash
docker compose ps                     # estado de los contenedores
docker compose logs -f waf            # logs del WAF (Ctrl+C para salir)
docker compose restart waf            # reiniciar solo el WAF
docker compose down                   # parar y borrar los contenedores y la red
```

El log de auditoría (`waf-audit-log/audit.log`) vive en tu ordenador y **no** se
borra al parar o reiniciar los contenedores. Para empezar de cero, ejecuta la celda
opcional del principio del notebook o, desde una terminal de Jupyter
(*File → New → Terminal*):

```bash
: > /workspace/waf-audit-log/audit.log
```

---

## Resolución de problemas

- **Una celda devuelve resultados raros o de una ejecución anterior.** Menú
  *Kernel → Restart Kernel* y vuelve a ejecutar desde el principio.
- **Un ataque no da 403.** Puede que se quedara una regla de práctica o un cambio
  sin restaurar: ejecuta la celda de la sección **2.Z** y vuelve a probar.
- **`nginx -t` dice que no puede abrir `tuning-local.conf` (`Not able to open file`).**
  El fichero se editó con una herramienta que cambia el inodo (p. ej. `sed -i`).
  Arréglalo con: `docker compose restart waf`.
- **Tras un ataque no aparecen entradas nuevas en el log** (por ejemplo, tras
  borrar `audit.log` a mano). Reabre el log con:
  `docker exec waf-proxy nginx -s reload`.
- **El WAF (`waf-proxy`) se reinicia o no arranca.** Revisa
  `docker compose logs waf` y vuelve a lanzar `docker compose up -d`.
- **Puerto 3000, 8080 u 8888 ocupado.** Cambia el puerto de la izquierda en
  `docker-compose.yml` (p. ej. `"8889:8888"`) o libera el puerto.
- **El notebook no ve `docker`.** El socket `/var/run/docker.sock` debe seguir
  montado en el servicio `jupyter` de `docker-compose.yml`.

> **Nota de seguridad.** El contenedor de Jupyter monta el socket de Docker de tu
> ordenador para poder hacer `docker exec` sobre el WAF. Es aceptable en un
> laboratorio local y desechable; **no** es un patrón para producción.

---

*OWASP Juice Shop y OWASP Core Rule Set son proyectos de OWASP pensados para la
formación en seguridad.*
