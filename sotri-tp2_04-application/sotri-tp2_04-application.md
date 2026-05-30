# sotri-tp2_04-application

## Análisis del código fuente — Consulta (Paso 06)

### A. `startup_stm32f446retx.s`

Es el archivo de arranque escrito en ensamblador ARM Cortex-M4. Sus responsabilidades son:

* **Tabla de vectores (`g_pfnVectors`):** Define el mapa de vectores de interrupción en la dirección 0x0000_0000. La primera entrada es el puntero inicial de pila (`_estack`) y la segunda la dirección de `Reset_Handler`. Le siguen las excepciones del núcleo (`NMI`, `HardFault`, `SVC`, `PendSV`, `SysTick`) y todas las IRQ externas del STM32F446.
* **`Reset_Handler`:** Es el punto de entrada físico del procesador tras un reset. Realiza en orden:
  1. Carga el stack pointer con `_estack`.
  2. Llama a `SystemInit` (CMSIS) para configuración básica del reloj.
  3. Copia la sección `.data` desde Flash hacia SRAM (`_sidata` → `_sdata`:`_edata`).
  4. Inicializa en cero la sección `.bss` (`_sbss`:`_ebss`).
  5. Llama a `__libc_init_array` (constructores estáticos de C).
  6. Salta a `main()`.
* **`Default_Handler`:** Bucle infinito que atrapa cualquier interrupción no implementada, útil para detectar vectores sin handler durante depuración.
* **Aliases débiles (`.weak`):** Cada IRQ tiene un alias débil hacia `Default_Handler`. Si el código define una función con el mismo nombre, la reemplaza automáticamente en tiempo de enlace.

---

### B. `main.c`

Es el punto de entrada en C. Su flujo:

* **`initialise_monitor_handles()`:** Si semihosting está activo (`LOGGER_CONFIG_USE_SEMIHOSTING`), inicializa el canal de comunicación con el depurador antes de cualquier otra cosa.
* **`HAL_Init()`:** Inicializa la HAL, configura la prioridad de grupos de interrupción y establece TIM1 como base de tiempo (liberando SysTick para FreeRTOS).
* **`SystemClock_Config()`:** Configura el PLL con HSI=16 MHz como fuente, M=16, N=336, P=4 → SYSCLK = **84 MHz**.
* **`MX_GPIO_Init()`:** Configura B1 (botón) como `GPIO_MODE_IT_FALLING` con interrupción EXTI habilitada en prioridad 5, y LD2 como salida push-pull.
* **`MX_USART2_UART_Init()`:** UART2 a 115200 bps para el logger.
* **`MX_TIM2_Init()`:** Configura TIM2 con prescaler=1 y periodo=4199 → desborda a ~10 kHz, utilizado como contador de alta frecuencia para `configGENERATE_RUN_TIME_STATS`.
* **`HAL_TIM_Base_Start_IT(&htim2)`:** Arranca TIM2 en modo interrupción.
* **`app_init()`:** Inicializa la aplicación (crea tareas, colas y semáforos).
* **`osKernelStart()`:** Transfiere el control al scheduler de FreeRTOS. El `while(1)` posterior nunca se ejecuta.
* **`HAL_TIM_PeriodElapsedCallback`:** TIM1 incrementa el tick de la HAL; TIM2 incrementa `ulHighFrequencyTimerTicks` para las estadísticas de tiempo de ejecución.

**Evolución de `SystemCoreClock` y `SysTick`:**

| Fase | `SystemCoreClock` | `SysTick` |
| :--- | :--- | :--- |
| `Reset_Handler` | 16 MHz (HSI por defecto) | Inactivo |
| Antes de `SystemClock_Config()` | 16 MHz | Inactivo |
| Después de `SystemClock_Config()` | 84 MHz (PLL activo) | Inactivo |
| Después de `osKernelStart()` | 84 MHz | Activo — 1 kHz (`configTICK_RATE_HZ`) |

---

### C. `stm32f4xx_it.c`

Implementa los manejadores físicos de interrupción:

* **`NMI_Handler`, `HardFault_Handler`, `MemManage_Handler`, `BusFault_Handler`, `UsageFault_Handler`:** Bucles infinitos para atrapar fallos catastróficos y permitir inspección con el depurador.
* **`TIM1_UP_TIM10_IRQHandler`:** Delega a `HAL_TIM_IRQHandler(&htim1)` → genera el tick de la HAL.
* **`TIM2_IRQHandler`:** Delega a `HAL_TIM_IRQHandler(&htim2)` → activa el callback que incrementa `ulHighFrequencyTimerTicks`.
* **`EXTI15_10_IRQHandler`:** Nuevo respecto al TP1. Atiende la interrupción del botón B1 (conectado a una línea EXTI entre 10 y 15) y delega a `HAL_GPIO_EXTI_IRQHandler(B1_Pin)`, que a su vez llama a `HAL_GPIO_EXTI_Callback` definido en `app_it.c`.

