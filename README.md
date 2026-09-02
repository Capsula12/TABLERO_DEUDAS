# Tablero de Deuda de Consumo

Sitio estático que muestra cómo está endeudada la gente según la Central de
Deudores del BCRA. Corre entero en el navegador: DuckDB-WASM levanta unos
parquets agregados y responde las consultas del lado del cliente. No hay
backend ni API.

Se publica en **https://labancaria.org/deudores/**. **El acceso es abierto**: se
entra por el link y listo. Hasta agosto de 2026 los datos iban cifrados y el
sitio pedía una contraseña; se sacó.

Los genera el repo privado `DEUDAS_ANALISIS`, que es donde vive la base con los
datos personales. Acá sólo llegan agregados. **Este repo también es privado**
desde septiembre de 2026: lo que es abierto es el sitio, no el código.

---

## Archivos

```
index.html      el sitio entero — HTML, CSS y JS en un solo archivo
index.json      qué períodos hay disponibles y cuál es el default

AAAAMM/         una carpeta por período
  tablero_cubo_general.parquet          pestaña General
  tablero_cubo_caba_cp.parquet          mapa de CABA por barrio (carga diferida)
  tablero_entidades.csv                 pestaña Entidades
  tablero_entidades_irregularidad.json  irregularidad por tipo de acreedor
  tablero_red.parquet                   simulador Modelo RED (hoy oculto)
  tablero_metadata.json                 totales y listas para los filtros

assets/         lo que no depende del período
  d3.v7.min.js                 build completo de D3
  provincias.geo.js            24 provincias
  partidos_pba.geo.js          135 partidos de Buenos Aires
  barrios_caba.geo.js          48 barrios de CABA
  caba_cp_barrio_weights.js    reparto CP4 → barrio

historico/      la serie histórica, una sola para todo el sitio
  hist_por_provincia.parquet   por período × provincia × partido
  hist_por_edad.parquet        por período × rango etario
  hist_por_tramo.parquet       por período × tramo de deuda (hoy no se baja)
  hist_flujo_mora.csv          cómo se mueve la mora (hoy no se grafica)
  hist_totales_mes.csv         un renglón por mes
  hist_metadata.json           qué generación aportó cada mes
  ipc.csv                      IPC del INDEC, para deflactar
```

Todo esto se versiona tal cual. Son ~131 MB de agregados: no hay ninguna fila
que identifique a una persona.

**Nunca publicar** `hist_persona.parquet`, `deudores_consumo.parquet` ni
`deuda_persona_entidad.parquet`: tienen CUILs. Viven en `Otras/` del repo padre,
el pipeline no los copia acá y el `.gitignore` los nombra igual por las dudas.

`index.json` es el manifiesto de períodos:

```json
{ "periodos": ["202605", "202606"], "default": "202606",
  "actualizado": "2026-08-08 21:27:00" }
```

Al abrir, el sitio usa `?periodo=AAAAMM` si viene en la URL y está en el
manifiesto; si no, el `default`. El selector de la cabecera cambia el
querystring y recarga la página entera — a propósito, porque DuckDB-WASM tiene
los archivos registrados con alias y reiniciar es más seguro que reemplazarlos
en caliente.

---

## Las pestañas

**General** — la foto del período elegido. KPIs, el mapa coroplético y después
los cortes por tramo de deuda, tipo de acreedor, provincia, edad, sexo y
situación. Los filtros de la izquierda afectan todo.

Los KPIs son cinco, y el último —**deuda por deudor**— es el único que no crece
con el tamaño del recorte: sirve para comparar un filtro contra otro. El KPI de
morosos agrega el "1 de cada N", que dice lo mismo que el porcentaje pero es la
forma en que la gente cuenta gente.

Desde agosto de 2026 la métrica de entrada de casi todos los cortes es **% de
mora sobre personas** (mapa, tramo, provincia, edad, sexo, y también el heatmap
de Cruces). Antes era la deuda en pesos, y ahí el gráfico terminaba mostrando
dónde hay más plata en vez de dónde la gente está peor: los porcentajes son
comparables entre unidades de cualquier tamaño y los pesos no. La deuda sigue a
un click, en el mismo selector.

