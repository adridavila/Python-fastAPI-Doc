# Python-fastAPI-Doc
Documentación técnica y arquitectura de herramienta CLI en Python con FastAPI y LMDB.

# Objetivo del proyecto

CLIColaDescarga es el cliente de línea de comandos del sistema DPM. Su función principal es permitir que el usuario opere desde terminal los flujos del backend de descarga y consulta de información: fuentes de YouTube, fuentes RSS de noticias, tareas programadas, exportación de archivos, prioridades, monitoreo por WebSocket, logs y configuración local del servicio.

La CLI no implementa por sí sola la descarga final de información. Actúa como una capa de operación que valida comandos, construye modelos de datos, consume endpoints HTTP, escucha canales WebSocket y presenta resultados en tablas interactivas usando Rich y prompt_toolkit.
Cómo leer este documento

Este archivo documenta la carpeta completa CLIColaDescarga. La lectura recomendada es:

    Revisar primero la estructura general y el flujo de ejecución.
    Consultar la guía de comandos para entender la superficie de uso.
    Leer las capas view, endpoints, models, enums y utils según el módulo que se vaya a mantener.
    Usar la sección de archivos como índice técnico cuando se necesite ubicar una clase o responsabilidad concreta.

Estructura general

CLIColaDescarga/
├── main.py
├── README.md
├── __init__.py
└── app/
    ├── endpoints/
    ├── enums/
    ├── models/
    ├── test/
    ├── utils/
    ├── view/
    └── instrucciones.txt

Flujo general de ejecución

    main.py crea una sesión HTTP asíncrona con aiohttp.ClientSession.
    Completer inicia la terminal interactiva y muestra notificaciones al arrancar.
    PromptManager construye el prompt dpm>, el autocompletado y la barra inferior de ayuda.
    El usuario escribe comandos que empiezan con dpm.
    ParserCLI convierte la entrada en un objeto argparse.
    Dispatcher selecciona la vista responsable del módulo solicitado.
    La vista valida, normaliza y renderiza datos.
    Los endpoints consumen el backend mediante MetodosGenerales o escuchan WebSockets.
    Las respuestas se muestran como tablas, paneles o mensajes enriquecidos en terminal.

Capas de la aplicación
main.py

Punto de entrada de la CLI. Define app() y main(). Crea la sesión HTTP compartida, instancia Completer, ejecuta la terminal y cancela tareas asíncronas pendientes al cerrar. También captura KeyboardInterrupt para terminar de forma controlada.
app/view

Contiene la interfaz de usuario de la terminal. Sus clases no deberían conocer detalles internos del backend; se encargan de recibir comandos ya parseados, llamar endpoints, normalizar respuestas y mostrarlas con Rich.
app/endpoints

Contiene adaptadores HTTP y WebSocket hacia el backend. Cada archivo agrupa rutas de un dominio: fuentes, tareas, archivos, noticias, prioridades, YouTube, logs o notificaciones.
app/models

Define modelos de transporte usados por la CLI para construir payloads y normalizar estructuras: tareas, periodos, logs, archivos, fuentes, videos y conteos.
app/enums

Centraliza valores constantes compartidos entre parser, modelos y vistas. Evita escribir cadenas manuales para plataformas, estados, tipos de tarea, periodos, módulos y niveles de log.
app/utils

Contiene utilidades transversales. MetodosGenerales concentra peticiones HTTP, stream NDJSON, manejo de errores, fechas y truncamiento de texto. Configuracion gestiona arranque, paro, puerto y binarios del servicio local.
app/test

Contiene scripts auxiliares de prueba manual. No son una suite automatizada completa; sirven para probar ejecución de comandos desde archivo y configuración del servicio.
Guía de comandos

El comando base es dpm. La terminal interactiva recibe una línea de texto, la normaliza con shlex, valida que empiece con dpm, la parsea con ParserCLI y la envía a Dispatcher. Los comandos clear, cls, exit y quit también pueden ejecutarse sin prefijo porque son comandos internos de Completer.

Convenciones usadas en esta guía:

    <valor> significa argumento obligatorio.
    [valor] significa argumento opcional.
    Las opciones con -- son banderas de argparse.
    Los valores de plataforma aceptados son YOUTUBE y NOTICIAS.
    Los valores de tipo de tarea aceptados son DESCARGA y EXPORTACION.
    Los valores de periodo aceptados son HORA, DIAS, SEMANA y MES.

Sistema
dpm help

Muestra la tabla de ayuda construida en Dispatcher._help(). No consume el backend; solo renderiza información local con Rich.
clear, cls o dpm clear

Limpia la pantalla desde Completer.__comandos_internos(). No pasa por ParserCLI ni por Dispatcher.
exit, quit o dpm exit

Cierra la sesión interactiva. Completer detecta el comando interno y termina el ciclo de lectura del prompt.
Configuración
dpm config iniciar

Inicia el backend local desde Configuracion.iniciar(). La clase busca el ejecutable dpm-service junto al paquete o, en modo desarrollo, intenta iniciar el backend con Python. Guarda el PID en ~/.dpm-base/dpm-service.pid para poder detenerlo después.
dpm config detener