---

### D. `FreeRTOSConfig.h`

Archivo de configuración que adapta FreeRTOS al hardware:

* `configUSE_PREEMPTION 1`: Scheduler preemptivo — una tarea de mayor prioridad desaloja inmediatamente a la de menor prioridad.
* `configTICK_RATE_HZ 1000`: Tick del sistema cada 1 ms.
* `configSUPPORT_STATIC_ALLOCATION 1` y `configSUPPORT_DYNAMIC_ALLOCATION 1`: Se admiten ambos esquemas de memoria.
* `configTOTAL_HEAP_SIZE 15360`: 15 KB de heap para objetos dinámicos (tareas, colas, semáforos).
* `configGENERATE_RUN_TIME_STATS 1`: Activa estadísticas de uso de CPU por tarea, apoyadas en TIM2.
* `configUSE_MUTEXES 1`: Habilita el uso de mutexes.
* `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY 5`: Ninguna ISR con prioridad numérica menor a 5 puede llamar a la API de FreeRTOS. El botón B1 está configurado en prioridad 5, por lo que está en el límite habilitado.
* Mapeo de vectores: `vPortSVCHandler` → `SVC_Handler`, `xPortPendSVHandler` → `PendSV_Handler`, `xPortSysTickHandler` → `SysTick_Handler`.

---

### E. `Core/Src/freertos.c`

Provee la infraestructura de soporte del kernel:

* **`vApplicationGetIdleTaskMemory`:** Suministra estáticamente la memoria (TCB + stack) para la tarea Idle, necesario cuando `configSUPPORT_STATIC_ALLOCATION = 1`.
* **`vApplicationIdleHook`** (débil): Punto de extensión ejecutado cada vez que la tarea Idle corre. Sobreescrito en `app/src/freertos.c`.
* **`vApplicationTickHook`** (débil): Ejecutado dentro de la ISR del SysTick en cada tick. Solo puede usar funciones `...FromISR()`. Sobreescrito en `app/src/freertos.c`.
* **`vApplicationStackOverflowHook`** (débil): Invocado si el kernel detecta desbordamiento de stack. Sobreescrito en `app/src/freertos.c`.
* **`configureTimerForRunTimeStats`** y **`getRunTimeCounterValue`** (débiles): Sobreescritos por las funciones en `main.c` que usan TIM2.

---

## Análisis del código fuente — Consulta (Paso 08)

### 1.1. `app.c` — Inicialización y orquestación

Este módulo prepara toda la infraestructura antes de que el scheduler arranque.

**Novedades respecto al TP1:**

* **`QueueHandle_t h_btn_led_q`:** Handle de la cola que se usará en la Actividad 02 para comunicar `task_btn` → `task_led`. Se crea con capacidad de 5 elementos del tipo `task_led_ev_t`.
* **`SemaphoreHandle_t h_btn_led_bin_sem`:** Handle del semáforo binario que se usará en las Actividades 03 y 04. Se crea en estado "vacío" — debe darse (`xSemaphoreGive`) antes de poder tomarse.
* Ambos se registran con `vQueueAddToRegistry` para ser visibles en el depurador (FreeRTOS Kernel Awareness).

**Flujo de `app_init()`:**
1. Inicializa contadores de diagnóstico.
2. Crea la cola `h_btn_led_q` (capacidad 5, tamaño de elemento = `sizeof(task_led_ev_t)`).
3. Crea el semáforo binario `h_btn_led_bin_sem`.
4. Crea `task_btn` y `task_led` con prioridad 1 (`tskIDLE_PRIORITY + 1`), misma pila (2 × `configMINIMAL_STACK_SIZE`).
5. Llama a `app_it_init()` para configurar interrupciones de la aplicación.
6. Llama a `cycle_counter_init()` para habilitar el DWT (contador de ciclos).

---

### 1.2. `app_it.c` — Inicialización de interrupciones y callback