El card **Por sexo** tiene una casilla *Otros*, apagada por default. El padrón
sólo distingue varón y mujer; el bucket "Otro/SD" son unas 4 mil personas sobre
20 millones y dibujado al lado de los otros dos es una barra invisible que
además estira el eje. La nota al pie dice siempre cuánta gente queda afuera.

El renglón **"Sin dato"** de provincia (unas 30 mil personas sin domicilio en el
padrón) queda fuera del card "Por provincia" salvo que el filtro lo pida
explícitamente: tiene la morosidad más alta de la tabla, así que ordenando por
porcentaje encabezaba el ranking como si fuera un distrito, y el mapa ni
siquiera puede dibujarlo.

El toggle **modo excluyente** del filtro por tipo de acreedor cambia la
pregunta: apagado muestra a las personas que deben a alguno de los tipos
elegidos (con toda su deuda); prendido muestra sólo la deuda con esos tipos.
En ese modo el conteo de morosos queda un poco alto, porque el cubo no guarda
la peor situación desagregada por tipo.

El card "Por tipo de acreedor" tiene dos vistas: **por tipo**, donde quien debe
a varios cuenta en cada uno (la suma da más que el total), y **por
combinación**, donde cada persona cae en un solo grupo exclusivo.

Ese card también arranca en % de deudores en mora, y con una métrica en
porcentaje **el anillo no se dibuja**: un doughnut reparte un todo entre sus
partes, y las tasas de mora de banco, financiera y PNFC no son partes de nada
(16% y 32% no suman 48% de nada). Quedan la leyenda y las barras, que sí sirven
para comparar tasas, y las barras se estiran para ocupar el lugar del anillo.

### El drill del card "Por provincia"

Elegir **una sola** provincia de las dos que tienen detalle propio abre ese
detalle en el mismo card, con un selector de nivel al lado del título:

| Filtro | Nivel grueso | Nivel fino | De dónde sale |
|---|---|---|---|
| sólo *Buenos Aires* | 9 regiones | 135 partidos | `region_pba` / `partido_pba` del cubo general |
| sólo *CABA* | 15 comunas | 48 barrios | cubo por CP + reparto en el navegador |

El nivel elegido se recuerda por provincia (`M.prov.nivelPBA` y
`M.prov.nivelCABA`), así que ir y volver no pierde la elección. Las regiones
salen en su orden de referencia; partidos, comunas y barrios se ordenan por la
métrica.

CABA es el caso especial: el cubo general no baja del nivel provincia, así que
el detalle viene del mismo lugar que el mapa de barrios —`tablero_cubo_caba_cp.parquet`
repartido con los pesos CP→barrio— y por eso la primera vez que se abre hay que
esperar la descarga del cubo (~5,5 MB, una sola vez). Las comunas no son otro
reparto: los pesos ya traen la comuna de cada barrio, así que es la misma cuenta
agrupada más grueso. Los valores son **estimaciones por reparto, no recuentos**,
y la nota del card lo dice.

**"Sin clasificar".** Los dos drills tienen un renglón así, y no es residual:

- en PBA son los CPs que no resuelven a ningún partido (~31 mil deudores);
- en CABA son los que **no tienen CP informado** en el padrón (4,4% de los
  deudores de la Ciudad) más los que tienen un CP **sin barrio asignado** en la
  tabla de reparto (1,3%): en total ~5,7%.

Esos casos **no aparecen en el mapa**, porque no hay polígono al que asignarlos.
El porcentaje que muestra la nota del card se calcula sobre las filas del
recorte activo, no está escrito a mano, así que acompaña a los filtros y a los
cambios de período.

### El mapa

Va **justo debajo de los KPIs**, arriba de todos los cortes por dimensión
(`drawGeoMap()` en `index.html`): es la entrada a la pestaña, no el cierre. D3
con `geoMercator`, unión por **nombre** contra el geojson que corresponda,
tooltip con el valor y el puesto en el ranking, y leyenda. Respeta los filtros.

