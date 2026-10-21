# Scraper de perfiles de Instagram

Aplicación de línea de comandos en Python que, con Playwright y Chrome, consulta un perfil de Instagram, guarda la información pública en un archivo JSON local y genera un análisis de personalidad (Big Five) con la API de Gemini en un archivo de texto. Proyecto del laboratorio de Scraping (Arquitectura, semestre 7).

## Contenido

- [Qué extrae](#qué-extrae)
- [Requisitos](#requisitos)
- [Instalación](#instalación)
- [Uso](#uso)
- [Sesión de Instagram](#sesión-de-instagram)
- [Análisis con Gemini](#análisis-con-gemini)
- [Formato del resultado](#formato-del-resultado)
- [Estructura del proyecto](#estructura-del-proyecto)
- [Límites y problemas conocidos](#límites-y-problemas-conocidos)
- [Qué no subir al repositorio](#qué-no-subir-al-repositorio)
- [Uso responsable](#uso-responsable)
- [Licencia](#licencia)

## Qué extrae

| Sección | Datos |
|---|---|
| Perfil | `username`, `bio`, `followers`, `following`, `posts` |
| Seguidores y seguidos | Nombres de usuario de las cuentas del listado |
| Publicaciones | `shortcode`, `url`, `caption`, `likes`, `comments_count`, `views`, `timestamp`, `date` |
| Comentarios | `username` y `text` de cada comentario, dentro de cada publicación |
| Engagement | Likes y comentarios totales, promedio, tasa sobre seguidores, top de publicaciones y varianza |
| Análisis | Texto de Gemini con los rasgos Big Five (OCEAN) inferidos del perfil |

Por defecto se obtienen hasta **10** seguidores, 10 seguidos, 10 publicaciones y 10 comentarios por publicación. El límite se cambia con la constante `LIMIT` de `main.py`.

Detalles de las publicaciones:

- `caption` es `"Sin texto"` si la publicación no tiene descripción.
- `views` es `"N/A"` cuando no aplica (fotos).
- `date` tiene el formato `YYYY-MM-DD HH:MM:SS`; `timestamp` es la marca original.
- Los comentarios llegan en el orden de Instagram (por relevancia) y no incluyen respuestas a comentarios.

## Requisitos

- Python 3.11 o superior.
- Google Chrome instalado (Playwright usa el canal `chrome`).
- Conexión a Internet y una cuenta de Instagram.
- Una API key de Gemini (gratuita en [Google AI Studio](https://aistudio.google.com/apikey)).

## Instalación

Windows (PowerShell), desde la carpeta del proyecto:

```powershell
py -m venv scrap
.\scrap\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

Si PowerShell bloquea la activación, usa directamente `.\scrap\Scripts\python.exe` en lugar de `python`.

macOS o Linux:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

## Uso

```bash
python main.py <usuario>
```

Si no pasas el usuario como argumento, el programa lo pide una vez. No hay menú: la ejecución es de principio a fin.

1. Indica el nombre de usuario de Instagram, con o sin `@`.
2. En la primera ejecución se abre Chrome. Si no hay sesión, inicia sesión en Instagram y vuelve a la consola para presionar Enter.
3. El programa obtiene perfil, seguidores, seguidos, publicaciones y comentarios sin más preguntas.
4. Calcula el engagement y envía los datos a Gemini para el análisis.
5. Guarda `<usuario>_data.json` (datos extraídos, engagement y análisis) y `<usuario>_analisis.txt` (solo el análisis) en la carpeta desde donde ejecutaste el programa. Si falla Gemini, el JSON se guarda igualmente y no se crea el `.txt`.

La sesión de Chrome se conserva en `.chrome_profile/`, por lo que en las siguientes ejecuciones normalmente no hace falta iniciar sesión otra vez.

## Sesión de Instagram

Hay dos formas de autenticarse:

- **Manual** (por defecto): inicia sesión en la ventana de Chrome que abre el programa.
- **Con cookies en `.env`**: evita el inicio manual. Copia la plantilla y rellénala:

```powershell
Copy-Item .env.example .env
```

```env
SESSIONID=tu_sessionid_aqui
DS_USER_ID=tu_ds_user_id_aqui      # opcional
CSRFTOKEN=tu_csrftoken_aqui        # opcional
# o la cadena completa de cookies:
# INSTAGRAM_COOKIES="sessionid=...; ds_user_id=...; csrftoken=..."
```

### Cómo obtener `sessionid`

`sessionid` es una cookie `HttpOnly`: JavaScript no puede leerla desde la consola, así que hay que copiarla desde DevTools.

1. Inicia sesión en [instagram.com](https://www.instagram.com) en tu navegador.
2. Abre DevTools con `F12` y entra en **Application** (Firefox: **Almacenamiento**).
3. Ve a **Storage > Cookies > https://www.instagram.com**.
4. Copia el valor de `sessionid` y pégalo en `.env`.

Alternativa: en la pestaña **Network**, recarga la página, abre cualquier petición a Instagram y copia el encabezado `Cookie` completo en `INSTAGRAM_COOKIES`.

`ds_user_id` y `csrftoken` se copian igual desde la tabla de cookies.

> Trata `sessionid` como una contraseña: quien lo tenga puede entrar a tu cuenta. Nunca lo compartas ni lo subas a Git.

## Análisis con Gemini

La clave y el modelo se leen del `.env`:

```env
GEMINI_API_KEY=tu_api_key_de_gemini_aqui
GEMINI_MODEL=gemini-3.8-flash      # opcional
```

Si Google retira el modelo (error 404), cambia `GEMINI_MODEL` por uno vigente; no hace falta tocar el código. El análisis es una inferencia conductual a partir de señales públicas, no una evaluación clínica.

## Formato del resultado

```json
{
  "username": "ejemplo",
  "bio": "Texto de la biografía",
  "followers": 1200,
  "following": 300,
  "posts": 85,
  "followers_list": ["usuario1", "usuario2"],
  "following_list": ["usuario3"],
  "publications": [
    {
      "shortcode": "AbCdEf123",
      "url": "https://www.instagram.com/p/AbCdEf123/",
      "caption": "Sin texto",
      "likes": 150,
      "comments_count": 12,
      "views": "N/A",
      "timestamp": 1735689600,
      "date": "2025-01-01 00:00:00",
      "comments": [{"username": "usuario1", "text": "¡Genial!"}]
    }
  ],
  "engagement": {
    "total_posts": 10,
    "total_likes": 1500,
    "total_comments": 120,
    "total_engagement": 1620,
    "avg_engagement": 162.0,
    "engagement_rate_followers": 1.35,
    "top_post": {"url": "...", "likes": 300, "comments_count": 25, "engagement": 325},
    "top_5_posts": [],
    "consistency_variance": 4200.5
  },
  "analysis": "**Apertura**: Alta — ..."
}
```

Los valores son ilustrativos. Si una sección falla, queda como lista vacía y el resto se guarda igualmente.

## Estructura del proyecto

| Archivo | Función |
|---|---|
| `main.py` | Todo el programa: navegador, scraping, engagement, análisis con Gemini y guardado de resultados |
| `.env.example` | Plantilla de cookies de Instagram y clave de Gemini |
| `requirements.txt` | Dependencias de Python |
| `LICENSE` | Licencia MIT |

## Límites y problemas conocidos

- Instagram cambia su sitio y sus endpoints internos con frecuencia; si cambian, algunas secciones pueden salir vacías.
- Pedir muchos datos seguidos puede hacer que Instagram limite o bloquee la cuenta. El programa espera entre peticiones, pero no elimina ese riesgo.
- Con cuentas privadas que no sigues solo se obtiene lo que Instagram muestra públicamente.
- El análisis depende de la API de Gemini: requiere clave válida, conexión y un modelo disponible.
- Si Chrome no abre, cierra la ventana anterior del scraper y vuelve a ejecutar.

## Qué no subir al repositorio

El `.gitignore` ya excluye estos elementos, pero si alguno se agregó antes a Git, ignorarlo no lo quita del historial y hay que retirarlo del seguimiento:

- `.env` y cualquier credencial.
- `.chrome_profile/`, que contiene datos de sesión.
- Archivos `*_data.json` y `*_analisis.txt`, que pueden contener datos personales.
- Entornos virtuales (`scrap/`, `.venv/`, `venv/`, `env/`) y `__pycache__/`.

## Uso responsable

Este proyecto es de carácter académico. Úsalo solo con cuentas y datos a los que tengas autorización para acceder. El scraping puede ir contra los Términos de Uso de Instagram, y los datos recopilados pueden ser personales: guárdalos de forma local, no los publiques y respeta la normativa de protección de datos que aplique. Los autores no se hacen responsables del uso que se le dé.

## Licencia

Distribuido bajo la licencia [MIT](LICENSE).
