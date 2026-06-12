Documentación de Arquitectura e Integración: API Transferencias

Este documento detalla la lógica de orquestación, validación y manejo de excepciones implementada en el flujo síncrono postValidar.msgflow para la API REST /transferencia/validar.

1. Arquitectura General del Flujo

El flujo ha sido diseñado siguiendo un patrón de orquestación secuencial (Pipeline), donde se priorizan las validaciones tempranas de negocio antes de realizar operaciones de persistencia o mensajería asíncrona. Toda la mensajería interna mantiene la trazabilidad mediante un identificador único global (correlationId).

Componentes Principales del Lienzo:

Entrada REST (Input): Punto de inicio síncrono que recibe el payload original en formato JSON.

Generación de Trazabilidad (JavaCompute - GenerarUUID): Genera de forma nativa un UUID único y lo inyecta en el árbol de entorno global (Environment.Variables.CorrelationId) para mantenerlo vivo durante todo el ciclo de vida de la transacción. Asimismo, almacena el RequestOriginal para su posterior uso.

Capa de Conectividad Externa (HTTPRequests): Enlaces directos a microservicios externos configurados con una política estricta de tiempo de espera.

Capa de Reglas de Negocio (Compute ESQL): Bloques encargados de evaluar las respuestas de los servicios web y decidir el enrutamiento.

Capa de Persistencia Asíncrona (JavaCompute + MQOutput): Construye la estructura final estandarizada y la deposita en el backbone de mensajería empresarial.

Manejador Centralizado de Excepciones (Subflow): Componente reutilizable desacoplado que intercepta fallas técnicas y de negocio para asegurar respuestas uniformes hacia el cliente.

2. Lógica Detallada de Cada Rama

El flujo se bifurca de manera controlada en tres escenarios operativos bien definidos:

🟢 Rama A: Camino Feliz (Proceso Exitoso)

Esta rama se ejecuta linealmente cuando la transacción supera todas las reglas técnicas y de negocio.

Consulta de Saldo: El nodo PrepararSaldo mapea la variable de ruta e invoca mediante un método GET al servicio http://mock-bank/saldo/{cuentaOrigen} a través del nodo ConsultarSaldo.

Evaluación de Fondos: El nodo ValidarSaldo extrae el balance del JSON de respuesta. Al verificar que saldo >= monto, el flujo continúa su curso regular.

Consulta de Fraude: El nodo PrepararListaNegra toma el RUC del destinatario del payload original y realiza un llamado POST hacia http://mock-bank/listanegra utilizando el nodo ConsultarListaNegra.

Evaluación de Seguridad: El nodo ValidarListaNegra confirma que el campo flag reportado es estrictamente FALSE.

Estructuración del Mensaje de Auditoría: El flujo avanza al nodo PrepararMensajeMQ (JavaCompute), el cual:

Recupera el payload inicial clonando cíclicamente el RequestOriginal.

Agrega en la raíz del objeto los campos obligatorios de auditoría: correlationId y timestampAprobacion (calculado dinámicamente con la zona horaria del sistema).

Publicación en MQ: El mensaje estructurado se deposita físicamente en la cola local TRF.APROBADAS a través del nodo MQOutput.

Respuesta Síncrona: Finalmente, el nodo ConstruirRespuesta202 (Compute) intercepta el éxito del proceso, inyecta en las cabeceras de transporte el código de estado HTTP 202 (Accepted) y retorna al cliente un JSON confirmando el procesamiento junto al correlationId asociado.

🟡 Rama B: Errores de Reglas de Negocio (Validaciones Síncronas)

Esta rama intercepta las transacciones que violan los acuerdos operativos, impidiendo que payloads inválidos consuman recursos en el backend de mensajería.

Sub-rama B1 (Saldo Insuficiente): Si en el nodo ValidarSaldo se detecta que el monto solicitado supera los fondos disponibles, el código ESQL interrumpe la secuencia principal y ejecuta un desvío explícito (PROPAGATE TO TERMINAL 'failure'). El flujo viaja inmediatamente hacia el subflujo centralizado de errores, configurando el payload con un código de estado HTTP 422 (Unprocessable Entity) y la constante SALDO_INSUFICIENTE.

Sub-rama B2 (Destinatario Bloqueado): Si el microservicio de prevención de fraude responde que el RUC consultado se encuentra penalizado, el nodo ValidarListaNegra corta la ejecución síncrona enviando el mensaje por su terminal de error (failure). El mensaje es procesado para retornar al cliente un código de estado HTTP 403 (Forbidden) acompañado del mensaje DESTINATARIO_BLOQUEADO.

🔴 Rama C: Contingencia Técnica (Timeouts de Infraestructura)

Diseñada para proteger la disponibilidad de la API ante caídas de proveedores de servicios de terceros.

Los nodos de conectividad ConsultarSaldo y ConsultarListaNegra cuentan con la propiedad Request timeout (sec) fijada en 5 segundos.

Si cualquiera de los dos servidores externos excede este límite o presenta una desconexión a nivel de red (Connection Refused), el nodo HTTP interrumpe la espera de forma segura y desvía el hilo de ejecución por su terminal Error.

Esta señal es capturada por el subflujo sf_error_handler, el cual de forma unificada anula el procesamiento normal, setea el código de estado global HTTP 504 (Gateway Timeout) e inyecta la nomenclatura estándar SERVICIO_NO_DISPONIBLE.


504 Gateway Timeout

{ "codigo": "SERVICIO_NO_DISPONIBLE", "mensaje": "...", "timestamp": "...", "correlationId": "UUID" }

Fin del documento.