Detiene el backend iniciado por la CLI. Configuracion.finalizar() lee el PID guardado, intenta terminar el proceso y limpia el archivo de PID cuando corresponde.
dpm config estado

Verifica si el servicio responde usando la configuración HTTP local. Sirve para confirmar que el backend está activo antes de ejecutar comandos que consumen endpoints.
dpm config estado_puerto

Muestra el puerto configurado en el archivo de entorno administrado por Configuracion. El puerto se usa para construir las variables HTTP y WSS.
dpm config modificar_puerto <puerto>

Actualiza el puerto de ejecución. ParserCLI recibe <puerto> como cmd.port; Dispatcher.config() llama Configuracion.actualizar_puerto(cmd.port), que actualiza PORT, HTTP y WSS, detiene el servicio actual y vuelve a iniciarlo con el nuevo puerto.

Ejemplo:

dpm config modificar_puerto 2055

dpm config actualizar_yt-dlp

Descarga o reemplaza el binario local de yt-dlp dentro de ~/.dpm-base/bin. Se usa para mantener actualizado el componente de descarga de YouTube usado por el backend.
dpm config agregar_script <ruta>

Ejecuta comandos desde un archivo de texto. ParserCLI parsea la ruta, pero Completer.__dispatch_part() intercepta este subcomando y llama Completer.run_file(ruta). El archivo debe contener un comando dpm por línea; las líneas vacías y las líneas que empiezan con # se omiten.

Ejemplo:

dpm config agregar_script C:\rutas\carga_fuentes.txt

Fuentes
dpm fuentes agregar YOUTUBE <url_o_txt>

Encola fuentes de YouTube. El argumento puede ser una URL de YouTube o la ruta de un archivo .txt.

Flujo técnico:

    ParserCLI guarda el valor en cmd.urls.
    Dispatcher._fuentes() llama Fuentes.encolar_fuentes(cmd.urls).
    Fuentes.encolar_fuentes() valida que el valor sea una URL de YouTube o un archivo .txt.
    Si es URL, construye una lista JSON con esa URL.
    Si es .txt, verifica que exista, que sea archivo, que no esté vacío y extrae una URL por línea.
    FuentesEndpoint.encolar_fuentes() envía POST /fuentes/insertar con el parámetro fuentes.
    La respuesta se normaliza y se muestra en una tabla de resultados.

Ejemplos:

dpm fuentes agregar YOUTUBE https://www.youtube.com/watch?v=VIDEO_ID
dpm fuentes agregar YOUTUBE C:\fuentes\youtube.txt

dpm fuentes agregar NOTICIAS <nombre> <url_rss>

Registra una fuente RSS para el módulo de noticias.

Flujo técnico:

    ParserCLI asigna cmd.nombre y cmd.url_rss.
    Dispatcher._fuentes() llama Fuentes.registrar_fuente_noticias(nombre, url_rss).
    FuentesEndpoint.registrar_fuente_noticias() construye el payload {"nombre": nombre, "url_rss": url_rss}.
    Se envía POST /fuentes/noticias.
    La CLI muestra una tabla con la fuente y el estado de registro.

Ejemplo:

dpm fuentes agregar NOTICIAS "El Universal" https://www.eluniversal.com.mx/rss.xml

dpm fuentes estado

Muestra el estado de fuentes en tiempo real. Dispatcher._fuentes() llama WebSocketsView.tabla_fuentes(), que abre el canal WebSocket /webSockets/tabla_fuentes, normaliza los paquetes recibidos y actualiza una tabla viva con Rich Live.
dpm fuentes modificar_prioridad --plataforma <plataforma> --prioridad <n>

Cambia la prioridad de fuentes pendientes por plataforma.

Flujo técnico:

    ParserCLI recibe cmd.plataforma y cmd.prioridad.
    Dispatcher._fuentes() detecta cmd.plataforma.
    Fuentes.prioridad_plataforma() llama FuentesEndpoint.actualizar_prioridad_plataforma().
    Se envía PATCH /fuentes/plataforma/{plataforma} con el parámetro prioridad.
    La respuesta se interpreta como éxito, sin cambios o error.

Ejemplo:

dpm fuentes modificar_prioridad --plataforma YOUTUBE --prioridad 2

dpm fuentes modificar_prioridad --nombre --nombre_video <nombre> --prioridad <n>

Cambia la prioridad por nombre de fuente o video. La bandera --nombre activa este modo; el texto real se pasa con --nombre_video.

Flujo técnico:

    ParserCLI marca cmd.nombre=True, guarda cmd.nombre_video y cmd.prioridad.
    Dispatcher._fuentes() llama Fuentes.prioridad_nombre(nombre_video, prioridad).
    FuentesEndpoint.actualizar_prioridad_nombre() envía PATCH /fuentes/fuente/{nombre} con parámetros nombre y prioridad.

Ejemplo:

dpm fuentes modificar_prioridad --nombre --nombre_video "Mi video" --prioridad 1

Prioridades
dpm prioridades estado

Consulta las prioridades globales por plataforma. Prioridades.mostrar_prioridades() consume el endpoint de prioridades, normaliza la respuesta y muestra la tabla correspondiente.