El selector **Ver** abre tres escalas. Cambian los polígonos, las filas y los
textos; el resto del dibujo —paletas, cuantiles, trama de "sin datos", tooltip,
ranking— es el mismo código para las tres.

**Zoom.** Los polígonos viven dentro de un `<g>` y el zoom es una `transform`
sobre ese grupo (`d3.zoom`), así que acercarse no reproyecta ni redibuja nada.
Los bordes van con `vector-effect: non-scaling-stroke`: sin eso, a 8× la línea
tapa al partido. Se puede acercar con los botones **+ / −**, con **Ctrl + rueda**
o pellizcando con dos dedos. Dos decisiones de gesto, ambas para no pelearse con
el scroll de una página larga:

- La rueda sola **no** acerca: scrollea la página, y un aviso efímero explica
  que hay que sumar Ctrl.
- Un dedo arrastra el mapa **sólo si ya está acercado**. En vista completa ese
  gesto es el scroll de la página. El `touch-action` del contenedor lo maneja el
  JS y acompaña ese cambio (`pan-y` ↔ `none`).

En la vista de partidos hay además un atajo **Conurbano**, que encuadra los 24
partidos del GBA: al lado del resto de la provincia son ilegibles, y era el
motivo más frecuente para querer acercarse.

**Referencias.** La leyenda se puede cerrar y volver a abrir. En pantallas de
menos de 760 px arranca cerrada —tapaba el sur del mapa— y en su lugar queda un
botón *Referencias*. Si el usuario la abre o la cierra a mano, esa decisión
manda y deja de auto-ajustarse.

**Ficha de la unidad.** Al tocar una provincia, partido o barrio se abre un panel
con todo lo que el color no puede mostrar: la métrica activa con su puesto en el
ranking y cuánto pesa sobre el total del recorte (o, si es un porcentaje, la
distancia en puntos contra el conjunto), las seis métricas juntas y la deuda
media por deudor, que no está en ningún otro gráfico. Desde ahí se puede acercar
a esa unidad o saltar a su detalle (Buenos Aires → partidos, CABA → barrios). En
pantalla grande flota sobre el mapa; en celular pasa a ser un bloque debajo, para
no tapar nada. En pantalla táctil no hay hover, así que esta ficha es además la
única forma de leer los valores: el tooltip se suprime cuando el último puntero
fue un dedo.

| Vista | Polígonos | De dónde salen los datos |
|---|---|---|
| **País por provincia** | `assets/provincias.geo.js`, 24 | `GROUP BY provincia` sobre el cubo general. Inset aparte para CABA. |
| **Buenos Aires por partido** | `assets/partidos_pba.geo.js`, 135 | `GROUP BY partido_pba` sobre el mismo cubo. |
| **CABA por barrio** | `assets/barrios_caba.geo.js`, 48 | `GROUP BY cp4` sobre `tablero_cubo_caba_cp.parquet` + reparto a barrio en el navegador. |

Los tres geojson los genera `Scripts/herramientas/generar_geo_mapas.py` en el
repo padre (partidos del WFS del IGN, barrios del GCBA) y no se tocan mes a mes.

**El domicilio es el fiscal.** La provincia sale del padrón de ARCA, que guarda
el domicilio declarado y no siempre el real: CABA queda algo sobredimensionada
frente al conurbano. Afecta al reparto de personas entre distritos, no de manera
apreciable a los porcentajes, que son los que el mapa colorea por default. La
nota al pie del mapa del país lo dice en un renglón (`MAP_GEOS.pais.aviso`).

**Partidos.** El cubo ya traía la columna `partido_pba` para el card de barras;
lo único que faltaba era la geometría. Los nombres del IGN calzan exacto con los
del cubo porque ambos vienen de la misma nomenclatura (georef/IGN). *Lezama*
aparece siempre con trama de "sin datos": no tiene código postal propio en las
fuentes postales —se separó de Chascomús en 2009 y el padrón sigue registrando a
sus vecinos con el CP 7130, que es Chascomús—, así que sus deudores están
contados adentro de Chascomús. Es el dato que no existe, no un filtro.

