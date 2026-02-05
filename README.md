# RabbitMQ: Introducción y Conceptos Avanzados

## ¿Qué es RabbitMQ?

RabbitMQ es un message broker, es decir, un software intermediario que recibe mensajes de aplicaciones y los distribuye a otras aplicaciones. Piensa en él como una oficina de correos: tú depositas cartas (mensajes) y el correo se encarga de llevarlas a sus destinatarios.

**¿Por qué necesitamos esto?** Porque permite que aplicaciones trabajen de forma independiente y asíncrona. Una aplicación puede enviar un mensaje y continuar su trabajo sin esperar respuesta. Si la aplicación receptora está caída o lenta, no afecta a quien envía el mensaje. El mensaje simplemente espera en RabbitMQ hasta que pueda ser procesado.

## Los 4 Componentes Fundamentales

### Producer (Productor)

Es cualquier aplicación que envía mensajes a RabbitMQ. Puede ser tu API web, un servicio de backend, un script, etc. El producer crea el mensaje con la información que quiere comunicar.

**Detalle importante:** El producer nunca envía mensajes directamente a una cola. Siempre los envía a un exchange. Esto puede parecer extraño al principio, pero la razón es el desacoplamiento: el producer no necesita saber dónde terminarán sus mensajes ni quién los leerá. Solo dice "aquí está mi mensaje" y el exchange decide qué hacer con él.

### Exchange (Intercambiador)

Es el componente que recibe mensajes de los producers y decide a qué cola(s) enviarlos. Es como un repartidor que lee la dirección de un paquete y decide qué ruta tomar.

**¿Cómo decide?** Cada mensaje lleva información de enrutamiento. La más común es la "routing key", que es simplemente una cadena de texto como "ventas.pedido" o "logs.error". El exchange compara esta routing key con reglas configuradas (llamadas "bindings") que le indican a qué colas debe enviar el mensaje.

**¿Por qué existe esta capa intermedia?** Porque separa completamente a quien produce mensajes de quien los consume. Puedes cambiar toda la lógica de distribución (agregar colas nuevas, cambiar destinos) sin modificar ni una línea de código del producer.

### Queue (Cola)

Es un buffer que almacena mensajes en orden hasta que un consumer los procese. Es literalmente una fila: los mensajes entran por un lado y salen por el otro.

**¿Por qué almacenar mensajes?** Porque los producers y consumers trabajan a velocidades diferentes. Si tu aplicación recibe 1000 pedidos por segundo pero tu sistema de facturación solo procesa 100 por segundo, la cola almacena temporalmente los 900 pedidos restantes. Sin esto, perderías datos o tu aplicación productora se bloquearía esperando.

Las colas pueden configurarse como persistentes (sobreviven reinicios del servidor) o transitorias (se pierden si RabbitMQ se reinicia).

### Consumer (Consumidor)

Es cualquier aplicación que lee y procesa mensajes de una cola. Puede ser un servicio que envía emails, actualiza bases de datos, genera reportes, etc.

**Característica clave:** Pueden existir múltiples consumers conectados a la misma cola. Cuando esto sucede, RabbitMQ distribuye los mensajes entre ellos (cada mensaje va a un solo consumer). Esto permite escalar: si un consumer no da abasto, agregas otro y automáticamente se reparte el trabajo.

## Tipos de Exchange: Las Reglas del Enrutamiento

Aquí es donde RabbitMQ se vuelve poderoso. Hay diferentes tipos de exchanges que implementan distintas lógicas de distribución.

### Direct Exchange

Este es el más simple. Funciona con coincidencia exacta: el mensaje tiene una routing key (por ejemplo "pagos"), y el exchange lo envía solo a las colas que están configuradas exactamente con esa misma key "pagos".

**¿Cuándo usarlo?** Cuando necesitas enrutamiento directo y específico. Si tienes departamentos diferentes y cada uno debe procesar solo sus mensajes.