Endpoint principal: GET /prioridad/prioridades.
dpm prioridades asignar_prioridad <plataforma> --prioridad <n>

Asigna una prioridad global a una plataforma.

Flujo técnico:

    ParserCLI guarda cmd.plataforma y cmd.prioridad.
    Dispatcher._prioridades() convierte la prioridad a entero.
    Prioridades.cambiar_prioridad() llama al endpoint de actualización.
    PrioridadEndpoints.cambiar_prioridad() envía PUT /prioridad/prioridades/{plataforma}.

Ejemplo:

dpm prioridades asignar_prioridad NOTICIAS --prioridad 3

Tareas
dpm tareas insertar_tarea [--tipo_tarea <tipo>] [--plataforma <plataforma>] [--periodo <periodo>] [--valor_periodo <n>] [--fuente <url>]

Crea una tarea programada. Todos los argumentos son opcionales en el parser; si falta alguno, Dispatcher._tareas() lo solicita con prompts interactivos y autocompletado.

Flujo técnico:

    Se resuelve tipo_tarea como EnumTarea.
    Se resuelve plataforma como EnumPlataforma.
    Se resuelve periodo como EnumPeriodo.
    Si la tarea es DESCARGA, se solicita o usa --fuente.
    Se construye un modelo Periodo(periodo, valor_periodo).
    Para DESCARGA, se construye TareaDescarga(tipo, plataforma, fuente, periodo).
    Para EXPORTACION, se construye Tarea(tipo, plataforma, periodo).
    Se envía POST /tareas/tarea/insertar con el diccionario del modelo.

Validación de periodo:

    HORA: valor entre 1 y 23.
    DIAS: valor mayor a 0.
    SEMANA: valor mayor a 0.
    MES: valor entre 1 y 12.

Ejemplos:

dpm tareas insertar_tarea --tipo_tarea DESCARGA --plataforma YOUTUBE --periodo HORA --valor_periodo 6 --fuente https://www.youtube.com/@canal
dpm tareas insertar_tarea --tipo_tarea EXPORTACION --plataforma NOTICIAS --periodo DIAS --valor_periodo 1

dpm tareas modificar_tarea [--clave_tarea <clave>] [--periodo <periodo>] [--valor_periodo <n>]

Modifica el periodo de una tarea existente. Si falta la clave, el periodo o el valor, la CLI lo solicita de forma interactiva.

Flujo técnico:

    Dispatcher._tareas() obtiene clave_tarea.
    Construye un nuevo Periodo.
    Tareas.actualizar_tarea() envía el diccionario del periodo.
    TareasEndpoint.actualizar_tarea() ejecuta PATCH /tareas/tarea/{clave}.

Ejemplo:

dpm tareas modificar_tarea --clave_tarea 2f7c... --periodo DIAS --valor_periodo 2

dpm tareas eliminar_tarea <clave>

Elimina una tarea programada por clave.

Flujo técnico:

    ParserCLI guarda la clave en cmd.clave.
    Dispatcher._tareas() llama Tareas.eliminar_tarea(cmd.clave).
    TareasEndpoint.eliminar_tarea() ejecuta DELETE /tareas/tarea/{clave}.

dpm tareas estado

Lista tareas programadas mediante stream HTTP. Tareas.mostrar_tareas() consume GET /tareas/tarea/consultar con _request_stream() y agrega filas a una tabla viva.
YouTube
dpm youtube listar

Lista metadatos de videos descargados.

Flujo técnico:

    Dispatcher._youtube() llama YoutubeView.mostrar_videos().
    YoutubeView.mostrar_tabla() consume el stream GET /youtube/videos/metadatos.
    Cada paquete se normaliza con campos como Nombre_Video, Nombre_Canal, enlaces y fechas.
    La información se renderiza en una tabla Rich Live.

dpm youtube buscar --video <texto>

Busca videos por nombre. YoutubeView.buscar("nombre", texto) codifica el texto con quote() y consulta GET /youtube/video/{texto}.

Ejemplo:

dpm youtube buscar --video "conferencia matutina"

dpm youtube buscar --canal <texto>

Busca videos por canal. YoutubeView.buscar("canal", texto) codifica el texto con quote() y consulta GET /youtube/canales/{canal}/videos.

Ejemplo:

dpm youtube buscar --canal "Canal Oficial"

Noticias
dpm noticias iniciar_descarga [intervalo_min]

Inicia el scheduler de noticias. Si no se indica intervalo, SchedulerNoticiasEndpoint.start_noticias() usa 60 minutos por defecto.

Flujo técnico:

    ParserCLI guarda el argumento opcional en cmd.intervalo_min.
    ScheduleView.mostrar_start_noticias() llama el endpoint.
    Se envía POST /schedulerNoticias/start con parámetro intervalo_min.
    La respuesta se muestra como tabla de campos y valores.

Ejemplo:

dpm noticias iniciar_descarga 30

dpm noticias detener_descarga

Detiene el scheduler de noticias mediante POST /schedulerNoticias/stop y muestra el resultado en tabla.
dpm noticias estado

Consulta el estado actual del scheduler mediante GET /schedulerNoticias/status.
dpm noticias listar

