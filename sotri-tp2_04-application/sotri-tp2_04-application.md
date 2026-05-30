## Actividad 04 — Interrupciones y FreeRTOS (Paso 02)

### ¿Qué funciones de la API de FreeRTOS se pueden usar dentro de una rutina de servicio de interrupción?

Dentro de una ISR **solo** pueden llamarse las variantes terminadas en `...FromISR`. Las versiones normales de la API nunca deben invocarse desde una interrupción, porque pueden intentar bloquear el contexto actual, lo que no tiene sentido en una ISR. Ejemplos de funciones seguras:

* Colas: `xQueueSendFromISR()`, `xQueueSendToBackFromISR()`, `xQueueReceiveFromISR()`.
* Semáforos: `xSemaphoreGiveFromISR()`, `xSemaphoreTakeFromISR()`.
* Notificaciones de tarea: `vTaskNotifyGiveFromISR()`, `xTaskNotifyFromISR()`.
* Grupos de eventos y timers: `xEventGroupSetBitsFromISR()`, `xTimerStartFromISR()`.

Estas funciones reciben un parámetro de salida `pxHigherPriorityTaskWoken`: si la operación desbloquea una tarea de mayor prioridad que la interrumpida, se setea en `pdTRUE`, y al final de la ISR debe llamarse `portYIELD_FROM_ISR()` con ese valor para forzar el cambio de contexto.

Además, hay una restricción de **prioridad de interrupción**: solo las ISR cuya prioridad sea numéricamente **igual o menor** (es decir, de igual o menor urgencia) que `configMAX_SYSCALL_INTERRUPT_PRIORITY` pueden llamar a la API `FromISR`. En este proyecto `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY = 5` y la EXTI del botón B1 está configurada en prioridad 5, justo en el límite habilitado.

### ¿Cuáles son los métodos para delegar el procesamiento de interrupciones a una Tarea?

La ISR debe ser lo más corta posible: capta el evento y delega el trabajo pesado a una tarea (patrón *deferred interrupt processing*). Los mecanismos típicos para esa delegación son:

* **Semáforo (binario o contador):** la ISR hace `give` y una tarea dedicada hace `take` (bloqueante). Es el método usado en esta actividad.
* **Notificación directa a tarea** (`vTaskNotifyGiveFromISR` / `ulTaskNotifyTake`): más liviana y rápida que un semáforo, ya que no requiere crear un objeto aparte.
* **Cola:** cuando además del aviso hay que pasar datos, la ISR los encola con `xQueueSendFromISR` y la tarea los recibe.
* **Función diferida** (`xTimerPendFunctionCallFromISR`): la ISR pide que una función se ejecute luego en el contexto de la tarea del *timer service*.

La tarea que procesa el evento suele tener prioridad alta para que, gracias a `portYIELD_FROM_ISR`, corra inmediatamente al salir de la ISR. En este proyecto, `HAL_GPIO_EXTI_Callback()` da el semáforo `h_btn_int_bin_sem` y `task_btn` lo toma con `portMAX_DELAY`, haciendo el trabajo fuera de la ISR.

### ¿Cómo usar una cola para transferir datos dentro y fuera de una rutina de servicio de interrupción?

Una cola, a diferencia de un semáforo, **transporta datos** (los copia por valor), por lo que sirve cuando la interrupción debe entregar información además de señalizar:

* **Fuera de la ISR (ISR → tarea):** la ISR escribe con `xQueueSendFromISR(handle, &dato, &higher_priority_task_woken)` y una tarea lo recibe con `xQueueReceive()`. Es el caso habitual: por ejemplo, una ISR de UART que encola cada byte recibido para que una tarea lo procese.
* **Dentro de la ISR (tarea → ISR):** una tarea encola datos con `xQueueSend()` y la ISR los lee con `xQueueReceiveFromISR(handle, &dato, &higher_priority_task_woken)`. Por ejemplo, una ISR de transmisión que toma de la cola el próximo byte a enviar.

En ambos sentidos, las funciones `FromISR` actualizan `pxHigherPriorityTaskWoken`, y al terminar la ISR debe llamarse `portYIELD_FROM_ISR()` con ese valor. La ventaja sobre el semáforo es el transporte de datos; la desventaja, un costo algo mayor (copia del elemento).

### ¿Cuál es el modelo de anidamiento de interrupciones disponible en algunas portaciones de FreeRTOS?

En portaciones como la de ARM Cortex-M, FreeRTOS soporta **anidamiento de interrupciones** (una ISR de mayor prioridad puede interrumpir a otra en curso) gobernado por dos umbrales de prioridad:

* **`configKERNEL_INTERRUPT_PRIORITY`** — la prioridad (la más baja del sistema) que usan el tick del kernel y `PendSV`. Garantiza que el cambio de contexto no interfiera con interrupciones de aplicación.
* **`configMAX_SYSCALL_INTERRUPT_PRIORITY`** — la prioridad máxima desde la cual se permite llamar a la API `FromISR`. Las secciones críticas del kernel enmascaran las interrupciones hasta este nivel.

