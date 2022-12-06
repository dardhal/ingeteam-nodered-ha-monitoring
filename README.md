# INGETEAM Node-Red Home Assistant Monitoring
Los inversores híbridos INGETEAM SUN STORAGE 1PLAY TL-M no cuentan, a día de hoy (Diciembre 2022) de una integración propiamente dicha en Home Assistant. Para su monitorización y el almacenaje de métricas se hace necesario consultar los valores que el inversor publica via MODBUS (accesibles desde el inversor a través de un puerto TCP), procesarlos, y publicarlos en Home Assistant (si se necesita fácil acceso a ellos) y / o almacenarlos para su utilización, por ejemplo, enviándolos a InfluxDB para su posterior explotación mediante Grafana.

Para ello se utiliza Node-Red (que en mi caso está instalado como add-on en Home Assistant) y algunos nodos adicionales.

Este repositorio y los flujos que en él se encuentran no son sino una actualización, limpieza, documentación y "puesta en bonito" del excelente trabajo inicial de "mainakae", disponible en el siguiente repositorio de GitHub, del que este respositorio es un fork :

https://github.com/mainakae/ingeteam

## Cómo funcionan los flujos de Node-Red

Mi instalación solar cuenta con los siguientes componentes relevantes para este repositorio, a saber :
* Inversor INGETEAM INGECON SUN STORAGE 1PLAY TL-M 6 kW, con la última versión del firmware disponible hasta la fecha (ABH1007_N)
* Batería de litio Pylontech Force L2 10.45 kWh nominales, conectada al inversor mediante CANBUS
* Vatímetro externo Carlo Gavazzi ET112, conectado al inversor mediante MODBUS

El inversor publica, en el puerto TCP/502, todos los valores MODBUS del inversor, tal cual están descritos en la documentación del fabricante :

http://www.ingeras.es/manual/ABH2010IMB08.pdf
