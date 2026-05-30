# sotri-tp2_02-application

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
