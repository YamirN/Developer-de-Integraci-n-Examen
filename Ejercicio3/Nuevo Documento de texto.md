Ejercicio 3: Configuración de User-Defined Policy y Sobreescritura de Propiedades en BAR

Este documento técnico detalla el diseño, la parametrización visual en el Toolkit y la documentación de los comandos teóricos para la sobreescritura de propiedades de la política de tipo User-Defined denominada PolicyAutenticacion.

1. Estructura y Parámetros del Proyecto de Políticas

Se ha creado un proyecto de políticas independiente en el espacio de trabajo del Toolkit llamado PoliticasSeguridad. Dentro de este proyecto se aloja el elemento de política PolicyAutenticacion, configurado bajo el tipo "User Defined".

Este elemento centraliza los parámetros de conectividad para el servicio de Lista Negra, eliminando los valores estáticos dentro del flujo de integración. Las propiedades declaradas de forma visual en la interfaz de diseño son las siguientes:

Property: tokenEndpoint

Value: http://mock-bank/listanegra

Property: clientId

Value: client-dev-uuid-12345

Property: timeoutMs

Value: 5000

2. Consumo Dinámico en el Flujo de Integración

Para la lectura de estos valores en tiempo de ejecución, se modificó el flujo original del Ejercicio 1 sustituyendo el nodo de transformación anterior por un nodo JavaCompute llamado PrepararListaNegra.

Este nodo interactúa con el motor de ejecución de IBM ACE mapeando el proyecto PoliticasSeguridad y la política PolicyAutenticacion. Al recuperar las propiedades en runtime, el nodo inyecta dinámicamente el valor de la URL recuperada dentro del árbol de entorno local, específicamente en la ruta de destino HTTP (Destination.HTTP.RequestURL). Con esto, el nodo HTTPRequest consecuente realiza la llamada utilizando el endpoint de la política en lugar de un valor de configuración rígido.

3. Estrategia de Parametrización por Entorno (.properties)

Para cumplir con el requerimiento de documentar una segunda configuración para un ambiente de producción ficticio utilizando valores distintos, se definen los siguientes archivos de mapeo de propiedades externas:

Archivo: dev.properties (Ambiente de Desarrollo Ficticio)

PoliticasSeguridad:PolicyAutenticacion#tokenEndpoint=http://dev-bank/listanegra

PoliticasSeguridad:PolicyAutenticacion#clientId=client-dev-12345

PoliticasSeguridad:PolicyAutenticacion#timeoutMs=5000

Archivo: prod.properties (Ambiente de Producción Ficticio)

PoliticasSeguridad:PolicyAutenticacion#tokenEndpoint=https://prod-bank-secure/listanegra

PoliticasSeguridad:PolicyAutenticacion#clientId=client-prod-99999

PoliticasSeguridad:PolicyAutenticacion#timeoutMs=10000

4. Comando de Sobreescritura en el BAR (mqsiapplybaroverride)

A continuación, se detalla el uso del comando estándar de IBM ACE para demostrar cómo se realizaría la inyección de estas propiedades externas sobre el archivo BAR compilado (API_Transferencias.bar) según el ambiente requerido.

Comando para aplicar entorno de Desarrollo:

mqsiapplybaroverride -b API_Transferencias.bar -p dev.properties -r


Comando para aplicar entorno de Producción:

mqsiapplybaroverride -b API_Transferencias.bar -p prod.properties -r


Descripción de los parámetros del comando:

-b: Especifica la ruta física del archivo ejecutable empaquetado (API_Transferencias.bar).

-p: Apunta al archivo de propiedades de texto externo (.properties) que contiene las claves y los nuevos valores a inyectar.

-r: Indica que el reemplazo se aplicará de forma de cascada o recursiva sobre todos los componentes del BAR.