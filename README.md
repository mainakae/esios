# Video explicativo de como utilizar este software

https://youtu.be/oh0Lvhbh_M4

# Cómo calcular el gasto eléctrico en la tarifa PVPC

* [Link web término de facturación de energía activa](https://www.esios.ree.es/es/pvpc)
* [Link web descarga ESIOS](https://www.esios.ree.es/es/descargas?date_type=publicacion&start_date=01-06-2021&end_date=01-06-2021)
* [Link BOE cálculo de tarifas autoconsumo](https://www.boe.es/buscar/act.php?id=BOE-A-2014-3376)

Para consultar en formato web en tiempo real los precios para las tarifas de autoconsumo etc: utlizar el primer link. Inspeccionando la conexión, podemos encontrar la petición al servicio web donde descargamos bastantes datos sobre las tarifas de autoconsumo, aunque la mayor cantidad de información (y mejor formateada que he encontrado) se puede obtener desde el área de descargas.

El segundo link, del área de descargas, nos permite descargar ficheros históricos y el precio de la luz subastado para el día siguiente (a partir de las 20h). Cada día viene en un fichero .xls, por lo que cuando solicitamos un rango de tiempo de más de un día, nos devuelve un zip comprimido con tantos ficheros xls como días hayamos solicitado.

## Recomposición y carga de históricos

Para la descarga de los precios de la luz, realizaremos búsqueda por **"DATOS"** y seleccionaremos el periodo de búsqueda (fecha inicio y fin). De esta forma es posible descargar un fichero comprimido con todos los días del periodo en .xls. He utilizado esta técnica para descargar los históricos de 2018 a 2021, que posteriormente he importado en InfluxDB utilizando Python, la librería de InfluxDB, Pandas (para leer xls y trabajar con ellos) y algunas librerías de manejo de timestamps para el manejo del DST (Daylight Savings Time), etc. El código se encuentra disponible en un [cuaderno de Jupyter](./work/Untitled.ipynb), que se puede ejecutar y jugar con el por medio de un contenedor Docker (ver [docker-compose](./docker-compose.yml) adjunto). La única particularidad es que es preciso instalar la librería de influxdb en el contenedor de Jupyter antes de poder ejecutar el cuaderno.

```bash
# Para instalar influxdb ejecutar esto
docker-compose exec jupyter bash -c "pip install influxdb-client[extra]"

# Y listo
```

## Actualización diaria de precios de subasta

Una vez descargados los históricos, es conveniente tener un proceso en marcha para su actualización diária, conforme se publiquen los datos de subasta para el día siguiente.

Para ello he creado un flujo de Node-Red, que utilizando algunos nodos extra ([InfluxDB](https://flows.nodered.org/node/node-red-contrib-influxdb) y [spreadsheet-in](https://flows.nodered.org/node/node-red-contrib-spreadsheet-in)) que aprovecha la potencia de las expresiones JSONata (que en Node-Red además incluyen la librería *moments*) para trabajar con DST y timezones a demás de extraer los datos y formatearlos para influx.

Para instalar todos los módulos de nodered más rápido, podemos hacerlo mediante los siguientes comandos:

```bash
    # instalamos las dependencias
    docker compose exec nodered npm i node-red-contrib-influxdb node-red-contrib-moment node-red-contrib-spreadsheet-in

    # y reiniciamos el servicio para que node tome los cambios
    docker compose restart nodered
```

Posteriormente es posible importar el flujo mediante el icono de la hamburguesa (esquina derecha superior)>import.

## Algunos cálculos

El precio de la electricidad por KWh depende en primer lugar del tipo de contrato de tarifa regulada que tenemos (PVPC: precio voluntario para el pequeño consumidor), que tiene 3 modalidades; normal, discriminación horaria y super-valle o vehículo eléctrico. El precio de la electricidad por horas se subasta cada día para el día siguiente y se hace disponible a las 20h. En el enlace de descargas, así como en la web y en el servicio web de recuperación de datos, tendremos el precio hora a hora para las 3 modalidades. Para calcular cuanto pagaremos, calcularemos el consumo para cada hora del día (24 cubos por así decirlo) y multiplicaremos por el precio de esa hora y posteriormente acumular todos los consumos de cada hora.

Además podemos calcular el **precio al que nos comprarán el excendente** en caso de que tengamos una instalación solar fotovoltaica (PV). El precio al que lo compran es el de la columna de precio de mercado diario horario (PMH o PMDh) menos los costes de desvíos (CDSVh). Todos estos cálculos están [especificados en el BOE](https://www.boe.es/buscar/act.php?id=BOE-A-2014-3376). Para calcular cuanto nos van a deducir de nuestra factura, será cuestión de multiplicar el excedente volcado a la red cada hora, por *(PMH - CDSVh)* correspondiente y sumarlos.

## Mostrar gráfica en influx

Para mostrar 

```flux
import "date"
import "experimental"

fin = experimental.addDuration( d: 1d, to: date.truncate(t: v.timeRangeStop, unit: 1d))
inicio = date.truncate(t: v.timeRangeStart, unit: 1d)

from(bucket: "electricidad")
  |> range(start: inicio, stop: fin)
  |> filter(fn: (r) => r["_measurement"] == "esios")
  |> filter(fn: (r) => r["_field"] == "PVPC")
  |> filter(fn: (r) => r["Peaje"] == "2.0.DHA" or r["Peaje"] == "2.0TD")
  //|> aggregateWindow(every: 1h, fn: last, createEmpty: false)
  |> group(columns: ["_field"])
  |> sort(columns: ["_time"])
  |> map(fn: (r) => ({_time: r._time, _field: "pvpc", _value: r._value / 1000.0}))
  |> yield(name: "pvpc")
```

He comentado el `aggregateWindow` porque realmente no es necesario y estaba causando que los datos aparezcan "retrasados" una hora en el panel de exploración de Influxdb. En grafana, con el aggregateWindow aparece perfectamente :)