**Barrios de CABA.** El cubo de CABA es un archivo aparte por período y se baja
**sólo la primera vez que hace falta** —el mapa de barrios o el drill del card
"Por provincia", lo que ocurra primero: son ~5,5 MB que no tiene por qué pagar
quien nunca los abre.

El padrón del BCRA guarda sólo el código postal de 4 dígitos, y en CABA un CP
cubre varios barrios (el `1407` toca doce). Así que los valores por barrio son
**estimaciones por reparto**, no recuentos: cada CP se distribuye entre sus
barrios según el peso de sus cuadras, con los pesos de
`assets/caba_cp_barrio_weights.js`. El reparto se hace en el navegador para que
un filtro de sexo o de edad se reparta con el mismo criterio que usó el pipeline
offline.

El reparto **conserva el total de cada CP** (se renormaliza, porque las
fracciones vienen redondeadas a 4 decimales), pero eso no quiere decir que la
suma de los 48 barrios dé el total de la Ciudad: **no lo da**. Quedan afuera los
deudores sin CP informado en el padrón (4,4%) y los de CPs que la tabla de
reparto no cubre (1,3%) — ~5,7% en total. En el mapa esos casos simplemente no
tienen polígono; en el card "Por provincia" con el drill de CABA aparecen como
"Sin clasificar", con su porcentaje calculado sobre el recorte activo.

De dónde salen esos pesos, con qué datos se derivan y cómo se validaron está en
`Scripts/referencia/LEEME_cp_barrio_caba.md` del repo padre. Resumen: se cruza el
callejero del GCBA con el CPA del Correo, y cada cuadra vota por el barrio de su
vereda. La ponderación por población censal se probó y **empeoraba** el
resultado, así que no se usa.

Si los filtros dejan a la Ciudad o a la Provincia fuera del recorte, todos los
polígonos van con trama y la leyenda lo dice, en vez de mostrar una escala vacía.

La escala de color tiene dos decisiones que conviene no perder de vista:

**El dominio arranca en el mínimo observado, no en cero.** Los porcentajes de
mora se mueven en una franja angosta —las provincias van de ~28% a ~38%— y
anclando la escala en cero salían todas casi del mismo color. Con el mínimo como
piso, esos 10 puntos de diferencia ocupan la paleta entera. El costo es que el
color siempre es **relativo al recorte que se esté mirando**: cambia con los
filtros, y "verde" quiere decir "la más baja de este conjunto", no "baja" en
términos absolutos. Por eso la leyenda muestra los dos extremos y el tooltip
agrega la posición en el ranking (`13° de 24`).

**Tres paletas, según qué mide la métrica** (`MAP_RAMPS`):

| Familia | Métricas | Colores |
|---|---|---|
| `riesgo` | los dos `% Mora` (deuda y personas) | verde → amarillo → rojo |
| `mora` | `Deuda en mora`, `N° morosos` | rojo secuencial (claro → oscuro) |
| `volumen` | `Deuda total`, `N° deudores` | celeste → navy |

El verde/rojo se reserva para los porcentajes, que sí son comparables entre
provincias. Los agregados de mora en pesos o personas van en rojo secuencial sin
verde: ahí "más" está confundido con el tamaño de la provincia, y pintar de verde
a la más chica diría algo que el dato no dice. (La rampa verde-rojo es la RdYlGn
de ColorBrewer invertida; no es a prueba de daltonismo, y por eso el tooltip y el
ranking están para leer el número exacto.)

El selector **Por cuantiles / Lineal mín–máx** del encabezado:

