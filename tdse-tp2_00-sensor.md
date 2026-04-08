Basado en los archivos proporcionados, el sistema implementa una máquina de estados para leer un botón (sensor) y un mecanismo de comunicación entre tareas (una cola de eventos) para avisarle al resto del sistema qué pasó con dicho botón.

A continuación, se detalla el análisis del funcionamiento y la evolución de las variables:

### 1. Análisis de los Archivos

* **`task_sensor_attribute.h`**: Define las piezas fundamentales para la tarea del sensor. Incluye los eventos que puede sufrir el botón (`EV_BTN_UP`, `EV_BTN_DOWN`), los estados en los que puede estar (`ST_BTN_IDLE`, `ST_BTN_ACTIVE`), la estructura de configuración constante (qué puerto y pin de hardware usa) y la estructura de datos variables (`tick`, `state`, `event`).
* **`task_system_attribute.h`**: Define los eventos (`EV_SYS_IDLE`, `EV_SYS_ACTIVE`) y estados (`ST_SYS_IDLE`, `ST_SYS_ACTIVE`) para la "Tarea del Sistema", que es la que va a recibir las señales de la tarea del sensor para accionar alguna lógica (como prender un LED).
* **`task_sensor.c`**: Contiene la lógica de la máquina de estados. Define la configuración de un botón (`ID_BTN_A`), inicializa su estado y periódicamente evalúa si fue presionado o soltado.
* **`task_system_interface.c`**: Implementa una "cola circular" (Buffer FIFO) para comunicar tareas. Permite que la tarea del sensor encole un mensaje (`put_event_task_system`) y que, más adelante, la tarea del sistema lo desencole para procesarlo.

---

### 2. Evolución de las variables en `task_sensor.c`

* **`index`**
    * **Evolución:** Es la variable para recorrer el arreglo de sensores en los bucles `for`. Inicia en `0` en `task_sensor_init()` y `task_sensor_update()`. Como en este código `SENSOR_DTA_QTY` vale 1 (solo hay definido un botón en `task_sensor_cfg_list`), el índice solo valdrá `0`.
* **`task_sensor_dta_list[index].tick`**
    * **Unidad:** Milisegundos (mS) (indicado en el comentario de la cabecera "Update by Time Code, period = 1mS" y las macros `DEL_BTN_MAX`).
    * **Evolución:** En el código actual, esta variable **no evoluciona durante el flujo normal**. La máquina de estados provista es directa y no tiene implementado un retardo de "antirrebote" (debouncing) en los `case`. El `tick` únicamente adquiere el valor `DEL_BTN_MIN` si el `switch` llegara a caer en el bloque `default` por algún error.
* **`task_sensor_dta_list[index].state`**
    * **Evolución:** Al arrancar el sistema en `task_sensor_init()`, se fuerza el estado inicial a `ST_BTN_IDLE`. Durante `task_sensor_update()`, pasará a `ST_BTN_ACTIVE` cuando el usuario presione el botón, y regresará a `ST_BTN_IDLE` cuando el usuario lo suelte.
* **`task_sensor_dta_list[index].event`**
    * **Evolución:** En `task_sensor_init()` arranca con el valor artificial `EV_BTN_UP`. Una vez que el loop principal llama a `task_sensor_update()`, este evento cambia continuamente en cada iteración leyendo directamente el hardware: toma el valor `EV_BTN_DOWN` si el GPIO indica que está presionado, o `EV_BTN_UP` si está suelto.

---

### 3. Comportamiento de `task_sensor_statechart(uint32_t index)`

Esta función es la máquina de estados de Mealy/Moore y actúa de la siguiente manera:
1.  **Lectura de Entradas:** Primero lee el estado eléctrico del pin a través de `HAL_GPIO_ReadPin`. Si el estado del pin coincide con la configuración de "presionado" (`pressed`), actualiza la variable `event` de la tarea a `EV_BTN_DOWN`; caso contrario, le asigna `EV_BTN_UP`.
2.  **Evaluación de Estados (Switch-Case):**
    * **Si está en `ST_BTN_IDLE`:** Verifica si ocurrió el evento `EV_BTN_DOWN`. Si es así, significa que se acaba de apretar el botón. La función envía la señal configurada (`signal_down`, que equivale a `EV_SYS_ACTIVE`) hacia la cola del sistema usando `put_event_task_system()`, y cambia su estado a `ST_BTN_ACTIVE`.
    * **Si está en `ST_BTN_ACTIVE`:** Verifica si ocurrió el evento `EV_BTN_UP`. Si es así, significa que se soltó el botón. Envía la señal `signal_up` (que equivale a `EV_SYS_IDLE`) a la cola, y retorna al estado `ST_BTN_IDLE`.
    * **Si cae en `default`:** Reinicia las variables a un estado seguro (`ST_BTN_IDLE`).

---

### 4. Evolución de la Cola de Eventos (`event_task_system_queue`)

Estas variables en `task_system_interface.c` manejan la transferencia segura de señales entre tareas:

* **Estado Inicial (antes o durante `task_sensor_init`)**:
    * Se asume que al arrancar se llama a `init_event_task_system()`.
    * `head = 0`, `tail = 0`, `count = 0`.
    * Todo el arreglo `queue[i]` se llena con el valor `EMPTY`.
* **En sucesivas ejecuciones de `task_sensor_update()`**:
    Cada vez que el usuario presiona o suelta el botón en la máquina de estados, se invoca `put_event_task_system(event)`. En ese instante:
    1.  **`count`**: Se incrementa en `1` (`count++`), indicando que hay un nuevo evento sin leer.
    2.  **`queue[i]`**: Se guarda el evento enviado (ej. `EV_SYS_ACTIVE`) en el índice marcado por `head` (`queue[head] = event`).
    3.  **`head` (Cabeza)**: Avanza una posición hacia adelante (`head++`) para apuntar al siguiente espacio vacío. Si llega al límite de la cola (`QUEUE_LENGTH`), vuelve a saltar a `0`, formando un anillo.
    4.  **`tail` (Cola)**: No se modifica en este flujo de inyección de datos. Solo avanzará cuando la "Tarea del Sistema" decida leer el evento llamando a `get_event_task_system()`.
