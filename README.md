# Estimación del rendimiento intraparcela del trigo mediante series temporales de Sentinel-2

Estimación del rendimiento de trigo a 10 m de resolución combinando series
temporales de Sentinel-2 L2A, variables climáticas, propiedades del suelo y
descriptores topográficos, sobre el conjunto de datos YieldSAT: 175 parcelas de
trigo de invierno en Alemania, campañas 2016–2022.

## Metodología

- **Control de calidad.** De las 188 parcelas que aparecían en el dataset
  originalmente, se fueron filtrando y descartando. En primer lugar, se
  filtraron las parcelas por la calidad del mapa de rendimiento procedente de la
  cosechadora, quedando 175. A continuación, se filtraron y descartaron los
  píxeles poco fiables a través del error estándar relativo, con un umbral del
  0,1, junto con un filtro "biológico" de plausibilidad de los rendimientos de 0
  a 13 t/ha.

- **Normalización fenológica.** Puesto que las parcelas no se habían sembrado en
  la misma fecha, para tener un eje común en el tiempo a todas las parcelas, se
  optó por emplear la acumulación térmica a través de la integral térmica,
  dividiendo el ciclo de cultivo de las parcelas en tramos de 300 grados-día
  acumulados desde la fecha de siembra hasta la recolección. De cada tramo se
  obtiene un compuesto por mediana, previo enmascaramiento de nubes con la capa
  SCL, y se descartan los tramos que cubren menos del 40 % de la parcela.

- **Variables explicativas.** Seis índices espectrales por ventana, agregados
  climáticos, ocho capas de suelo de SoilGrids y descriptores topográficos
  derivados del modelo digital de elevaciones: pendiente, exposición, curvatura
  e índice topográfico de humedad.

- **Modelo.** Se escogió el LightGBM por su rendimiento en datos tabulares, su
  tolerancia a valores ausentes y su coste computacional, que permite repetir la
  validación cruzada sobre 148.000 píxeles en tiempos razonables.

- **Validación.** Tres niveles de agrupación, cada uno más exigente que el
  anterior según qué información comparten entrenamiento y prueba:
  - *Píxel*: partición aleatoria. Píxeles del mismo campo caen a ambos lados,
    así que el modelo evalúa parcelas que ya ha visto. Es el escenario optimista
    y sirve de referencia, no de medida.
  - *Parcela*: cada campo-año va entero a un lado. El modelo no ha visto esa
    parcela, pero sí otras de la misma explotación y la misma parcela en otras
    campañas.
  - *Explotación*: todos los campos de una finca, de todas las campañas, van
    juntos. El modelo se enfrenta a una explotación completamente nueva. Es el
    único nivel que responde a si el modelo serviría en una finca distinta.

## Diagnóstico

El modelo de rendimiento absoluto presentó una degradación progresiva de las
métricas conforme se endurece el criterio de agrupación, hasta situarse por
debajo de predecir la media.

| Agrupación | R² | MAE (t/ha) | r de Pearson |
|---|---|---|---|
| Píxel | +0,70 | 0,82 | +0,83 |
| Parcela | +0,47 | 1,16 | +0,67 |
| Explotación | **−0,33** | 1,85 | +0,34 |

Este último es el caso de la agrupación por explotación: cuando el modelo se
evalúa sobre explotaciones completas que no ha visto durante el entrenamiento,
el rendimiento se desploma, lo que indica que parte de su acierto procedía de
reconocer la unidad de gestión y no de interpretar el estado del cultivo.

Esto puede deberse a que una fracción considerable de las variables explicativas
no describe el estado del cultivo en cada píxel sino la identidad del campo que
lo contiene.

De las 71 variables, 30 son constantes dentro de cada parcela: todas las
climáticas, que proceden de una única serie por campo. A ellas se suma la
elevación, que varía 3,9 m dentro de una parcela frente a los 140 m que separan
las explotaciones, y que resulta ser la variable más influyente del modelo.
Eliminarlas eleva el coeficiente agrupado por explotación hasta +0,05, mejora
insuficiente que sitúa la causa en la formulación del objetivo y no en la
selección de variables.

## Reformulación

La descomposición de la varianza del rendimiento sitúa la mayor parte en la
componente interna de las parcelas, por encima de la que separa unas de otras.
Son componentes de naturaleza distinta. El nivel medio de un campo lo fijan la
variedad, la dosis de fertilización y el manejo, variables que el conjunto de
datos no recoge, y el muestreo es de una observación por parcela. La variación
interna, en cambio, queda registrada en la serie temporal de reflectancia con
varios miles de observaciones por campo. De las dos, solo esta última es
explicable con la información disponible.

El objetivo se reformula entonces para aislarla, y pasa a ser el rendimiento
relativo intra-parcela: el rendimiento de cada píxel dividido por la mediana de
su parcela, y cada variable explicativa centrada restándole la mediana de la
suya. La división elimina el nivel del campo, que era la parte no explicable, y
el centrado deja a cero las 30 variables constantes, de modo que el filtro de
varianza las descarta sin necesidad de seleccionarlas a mano.

## Resultados

| Agrupación | Absoluto | Sin identificadoras | Relativo |
|---|---|---|---|
| Píxel | +0,70 | — | +0,69 |
| Parcela | +0,47 | +0,42 | +0,42 |
| **Explotación** | **−0,33** | **+0,05** | **+0,27** |

Al cambiar el objetivo del modelo de rendimiento absoluto a rendimiento relativo
por parcela, las métricas obtenidas respecto a la validación aleatoria apenas
varían, como se puede ver en la tabla. Algo similar ocurre con la validación por
parcela, donde el rendimiento del modelo relativo apenas varía con respecto al
modelo de rendimientos absolutos. Sin embargo, cuando se realiza la validación
por explotación, el modelo relativo funciona bastante mejor respecto al
absoluto.

Sin embargo, para el modelo de rendimiento relativo, que trata de zonificar la
parcela según su rendimiento, el R² y otras métricas no son las más
informativas. Para ver si el modelo reproduce el orden correcto de los píxeles
dentro de un campo, se empleó la correlación de Spearman: mediana de 0,51, con
el 92 % de las parcelas por encima de 0,3.

| Correlación de Spearman intra-parcela | |
|---|---|
| Mediana | +0,51 |
| Parcelas con rho > 0,3 | 92 % |
| Parcelas con rho < 0 | 1 % |

El NDWI y el NDRE del tramo 1800–2400 GDD corresponden a la fase de llenado del
grano, y el análisis de importancia por permutación los sitúa como las variables
más determinantes del modelo, lo que señala el estado hídrico en esa fase como
principal factor del rendimiento dentro de un campo. La posición
topográfica relativa queda en posiciones intermedias.

## Limitaciones

La media de los píxeles de una parcela y el rendimiento que declara el
agricultor solo comparten un 30 % de varianza, con un sesgo de +1,08 t/ha. El
propio dato de referencia tiene error, y eso pone un techo a lo que se puede
afirmar sobre rendimiento real.

## Contenido

`nb_rend_yieldsat.ipynb` recorre el trabajo completo en diez secciones.

**Dependencias:** numpy, pandas, scipy, matplotlib, rasterio, pysheds,
scikit-learn, lightgbm.

Los datos no se incluyen. La variable `ruta` del notebook apunta al directorio
local del conjunto YieldSAT.