- **Por cuantiles** (default) parte las provincias en 6 grupos de igual tamaño,
  uno por color. Funciona igual de bien con datos apretados (% de mora) que muy
  sesgados (deuda total, donde Buenos Aires aplasta al resto). Exagera
  diferencias chicas si los datos son parejos: el rango de cada grupo está en el
  `title` de su escalón de la leyenda.
- **Lineal mín–máx** es fiel a la distancia numérica. En las magnitudes
  (`volumen`) usa raíz cuadrada, para que el sesgo no deje a media Argentina en
  el color más claro.

Casos borde ya contemplados: provincias fuera del recorte van con trama diagonal
gris ("sin datos"); si queda una sola provincia o todas tienen el mismo valor, se
pinta un tono medio y la leyenda dice "un solo valor". Cambiar la escala sólo
redibuja, no vuelve a consultar DuckDB; cambiar la métrica sí dispara
`refreshGeneral()`.

**Cruces** — cualquier par de dimensiones contra cualquier métrica, como
heatmap o como barras apiladas. Arranca en **% Mora (personas)**.

**Histórico** — la serie mes a mes, siempre sobre **deuda total informada**: todo
lo que la persona debe, medido igual en todos los meses, que es lo que la vuelve
comparable en el tiempo. Un solo toggle, *ajustar por IPC* (pesos constantes), y
dos advertencias que el frontend muestra solo, **al pie de la pestaña** (arriba
empujaban los gráficos abajo del pliegue, y son avisos para interpretar lo que ya
se vio):

- La línea punteada de julio de 2024 marca el mes en que el BCRA subió de $1.000
  a $25.000 el importe mínimo que las entidades tienen que informar a la Central
  de Deudores. Se cayeron 5,4 millones de registros y 3,1 millones de personas de
  un mes al otro, casi todas con deudas chicas (mediana $12.000), y más de un
  millón de ellas estaba en mora. **No cambió quién informa**: las entidades
  pasaron de 432 a 454. Se compara dentro de cada tramo, no entre tramos. El
  detalle y la verificación están en el `leeme.md` del repo padre.
- El último mes es provisorio: hay entidades que informan con un mes de atraso
  y se las completa con su último dato.

El toggle *consumo puro* (dejar afuera hipotecas y prendas, el recorte de la
pestaña General) está **oculto desde agosto de 2026**: en los meses anteriores a
la primera base propia es una estimación que se queda cada vez más corta, y
convivir con la serie comparable confundía más de lo que aportaba. La lógica
sigue entera (`H.excluirHip`, `histDeudaField()` y compañía) y los datos se
siguen publicando: para reactivarlo hay que descomentar el bloque del toggle en
la sidebar, el aviso `#hist-hip-info` del banner y las dos líneas marcadas en
`initHistoricoControls()` y `refreshHistorico()`.

#### El encabezado del Histórico

Cinco KPIs, cada uno con dos variaciones: contra el mes anterior y contra el
mismo mes del año pasado. El nivel solo envejece rápido y no dice a qué
velocidad se mueve nada; las variaciones sí.

| KPI | Variación |
|---|---|
| Deuda total | % (aclara si es en términos reales o nominales) |
| Deudores | % |
| Deudores en mora | % |
| % de la deuda en mora | puntos porcentuales |
| % de deudores en mora | puntos porcentuales |

El color de cada pill no es neutral: **subir es la mala noticia**, así que verde
es baja, gris es sin cambios de magnitud y ámbar/rojo es suba. Los umbrales
están en `pillVar()`.

#### Los gráficos del Histórico

- **Peor situación BCRA de los deudores** (área apilada) reemplazó al viejo
  apilado por tramo de deuda. Cuenta **personas**, no pesos, y arranca en
  **% del mes**: cinco colores fijos, uno por situación, los mismos del resto
  del tablero. El tramo decía cuánta plata hay arriba de cada franja; esto dice
  cuánta gente pasó de estar al día a estar en problemas.
- **Por provincia** y **por rango etario** ahora grafican el **% de deudores en
  mora**, no la deuda. Con pesos, Buenos Aires siempre arriba y el resto
  amontonado abajo; con porcentaje se ve quién está peor y no quién es más
  grande. En provincia se excluye "Sin dato".
