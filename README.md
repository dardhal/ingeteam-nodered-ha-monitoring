# INGETEAM Node-Red Home Assistant Monitoring
Los inversores híbridos INGETEAM SUN STORAGE 1PLAY TL-M no cuentan, a día de hoy (Diciembre 2022) de una integración propiamente dicha en Home Assistant. Para su monitorización y el almacenaje de métricas se hace necesario consultar los valores que el inversor publica via MODBUS (accesibles desde el inversor a través de un puerto TCP), procesarlos, y publicarlos en Home Assistant (si se necesita fácil acceso a ellos) y / o almacenarlos para su utilización, por ejemplo, enviándolos a InfluxDB para su posterior explotación mediante Grafana.

Para ello se utiliza Node-Red (que en mi caso está instalado como add-on en Home Assistant) y algunos nodos adicionales.

Este repositorio y los flujos que en él se encuentran no son sino una actualización, limpieza, documentación y "puesta en bonito" del excelente trabajo de "mainakae", disponible en el siguiente repositorio de GitHub, del que este respositorio es un fork :

https://github.com/mainakae/ingeteam

_Por interés histórico y en reconocimiento al autor se han dejado en este repositorio los dos archivos .json creados por él (original-*.json). El archivo que usar, no obstante, sería el de nombre "ingecon-vatimetro-modbus-influxdb.json"._


## Cómo funcionan los flujos de Node-Red

Mi instalación solar cuenta con los siguientes componentes relevantes para este repositorio, a saber :
* Inversor INGETEAM INGECON SUN STORAGE 1PLAY TL-M 6 kW, con la última versión del firmware disponible hasta la fecha (ABH1007_N)
* Batería de litio Pylontech Force L2 10.45 kWh nominales, conectada al inversor mediante CANBUS
* Vatímetro externo Carlo Gavazzi ET112, conectado al inversor mediante MODBUS

El inversor publica, en el puerto TCP/502, todos los valores MODBUS del inversor, tal cual están descritos en la documentación del fabricante :

http://www.ingeras.es/manual/ABH2010IMB08.pdf

Pero también publica, en el puerto TCP/503, los valores MODBUS que lee desde el vatímetro instalado. En el caso del vatímetro usado, la descripción de los valores disponibles se encuentra en el siguiente documento :

https://gavazzi.se/app/uploads/2020/11/em111_em112_et112_cp.pdf


De tal manera que podemos usar las facilidades propias de Node-Red para consultar todos estos valores, y procesarlos según se desee. En el flujo publicado en este repositorio hay por lo tanto dos conjuntos de nodos :

* Uno para consultar y procesar los valores MODBUS propios del inversor
* Otro para consultar y procesar los valores MODBUS que, publicados por el inversor, provienen del vatimetro externo


En ambos casos la lógica de los flujos es la misma : 

* Se usa el nodo Modbus-Read para leer, desde la dirección IP del inversor y puerto TCP correspondiente, el conjunto de "input registers" MODBUS con los valores que nos interesan. La salida es una cadena de bytes que necesita ser procesada
* Se usa a continuación el nodo buffer-parse para procesar la cadena de bytes y mapear cada conjunto de bytes a variables específicas, asignando a éstas nombres relevantes, y aplicando las conversiones necesarias : en MODBUS parece no haber un tipo para números no enteros, así que cualquier registro que represente un número no entero se debe publicar un entero y luego escalar en software. Por ejemplo, para publicar "123,456" se publicaria "123456" y se post-procesaría dividiendo el valor por 1000 en software, en este nodo
* El resultado es un objeto con parejas de "variable: valor" que se procesa con un nodo "function" para que los parámetros que representan valores discretos (de una lista de valores posibles) generen, cada uno de ellos, parámertos del mismo nombre acabados en "\_str" con la descripción textual
* Mediante un nodo "change" adicional se efectúan cálculos para añadir parámetros adicionales al objeto, a partir de parámetros ya existentes
* Finalmente, se usa el nodo Influxdb-Out para escribir en una instancia existente de InfluxDB, en la base detos que se indique, y con el nombre de métrica correspondiente, una entrada cada ejecución con el total de parejas "clave: valor" en el "payload" del mensaje

En mi caso además quería tener disponibles en Home Assistant alguno de los valores, aparte de tenerlos todos almacenados en InfluxDB. Para ello la opción más sencilla fue usar nodos "API" de Home Assistant para crear y actualizar el valor de sensores que usar también desde HA.


## Configuración del flujo

Copia el contenido del .json en una sola línea, ve al menú superior derecho de Node-Red (las tres rayas horizontales), elige "Import" y pega el contenido en el espacio correspondiente, y confirma la importación como un "nuevo flujo", para tenerlo en una pestaña separada. Es posible que Node-Red te advierta de la necesidad de instalar nodos que aún no tienes instalados, pues algunos no vienen de serie con el add-on de Node-Red para Home Assistant.