De aquí surgen dos categorías de interrupción (recordando que en Cortex-M **menor número = mayor prioridad**):

1. Interrupciones con prioridad **igual o por debajo** de `configMAX_SYSCALL_INTERRUPT_PRIORITY`: pueden usar la API `FromISR` y son enmascaradas por las secciones críticas del kernel.
2. Interrupciones con prioridad **por encima** (más urgentes): nunca son enmascaradas por FreeRTOS —ofrecen latencia mínima y pueden anidar incluso durante secciones críticas—, pero **no pueden** llamar a ninguna función de la API.

Para que el esquema funcione, el agrupamiento del NVIC debe usar todos los bits como prioridad de *preempción* (sin subprioridad); en este proyecto `HAL_Init()` configura `NVIC_PRIORITYGROUP_4`.

---

## Actividad 04 — Comportamiento observado (Paso 03)

Se modificó la forma de atender el botón: en lugar del **polling** de la Actividad 03 (donde `task_btn` leía el pin cada 50 ms), ahora el botón genera una **interrupción externa (EXTI)** y la ISR delega el procesamiento a `task_btn` mediante un semáforo binario. La infraestructura EXTI ya venía cableada en el proyecto base (B1 configurado como `GPIO_MODE_IT_FALLING`, prioridad 5, atendido por `EXTI15_10_IRQHandler` → `HAL_GPIO_EXTI_IRQHandler` → `HAL_GPIO_EXTI_Callback`).

El sistema usa ahora **dos semáforos binarios** encadenados:

* `h_btn_int_bin_sem` (nuevo) — comunica la **ISR → `task_btn`**.
* `h_btn_led_bin_sem` (de la Actividad 03) — comunica **`task_btn` → `task_led`** (el toggle, sin cambios).

**Cambios realizados:**

* `app.h` / `app.c` — se declara, define y crea el nuevo semáforo `h_btn_int_bin_sem` con `xSemaphoreCreateBinary()` (estado inicial vacío).
* `app_it.c` — `HAL_GPIO_EXTI_Callback()`: dentro del `if (GPIO_Pin == BTN_A_PIN)` se da el semáforo desde la ISR:
  ```c
  BaseType_t higher_priority_task_woken = pdFALSE;
  xSemaphoreGiveFromISR(h_btn_int_bin_sem, &higher_priority_task_woken);
  portYIELD_FROM_ISR(higher_priority_task_woken);
  ```
  Fue necesario agregar `#include "cmsis_os.h"` (los tipos y la API de FreeRTOS no estaban incluidos en este archivo) y `#include "app.h"` (para ver el handle).
* `task_btn.c` — el lazo principal pasa de sondear a **bloquearse** esperando la interrupción:
  ```c
  xSemaphoreTake(h_btn_int_bin_sem, portMAX_DELAY);
  put_event_task_led();
  ```
  Se eliminaron el `task_btn_statechart()` (polling con antirrebote) y el `vTaskDelay(50 ms)`.

**¿Por qué `portMAX_DELAY` (bloqueante) en `task_btn`, a diferencia del timeout `0` de `task_led`?**

`task_btn` no tiene ningún trabajo que hacer mientras no se presione el botón, por lo que conviene que se bloquee indefinidamente y no consuma CPU; despierta solo cuando la ISR da el semáforo. `task_led`, en cambio, debe seguir parpadeando, así que consulta su semáforo sin bloquearse (timeout `0`).

**Comportamiento observado al depurar:**

El comportamiento funcional es el mismo *toggle* de la Actividad 03 (primera pulsación enciende y hace parpadear; segunda pulsación apaga), pero ahora **disparado por interrupción** en lugar de por sondeo:

1. Al presionar B1, la EXTI dispara la ISR, que da `h_btn_int_bin_sem`.
2. `task_btn`, que estaba bloqueada, despierta inmediatamente (gracias a `portYIELD_FROM_ISR`), registra `"BTN PRESSED (ISR)"` y llama a `put_event_task_led()`.
3. `task_led` recibe la señal por `h_btn_led_bin_sem` y alterna el estado del LED.

La diferencia estructural respecto de la Actividad 03 es la **eficiencia**: ya no hay polling cada 50 ms; `task_btn` permanece en estado *Blocked* y la CPU queda libre (mayor tiempo en la tarea Idle) hasta que ocurre la interrupción. Esto reduce la latencia de respuesta y el consumo.

Como la EXTI dispara por flanco, un pulsador mecánico podría generar rebotes (varios flancos por pulsación). Al ser `h_btn_int_bin_sem` un semáforo binario, su cuenta satura en 1, por lo que gives sucesivos muy próximos se descartan; en las pruebas en placa el comportamiento resultó estable. De observarse toggles erráticos, el refinamiento sería agregar un pequeño antirrebote (un `vTaskDelay` breve) en `task_btn` tras el `take`.