* **`app_it_init()`:** Reserva para lógica de inicialización de interrupciones. Actualmente deshabilita y vuelve a habilitar interrupciones globales (`CPSID i` / `CPSIE i`) como estructura placeholder.
* **`HAL_GPIO_EXTI_Callback()`:** Callback invocado por la HAL cuando se detecta el flanco en B1 (vía `EXTI15_10_IRQHandler` → `HAL_GPIO_EXTI_IRQHandler` → aquí). Actualmente tiene el cuerpo vacío — aquí es donde la Actividad 04 deberá enviar el semáforo binario desde la ISR usando `xSemaphoreGiveFromISR()`.

---

### 1.3. `task_btn.c` — Polling del botón con antirrebote

Respecto al TP1, cambió el mecanismo de temporización:

* Ahora usa `vTaskDelay(BTN_TICK_DEL_MAX)` (50 ms) en lugar de `vTaskDelayUntil`. La diferencia: `vTaskDelay` bloquea la tarea por 50 ms a partir del momento en que se llama (relativo), mientras que `vTaskDelayUntil` mantiene una periodicidad absoluta. Para el antirrebote por polling, ambos son equivalentes en la práctica.
* La máquina de estados es la misma: ST_BTN_UP → ST_BTN_FALLING (validación 50 ms) → ST_BTN_DOWN → ST_BTN_RISING (validación 50 ms) → ST_BTN_UP.
* Al confirmar pulsación llama `put_event_task_led(EV_LED_BLINK)` y al liberar `put_event_task_led(EV_LED_OFF)` — mecanismo que será reemplazado en las Actividades 02 y 03.

---

### 1.4. `task_led.c` — Control del LED

* Usa `vTaskDelayUntil` con período de 50 ms para ejecución periódica precisa.
* Máquina de estados:
  * `ST_LED_OFF`: si recibe `EV_LED_BLINK` con `flag=true`, enciende el LED y pasa a `ST_LED_BLINK`.
  * `ST_LED_BLINK`: si recibe `EV_LED_OFF` con `flag=true`, apaga el LED. Si no, cada 500 ms hace toggle del pin (parpadeo a 1 Hz).
* La comunicación actual llega a través de `task_led_dta.flag` y `task_led_dta.event` — variables compartidas escritas directamente por `put_event_task_led()`.

---

### 1.5. `task_led_interface.c` — Interfaz de comunicación actual

```
void put_event_task_led(task_led_ev_t event)
{
    task_led_dta.event = event;
    task_led_dta.flag  = true;
}
```

Escribe directamente sobre la estructura interna de `task_led`. No utiliza ningún primitivo de FreeRTOS, por lo que no tiene protección contra condiciones de carrera. Esta función es el punto exacto que será reemplazado en la Actividad 02 (por una escritura en cola) y en la Actividad 03 (por un `xSemaphoreGive`).

---

### 1.6. `app/src/freertos.c` — Hooks de la aplicación

* **`vApplicationIdleHook`:** Incrementa `g_task_idle_cnt` en cada iteración de la tarea Idle. Sirve para medir cuánto tiempo el sistema no tiene trabajo útil.
* **`vApplicationTickHook`:** Incrementa `g_app_tick_cnt` en cada tick (1 ms). Corre dentro de la ISR — solo puede usar funciones `FromISR`.
* **`vApplicationStackOverflowHook`:** Entra en sección crítica y hace `configASSERT(0)` — cuelga el sistema de forma controlada para poder depurar qué tarea desbordó su stack.

---

## Resumen del flujo de eventos

```
[ Botón B1 ]
     │  (polling cada 50 ms por task_btn)
     ▼
[ task_btn_statechart ] ──(antirrebote 50 ms)──► [ Confirma evento ]
                                                        │
                                               put_event_task_led()
                                               (escribe en task_led_dta)
                                                        │
                                                        ▼
                                             [ task_led_statechart ]
                                                        │
                                                        ▼
                                               [ LED físico LD2 ]
```

**Estado de reposo:** `task_btn` hace polling cada 50 ms. `task_led` evalúa su FSM cada 50 ms. La mayor parte del tiempo corre la tarea Idle, incrementando `g_task_idle_cnt`.

**Al presionar el botón:** `task_btn` detecta flanco descendente, espera 50 ms, confirma. Llama `put_event_task_led(EV_LED_BLINK)`. `task_led` detecta `flag=true`, enciende el LED y comienza a hacer toggle cada 500 ms.

**Al soltar el botón:** `task_btn` detecta flanco ascendente, espera 50 ms, confirma. Llama `put_event_task_led(EV_LED_OFF)`. `task_led` apaga el LED y vuelve a `ST_LED_OFF`.

---

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
