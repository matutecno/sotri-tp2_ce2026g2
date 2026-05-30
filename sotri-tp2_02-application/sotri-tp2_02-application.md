## Actividad 02 — Comunicación por Cola (Paso 02)

### ¿Cómo crear una Cola?

Se crea con `xQueueCreate(uxQueueLength, uxItemSize)`. El primer parámetro es la cantidad máxima de elementos que puede contener; el segundo, el tamaño en bytes de cada elemento. Devuelve un `QueueHandle_t`, que es `NULL` si no hay memoria suficiente en el heap.

En este proyecto se crea en `app_init()`:
```c
h_btn_led_q = xQueueCreate(5, sizeof(task_led_ev_t));
```

### ¿Cómo eliminar una Cola?

Se elimina con `vQueueDelete(QueueHandle_t xQueue)`. Libera la memoria ocupada por la cola en el heap dinámico. Solo debe llamarse cuando ninguna tarea está bloqueada esperando leer o escribir en ella, ya que destruir una cola con tareas bloqueadas produce comportamiento indefinido.

### ¿Cómo gestiona una Cola los datos que contiene?

Las colas de FreeRTOS implementan una política **FIFO** (First In, First Out): los datos se leen en el mismo orden en que fueron escritos. Además, los datos se copian **por valor** dentro del buffer interno de la cola — no por referencia. Esto significa que la variable original puede modificarse o salir de scope después de `xQueueSend` sin afectar el dato ya encolado.

### ¿Cómo enviar datos a una Cola?

Con `xQueueSend(xQueue, pvItemToQueue, xTicksToWait)`. Copia el dato apuntado por `pvItemToQueue` al final de la cola. Si la cola está llena, la tarea se bloquea hasta `xTicksToWait` ticks. Valores típicos de timeout:

* `0`: no bloquear; retorna `errQUEUE_FULL` si no hay espacio.
* `portMAX_DELAY`: bloquear indefinidamente hasta que haya espacio.

Retorna `pdPASS` si el envío fue exitoso.

### ¿Cómo recibir datos de una Cola?

Con `xQueueReceive(xQueue, pvBuffer, xTicksToWait)`. Copia el elemento más antiguo de la cola en `pvBuffer` y lo elimina de la cola. Si la cola está vacía, la tarea se bloquea hasta `xTicksToWait` ticks. Retorna `pdTRUE` si recibió un dato, `pdFALSE` si expiró el timeout sin recibir nada.

### ¿Qué significa bloquearse en una Cola?

Una tarea se bloquea cuando intenta leer de una cola vacía (o escribir en una cola llena) y especifica un timeout mayor a `0`. Mientras está bloqueada, pasa al estado **Blocked** y no consume tiempo de CPU: el scheduler selecciona otra tarea para ejecutar. La tarea vuelve al estado **Ready** cuando el evento esperado ocurre (dato disponible o espacio libre) o cuando vence el timeout.

### ¿Cómo bloquearse en varias Colas?

Mediante **Queue Sets** (`xQueueCreateSet`, `xQueueAddToSet`, `xQueueSelectFromSet`). Un Queue Set agrupa varias colas y/o semáforos. La tarea llama a `xQueueSelectFromSet` con un timeout y queda bloqueada hasta que *alguno* de los objetos del set tenga datos disponibles. La función retorna el handle del objeto que disparó el desbloqueo, permitiendo luego leer de él específicamente.

### ¿Cómo sobrescribir datos en una Cola?

Con `xQueueOverwrite(xQueue, pvItemToQueue)`. Escribe en una cola de capacidad 1 sin bloquearse, incluso si ya contiene un elemento (lo sobreescribe). Es útil para colas de "último valor conocido" donde no importa el historial sino solo el dato más reciente, como el estado actual de un sensor.

### ¿Cómo vaciar una Cola?

Con `xQueueReset(xQueue)`. Descarta todos los elementos presentes y devuelve la cola a su estado inicial vacío. Las tareas bloqueadas esperando escribir en ella no son desbloqueadas por esta operación.