Lista noticias descargadas. EditorialView.mostrar_tabla_noticias() ejecuta GET /editorialConsulta/noticias, normaliza fuente, titulo, url y fechaPublicacion, y renderiza una tabla.
dpm noticias analizar_editorial [fuente] [dias]

Ejecuta el análisis editorial manual. Los argumentos son opcionales en ParserCLI; si se proporcionan, se envían a EditorialNoticiasEndpoint.analizar_editorial().

Endpoint principal: /editorialConsulta/analizar_editorial.

Ejemplo:

dpm noticias analizar_editorial "El Universal" 7

dpm noticias resumir_editorial

Obtiene un resumen del módulo editorial. EditorialView.mostrar_resumen_noticias() consulta /editorialConsulta/resumen y renderiza tablas para salud general, noticias, editorial, fuentes clave y diagnóstico.
dpm noticias listar_fuentes

Lista fuentes RSS registradas y permite eliminar una fuente desde la misma interfaz. Primero ejecuta GET /editorialConsulta/fuentes; después solicita por consola el nombre a eliminar. Si se escribe un nombre válido, ejecuta DELETE /editorialConsulta/fuentes/{nombre}.
dpm noticias limpiar_exportadas

Elimina noticias que ya fueron exportadas, conservando las llaves históricas necesarias para evitar duplicados. Ejecuta DELETE /editorialConsulta/noticias/exportadas y muestra métricas como eliminadas, conservadas y exportadas detectadas.
Archivos
dpm archivos estado

Muestra archivos en tiempo real. Dispatcher._archivos() llama WebSocketsView.tabla_archivos(), que escucha /webSockets/tabla_archivos, normaliza eventos y actualiza la tabla con estados como PENDIENTE, DESCARGANDO, ERROR, CANCELAR y EXITO.
dpm archivos generar <YOUTUBE|NOTICIAS>

Solicita al backend generar un archivo de exportación para una plataforma.

Flujo técnico:

    ParserCLI convierte el argumento a mayúsculas y valida que sea YOUTUBE o NOTICIAS.
    ArchivosView.insertar_archivo() llama ArchivosData.insertar_archivo(plataforma).
    Se envía POST /archivos/insertar con parámetro plataforma.
    La respuesta se muestra como tabla.

Ejemplo:

dpm archivos generar NOTICIAS

dpm archivos cancelar <nombre>

Solicita cancelar o cambiar el estado de un archivo por nombre.

Flujo técnico:

    ParserCLI guarda cmd.nombre.
    ArchivosData.cancelar_archivo() codifica el nombre con quote().
    Se envía PATCH /archivos/archivo/{nombre}.

dpm archivos importar_archivo <YOUTUBE|NOTICIAS> [--limpiar-bd]

Descarga el archivo exportado de una plataforma al directorio de descargas del usuario. La opción --limpiar-bd solo existe para NOTICIAS.

Flujo técnico:

    ParserCLI usa subparsers por plataforma y define cmd.limpiar_bd.
    ArchivosData.exportar_archivo() ejecuta GET /archivos/exportar/{plataforma}.
    Si la respuesta contiene bytes de archivo, lee Content-Disposition para obtener el nombre.
    Guarda el archivo en Downloads, Descargas o la carpeta configurada por XDG_DOWNLOAD_DIR.
    Si la plataforma es NOTICIAS y se usa --limpiar-bd, ejecuta DELETE /editorialConsulta/noticias/exportadas.

Ejemplos:

dpm archivos importar_archivo YOUTUBE
dpm archivos importar_archivo NOTICIAS --limpiar-bd

dpm archivos importar_archivos

Exporta en secuencia los archivos de YOUTUBE y NOTICIAS. Internamente llama dos veces a ArchivosView.exportar_archivo(): primero con YOUTUBE y después con NOTICIAS.
Logs
dpm logs estado

Muestra logs en tiempo real. Dispatcher._logs() llama WebSocketsView.tabla_logs(), que abre /webSockets/tabla_logs, agrupa eventos por módulo y nivel, y renderiza paneles con Panel_logs_View.
Configuración local

La clase Configuracion trabaja con archivos dentro del home del usuario:

    ~/.dpm-base/.env: guarda PORT, HTTP y WSS.
    ~/.dpm-base/dpm-service.pid: guarda el PID del backend iniciado desde la CLI.
    ~/.dpm-base/bin/yt-dlp o yt-dlp.exe: binario descargado o actualizado por la CLI.

El backend esperado se busca junto al ejecutable de Python o del paquete, en dpm-service/dpm-service.exe para Windows o dpm-service/dpm-service para Linux.

Nota tecnica: MetodosGenerales lee por default ~/.dpm/.env, mientras que Configuracion crea y actualiza ~/.dpm-base/.env. Conviene revisar está diferencia si la CLI no encuentra HTTP o WSS despues de configurar el puerto.
Documentación por archivo
Raíz del proyecto
main.py

Inicializa la aplicación asíncrona. Su responsabilidad es crear la sesión HTTP global, delegar la ejecución a Completer y cerrar correctamente tareas pendientes.
README.md

Documento previo del proyecto. Resume objetivo, estructura, comandos, dependencias y algunos componentes principales. Este archivo nuevo actualiza y amplia esa información para cubrir todos los archivos actuales.
__init__.py

