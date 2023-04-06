# Analisis de clientes Cyclistics

Objetivo:

El Objetivo del proyecto es analizar a los clientes de la empresa, diferenciados entre los que pagan la membresía anual y los clientes ocasionales, para aumentar la tasa de clientes con membresía anual. Las conclusiones obtenidas del análisis se utilizarán para dirigir la campaña de marketing de la empresa hacía la conversión de los clientes ocasionales en clientes con membresía. 

Interesados:

La directora de marketing de la empresa.
El equipo de análisis de marketing.
El equipo ejecutivo de la empresa.

Fuentes de datos:

https://divvy-tripdata.s3.amazonaws.com/index.html
Se analizan los 12 últimos meses de los registros de los clientes de la empresa. Desde enero de 2022 hasta diciembre de 2022.

Preparación de los datos:

En esta sección se exploran los datos para conocer como están estructurados, obtener los datos y reunirlos en una misma tabla, además se realizan los cálculos para crear las nuevas columnas.
Lo primero es cargar los datos en Excel a partir de los archivos .csv como primera toma de contacto.  Vemos que hay muchas filas y que no vamos a poder trabajar los datos con Excel.
Al ser archivos con tantos datos se trabajan con SQL, utilizando BigQuery de Google. El primer paso es crear un database llamado “ciclystic”, a continuación, se crean una tabla para cada mes importando el archivo csv a SQL, para los archivos de más de 100 Mb se cargan a través de Google Drive.
Las tablas contienen las siguientes variables:

fullname |	mode |	type |	description

ride_id	--> NULLABLE -->	STRING -->	 ID único de cada viaje.

rideable_type -->	NULLABLE -->	STRING -->	tipo de bicicleta usada en el viaje

started_at -->	NULLABLE -->	TIMESTAMP -->	Fecha y hora de comienzo del viaje.

ended_at -->	NULLABLE -->	TIMESTAMP	--> Fecha y hora de finalización del viaje.

start_station_name -->	NULLABLE -->	STRING -->	Nombre de la estación desde la que comienza el viaje.

start_station_id -->	NULLABLE -->	STRING -->	ID único de la estación de comienzo del viaje.

end_station_name -->	NULLABLE -->	STRING -->	Nombre de la estación donde acaba el viaje.

end_station_id -->	NULLABLE -->	STRING -->	ID único de la estación donde acaba el viaje.

start_lat -->	NULLABLE -->	FLOAT	--> Latitud de la posición de inicio.

start_lng	--> NULLABLE	--> FLOAT	--> Altitud de la posición de inicio.

end_lat	--> NULLABLE	--> FLOAT	--> Latitud de la posición final.

end_lng	--> NULLABLE	--> FLOAT	Altitud de la posición final.

member_casual	--> NULLABLE	--> STRING	--> Tipo de cliente que realiza el viaje

Para no realizar las mismas consultas en las 12 tablas para crear nuevas variables, lo primero es juntarlas en una sola tabla mediante la función UNION ALL.
El código es el siguiente:

SELECT 
  ride_id, 
  rideable_type, 
  started_at, 
  ended_at, 
  start_station_name, 
  start_station_id, 
  end_station_name, 
  end_station_id, start_lat, 
  start_lng, 
  end_lat, 
  end_lng, 
  member_casual 
 FROM `bloque4semana3.cyclistic.enero2022`
 UNION ALL
 SELECT 
  ride_id, 
  rideable_type, 
  started_at, 
  ended_at, 
  start_station_name, 
  start_station_id, 
  end_station_name, 
  end_station_id, start_lat, 
  start_lng, 
  end_lat, 
  end_lng, 
  member_casual 
 FROM `bloque4semana3.cyclistic.febrero2022`
 UNION ALL
 (... hasta unir las 12 tablas)

Una vez unidas tiene más de 5 millones de filas, así que se guarda la tabla como datos2022 para no tener que repetir la unión en el código de futuras consultas.
A continuación, se crean las columnas con las nuevas variables duración_viaje, mes y dia_semana. Utilizando las funciones TIMESTAMP_DIFF, DAYOFWEEK y MONTH, obtenemos el siguiente código:

SELECT ride_id,
  rideable_type, 
  started_at, 
  ended_at,
  ROUND (TIMESTAMP_DIFF(ended_at, started_at, minute)) AS duracion_viaje,
  EXTRACT(DAYOFWEEK FROM started_at) AS dia_semana,
  EXTRACT(MONTH FROM started_at) AS mes,
  start_station_name, 
  start_station_id, 
  end_station_name, 
  end_station_id, start_lat, 
  start_lng, 
  end_lat, 
  end_lng, 
  member_casual 
FROM `bloque4semana3.cyclistic.datos2022`;

La columna de dia_semana muestra un valor numérico entre el 1 (domingo) y el 7 (sábado). La consulta se guarda en la base de datos como una tabla nueva llamada tabladatos2022.

Limpieza de los datos:

Una vez que tenemos todos los datos en la misma tabla y con todas las variables creadas podemos empezar el proceso de limpieza.
Se comprueban el número de registros de ride_id al ser la variable principal de los datos, por si existen duplicados:

SELECT DISTINCT COUNT (ride_id) AS numero_viajes
FROM `bloque4semana3.cyclistic.datos2022`;

El resultado nos da el mismo número de filas que tiene el archivo (5667717), por lo tanto, no hay ningún viaje repetido.
Además, queremos revisar que no haya valores nulos en las variables que usaremos para crear nuevas columnas que nos servirán en el posterior análisis. El código utilizado es el siguiente:

SELECT COUNT(*) AS valores_nulos,
FROM `bloque4semana3.cyclistic.datos2022`
WHERE ride_id IS NULL;

El resultado de esta consulta es una columna con el número de valores nulos en los datos. Repitiendo la consulta para cada una de las distintas variables se obtiene el número de valores nulos en cada columna de la tabla. En las variables que nos interesan como son ride_id, rideable_type, started_at, ended_at y member_casual no hay valores nulos. Por lo tanto, se considera que no es necesario eliminar ningún registro de los datos utilizados.
En las variables start_station_name y end_station_name sí que aparecen valores nulos, pero no los vamos a borrar porque son muchos registros. Se filtrarán las posteriores consultas para que no visualicen estos valores nulos.
Para la variable creada duración_viaje procedemos a revisar los resultados obtenidos para encontrar posibles fallos en el cálculo debido a valores erróneos. El primer caso es contar todos los valores nulos o negativos en la duración del viaje:

SELECT COUNT(*) AS viajes_negativo,
FROM `bloque4semana3.cyclistic.tabladatos2022`
WHERE duracion_viaje <= 0;

El resultado obtenido son 121089 filas con duración del viaje en negativo, con lo que estos datos no sirven para el análisis. Así que se eliminan de la tabla obtenida anteriormente:

DELETE FROM `bloque4semana3.cyclistic.tabladatos2022`
WHERE duracion_viaje <= 0;

El siguiente paso es determinar los viajes máximos y mínimos, y la media de duración de todos los viajes:

SELECT ROUND(AVG(duracion_viaje),1) AS media_duracion,
    MAX (duracion_viaje) AS max_duracion,
    MIN (duracion_viaje) AS min_duracion,
FROM bloque4semana3.cyclistic.tabladatos2022;

El resultado de la consulta es que el tiempo medio de viaje es de 19.4 minutos, la duración máxima es de 41387 minutos (28 días), y la duración mínima es de 1 minuto. Como se puede observar hay valores extraños en los datos de la hora de finalización de los viajes, ya que aparecen datos de duración de varios días o incluso un mes entero. No se borran estos valores porque no podemos determinar a partir de que cantidad de tiempo se considera error.

Análisis de los datos:

Realizando la misma consulta que en el paso anterior se obtiene la media de los viajes para cada tipo de cliente en la empresa:

SELECT ROUND(AVG(duracion_viaje),2) AS media_viaje,
  member_casual
FROM `bloque4semana3.cyclistic.tabladatos2022`
GROUP BY member_casual;

Obtenemos una media de duración de viaje para los miembros de 12,5 minutos, y para los usuarios casuales de 29,26 minutos.
Creamos 4 tablas a partir de las consultas de promedio de duración del viaje, por mes y por día de la semana, para miembros y clientes casuales.

SELECT
  ROUND(AVG(duracion_viaje),2) AS media_duracion,
  dia_semana
FROM `bloque4semana3.cyclistic.tabladatos2022`
WHERE member_casual = 'casual'
GROUP BY dia_semana;

También se realiza la consulta con el número de viajes realizado para cada tipo de cliente:

SELECT member_casual,
  COUNT(*) AS num_viajes
FROM `bloque4semana3.cyclistic.tabladatos2022`
GROUP BY member_casual;

El resultado es: 3272486 viajes realizados por miembros y 2274142 viajes realizados por clientes casuales.
Para obtener el periodo del año en el que se realizan más viajes se crea una consulta para cada tipo de cliente con la cuenta del total de viajes por mes y por día de la semana.

SELECT COUNT(*) AS num_viajes,
dia_semana
FROM `bloque4semana3.cyclistic.tabladatos2022`
WHERE member_casual = 'member'
GROUP BY dia_semana;

Se realiza la consulta las cuatro veces cambiando el tipo de cliente y la columna a filtrar.
Una vez obtenida toda la información del número de viajes, los promedios y las cantidades, pasamos a analizar el tipo de bicicleta que más utilizan los clientes:

SELECT rideable_type,
COUNT(*) AS viajes_clientes
FROM `bloque4semana3.cyclistic.tabladatos2022`
WHERE member_casual='member'
GROUP BY rideable_type;

Como último paso procedemos a analizar las estaciones más populares entre los dos tipos de clientes:

SELECT 
  start_station_name,
  COUNT(*) AS viajes_miembros
FROM `bloque4semana3.cyclistic.tabladatos2022`
WHERE member_casual = 'member'
  AND start_station_name != 'null'
GROUP BY start_station_name
ORDER BY COUNT(*) DESC
LIMIT 10;

Obtenemos las 10 estaciones desde donde empiezan los viajes para los clientes que son miembros. Realizamos esta consulta con los clientes casuales, y para las estaciones de finalización de los viajes.
Del mismo modo obtenemos las 10 estaciones donde finalizan más viajes:

SELECT 
  start_station_name,
  COUNT(*) AS viajes_miembros
FROM `bloque4semana3.cyclistic.tabladatos2022`
WHERE member_casual = 'member'
  AND start_station_name != 'NULL'
GROUP BY start_station_name
ORDER BY COUNT(*) DESC
LIMIT 10;

Y por último unimos estas dos consultas para obtener una tabla con la suma de las estaciones en las que más bicicletas circulan:

SELECT 
station_name,
SUM (viajes_miembros) AS total_viajes
FROM `bloque4semana3.cyclistic.stations_members` 
GROUP BY station_name
ORDER BY total_viajes DESC

Obtenemos dos tablas con el número de viajes en las 10 estaciones más visitadas por las clientes ordenadas de mayor a menor.

Visualización de los datos:

Para crear el informe con el resultado del análisis utilizaremos Power Bi, que se conecta con la base de datos en Google BigQuery y podemos cargar las tablas obtenidas de las consultas anteriores.
Visualizaciones de datos con Power BI:
Gráfico circular con el % de los viajes realizados por cada tipo de cliente.
Gráfico de líneas con la variación del número de viajes por mes. 
Gráfico de líneas con la variación del número de viajes en cada día de la semana.
Gráfico circular con la clase de bici más usado para los dos tipos de usuario.
Gráfico de barras con la duración media de los viajes para los tipos de clientes, en cada mes del año, y en cada día de la semana.
Tablas con las estaciones más visitadas para los dos tipos de clientes.