### ¿Cuál es el efecto de las prioridades de las Tareas al escribir y leer en una Cola?

Las prioridades determinan qué tarea obtiene la CPU después de un evento en la cola:

* Si una tarea de **baja prioridad** escribe en una cola donde una tarea de **alta prioridad** estaba bloqueada esperando datos, FreeRTOS desbloquea a la tarea de alta prioridad y realiza un cambio de contexto inmediato — la tarea de alta prioridad corre antes de que la escritora continúe.
* Si varias tareas están bloqueadas esperando leer de la misma cola, cuando llegue un dato se desbloqueará la de **mayor prioridad**; en caso de igual prioridad, la que lleva más tiempo esperando.

Esto garantiza que el sistema sea reactivo: la tarea más urgente procesa el evento tan pronto como este está disponible.

---

## Actividad 02 — Comportamiento observado (Paso 03)

Se modificó el mecanismo de comunicación entre `task_btn` y `task_led` reemplazando las variables compartidas (`task_led_dta.flag` / `task_led_dta.event`) por una **cola FreeRTOS** (`h_btn_led_q`, capacidad 5 elementos de tipo `task_led_ev_t`).

**Cambios realizados:**

* `task_led_interface.c` — `put_event_task_led()`: reemplaza la escritura directa en la estructura por `xQueueSend(h_btn_led_q, &event, 0)`.
* `task_led.c` — `task_led_statechart()`: en cada estado del `switch`, la condición del `if` pasa de chequear `task_led_dta.flag` a llamar `xQueueReceive(h_btn_led_q, &event, 0)` con timeout `0` (no bloqueante).

**¿Por qué timeout `0` en `xQueueReceive`?**

La tarea `task_led` tiene trabajo propio que realizar independientemente de si llega un evento: el parpadeo del LED cada 500 ms. Si se usara `portMAX_DELAY`, la tarea quedaría bloqueada esperando un evento y el toggle nunca se ejecutaría mientras el botón no se presione. Con timeout `0`, la tarea pregunta si hay algo en la cola y continúa su ciclo de 50 ms pase lo que pase.

**Comportamiento observado al depurar:**

Tras cargar el firmware en la placa, el comportamiento funcional resultó idéntico al de la Actividad 01: al presionar el botón B1, el LED LD2 se enciende y comienza a parpadear a 1 Hz; al soltarlo, el LED se apaga. No se apreció ninguna diferencia visible respecto del mecanismo anterior basado en variables compartidas.

Este resultado es el esperado y correcto. Lo que se modificó fue únicamente el mecanismo de transporte del evento entre `task_btn` y `task_led` (variable compartida → cola), no la lógica de la máquina de estados del LED ni la relación causa-efecto entre el botón y el LED. Por lo tanto, la conducta observable a simple vista debía permanecer igual.

La diferencia real introducida por la cola no es perceptible a simple vista, sino estructural:

* La comunicación pasa a ser segura ante concurrencia (*thread-safe*): `xQueueSend` y `xQueueReceive` copian el dato de forma atómica dentro del buffer interno de la cola, lo que elimina la condición de carrera latente que existía al escribir directamente sobre `task_led_dta`.
* El productor y el consumidor quedan desacoplados, y la cola (de capacidad 5) podría almacenar varios eventos pendientes. En esta aplicación, al usar `xQueueReceive` de forma no bloqueante (timeout `0`), el parpadeo periódico de 500 ms se mantiene intacto independientemente de la llegada de eventos.

Como observación complementaria sobre la configuración de reloj, durante la depuración se verificó la evolución de `SystemCoreClock`: pasa de 16 MHz (HSI por defecto, en el `Reset_Handler`) a 84 MHz una vez ejecutado `SystemClock_Config()` con el PLL activo, según la tabla de la sección B. Este cambio de frecuencia es independiente de la migración a cola, ya que afecta la velocidad de ejecución del sistema y no la lógica de comunicación entre tareas.
