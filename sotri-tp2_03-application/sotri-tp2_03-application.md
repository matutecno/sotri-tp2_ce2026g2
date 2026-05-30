## Actividad 03 — Semáforos binarios y contadores (Paso 02)

### ¿Cómo crear y usar semáforos binarios y semáforos contadores?

**Semáforo binario**

Se crea con `xSemaphoreCreateBinary()` (asignación dinámica; devuelve un `SemaphoreHandle_t`, o `NULL` si no hay memoria en el heap) o con `xSemaphoreCreateBinaryStatic()` (asignación estática). Se crea en estado **vacío**: debe darse al menos una vez con `xSemaphoreGive()` antes de poder tomarse con `xSemaphoreTake()`.

Para usarlo:

* `xSemaphoreGive(xSemaphore)` — libera/señaliza el semáforo (lleva la cuenta de 0 a 1).
* `xSemaphoreTake(xSemaphore, xTicksToWait)` — toma el semáforo (lleva la cuenta de 1 a 0). Si está vacío, la tarea se bloquea hasta `xTicksToWait` ticks. Retorna `pdTRUE` si lo obtuvo, `pdFALSE` si venció el timeout.
* Desde una rutina de servicio de interrupción (ISR) deben usarse las variantes `xSemaphoreGiveFromISR()` y `xSemaphoreTakeFromISR()`.

En este proyecto el semáforo binario se crea en `app_init()`:
```c
h_btn_led_bin_sem = xSemaphoreCreateBinary();
```

**Semáforo contador**

Requiere `configUSE_COUNTING_SEMAPHORES 1` en `FreeRTOSConfig.h`. Se crea con `xSemaphoreCreateCounting(uxMaxCount, uxInitialCount)` (o su variante `...Static`), donde `uxMaxCount` es la cuenta máxima que puede alcanzar y `uxInitialCount` la cuenta con la que arranca.

Se usa con la misma API que el binario: `xSemaphoreGive()` incrementa la cuenta (hasta `uxMaxCount`) y `xSemaphoreTake()` la decrementa (se bloquea o falla si la cuenta es 0). Tiene dos usos típicos:

* **Conteo de eventos:** un evento (por ejemplo, en una ISR) hace `give` por cada ocurrencia, y una tarea hace `take` por cada uno que procesa. La cuenta representa los eventos pendientes de atender.
* **Gestión de recursos:** la cuenta representa la cantidad de recursos disponibles de un pool de `uxMaxCount` elementos.

Ambos tipos se eliminan con `vSemaphoreDelete(xSemaphore)`. Un semáforo binario es, conceptualmente, un semáforo contador con cuenta máxima 1.

### ¿Cuáles son las diferencias entre semáforos binarios y semáforos contadores?

| Aspecto | Binario | Contador |
| :--- | :--- | :--- |
| Cuenta máxima | 1 (disponible / no disponible) | `uxMaxCount`, configurable (N) |
| Estado inicial | Siempre vacío al crearse | Definido por `uxInitialCount` |
| Acumula señales | No: si se da varias veces antes de tomarlo, satura en 1 y las señales sobrantes **se pierden** | Sí: acumula hasta `uxMaxCount` sin perder eventos |
| Uso típico | Sincronización 1-a-1 (señalar "ocurrió un evento"), exclusión mutua simple | Conteo de eventos repetidos, gestión de un pool de N recursos |
| Config. necesaria | — | `configUSE_COUNTING_SEMAPHORES 1` |

La diferencia clave es la **capacidad de conteo**: el binario solo distingue "hay señal" / "no hay señal", mientras que el contador lleva la cuenta de cuántas señales pendientes hay. Esto último es relevante cuando los eventos pueden llegar más rápido de lo que se procesan y no se quiere perder ninguno.

---

## Actividad 03 — Comportamiento observado (Paso 03)

Se modificó el mecanismo de comunicación entre `task_btn` y `task_led` reemplazando la cola de la Actividad 02 por el **semáforo binario** `h_btn_led_bin_sem` (ya creado en `app.c`).

A diferencia de una cola, un semáforo binario **no transporta datos**: solo sincroniza, señalando que "ocurrió un evento" sin indicar cuál. Como `task_btn` generaba dos eventos distintos (`EV_LED_BLINK` al presionar y `EV_LED_OFF` al soltar), un único semáforo binario no puede distinguirlos. Por eso se rediseñó la lógica a un esquema **toggle**: cada pulsación alterna el estado del LED.

**Cambios realizados:**

* `task_led_interface.h` / `task_led_interface.c` — `put_event_task_led()` pierde su parámetro (`void`) y reemplaza la escritura en la cola por `xSemaphoreGive(h_btn_led_bin_sem)`.
* `task_btn.c` — se da el semáforo **únicamente** al confirmar la pulsación (estado PRESSED); se quitó la señalización al soltar (estado HOVER). Así se genera **un solo `give` por pulsación**, condición necesaria para que el toggle funcione (dos `give` por ciclo dejarían el LED en su estado original).
* `task_led.c` — `task_led_statechart()`: en cada estado, la condición del `if` pasa de `xQueueReceive(...)` a `xSemaphoreTake(h_btn_led_bin_sem, 0)` con timeout `0` (no bloqueante). Cada `take` exitoso alterna `ST_LED_OFF` ↔ `ST_LED_BLINK`.

**¿Por qué timeout `0` en `xSemaphoreTake`?**

Por la misma razón que en la Actividad 02: `task_led` debe seguir ejecutando el parpadeo del LED cada 500 ms independientemente de si llega una señal. Si se bloqueara con `portMAX_DELAY`, el toggle de parpadeo no se ejecutaría mientras no se presione el botón. Con timeout `0`, la tarea consulta el semáforo y continúa su ciclo de 50 ms en cualquier caso.

**Comportamiento observado al depurar:**

El comportamiento cambió respecto de la Actividad 02 y pasó a ser de tipo **toggle**, tal como se esperaba del diseño:

1. Primera pulsación → el LED LD2 se enciende y comienza a parpadear a 1 Hz.
2. Al soltar el botón → el LED **sigue parpadeando** (soltar ya no genera ninguna señal).
3. Segunda pulsación → el LED se apaga.

Es decir, el LED reacciona al **flanco de presión** (el evento de presionar, validado tras el antirrebote de 50 ms), no al **nivel** (mantener el botón apretado). Esta es la diferencia esencial con la Actividad 02, donde el evento transportado por la cola permitía que presionar encendiera y soltar apagara.

Una consecuencia de usar un semáforo binario es que la cuenta **satura en 1**: si se generaran dos `give` consecutivos antes de que `task_led` tomara el primero, el segundo se perdería. En la práctica esto no ocurre, porque el antirrebote de 50 ms en `task_btn` y el ciclo de 50 ms de `task_led` garantizan que cada señal se procese antes de la siguiente. Un semáforo contador, en cambio, no perdería esas señales.