Marca la carpeta como paquete Python.
app/endpoints
ArchivosData.py

Clase ArchivosData. Consume rutas de archivos del backend. Permite crear registros de exportación, cancelar archivos y descargar archivos generados. Al exportar detecta el directorio de descargas del usuario, interpreta Content-Disposition para obtener el nombre del archivo y, si se usa --limpiar-bd con Noticias, ejecuta limpieza posterior de noticias exportadas.

Metodos principales:

    _directorio_descargas(): resuelve la carpeta de descargas.
    insertar_archivo(plataforma): POST /archivos/insertar.
    cancelar_archivo(nombre): PATCH /archivos/archivo/{nombre}.
    exportar_archivo(plataforma, limpiar_bd=False): GET /archivos/exportar/{plataforma}.

EditorialNoticias.py

Clase EditorialNoticiasEndpoint. Encapsula operaciónes editoriales de noticias.

Metodos principales:

    analizar_editorial(fuente=None, días=30): ejecuta análisis manual en /editorialConsulta/analizar_editorial.
    obtener_resumen_editorial(): consulta /editorialConsulta/resumen.
    limpiar_noticias_exportadas(): elimina noticias ya exportadas en /editorialConsulta/noticias/exportadas.

Fuentes.py

Clase FuentesEndpoint. Comunica la CLI con las rutas de administración de fuentes.

Metodos principales:

    encolar_fuentes(fuente): envia fuentes de YouTube a /fuentes/insertar.
    actualizar_prioridad_nombre(nombre, prioridad): intenta actualizar prioridad por nombre.
    actualizar_prioridad_plataforma(plataforma, prioridad): actualiza prioridad por plataforma.
    registrar_fuente_noticias(nombre, url_rss): registra una fuente RSS en /fuentes/noticias.

Notificaciones.py

Clase NotificacionesEndpoint. Consume un stream HTTP desde /notificaciones/consulta usando _request_stream. Cada linea se interpreta como JSON y se entrega con yield para que la vista la muestre en vivo.
Prioridad.py

Clase PrioridadEndpoints. Consulta y actualiza prioridades globales.

Metodos principales:

    obtener_prioridades(): GET /prioridad/prioridades.
    cambiar_prioridad(plataforma, prioridad): PUT /prioridad/prioridades/{plataforma}.

SchedulerNoticias.py

Clase SchedulerNoticiasEndpoint. Administra el scheduler del módulo Noticias.

Metodos principales:

    start_noticias(intervalo_min): inicia scheduler en /schedulerNoticias/start.
    stop_noticias(): detiene scheduler en /schedulerNoticias/stop.
    status_noticias(): consulta estado en /schedulerNoticias/status.

Tareas.py

Clase TareasEndpoint. Consume rutas de tareas programadas.

Metodos principales:

    insertar_tarea(tarea): POST /tareas/tarea/insertar.
    actualizar_tarea(clave, periodo): PATCH /tareas/tarea/{clave}.
    eliminar_tarea(clave): DELETE /tareas/tarea/{clave}.

WebSockets.py

Clase WebSocketsEndpoint. Abre conexiónes WebSocket usando la URL WSS del entorno.

Canales principales:

    /webSockets/encolar_websocket: fuentes encoladas.
    /webSockets/procesadas_websocket: eventos de procesamiento.
    /webSockets/tabla_fuentes: tabla de fuentes.
    /webSockets/tabla_archivos: tabla de archivos.
    /webSockets/tabla_logs: logs en tiempo real.

YoutubeData.py

Clase YoutubeData. Contiene adaptador para datos de YouTube. Actúalmente implementa exportar_videos() contra /youtube/videos/export; las consultas de listado y busqueda se realizan directamente desde YoutubeView usando MetodosGenerales.
___init__.py

Archivo de inicialización del paquete endpoints. El nombre contiene tres guiones bajos al inicio; si se esperaba un inicializador estándar, deberia llamarse __init__.py.
app/view
ArchivosView.py

Vista para generar, cancelar, exportar y observar archivos. Normaliza respuestas del backend, construye tablas de estado y delega la descarga a ArchivosData.
Completer.py

Controla la sesión interactiva completa. Normaliza entradas con shlex, ejecuta comandos internos, valida que los comandos empiecen con dpm, permite ejecutar scripts con run_file() y delega comandos a Dispatcher.
Dispatcher.py

Router interno de la CLI. Recibe el objeto parseado y llama la vista correcta para config, youtube, logs, noticias, archivos, tareas, fuentes, prioridades y help. También solicita datos faltantes de tareas mediante prompts interactivos.
EditorialView.py

Vista de noticias editoriales. Muestra análisis editorial, resumen del módulo, tabla de noticias, limpieza de noticias exportadas y menú para listar/eliminar fuentes RSS.
Fuentes.py

Vista para fuentes de YouTube y Noticias. Valida si una entrada es URL de YouTube o archivo .txt, extrae URLs, envia fuentes al backend, registra fuentes RSS y muestra resultados en tabla.
NotificacionesView.py

Muestra notificaciones recibidas desde stream HTTP. Usa Live y Panel de Rich para presentar mensajes acumulados sin duplicarlos.
Panel_Logs.py

