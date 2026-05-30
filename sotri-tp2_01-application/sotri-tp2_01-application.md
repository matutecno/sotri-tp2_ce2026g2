# sotri-tp2_01-application

## 1. Análisis del código fuente — Consulta (Paso 06)

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

## 2. Indicar la evolución de las variables `SysTick` y `SystemCoreClock` desde `Reset_Handler` hasta el `while(1)` de `main.c`

- **Respuesta:**  
  Al inicio del programa, luego del reset, el microcontrolador trabaja con el reloj interno por defecto. En ese punto `SystemCoreClock` todavía representa la frecuencia inicial del sistema y `SysTick` aún no está funcionando como tick de FreeRTOS.

  Durante `Reset_Handler`, se llama a `SystemInit()`, que realiza una configuración básica del sistema antes de entrar a `main()`. Luego, dentro de `main.c`, `HAL_Init()` inicializa la HAL y prepara la base de tiempo de la librería. Más adelante, `SystemClock_Config()` configura el PLL y actualiza la frecuencia principal del sistema, por lo que `SystemCoreClock` pasa a representar la frecuencia final configurada para la CPU.

  Antes de iniciar FreeRTOS, `SysTick` todavía no actúa como tick del scheduler. Cuando se llama a `osKernelStart()`, FreeRTOS configura `SysTick` para generar interrupciones periódicas según `configTICK_RATE_HZ`. En este proyecto, esa frecuencia es de 1000 Hz, por lo que el tick del RTOS ocurre cada 1 ms.

| Etapa | `SystemCoreClock` | `SysTick` |
|---|---|---|
| Inicio en `Reset_Handler` | Frecuencia inicial del sistema | Inactivo para FreeRTOS |
| Después de `SystemInit()` | Configuración básica del sistema | Aún no controla el scheduler |
| Después de `SystemClock_Config()` | Frecuencia final configurada por PLL | Aún no es el tick de FreeRTOS |
| Después de `osKernelStart()` | Se mantiene en la frecuencia final | Activo como tick de FreeRTOS |

---

## 3. Indicar el comportamiento del programa desde `Reset_Handler` hasta antes de llegar al `while(1)` de `main.c`

- **Respuesta:**  
  La ejecución comienza en `Reset_Handler`, que prepara la memoria y el entorno de C. Primero se configura el stack, luego se inicializa el sistema, se copian las variables inicializadas desde Flash a RAM y se limpian las variables no inicializadas en `.bss`. Después se ejecutan las inicializaciones de la librería C y se llama a `main()`.

  Al entrar en `main.c`, el programa inicializa la HAL, configura el reloj principal del sistema, inicializa los periféricos usados por la aplicación y arranca TIM2 en modo interrupción. Luego se inicializan los recursos propios de la aplicación, como tareas, colas, semáforos y callbacks.

  Finalmente, `osKernelStart()` inicia el scheduler de FreeRTOS. Desde ese punto, el control del programa pasa al RTOS. Por eso, el `while(1)` que aparece al final de `main.c` queda como código de respaldo, pero normalmente no se ejecuta.

---

## 4. Indicar cómo y para qué `SysTick` y `Timer 1 / TIM2` interactúan con FreeRTOS

- **Respuesta:**  
  `SysTick` interactúa directamente con FreeRTOS porque funciona como la base de tiempo del sistema operativo. Cada interrupción de `SysTick` genera un tick del kernel. Ese tick permite que FreeRTOS controle retardos, desbloquee tareas, actualice temporizadores internos y decida si debe realizar un cambio de contexto.

  TIM2 se usa como contador auxiliar de alta frecuencia para las estadísticas de ejecución de FreeRTOS. Cuando `configGENERATE_RUN_TIME_STATS` está habilitado, FreeRTOS necesita una fuente de tiempo más precisa que el tick normal para medir cuánto tiempo de CPU consume cada tarea. En este proyecto, TIM2 cumple ese rol incrementando `ulHighFrequencyTimerTicks`.

  En resumen, `SysTick` mide el tiempo del scheduler, mientras que TIM2 mide tiempo de ejecución para diagnóstico y estadísticas.

---

## 5. Indicar cómo y para qué `TIM2` interactúa con la HAL del proyecto STM32

- **Respuesta:**  
  TIM2 interactúa con la HAL a través de su interrupción. Cuando TIM2 genera un evento de actualización, se ejecuta `TIM2_IRQHandler()` en `stm32f4xx_it.c`. Este handler no procesa directamente toda la lógica, sino que llama a `HAL_TIM_IRQHandler(&htim2)`.

  La HAL identifica el evento del temporizador y ejecuta el callback correspondiente: `HAL_TIM_PeriodElapsedCallback()`. Dentro de ese callback, el código verifica si el temporizador que produjo la interrupción es TIM2. Si lo es, incrementa `ulHighFrequencyTimerTicks`.

  Por lo tanto, TIM2 se usa mediante la infraestructura de la HAL para generar interrupciones periódicas y actualizar un contador de alta frecuencia usado por FreeRTOS para estadísticas de tiempo de ejecución.