- **Cómo se mueve la mora** está **oculto desde agosto de 2026**, apenas
  después de estrenarse. Descomponía la variación del stock de morosos en siete
  series que cerraban exactamente contra el total, y seis de ellas están bien.
  La séptima no: la "baja como incobrable" afirma más de lo que el dato puede
  sostener. La Central de Deudores informa deuda vigente y no publica ningún
  evento de "canceló" ni de "castigó" — las dos cosas se ven igual, la persona
  deja de ser informada y listo. Separar por la situación previa (5 contra 3 o
  4) es una inferencia razonable, no una medición, y no alcanza para ponerle
  "incobrable" a una barra de un tablero público.

  Todo lo demás sigue entero: `pipeline/7_flujo_mora.py` se sigue corriendo,
  `hist_flujo_mora.csv` se sigue publicando, y la carga y el dibujo siguen en el
  `index.html`. Para reactivarlo hay que descomentar el bloque del card y la
  llamada a `drawHistFlujoMora()` en `refreshHistorico()`. Lo que faltaría para
  que se sostenga es una forma de distinguir cancelación de castigo: los campos
  `irrecuperables`, `previsiones` y `sit_juridica` de `deudores` son el lugar
  donde buscarla.
- **Cuánto debe cada deudor** son dos líneas, deuda total sobre deudores y deuda
  en mora sobre morosos. Es lo único de la pestaña que no crece cuando entra
  gente nueva a la base, así que separa "hay más deudores" de "cada uno debe
  más".
- El gráfico de cierre, **"Las dos moras"**, superpone el % de deudores en mora
  y el % de la plata en mora. La brecha entre las dos líneas es el punto.

Como el apilado por tramo ya no existe, `hist_por_tramo.parquet` **no se baja
más** (eran 1,2 MB por visita). El archivo se sigue publicando por si vuelve un
gráfico por tramo; la línea está comentada en `loadData()`.

**Entidades** — irregularidad por tipo de acreedor y el listado entidad por
entidad. La sección de arriba mide cuántos **deudores** están en mora; el BCRA
publica cuánta **plata** está en mora. Son preguntas distintas y la primera da
siempre más alto. Si falta el JSON de irregularidad, la sección se oculta sola.

La pestaña se dibuja en el arranque, con su sección todavía oculta, así que sus
charts nacen midiendo 0x0 y quedan a merced del `ResizeObserver` de Chart.js
para acomodarse cuando la pestaña aparece. `setupTabs()` los redibuja al entrar
(`renderIrregularidadEnt()` y `renderEntTop()`) para no depender de eso: cuesta
nada, porque los datos ya están en memoria. Cualquier chart nuevo en una pestaña
que no se refresque sola conviene que haga lo mismo.

Arriba del listado hay un ranking de **los 10 acreedores con más gente en
mora**, con un selector para pasar a los 10 con más plata en mora. Las dos
listas no son la misma y ese es el motivo del card: en 2026/06 MercadoLibre es
primero por personas y octavo por monto. Respeta la pestaña de tipo (bancos,
financieras, PNFCs) pero no el buscador, que dejaría el ranking en una sola
barra. La nota calcula cuánto concentran esos diez sobre el recorte activo;
como una persona puede estar en mora con varias entidades, la suma no es un
recuento de personas distintas.

Los valores publicados por el BCRA que se muestran como contraste salen de
`BCRA_REFERENCIA`, en `Scripts/pipeline/5_entidades.py` del repo padre, y se
actualizan a mano. Familias, empresas y total del sector privado están en el PDF
del Informe sobre Bancos; el desglose por línea (personales, tarjetas,
hipotecarios, prendarios) sólo está en el anexo estadístico, que suele ir un mes
atrás. Por eso el payload tiene `periodo_familias` y `periodo_resto` separados y
el frontend los muestra como dos frases distintas cuando no coinciden.