Constructor visual del panel de logs. Formatea niveles, módulos, fechas, conteos y descripciónes. Genera un layout con paneles por módulo y una tabla inferior de logs generales.
ParserCLI.py

Define la gramatica de comandos con argparse. Registra subcomandos de configuración, logs, noticias, archivos, tareas, fuentes, prioridades, YouTube y ayuda.
Prioridad.py

Vista de prioridades. Normaliza datos, muestra tabla de prioridades por plataforma y aplica cambios de prioridad mostrando mensajes según el estado devuelto por el backend.
PromptManager.py

Construye la experiencia interactiva del prompt. Define autocompletadores para EnumTarea, EnumPlataforma y EnumPeriodo, genera un NestedCompleter, configura estilos y muestra ayuda contextual en la barra inferior usando COMMANDS.
SchedulerView.py

Vista del scheduler de noticias. Renderiza tablas simples para inicio, detención y estado del scheduler.
Tareas.py

Vista para tareas programadas. Inserta, actualiza, elimina y lista tareas. La consulta de tareas se hace mediante stream y se presenta en una tabla viva.
WebSocketsView.py

Vista de monitoreo en tiempo real. Escucha canales WebSocket y mantiene snapshots de fuentes, archivos y logs. Clasifica estados, elimina filas exitosas cuando corresponde, normaliza paquetes heterogéneos y actualiza tablas/paneles con Live.
YoutubeView.py

Vista de consulta de YouTube. Lista videos descargados y busca por nombre de video o canal. Consume endpoints de metadatos con stream y renderiza filas con datos de video, canal, enlaces y fechas.
command_help.py

Diccionario COMMANDS usado por PromptManager para mostrar ayuda contextual en la barra inferior del prompt.
__init__.py

Inicializador del paquete view.
app/utils
Configuración.py

Gestiona la configuración local del servicio. Puede iniciar y detener el backend, validar si responde, consultar o modificar puerto, crear .env, descargar/actualizar yt-dlp e instalar ffmpeg mediante el gestor del sistema.
MetodosGenerales.py

Utilidad transversal para operaciónes HTTP y formato de datos.

Metodos principales:

    _get_sesión(): valida la sesión HTTP inyectada.
    close(): cierra la sesión si sigue abierta.
    _request(): ejecuta peticiones HTTP y retorna JSON, texto o archivos.
    _request_stream(): consume respuestas NDJSON por chunks.
    _log_error(): imprime errores HTTP, de conexión, timeout o cliente.
    _redact_urls(): oculta URLs en errores.
    format_date(), format_complete(), format_fecha(): formatean fechas.
    normalize_collection(): normaliza listas o diccionarios usando una funcion de mapeo.
    short(): acorta textos para tablas.

__init__.py

Inicializador del paquete utils.
app/models
Archivo.py

Modelo para archivos exportables. Contiene nombre, estado, plataforma y fecha. Puede convertirse a diccionario o JSON y reconstruirse desde JSON.
ConteoLog.py

Modelo de conteo por nivel de log. Guarda EnumLog y cantidad, permite incrementar y serializar.
ConteoLogModule.py

Modelo de conteo de logs por módulo. Guarda EnumModules y un diccionario de conteos por nivel de log.
Fuente.py

Modelo simple para representar una fuente anterior y una fuente nueva. Incluye transformacion a JSON y reconstrucción desde diccionario.
Log.py

Modelo de log operativo. Guarda clave, módulo, nivel, descripción y fecha en zona horaria America/Mexico_City. Puede serializarse y reconstruirse.
Periodo.py

Modelo de periodicidad. Valida que los valores sean correctos según el periodo: horas entre 1 y 23, meses entre 1 y 12, días y semanas mayores a 0.
Prioridad.py

Modelo Priority para representar enlace, prioridad actual y nueva prioridad. Actúalmente solo almacena valores y no expone metodos de serialización.
Tarea.py

Modelo base de tarea programada. Incluye clave UUID, tipo de tarea, plataforma, fechas de próxima y última descarga, y periodo.
TareaDescarga.py

Extiende Tarea agregando la propiedad Fuente. Se usa para tareas de tipo DESCARGA, donde se requiere URL o fuente origen.
Youtube.py

Modelo para metadatos de video de YouTube: canal, enlaces, fechas, comentarios y subtitulos. Permite construir un diccionario y JSON.
__init__.py

Inicializador del paquete models.
app/enums
EnumEnlace.py

Tipos de enlace soportados: INDIVIDUAL, PLAYLIST, CANAL y RSS.
EnumEstados.py

Estados operativos: PENDIENTE, DESCARGANDO, ERROR, CANCELAR y EXITO. Incluye propiedad descripción para mensajes legibles.
EnumLog.py

Niveles o tipos de log: INFO, SUCCESS, WARNING, ERROR, DOWNLOAD, PROCESS, UPDATE y DELETE.
EnumModules.py

Módulos del sistema: YOUTUBE, NOTICIAS, TAREAS, DESCARGA, ARCHIVOS, NOTIFICACION y FUENTES.
EnumPeriodo.py