### Topic Exchange

Similar al Direct, pero permite patrones con comodines en las routing keys. Los comodines son: asterisco (*) que representa exactamente una palabra, y almohadilla (#) que representa cero o más palabras.

Por ejemplo, si una cola está vinculada con el patrón "logs.*.error", recibirá mensajes con routing keys como "logs.database.error" o "logs.api.error", pero no "logs.info" ni "logs.database.error.critical".

**¿Cuándo usarlo?** Para sistemas de categorización jerárquica, como logs donde quieres suscribirte a "todos los errores de cualquier componente" o "todo lo relacionado con la base de datos".

### Fanout Exchange

Este ignora completamente las routing keys. Envía una copia del mensaje a todas las colas que están vinculadas a él, sin excepciones.

**¿Cuándo usarlo?** Para broadcasting, cuando el mismo evento debe ser procesado por múltiples sistemas independientes. Por ejemplo, cuando un usuario se registra, necesitas: enviar email de bienvenida, actualizar analytics, notificar al CRM, etc.

### Headers Exchange

En lugar de usar routing key, este tipo enruta basándose en atributos del mensaje llamados headers (metadatos clave-valor). Puedes especificar que coincidan todos los headers necesarios o al menos uno.

**¿Cuándo usarlo?** Cuando necesitas enrutamiento complejo con múltiples criterios que no se expresan bien con routing keys simples.

## Conceptos Avanzados

### Bindings (Vinculaciones)

Un binding es la regla que conecta un exchange con una cola. Le dice al exchange "cuando recibas un mensaje que cumpla X condición, envíalo a esta cola". La condición depende del tipo de exchange: puede ser una routing key exacta, un patrón, o atributos de headers.

### Routing Key (Clave de Enrutamiento)

Es un atributo del mensaje, una cadena de texto que el producer incluye cuando publica. El exchange usa esta información para decidir a qué colas enviar el mensaje. En exchanges tipo fanout se ignora. En topic exchanges suele seguir un formato jerárquico separado por puntos, como "ventas.europa.pedido".

### Acknowledgments (Confirmaciones)

Cuando un consumer recibe un mensaje, debe confirmar a RabbitMQ que lo procesó correctamente. Esto se llama ACK (acknowledgment).

**Modo automático:** RabbitMQ considera el mensaje procesado apenas lo envía al consumer. Si el consumer falla inmediatamente después, el mensaje se pierde para siempre.

**Modo manual:** El consumer debe enviar explícitamente un ACK después de procesar. Si el consumer falla antes de enviar el ACK, RabbitMQ entiende que algo salió mal y reencola el mensaje para que otro consumer lo intente.

**¿Por qué el modo manual?** Porque garantiza que ningún mensaje se pierda ante fallos. El trade-off es complejidad: tu código debe manejar los ACKs correctamente.

### Prefetch Count

Es un límite que estableces en el consumer: "no me envíes más de N mensajes sin confirmar simultáneamente". Por ejemplo, con prefetch=10, RabbitMQ enviará 10 mensajes al consumer, y no enviará el mensaje 11 hasta que el consumer confirme al menos uno de los primeros 10.

**¿Por qué es necesario?** Sin esto, un consumer lento podría recibir miles de mensajes, dejando otros consumers ociosos. El prefetch balancea la carga automáticamente.

### Publisher Confirms

Es el ACK pero del lado del producer. El producer espera confirmación de RabbitMQ de que recibió y persistió el mensaje correctamente antes de continuar.

**¿Por qué?** Sin esto, si RabbitMQ falla justo después de recibir un mensaje (antes de guardarlo), el producer cree que lo envió exitosamente pero en realidad se perdió. Publisher confirms garantizan escrituras confiables.

### Message Persistence

Los mensajes normalmente viven en memoria RAM para rapidez. Si marcas un mensaje como persistente, RabbitMQ lo escribe a disco. Si el servidor se reinicia, el mensaje sigue ahí.

**El costo:** Escribir a disco es muchísimo más lento que RAM. Solo usa persistencia para mensajes críticos que no pueden perderse bajo ninguna circunstancia.

### Dead Letter Exchange (DLX)

Es un exchange especial configurado como destino de mensajes "muertos". Un mensaje muere cuando: el consumer lo rechaza, expira su tiempo de vida (TTL), o la cola está llena.

**¿Por qué?** Permite manejar errores de forma elegante. En lugar de perder mensajes problemáticos, los envías a un DLX donde puedes analizarlos, reintentarlos más tarde, o alertar a un humano.

### TTL (Time To Live)

Es el tiempo máximo que un mensaje puede existir. Puedes configurarlo por mensaje individual o por cola completa. Pasado ese tiempo, el mensaje se descarta (o se envía al DLX si está configurado).

**¿Por qué?** Datos que pierden validez con el tiempo no deberían procesarse. Por ejemplo, una cotización de precio de hace 3 horas puede ser irrelevante.

### Priority Queues

Colas que procesan mensajes según prioridad asignada en lugar de orden de llegada. Mensajes con prioridad 10 se procesan antes que los de prioridad 1.

**El costo:** Más complejidad interna y menor rendimiento general. Solo úsalas cuando realmente necesitas procesamiento urgente de ciertos mensajes.

### Clustering

Varios servidores RabbitMQ trabajando juntos como uno solo. Comparten configuración de exchanges y bindings. Aumenta capacidad de procesamiento.

**Importante:** Por defecto, las colas NO se replican entre nodos. Si un nodo falla, sus colas se pierden. Para alta disponibilidad necesitas quorum queues.

### Quorum Queues

Tipo especial de cola que se replica automáticamente en varios nodos del cluster usando consenso Raft. Si un nodo falla, los otros tienen copia de los mensajes.

**¿Por qué?** Para aplicaciones críticas donde perder mensajes es inaceptable. Son más lentas que colas normales pero mucho más confiables.

### Lazy Queues

Colas configuradas para escribir mensajes a disco inmediatamente en lugar de mantenerlos en RAM. Reduce consumo de memoria drásticamente.

**¿Cuándo?** Cuando tienes millones de mensajes acumulados y no caben en RAM, pero puedes tolerar mayor latencia de procesamiento.

---

## TUTORIAL 1: "Hello World"
**Patrón:** Productor Simple → Cola → Consumidor Simple.

Este tutorial establece la comunicación más básica posible en RabbitMQ. Aunque parece trivial, introduce conceptos arquitectónicos fundamentales que se aplican en todos los patrones avanzados.

### Resumen Teórico y Aprendizajes Clave

#### 1. El Default Exchange (Intercambio Predeterminado)

Cuando el productor publica con `exchange=''`, la cadena vacía no significa "sin exchange". Identifica al **Default Exchange**, un exchange especial preconfigurado en RabbitMQ.

**Comportamiento único:** Es de tipo Direct Exchange con bindings automáticos a todas las colas. Cuando usas `routing_key='hello'`, enruta directamente a la cola llamada 'hello'. Esto crea la ilusión de enviar "directamente a la cola", pero internamente sigue el patrón exchange → binding → cola.

**¿Por qué existe?** Para simplificar casos simples donde no necesitas lógica de enrutamiento compleja. Es un atajo conveniente para principiantes, pero en producción normalmente declaras exchanges explícitos con nombres significativos.

#### 2. Declaración de Colas: Idempotencia e Independencia

El comando `queue_declare(queue='hello')` aparece tanto en productor como consumidor. Esto no es redundancia, es diseño deliberado para sistemas distribuidos.

**Idempotencia:** Ejecutar `queue_declare` múltiples veces es seguro. Si la cola existe, no hace nada. Si no existe, la crea. Esta propiedad permite que cualquier componente declare los recursos que necesita sin preocuparse por el estado actual.

**¿Por qué ambos lados declaran?** Porque no existe orden garantizado de inicio. Si el productor arranca primero y envía a una cola inexistente usando el default exchange, RabbitMQ descarta silenciosamente el mensaje. Si el consumidor arranca primero e intenta consumir de una cola inexistente, la conexión falla. Solución: cada componente declara todos los recursos que necesita para funcionar, haciéndolos independientes y resilientes.

#### 3. Conexiones y Canales: Optimización de Recursos

La estructura muestra dos niveles de abstracción: Connection (conexión TCP física) y Channel (canal lógico).

**¿Por qué esta separación?** Crear conexiones TCP es costoso: requiere handshake, autenticación, negociación de protocolo. Mantener miles de conexiones saturaría el sistema operativo y RabbitMQ. Los channels permiten multiplexar muchas operaciones lógicas (publicar, consumir, declarar) sobre una sola conexión física, optimizando recursos. Una Connection puede tener cientos de Channels ligeros.

#### 4. Modelo de Consumo: Push API

El consumidor define una función callback y RabbitMQ envía mensajes automáticamente cuando llegan. No hay polling activo ni bucles infinitos preguntando "¿hay mensajes?".

**¿Por qué Push en lugar de Pull?** Reduce latencia significativamente y simplifica el código del consumidor. RabbitMQ gestiona eficientemente el flujo de mensajes sin que tu aplicación deba manejar timeouts ni lógica de polling. El método `start_consuming()` inicia un bucle bloqueante que escucha eventos del servidor.

#### 5. Auto-acknowledgment: Simplificación Peligrosa

El parámetro `auto_ack=True` le dice a RabbitMQ "considera este mensaje procesado exitosamente en el momento que lo envíes por la red".

**El problema:** Si la función callback falla (excepción, crash, pérdida de conexión), RabbitMQ ya borró el mensaje. Se perdió permanentemente. Este tutorial usa auto-ack solo para simplificar el código inicial. El Tutorial 2 introduce acknowledgments manuales que garantizan confiabilidad.

---

## TUTORIAL 2: "Work Queues" (Colas de Trabajo)
**Patrón:** Productor → Cola → Múltiples Consumidores Competidores.

Transforma el sistema básico en arquitectura distribuida para procesamiento paralelo de tareas. Introduce conceptos críticos de fiabilidad y balanceo de carga.

### Resumen Teórico y Aprendizajes Clave

#### 1. Competencia por Mensajes (Competing Consumers)

Cuando múltiples consumidores se conectan a la misma cola, cada mensaje se entrega a **exactamente uno** de ellos, nunca a todos.

**¿Por qué este comportamiento?** Para distribución de carga horizontal. Si tienes 1000 tareas y 10 workers, cada worker procesará aproximadamente 100. Esto permite escalar simplemente agregando más consumidores. Es fundamentalmente diferente del patrón Publish/Subscribe donde múltiples consumidores reciben copias del mismo mensaje.

#### 2. Round-Robin Dispatching (Distribución Cíclica)

Por defecto, RabbitMQ distribuye mensajes en rotación estricta: mensaje 1 a Worker A, mensaje 2 a Worker B, mensaje 3 a Worker A, etc.

**Limitación crítica:** Es completamente ciego a la carga computacional real. Si Worker A recibe una tarea de 1 hora y Worker B una de 1 segundo, el siguiente mensaje va al siguiente worker en rotación aunque Worker B esté ocioso. Solo cuenta mensajes entregados, no tiempo de procesamiento ni carga actual. Resultado: balanceo injusto donde algunos workers se saturan mientras otros están inactivos.

#### 3. Message Acknowledgment: El Corazón de la Confiabilidad

Este es el concepto más crítico para sistemas de producción.

**El problema:** Con `auto_ack=True`, RabbitMQ marca el mensaje como "entregado exitosamente" apenas lo envía por la red. Si el worker crashea inmediatamente después, el mensaje se pierde permanentemente.

**Solución con Manual ACK:** El worker debe enviar explícitamente `basic_ack()` después de procesar exitosamente.

**Mecánica detallada:**
1. RabbitMQ envía el mensaje y lo marca "Unacknowledged" (en tránsito)
2. El mensaje permanece en este estado hasta recibir confirmación
3. Si llega `basic_ack()`, RabbitMQ elimina el mensaje de la cola
4. Si la conexión TCP se cierra sin ACK (crash, red caída), RabbitMQ reencola automáticamente el mensaje como "Ready"
5. Otro worker disponible lo procesa

**Implicación crucial:** Garantiza entrega "at-least-once". Un mensaje puede procesarse múltiples veces si el worker crashea después de procesarlo pero antes de enviar ACK. Tu lógica de negocio debe ser idempotente para manejar esto correctamente.

**Forgotten Acknowledgment:** Si olvidas llamar `basic_ack()`, los mensajes quedan "Unacknowledged" indefinidamente. RabbitMQ no liberará memoria ni enviará nuevos mensajes al consumidor (con prefetch configurado). Eventualmente agotarás recursos del servidor.

#### 4. Message Durability: Sobreviviendo Reinicios

Los acknowledgments protegen contra fallos del worker, pero no contra fallos del servidor RabbitMQ. Si RabbitMQ se reinicia, todas las colas y mensajes en memoria se pierden.

**Persistencia requiere dos configuraciones independientes:**

**Cola durable:** `queue_declare(queue='task_queue', durable=True)` guarda la definición de la cola en disco. Después de reiniciar, la cola sigue existiendo pero vacía. No puedes cambiar la durabilidad de una cola existente; debes crear una nueva con nombre diferente.

**Mensajes persistentes:** `delivery_mode=2` en las propiedades del mensaje. RabbitMQ escribe el contenido a disco antes de confirmar recepción.

**¿Por qué son independientes?** Porque resuelven problemas diferentes. Una cola durable sin mensajes persistentes: la cola existe después del reinicio pero vacía. Mensajes persistentes en cola no-durable: ambos se pierden al reiniciar. Necesitas ambas para supervivencia completa.

**Advertencia sobre garantías:** Existe una pequeña ventana entre que RabbitMQ acepta el mensaje y lo escribe a disco. Para garantías absolutas necesitas Publisher Confirms. El costo de durabilidad es significativo: escribir a disco es 100-1000 veces más lento que memoria.

#### 5. Fair Dispatch con QoS (Quality of Service)

`channel.basic_qos(prefetch_count=1)` resuelve el problema del Round-Robin.

**Mecánica:** Sin QoS, RabbitMQ envía mensajes agresivamente. Con 100 mensajes y 2 workers, envía 50 a cada uno inmediatamente. Si Worker A es lento, Worker B termina sus 50 y queda ocioso mientras A sigue trabajando.

Con `prefetch_count=1`, RabbitMQ mantiene exactamente 1 mensaje "Unacknowledged" por worker. Cuando un worker envía ACK, inmediatamente recibe el siguiente mensaje disponible. Los workers rápidos procesan más mensajes automáticamente.

**¿Por qué funciona?** Porque transforma el modelo de "empujar todos los mensajes" a "empujar bajo demanda". RabbitMQ solo envía nuevos mensajes cuando el worker demuestra (vía ACK) que tiene capacidad.

**Valores más altos:** `prefetch_count=10` permite hasta 10 mensajes sin confirmar. Reduce latencia de red en procesamiento rápido, pero hace el balanceo menos preciso. Es un trade-off entre eficiencia de red y precisión de distribución.

**Advertencia:** Si todos los workers están ocupados y los mensajes siguen llegando, la cola crecerá indefinidamente. QoS distribuye carga entre workers existentes, pero no crea workers automáticamente. Debes monitorear tamaño de cola y agregar capacidad cuando sea necesario.

---