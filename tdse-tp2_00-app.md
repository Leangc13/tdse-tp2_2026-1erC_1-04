¡Hola! Analizando en detalle los archivos adjuntos, podemos ver que en su conjunto forman un sistema de planificación de tareas (scheduler) disparado por tiempo (Time-Triggered) con un sistema integrado de perfilado (profiling) para medir cuánto tarda en ejecutarse cada bloque de código.

A continuación, te explico la función de cada módulo y cómo evolucionan las variables solicitadas.

### 1. Análisis del funcionamiento por archivo

* **`dwt.h`**: Este archivo configura y maneja el Data Watchpoint and Trace (DWT), un periférico interno de los núcleos ARM Cortex-M. Su función principal aquí es habilitar el registro de hardware `CYCCNT`, un contador que se incrementa en cada ciclo de reloj de la CPU. Al dividir estos ciclos por la frecuencia del sistema (en MHz), se obtienen mediciones de tiempo de altísima precisión en microsegundos.
* **`systick.c`**: Contiene una función (`systick_delay_us`) que utiliza el timer del sistema (SysTick) para generar retardos bloqueantes (delays) en microsegundos. El SysTick lee su valor actual y calcula cuánto falta para alcanzar el objetivo, bloqueando el código en un ciclo `while(1)` hasta que pasa el tiempo requerido.
* **`logger.c` y `logger.h`**: Conforman el módulo de depuración para enviar mensajes a la PC. La macro `LOGGER_INFO` deshabilita las interrupciones globales (`__asm("CPSID i")`), formatea un string usando `snprintf`, lo imprime vía Semihosting (usando `printf`) y luego vuelve a habilitar las interrupciones (`__asm("CPSIE i")`).
* **`app.c`**: Es el planificador principal del sistema bare-metal. 
    * En `app_init()`, se inicializan los contadores, se preparan las tareas y se configuran sus métricas iniciales.
    * En `app_update()`, se evalúa constantemente si ocurrió un "tick" del sistema (disparado por `HAL_SYSTICK_Callback()`). Si es así, se itera sobre todas las tareas definidas, se resetea el contador DWT, se ejecuta la tarea y se miden sus tiempos para actualizar sus estadísticas.

---

### 2. Evolución de las variables desde `app_init()` hasta `app_update()`

* **`g_app_tick_cnt`**
    * **Unidad:** Ticks del sistema (por el contexto, representa **milisegundos**).
    * **Evolución:** En `app_init()`, se inicializa en `0`. Durante la ejecución normal, la función `HAL_SYSTICK_Callback()` lo incrementa asíncronamente en cada interrupción del timer. En el loop de `app_update()`, la variable se decrementa (`g_app_tick_cnt--`) para "consumir" los ticks pendientes y habilitar la ejecución de las tareas.
* **`index`**
    * **Unidad:** Adimensional (índice de un arreglo).
    * **Evolución:** Es la variable local utilizada en los bucles `for` tanto en `app_init()` como en `app_update()`. Comienza en `0` y aumenta secuencialmente de a uno hasta alcanzar `TASK_QTY - 1` (la cantidad total de tareas configuradas).
* **`g_app_runtime_us`**
    * **Unidad:** Microsegundos (us).
    * **Evolución:** En cada "tick" habilitado dentro de `app_update()`, se reinicia a `0` antes de iterar sobre las tareas. Luego, a medida que las tareas se ejecutan, se le suma progresivamente el tiempo medido de cada una (`task_dta_list[index].LET`). Representa el tiempo total que tomó ejecutar todas las tareas en esa pasada.
* **`task_dta_list[index].NOE` (Number of Executions)**
    * **Unidad:** Adimensional (cantidad).
    * **Evolución:** En `app_init()`, arranca en `0`. Dentro de `app_update()`, se incrementa en `1` (`++`) justo después de que la tarea especificada ha sido ejecutada.
* **`task_dta_list[index].LET` (Last Execution Time)**
    * **Unidad:** Microsegundos (us).
    * **Evolución:** Inicia en `0` durante `app_init()`. En `app_update()`, se sobreescribe en cada ciclo con el valor arrojado por `cycle_counter_get_time_us()`, registrando exactamente cuánto tardó la tarea en esa iteración particular.
* **`task_dta_list[index].BCET` (Best-case Execution Time)**
    * **Unidad:** Microsegundos (us).
    * **Evolución:** En `app_init()`, se inicializa intencionalmente con un valor alto (`1000`). En `app_update()`, se compara con el tiempo actual medido (`LET`); si el nuevo `LET` es menor que el `BCET` actual, se actualiza (`BCET = LET`). Guarda el récord del tiempo más rápido.
* **`task_dta_list[index].WCET` (Worst-case Execution Time)**
    * **Unidad:** Microsegundos (us).
    * **Evolución:** En `app_init()`, inicia en `0`. Durante `app_update()`, si el tiempo recién medido (`LET`) es mayor que el `WCET` actual, este último se actualiza (`WCET = LET`). Registra el peor tiempo (el más lento) que tardó la tarea en ejecutarse.

---

### 3. Impacto de usar `LOGGER_INFO()`

Usar `LOGGER_INFO()` dentro de la rutina de una tarea (es decir, el bloque de código cuyo tiempo está siendo cronometrado por el DWT) tiene un impacto destructivo en las mediciones del sistema:

1.  **En `g_app_runtime_us`:** La macro `LOGGER_INFO` usa la función `snprintf` y realiza comunicación por semihosting (pausando la ejecución para hablar con la PC a través del debugger). Esto es un proceso increíblemente lento desde la perspectiva del microcontrolador. Al estar dentro de la tarea, el contador de ciclos (DWT) no se detiene. Por ende, el tiempo de la tarea (`LET`) dará un salto gigantesco, lo que a su vez causará que el total `g_app_runtime_us` se dispare a valores muy elevados.
2.  **En `task_dta_list[index].WCET`:** Como el tiempo de la última ejecución (`LET`) aumentó artificialmente por culpa del envío de caracteres a la consola, el sistema registrará este valor como un "nuevo peor tiempo". En consecuencia, el `WCET` quedará manchado y reflejará el tiempo de transmisión del log, perdiendo por completo la capacidad de mostrarte el verdadero tiempo del peor caso de la lógica de tu máquina de estados o algoritmo.
3.  **En el comportamiento del sistema (Interrupciones):** Además del tiempo perdido, `LOGGER_INFO` incluye instrucciones de ensamblador para deshabilitar las interrupciones (`__asm("CPSID i")`) mientras procesa e imprime. Si el log es muy extenso, el sistema estará sordo (incluso ante el SysTick) por periodos de tiempo no deseados, afectando la determinismo del sistema.