**Modelo RED** — oculta desde julio de 2026. Para volver a activarla hay que
descomentar el botón de la pestaña y las llamadas correspondientes en
`loadData()` y `bootApp()`; los datos se siguen publicando.

---

## Ajuste por IPC

Se aplica en el frontend, no en los datos. El período base es el más reciente
de la serie, así que los montos quedan en pesos de hoy:

```
monto_ajustado = monto × (IPC_base / IPC_mes)
```

La serie sale de `IPC/ipc_indec_nacional_nivel_general.csv` en el repo padre y
la actualiza `Scripts/pipeline/1_ipc.py`, que corre como primer paso de cada
regeneración.

---

## Acceso abierto y privacidad

Hasta agosto de 2026 los datos se publicaban cifrados (AES-256-GCM con clave
PBKDF2) y el sitio pedía una contraseña. Se eliminó: el tablero se entra por el
link, sin gate.

Ese cifrado nunca fue una medida de privacidad de datos personales — acá no hay
datos personales. Era una restricción de acceso, y tenía los límites de siempre:
un secreto compartido, imposible de revocar por persona, y sujeto a fuerza bruta
offline sobre archivos que cualquiera podía bajarse. Sacarlo simplifica el
pipeline y hace el tablero usable desde cualquier lado.

Lo que sí sigue firme es qué se publica. Todo lo que llega a este repo son
**agregados**: cubos con conteos y montos por combinación de dimensiones. La
mínima unidad geográfica es el código postal, y la mínima temporal el mes. Los
parquets a nivel persona, que tienen CUILs y son datos personales (Ley 25.326),
quedan en `Otras/` del repo privado y el pipeline no los toca.

---

## Publicar

Son dos pasos y el segundo no lo hacemos nosotros.

**1. Pushear.** Lo normal es correr `publicar_tablero.bat` desde el repo padre.
A mano:

```bash
cd TABLERO_DEUDAS
git add -A
git commit -m "Período AAAAMM"
git push origin main
```

**2. Pullear en el servidor.** El sitio es un clone de este repo en el servidor
de La Bancaria, y no se actualiza solo: alguien de sistemas tiene que correr el
`git pull`. Hasta que eso pase, el push no cambió nada de lo que se ve. El clone
autentica con un deploy key de solo lectura, así que el servidor puede leer este
repo y nada más.

Hasta septiembre de 2026 esto también se publicaba en GitHub Pages, servido
desde la rama `page`; al pasar el repo a privado la page se despublicó. La rama
sigue existiendo pero quedó congelada. El `.nojekyll` es un resto de esa época:
no molesta y se deja por si alguna vez vuelve.

---

## Verlo localmente

```bash
python -m http.server 8000
```

Y abrir http://localhost:8000/. Tiene que ser por HTTP: DuckDB-WASM usa workers
y no funciona abriendo el archivo directo.

Todos los períodos están en claro en el repo, así que cualquier cambio se puede
probar contra los datos reales sin más trámite.

### Probar la lógica de dibujo

Todo el JS del tablero vive en un solo `<script type="module">`, así que sus
funciones **no** quedan en `window` y no se las puede llamar desde la consola.
Para barrer casos borde de un gráfico conviene armar un banco de pruebas aparte:
un HTML servido desde la raíz del repo (para que resuelvan `assets/d3.v7.min.js`
y los `*.geo.js`) que **extraiga textualmente** del `index.html` los bloques que
hacen falta —el CSS del mapa, `METRICS`, los formatters, `computeValueFn` y la
sección del mapa— y los corra como script clásico contra filas sintéticas
`{ k, nd, nm, dt, dm }`, que es la forma exacta que devuelven las consultas.

Extraer en vez de copiar y pegar importa: se prueba el código que se editó, no
una copia que envejece. Vale barrer de una las tres escalas del mapa, todas las
métricas, las dos escalas de color, unidades faltantes, todas iguales y una
sola, y mirar cuántos colores distintos salieron y qué dice la leyenda. El
archivo del banco es temporal y el `.gitignore` cubre `_*.html`.
