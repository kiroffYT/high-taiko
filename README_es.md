# high-taiko

Un renderizador interno de replays de osu!taiko a MP4 (fork de **osu-taiko-renderer**). Analiza un archivo de replay `.osr` con respecto a su beatmap, simula y evalúa la jugada, dibuja el área de juego de taiko con desplazamiento de derecha a izquierda fotograma por fotograma y lo codifica en video mediante ffmpeg.

> **Característica del fork**: Si no se especifica el parámetro `BEATMAP_DIR`, **high-taiko** descargará automáticamente el beatmap correspondiente a través de la API de **osu.direct**, utilizando el hash MD5 que se encuentra dentro del propio archivo de replay.

## Parte del R3D Renderer

Este es el **motor de taiko** para [R3D Renderer](https://renderer.r3dwolfie.com), un servicio independiente de conversión de replays de osu! a MP4 (bot de Discord + sitio web, paquete `mania_ordr`) que envía cada renderizado a un repositorio de motor específico según el modo. El núcleo ejecuta este motor como un subproceso independiente por cada renderizado, por lo que cualquier cambio se aplica en el siguiente renderizado sin necesidad de reiniciar el servicio. En producción, este motor se ejecuta en el entorno virtual de Python del motor **catch** (intérprete compartido; el código reside en su propio repositorio).

> Nota: partes de este repositorio provienen del renderizador de catch — algunas cadenas de documentación de módulos (por ejemplo, `osu_taiko_renderer/__init__.py`, `render/gl.py`) todavía mencionan "catch". El código en tiempo de ejecución es específico para taiko.

## Qué renderiza / Fidelidad

* **Objetos de Taiko**: don (centro rojo), kat (borde azul), notas grandes/finales (ambas teclas), redobles de tambor (+ ticks) y swells/denden (agitadores de golpes alternos). Se modelan cuatro entradas: centro-izquierda/derecha y borde-izquierda/derecha.
* **Simulación ordenada y precisa + Juicio** (`render/scene.py`): cada pulsación consume la nota más avanzada del mismo color disponible dentro de la ventana de impacto OD (notelock, sin robo de pulsaciones); una nota cuya ventana pasa sin ser golpeada se considera MISS. Esto ubica los cortes de combo en las notas que el jugador realmente falló.
* **Reconciliación del encabezado** (`beatmap/score_fidelity.py`): el encabezado del archivo `.osr` mantiene la autoridad sobre el recuento — un paso de reconciliación ajusta los totales de GREAT/OK/MISS y el combo máximo con el encabezado, garantizando precisión exacta en precisión, puntuación y recuentos, mientras que la simulación determina *qué* notas rompieron el combo.
* **HUD Argon (lazer) por defecto** (`argon/` — compositor, HUD, contadores, glifos procedimentales). No se incluyen recursos del juego osu!; la salida sin skin utiliza arte generado procedimentalmente de Argon y la fuente Nunito incluida. Se puede proporcionar un skin personalizado mediante `--skin`.
* **pp / SR**: contador de pp en vivo y calificación de estrellas estimada mediante `rosu_pp_py`; `--pp` / `--sr` fijan la tarjeta de resultados (y el extremo del contador pp) a los valores oficiales de osu! manteniendo la forma de curva de rosu.
* **Storyboards**: motor interno opcional de storyboards (`--storyboard`, desactivado por defecto; paridad con el motor std). Cuando está desactivado, la salida es idéntica byte a byte.
* **Extras**: pantalla de resultados final con tabla de clasificación por mapa (BD local de renderizados, o mejores puntuaciones globales de osu! mediante `--leaderboard-source osu`), oscurecimiento/desenfoque de fondo, bandas negras en descansos, efectos de sonido de beatmap/fallo/nightcore, marca de agua y un archivo JSON por fotograma (`--score-json`) para el compositor en vivo.

## Uso

Invocado como módulo:

```
python -m osu_taiko_renderer REPLAY.osr [BEATMAP_DIR] -o out.mp4 \
    [--resolution 1920x1080] [--fps 60] [--encoder auto] [--skin DIR]

```

* `REPLAY.osr` — el archivo de replay (posicional).
* `BEATMAP_DIR` — directorio que contiene el `.osu` + audio + fondo (posicional opcional; si se omite, se descarga automáticamente desde osu.direct mediante el MD5 del replay).
* `-o, --output` — ruta del archivo de salida (**requerido**).

Opciones seleccionadas (verificadas desde `osu_taiko_renderer/cli.py`):

| Opción | Por defecto | Propósito |
| --- | --- | --- |
| `--resolution WxH` | `1920x1080` | resolución de salida |
| `--fps N` | `60` | cuadros por segundo |
| `--encoder` | `auto` | `auto | h264_vaapi | h264_nvenc | libx264` |
| `--encoder-device` | ninguno | ej. `/dev/dri/renderD128` |
| `--skin DIR` / `--default-skin DIR` | ninguno | directorio de skin extraído / alternativa |
| `--scroll-time MS` | `1600` | ms que una nota a 1.0× es visible (menor = más rápido) |
| `--pp FLOAT` / `--sr FLOAT` | ninguno | fijar pp / dificultad de estrellas oficiales en resultados |
| `--storyboard` | off | renderizar el storyboard del mapa |
| `--[no-]results`, `--[no-]pp-counter`, `--[no-]hit-counter`, `--[no-]leaderboard` | on | conmutadores del HUD / final |
| `--leaderboard-source {r3d,osu}` | `r3d` | fuente de la tabla en resultados (`osu` lee `--leaderboard-json`) |
| `--watermark`, `--music-volume`, `--general-volume`, `--audio-offset`, `--bg-dim-*`, `--bg-blur` | — | ajuste de audio y fondo |

Los conmutadores booleanos utilizan `BooleanOptionalAction` de argparse (`--flag` / `--no-flag`). Las opciones desconocidas son aceptadas e ignoradas.

Variables de entorno relevantes: `R3D_EGL_DEVICE_INDEX` (selección del dispositivo GPU/EGL), `R3D_FRAME_MD5` (hash de fotogramas sin procesar).

## Requisitos

* **Python 3.10+** (utiliza uniones de tipo `X | Y` y `BooleanOptionalAction`).
* Paquetes de terceros (importados directamente; sin `requirements.txt`/`pyproject.toml` — las dependencias vienen del entorno virtual compartido):
* `numpy`
* `Pillow` (PIL — composición del HUD por CPU)
* `moderngl` (contexto **EGL** independiente / framebuffer RGBA fuera de pantalla)
* `osrparse` (análisis de replays)
* `rosu_pp_py` (cálculo de pp y estrellas; opcional)
* `osu_renderer` — paquete central de R3D para codificación ffmpeg y uso de skins (`wiki_renderer.SkinPair`)


* **ffmpeg** en el `PATH` del sistema (el renderizador maneja su propio subproceso de ffmpeg con rgb24 por entrada estándar).

## Estructura

```
osu_taiko_renderer/
  __main__.py, cli.py       Punto de entrada CLI (argparse) → render_taiko()
  beatmap/                  modelos de análisis y simulación
    beatmap.py, models.py, replay.py, sliderpath.py,
    storyboard.py, score_fidelity.py, legacy_random.py
  render/                   orquestación + GL + codificación
    render.py               análisis → simulación → dibujo GL cuadro por cuadro + HUD → ffmpeg
    scene.py                simulación taiko + constructor de escena
    gl.py                   lote de sprites moderngl/EGL
    effects.py, flashlight.py, dim.py, hitsounds.py,
    storyboard_engine.py, storyboard_render.py
  argon/                    HUD procedimental Argon (lazer) — compositor, HUD,
                            contadores, geometría, texturas, fuentes, glifos
  hud/                      HUD y resultados: hud.py, lazer_results.py,
                            leaderboard.py, lb_cards.py, break_overlay.py, …
  skin/                     carga de skins: taiko_skin.py, lazer_skin.py,
                            assets.py, fonts.py
  assets/                   logo, fuente Nunito (OFL), efectos de sonido nightcore
  fonts_data/               atlasses de la fuente Torus
tests/                      pruebas de storyboard e intro

```

## Licencia

**GNU Affero General Public License v3.0 or later (AGPL-3.0-or-later)** — ver `LICENSE`. Copyright (C) 2026 Cool Adults (ver `COPYRIGHT`).

Atribución (según `COPYRIGHT`): la jugabilidad, sincronización, juicio, puntuación y HUD son adaptaciones de osu! / osu-framework por peppy / ppy (MIT); danser-go por Wieku (GPL-3.0) se estudió como referencia. **No se incluyen recursos del juego osu!** — los skins, mapas, audio y arte de osu! tienen licencia CC BY-NC y no están incluidos. La fuente variable **Nunito** incluida está bajo la Licencia SIL Open Font 1.1 (`assets/fonts/OFL.txt`).