Periodos de tareas: HORA, DIAS, SEMANA y MES.
EnumPlataforma.py

Plataformas principales: YOUTUBE y NOTICIAS.
EnumTarea.py

Tipos de tarea: DESCARGA y EXPORTACION.
app/test
main.py

Script manual para probar ejecución de comandos desde archivo mediante Completer.run_file().
main_config.py

Script manual para probar la clase Configuracion, especialmente cambio de puerto.
__init__.py

Inicializador del paquete test.
Otros archivos
app/instrucciones.txt

Plantilla de comandos DPM. Sirve como ejemplo para dpm config agregar_script <ruta>, aunque algunos comandos comentados conservan nombres antiguos y deben alinearse con ParserCLI.py antes de usarse como script operativo.
.gitignore

Define archivos ignorados por Git.
.vscode/

Configuración local del editor. No forma parte del comportamiento de la CLI.
venv/ y __pycache__/

Carpetas generadas localmente. No deben documentarse como código fuente ni versionarse.
Dependencias principales

    aiohttp: sesión HTTP asíncrona compartida.
    httpx: descargas y validaciones puntuales dentro de Configuracion.
    websockets: escucha de canales WebSocket.
    python-dotenv: lectura y actualizacion de variables de entorno.
    rich: tablas, paneles, layouts y vistas en vivo.
    prompt_toolkit: prompt interactivo, autocompletado y barra de ayuda.
    pandas: construccion auxiliar de tablas en Fuentes.
    pytz y zoneinfo: manejo de zona horaria y fechas.

Relación con el backend

La CLI depende de que el backend exponga rutas HTTP y WebSocket compatibles con los endpoints usados. Las URLs base se leen desde variables de entorno:

    HTTP: base HTTP, por ejemplo http://127.0.0.1:2055.
    WSS: base WebSocket, por ejemplo ws://127.0.0.1:2055.
    PORT: puerto local del servicio.

