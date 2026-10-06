# Sugerencias de Música Local

## Motivación

Soy una persona del milenio pasado. Para mí, mi colección musical son las aproximadamente 1,700 canciones que tengo en mi computadora: música que he ido reuniendo poco a poco, a lo largo de los años, de amigos que compartieron conmigo algo que les gustaba, de discos que descubrí y de etapas de mi vida.

Esa colección tiene valor precisamente porque no fue creada por un algoritmo. Sin embargo, sí echo en falta la posibilidad de recibir sugerencias de música nueva basadas en las canciones que ya conservo.

Por eso, y aprovechando mis nuevas habilidades con Inteligencia Artificial, decidí construir este proyecto: una herramienta que use mi biblioteca como punto de partida para crear playlists propias y descubrir canciones que probablemente encajen con ellas.

## Qué hace este proyecto

El sistema produce dos resultados complementarios:

1. **Playlists locales:** grupos de canciones que ya existen en la biblioteca personal.
2. **Canciones faltantes:** recomendaciones para cada playlist; canciones que aún no están en la colección, pero que suelen aparecer junto con canciones similares en playlists de otras personas.

No se limita a agrupar por género, artista, década o álbum. Aprende relaciones de **coocurrencia**: qué canciones suelen estar juntas en playlists humanas. Esto permite detectar afinidades menos obvias entre canciones de distintos estilos, épocas o artistas.

Los resultados son borradores para revisión humana; no pretenden sustituir el criterio personal ni afirmar con certeza que una canción gustará.

~~~mermaid
flowchart LR
  A[Biblioteca musical local] --> B[Metadatos y coincidencias]
  C[Playlists externas autorizadas] --> D[Corpus de canciones]
  D --> E[Embeddings item2vec]
  B --> F[Catálogo local resuelto]
  E --> G[Comunidades de canciones propias]
  F --> G
  G --> H[Playlists locales]
  G --> I[Recomendaciones de canciones faltantes]
~~~

## Cómo funciona

Cada canción se representa como un vector numérico, también llamado *embedding*. Dos canciones quedan cerca en ese espacio cuando aparecen con frecuencia en contextos de playlist similares.

El proceso identifica coincidencias de la biblioteca local con un catálogo de referencia, aprende relaciones usando playlists externas autorizadas, encuentra comunidades entre las canciones propias y busca canciones externas cercanas al perfil de cada comunidad.

La puntuación expresa **afinidad dentro del modelo**, no una probabilidad. Una puntuación alta significa que la canción encaja bien con el perfil de la lista según los patrones aprendidos; no equivale a “95 % de certeza de que te gustará”.

~~~mermaid
flowchart TD
  A[1. Auditar y normalizar<br/>la biblioteca local] --> B[2. Resolver coincidencias<br/>con el catálogo]
  C[3. Aplanar playlists<br/>externas autorizadas] --> D[4. Preparar corpus en DuckDB]
  B --> D
  D --> E[5. Entrenar item2vec]
  E --> F[6. Encontrar comunidades<br/>en canciones propias]
  F --> G[7. Crear borradores<br/>de playlists]
  G --> H[8. Recomendar canciones faltantes]
  H --> I[9. Revisar e importar<br/>a Music]
~~~

## Herramientas incluidas

| Herramienta | Para qué se utiliza |
| --- | --- |
| flatten_spotify_mpd.py | Convierte playlists externas autorizadas a JSON Lines para procesarlas de forma incremental. |
| export_local_tracks.py | Convierte una auditoría de biblioteca local en un catálogo compacto de canciones. |
| spotify_track_lookup.py | Busca coincidencias de título y artista en Spotify. Las credenciales se leen sólo desde variables de entorno. |
| train_item2vec.py | Entrena embeddings de canciones a partir de secuencias de playlists. |
| analyze_item2vec.py | Calcula vecinos, crea un grafo mutuo y encuentra comunidades de canciones propias. |
| recommend_from_embeddings.py | Genera borradores de playlists y recomendaciones de canciones faltantes. |
| export_playlists_xml.py | Exporta playlists como XML compatible con Music/iTunes. |
| import_playlists_to_music.py | Crea borradores directamente en Música de macOS, sin duplicar archivos de audio. |

## Flujo de uso

### 1. Preparar la biblioteca local

La biblioteca se audita y se corrigen, en la medida de lo posible, sus metadatos: título, artista, álbum y ubicación de cada archivo. Las canciones nunca se suben a este repositorio ni a un servicio externo como parte de este proceso; la biblioteca permanece en la computadora del usuario.

### 2. Preparar playlists externas

Se necesita un corpus de playlists al que se tenga acceso legítimo. El código puede procesar el Spotify Million Playlist Dataset, pero el dataset no se incluye ni se distribuye con este proyecto.

~~~bash
python3 tools/flatten_spotify_mpd.py \
  /ruta/al/dataset/Data \
  /ruta/privada/mpd-tracks.jsonl.gz
~~~

Para una prueba rápida sobre un fragmento:

~~~bash
python3 tools/flatten_spotify_mpd.py \
  /ruta/al/dataset/Data \
  /ruta/privada/mpd-tracks.jsonl.gz \
  --limit-files 1
~~~

### 3. Entrenar el modelo de afinidad

Con el corpus de playlists preparado en DuckDB, se entrena un modelo item2vec. El modelo aprende de qué canciones suelen aparecer cerca unas de otras.