* Necesitarás editar los dos nodos "modbus-client", que representan la dirección IP y puerto del acceso a datos MODBUS del inversor (TCP/502) y vatímetro (TCP/503)
* También podrías necesitar editar los dos nodos "Modbus-Read" si el inversor es una versión anterior (que tiene menos valores expuestos) o el vatímetro es otro modelo diferente. Consulta la documentación del fabricante para más datos
* Edita también los nodos "influxdb" e "influxdb-out" para inversor y vatímetro, para apuntar a tu instalación de InfluxDB y usar las credenciales y "measurement" de tu elección
* Una vez hecho el "deploy", activa y desactiva los nodos "Debug" a tu elección para ver el contenido de los objetos



## Otros detalles adicionales

* El inversor no publica valores acumulados de energía (Wh o kWh). El vatímetro externo lo hace pero con una resolución pobre (sólo de décimas de kWh) y obviamente sólo cubre los netos desde y hacia la red eléctrica. En versiones posteriores del firmware del vatímetro (B.10 o posterior) sí proporciona valoes de consumos acumulados con precisión de Wh, pero en mi caso el vatímetro va con la versión B.4, y no los soporta
* La consulta de los valores MODBUS para inversor y vatímetro está configurada cada 30 segundos. Suficiente para mi, mucho para otros, insuficiente si se quiere una precisión mayor en el cálculo de los consumos
* Para el vatímetro existen flujos adicionales configurados para ejecutar cada 24 horas porque realmente sólo me fueron útiles a efectos de "troubleshooting"
* Generar sensores en Home Assistant desde Node-Red se puede hacer de varias otras maneras diferentes, pero a mi usando la API me funcionó y me resultó simple y correcta para mis propópsitos

## Nota sobre las versiones de firmware del INGECON SunStorage 1Play TL-M
Alunas nuevas versiones del firmware para el inversor, que la propia interfaz web sugiere actualizar cuando están disponibles, pueden introducir cambios o añadir funcionalidad en los "input registers" publicados por MODBUS.

Por ejemplo, entre las versione _G y _H del firmware, se documentan los siguientes cambios :

 - *EV Charger. Active Power [30081] added*
 - *RMS Differential Current [30062] changed to mA x10*
![enter image description here](https://github.com/dardhal/ingeteam-nodered-ha-monitoring/blob/main/changelog-G_H.png?raw=true)
El segundo de ellos supone que los valores previos del parámetro hay que convertirlos en el objeto "buffer-parse" de Node-Red, para que el resultado final sea correcto (de lo contrario un valor real de 12 mA aparecerá como 120 mA) :
![enter image description here](https://github.com/dardhal/ingeteam-nodered-ha-monitoring/blob/main/differential-current-parse-buffer-H.png?raw=true)
En mis pruebas , el nuevo parámetro "EV Charger. Active Power" no se publica, aún con la versión _H insalada.


## Problemas insertando valores en InfluxDB tras actualización del inversor a firmware versión ABH1007_W

A principios de 2025 INGECON publica la versión indicada del firmware, sin que conste cambio alguno en el listado publicado de "input registers", que de hecho, data en la versión enlazada en su web del año 2023 :
[Información técnica de INGECON SUN STORAGE 1Play TL M](https://www.ingeconsuntraining.info/?page_id=25442)
[Input registers](https://www.ingeras.es/manual/ABH2010IMB08.pdf)

Sin embargo tras actualizar el inversor se comienza a obtener errores (HTTP/400 Bad Request) al intentar insertar los valores obtenidos por MODBUS desde el inversor.

La investigación muestra que el problema deriva del valor obtenido para el parámetro "battery_discharge_power_reduction_reason", que devuelve un valor numérico 13. Según la documentación del fabricante indicada arriba, este parámetro (30078 : Battery. Charge Limitation Reason) está descrito en la nota 9, que da una lista de los valores posibles de este parámetro, entre los cuales no está el valor numérico 13 :
![enter image description here](https://github.com/dardhal/ingeteam-nodered-ha-monitoring/blob/main/Input-Registers-Table-9.png?raw=true)

En el flujo de Node-Red, la función de "Value Mapping", no encuentra ningún índice de valor 13 para el parámetro en cuestión, deja la clave "battery_discharge_power_reduction_reason_str" como "undefined", y a la hora de insertar en InfluxDB falla la inserción con el error "invalid boolean".

La solución de emergencia consiste en definir un valor adicional con clave 13 para el "battery_discharge_power_reduction_reason" de la tabla con un valor textual cualquiera.

### Actualización a 24 de abril de 2025
A resultas del correo enviado al fabricante (INGECON), al cabo de unos días publicaron la versión actualizada del documento de "Input Registries", donde consta la descripción del parámetro que causó problemas, así como otro añadido , que también se ha incorporado al flujo de Node-Red :

[Documento actualizado de input registries del INGECON SunStorage 1PLay TL-M](https://www.ingeras.es/manual/ABH2010IMB08.pdf)

![image](https://github.com/user-attachments/assets/bc266ab7-544d-4526-b5a8-4817d7a32885)