Los principales contratos consumidos son:

    /fuentes/* para fuentes y prioridades por fuente.
    /prioridad/* para prioridades globales.
    /tareas/* para tareas programadas.
    /youtube/* para metadatos de YouTube.
    /schedulerNoticias/* para scheduler de noticias.
    /editorialConsulta/* para noticias, resumen y análisis editorial.
    /archivos/* para exportación y descarga de archivos.
    /notificaciones/consulta para notificaciones.
    /webSockets/* para monitoreo en tiempo real.

Consideraciones de mantenimiento

    Mantener sincronizados ParserCLI.py, PromptManager.py, command_help.py y la tabla de Dispatcher._help; si se agrega un comando en un lugar, conviene actualizar los cuatro.
    Revisar la diferencia entre ~/.dpm/.env y ~/.dpm-base/.env para evitar errores de configuración.
    Evitar duplicar reglas de renderizado; ArchivosView, WebSocketsView y Panel_Logs ya contienen helpers reutilizables.
    Mantener las rutas HTTP en endpoints y no en vistas, excepto en casos legacy como consultas directas de YoutubeView y EditorialView.
    Si se formalizan pruebas, mover los scripts de app/test a una suite con pytest y mocks de HTTP/WebSocket.
    Corregir nombres o textos antiguos en app/instrucciones.txt para que coincidan con los comandos reales: por ejemplo listar en lugar de list, estado en lugar de status en la mayoría de módulos, status en tareas, e iniciar_descarga en lugar de start_scheduler.

    --------------------------------------------------------------------------------------------------------------------------------------------
    --------------------------------------------------------------------------------------------------------------------------------------------

# Forma de trabajo del sistema

El sistema implementa una cola de prioridad para gestionar las solicitudes de descarga de los usuarios. Cada enlace enviado es encolado junto con un nivel de prioridad que determina el orden de procesamiento.

Un componente orquestador se encarga de clasificar los enlaces recibidos y asignarlos a la cola correspondiente según su tipo y prioridad. Los consumidores de eventos procesan las colas de forma asíncrona, ejecutando las descargas sin bloquear la interacción del usuario.

Este diseño permite desacoplar la recepción de solicitudes de su ejecución, mejorando la escalabilidad y garantizando un procesamiento eficiente de las descargas.

![[cola-prioridad.png]]
Arquitectura del sistema

Las arquitectura del sistema se describe mediante múltiples vistas arquitecturas. A continuación se describe cada arquitectura y las razones de su uso en el sistema.
Arquitectura monolítica modular

![[arquitectura-modular.png]]
¿Qué es la arquitectura monolítica?

La arquitectura monolítica es una forma de organizar un proyecto, la cual esta acoplada para trabajar en conjunto, lo cual la hace fácil de utilizar para proyectos pequeños. Las limitaciones que tiene son bastantes, al ir creciendo la aplicación o el sistema no es fácil de escalar el proyecto, por lo tanto darle mantenimiento a este tipo de arquitectura se vuelve un reto.
Monolítico modular

Para solucionar alguno de los inconvenientes de un monolítico se pude desacoplar la lógica del proyecto en base a módulos, el cual permite una mejor distribución de carga de trabajo, darle mantenimiento a una aplicación ya no es tan complicado, al tener varios módulos, se puede trabajar con la parte del sistema a refactorizar, actualizar o incluso a eliminar además es posible escalar este tipo de arquitectura a una arquitectura de microservicios, al tener en varias partes el sistema se puede poner en un servicio independiente.

Para tener un mejor control y poder escalar el proyecto a microservicios en caso de ser necesario se decidió trabajar con la arquitectura monolítica modular.
Vistas de la organización interna del Back-End

Esta vista describe la organización interna del Back-End como un monolito modular, donde cada módulo encapsula un dominio específico del negocio.

Cola-Descarga/
│── app/
│   │
│   ├── core/    #Hilo de conexión para el funcionamiento del sistema        
|   |   ├── enums/
|   |   ├── models/
|	|   ├── app_state.py 
|	|   ├── main.py 
│   │
│   ├── db/      # Conexión a la base de datos (lmdb)       
│   │   ├── Conexion.py
│   ├── modules/ # Modulos del sistema  
│   │   ├── archivos/ 
│   │   ├── coladedescarga/
│   │   ├── fuentes/
│   │   ├── logs/
│   │   ├── noticias/
│   │   ├── notificaciones/
│   │   ├── prioridad/
│   │   ├── tareas/
│   │   ├── websockets/
│   │   ├── youtube/
|   |   ├── tests/ 
│   |   ├── workers/ # Levantar procesos de cada modulo  
│   |   ├── main.py # Archivo principal del proyecto  
|	|	    ├── WorkerManagerService.py
|	|            
│   ├── main.py  # Procesamiento de la API                           
├── .gitignore                # Ignorar carpetas y archivos
├── poetry.lock              # Dependencias con poetry
└── requirements.txt         # Dependencias

Estructuración del proyecto de manera visual para su mejor entendimiento, cada archivo y carpeta es fundamental para elaborar un control mas adecuado el desarrollo y mantenimiento del sistema.
Arquitectura cliente servidor

El funcionamiento para la arquitectura de comunicación con el sistema es una arquitectura cliente servidor, para tener menos riesgos.

El cliente consulta o realizar una operación al servidor, el servidor recibe la pregunta del cliente el cual envía una petición a la base de datos para poder regresar una respuesta al cliente.

En los clientes que pueden comunicarse con el sistema son los siguientes:

    Navegador web
    App móvil
    App de escritorio

El servidor por nuestra parte es una:

    API REST maneja que solicitudes HTTP, WSS y devuelve archivos JSON
    La lógica consiste en enviar validar enlaces estilo URL para retornar la información correspondiente.
    La información será gestionada por medio de consultas desde el servidor a la base de datos, el servidor recibirá una respuesta donde será valida para enviar el mensaje correspondiente al cliente.

El cliente y el servidor se comunicaran por el protocolo HTTP O WSS.

![[arquitectura_cliente_servidor.png]] Se muestra una imagen de la arquitectura cliente servidor del sistema.
Arquitectura por capas

Cada módulo del sistema implementa internamente una arquitectura por capas, donde las responsabilidades se separan en presentación, aplicación, dominio y persistencia. Esta estructura se replica de forma consistente en todos los módulos, favoreciendo la mantenibilidad y escalabilidad del sistema.
Estructura de cada modulo por capas

nombre_modulo/
 ├── routers/        → Capa de Presentación
 ├── services/       → Capa de Aplicación / Negocio
 ├── models/         → Capa de Dominio / Datos
 ├── repositories/   → Persistencia (si aplica)
 ├── utils/          → Soporte transversal

A continuación se muestra la descripción de cada sección de la arquitectura en capas por modulo.
Carpeta 	Capa arquitectónica 	Responsabilidad
routers 	Presentación 	Entrada HTTP, validación básica
services 	Aplicación 	Lógica de negocio
models 	Dominio / Datos 	Entidades, esquemas
enums 	Dominio 	Estados, reglas
repositories 	Persistencia 	Acceso a datos
utils 	Infraestructura 	Funciones comunes
El sistema no esta atado a no crear mas carpetas para el sistema, cada modulo puede tener varias carpetas. 		
Protocolos de comunicación
Protocolo HTTP

El sistema recibe respuestas http para poder comunicarse con la base de datos, entre el cliente y el servidor. Haciendo mas cómodo la forma de enviar y recibir información. Lo cual facilita la forma de migrar de tener un cliente CLI a tener distintos clientes, webs, aplicaciones móviles o de escritorio.

El cliente realiza un petición sea GET, POST, DELETE, PUT, PATCH al servidor. Este analiza el registro el tipo de ruta que se hace para enviar la información solicitada y que el usuario pueda visualizarla.

![[protocolo-http.png]]
Protocolo WSS

Para la notificación de eventos en tiempo real, el sistema incorpora comunicación mediante WebSockets. Este mecanismo permite mantener una conexión persistente entre el cliente y el servidor, facilitando la transmisión inmediata de cambios de estado asociados a las descargas en cola.

Los eventos generados por el procesamiento asíncrono (inicio, progreso o finalización de una descarga) son emitidos por el sistema y enviados a los clientes conectados a través del canal WebSocket. De esta manera, los clientes reaccionan a los eventos sin necesidad de realizar consultas constantes al servidor.

![[protocolo-wss.png]]