~~~bash
python3 tools/train_item2vec.py \
  --db /ruta/privada/catalogo.duckdb \
  --output-dir /ruta/privada/modelo
~~~

Por defecto usa vectores de 64 dimensiones. El tamaño, ventana de contexto, número de épocas y otros parámetros pueden ajustarse desde la línea de comandos.

### 4. Encontrar comunidades en la colección

El análisis se hace únicamente sobre canciones que ya pertenecen a la biblioteca y que tienen una coincidencia válida en el catálogo de referencia.

~~~bash
python3 tools/analyze_item2vec.py \
  --db /ruta/privada/catalogo.duckdb \
  --vectors /ruta/privada/modelo/mpd_item2vec_64d.kv \
  --output-dir /ruta/privada/analisis
~~~

La herramienta calcula vecinos cercanos, conserva relaciones mutuas y aplica Louvain para detectar comunidades. También ejecuta una prueba de cohesión: oculta una canción dentro de una playlist candidata y mide si el modelo puede volver a encontrarla a partir de las demás.

Esto es un diagnóstico técnico del modelo, no una medida independiente del gusto personal.

### 5. Generar playlists y recomendaciones

~~~bash
python3 tools/recommend_from_embeddings.py \
  --db /ruta/privada/catalogo.duckdb \
  --vectors /ruta/privada/modelo/mpd_item2vec_64d.kv \
  --communities /ruta/privada/analisis/local_embedding_communities.csv \
  --output-dir /ruta/privada/borradores
~~~

La herramienta descarta comunidades muy pequeñas o excesivamente concentradas en un solo artista. Para cada playlist aceptada genera canciones propias y recomendaciones externas.

También aplica reglas de diversidad:

- excluye canciones que ya están en la colección, incluso si son variantes como “remaster”, “live” o “deluxe”;
- exige una presencia mínima en el corpus de playlists;
- limita la repetición temprana de un mismo artista;
- favorece artistas nuevos en las primeras posiciones.

Los archivos principales generados son:

| Archivo | Contenido |
| --- | --- |
| embedding_playlist_profiles.csv | Perfil de cada borrador de playlist. |
| embedding_playlist_owned_tracks.csv | Canciones propias asignadas a cada playlist. |
| embedding_playlist_missing_recommendations.csv | Canciones sugeridas con su afinidad y contexto. |
| embedding_excluded_communities.csv | Comunidades descartadas y la razón. |
| embedding_playlist_drafts_summary.json | Resumen de parámetros y resultados de la ejecución. |

## Llevar las playlists a Music en macOS

Después de revisar los borradores hay dos opciones.

### Exportar XML

export_playlists_xml.py genera un archivo compatible con Music/iTunes. Sólo hace referencia a canciones ya presentes en la biblioteca; no copia ni mueve audio.

~~~bash
python3 tools/export_playlists_xml.py \
  --owned-csv /ruta/privada/borradores/embedding_playlist_owned_tracks.csv \
  --names-csv /ruta/privada/borradores/playlist_names.csv \
  --music-library-export /ruta/a/Music-Library.xml \
  --output "/ruta/privada/Sugerencias de Música Local.xml"
~~~

Después, en Música: **Archivo → Biblioteca → Importar playlist**.

### Crear borradores directamente

En macOS, import_playlists_to_music.py puede localizar las canciones por su ruta exacta y crear las playlists. Por seguridad, primero hace una simulación; sólo modifica la biblioteca cuando se añade --apply.

~~~bash
python3 tools/import_playlists_to_music.py \
  --owned-csv /ruta/privada/borradores/embedding_playlist_owned_tracks.csv \
  --names-csv /ruta/privada/borradores/playlist_names.csv
~~~

## Instalación

Se recomienda Python 3.11 o posterior.

~~~bash
python3 -m venv .venv
. .venv/bin/activate
pip install -r tools/requirements-recommender.txt
~~~

La guía técnica detallada está en [tools/README-spotify-mpd.md](tools/README-spotify-mpd.md).

## Privacidad, datos y licencias

Este repositorio contiene sólo código y documentación. No incluye ni debe incluir:

- canciones, archivos de audio o rutas de la biblioteca;
- metadatos personales;
- credenciales, tokens o secretos;
- datasets externos;
- bases DuckDB, embeddings, índices, CSV, XML o resultados generados;
- recomendaciones producidas a partir de una biblioteca privada.

El archivo .gitignore protege las ubicaciones locales más habituales, pero cada persona debe verificar lo que añade a Git antes de publicarlo.

Los datasets externos conservan sus propios términos de uso. Tener este código no concede derechos sobre esos datos.

## Límites

- La calidad depende de qué tan buenas sean las coincidencias entre canciones locales y el catálogo de referencia.
- Una coocurrencia frecuente puede reflejar popularidad, época o contexto cultural, no necesariamente una conexión musical profunda.
- Las comunidades son puntos de partida: conviene revisarlas, darles nombre y ajustar su tamaño.
- La afinidad del embedding no está calibrada como porcentaje de certeza.
- El sistema mejora cuando el usuario acepta, mueve o rechaza sugerencias y ajusta los parámetros en consecuencia.

## Licencia

El código se distribuye bajo licencia [MIT](LICENSE). Los datos externos y las bibliotecas personales no forman parte de esta licencia.
