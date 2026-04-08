Basado en los archivos proporcionados y el contexto del proyecto, este módulo se encarga de controlar las salidas físicas del sistema (en este caso, un LED) mediante una máquina de estados orientada a eventos. A diferencia del Sistema (que usa una cola FIFO), el Actuador recibe órdenes a través de un mecanismo de buzón directo usando una "bandera" (flag).

Aquí tienes el análisis detallado:

### 1. Análisis General de los Archivos

* **`task_actuator_attribute.h`**: Define el esqueleto del módulo. Declara los estados posibles (`ST_LED_IDLE`, `ST_LED_ACTIVE`), los eventos que puede recibir (`EV_LED_IDLE`, `EV_LED_ACTIVE`), y el identificador de los actuadores (ej. `ID_LED_A`). También define las estructuras de configuración (pines, puertos) y de datos (`tick`, `state`, `event`, `flag`).
* **`task_actuator.c`**: Contiene la lógica interna. Inicializa las variables del actuador en `task_actuator_init()`, y en el bucle principal evalúa la máquina de estados dentro de `task_actuator_update()`.
* **`task_actuator_interface.c`**: Es el punto de acceso público. Provee la función `put_event_task_actuator()` para que otras tareas (como el Sistema) puedan enviarle órdenes al actuador sin romper el encapsulamiento.

---

### 2. Evolución de variables (index, tick, state, event, flag)

Desde que arranca el sistema en `task_actuator_init()` y durante las iteraciones de `task_actuator_update()`, estas variables que componen la estructura de la tarea evolucionan así:

* **`index`**:
    * **Evolución:** Es la variable local que controla los bucles `for` para iterar sobre todos los actuadores. Tanto en el `init` como en el `update`, comienza en `0` y se incrementa hasta `ACTUATOR_DTA_QTY - 1`. Dado que solo tienes un LED (`ID_LED_A`), generalmente vale `0`.
* **`task_actuator_dta_list[index].tick`**:
    * **Unidad:** Milisegundos (ms).
    * **Evolución:** En `task_actuator_init()` se inicializa a `0`. Durante el loop principal en `task_actuator_update()`, esta variable **no evoluciona** ni se incrementa. Esto ocurre porque la máquina de estados del actuador es puramente dependiente de eventos (Event-Triggered), no tiene temporizadores internos ni demoras programadas.
* **`task_actuator_dta_list[index].state`**:
    * **Evolución:** En la inicialización se fuerza al estado de reposo `ST_LED_IDLE`. Evolucionará a `ST_LED_ACTIVE` solo cuando la máquina de estados procese el evento de encendido, y regresará a `ST_LED_IDLE` cuando procese el de apagado.
* **`task_actuator_dta_list[index].event`**:
    * **Evolución:** Se inicializa por defecto en `EV_LED_IDLE`. Su valor permanecerá estático hasta que, asíncronamente, alguna otra tarea llame a la función de la interfaz para sobreescribir este evento con una nueva orden.
* **`task_actuator_dta_list[index].flag`**:
    * **Evolución:** Es la variable clave del buzón. En `init` se inicializa en `false`. Permanece en `false` durante el `update` continuo, indicando que "no hay novedades". Cambiará a `true` (por acción externa) cuando llegue un mensaje, y la propia máquina de estados la devolverá a `false` apenas consuma dicho mensaje.

---

### 3. Comportamiento de `task_actuator_statechart(uint32_t index)`

Esta función es la Máquina de Estados del actuador y su comportamiento es directo y reactivo:

1.  **Mapeo de Datos:** Primero toma el `index` y crea punteros locales (`p_task_actuator_cfg` y `p_task_actuator_dta`) para acceder rápidamente a qué pin debe mover y cuál es su estado actual.
2.  **Evaluación (Switch-Case):**
    * **Si está en `ST_LED_IDLE` (LED apagado):**
        Pregunta si hay un nuevo mensaje sin leer (`flag == true`) y si ese mensaje es una orden de encendido (`event == EV_LED_ACTIVE`). Si se cumplen ambas, el actuador:
        * "Consume" el evento bajando la bandera (`flag = false`).
        * Ejecuta la acción física: enciende el LED a través de `HAL_GPIO_WritePin()`.
        * Actualiza su estado interno a `ST_LED_ACTIVE`.
    * **Si está en `ST_LED_ACTIVE` (LED encendido):**
        Pregunta si hay un mensaje nuevo (`flag == true`) y si es una orden de apagado (`event == EV_LED_IDLE`). De ser así:
        * Baja la bandera (`flag = false`).
        * Apaga el LED físicamente usando la HAL.
        * Vuelve al estado `ST_LED_IDLE`.

---

### 4. Evolución de variables de Interfaz (identifier, event, flag)

Esta parte explica cómo interactúa el entorno exterior con el actuador a través de `task_actuator_interface.c` (`put_event_task_actuator`):

* **`identifier`**:
    * **Evolución:** Es simplemente el argumento que se le pasa a la función (por ejemplo, el sistema le pasa `ID_LED_A` que equivale a `0`). No evoluciona, sirve únicamente como índice para saber a qué posición del arreglo `task_actuator_dta_list[]` se le debe escribir el mensaje.
* **`task_actuator_dta_list[identifier].event`**:
    * **Evolución:** A diferencia de la variable del inciso 2 (que se lee), aquí es donde **se escribe**. Cuando se llama a esta función (ej. desde el `task_system.c`), el evento actual es pisado instantáneamente por la nueva orden (ej. de `EV_LED_IDLE` a `EV_LED_ACTIVE`).
* **`task_actuator_dta_list[identifier].flag`**:
    * **Evolución:** Al mismo tiempo que se sobrescribe el evento, esta bandera se fuerza a `true`. Esto es el equivalente a levantar la bandera de un buzón postal. 

**Resumen del ciclo:**
Al arrancar, `flag` es `false`. De repente, el Sistema llama a la interfaz y pone `event = EV_LED_ACTIVE` y `flag = true`. Milisegundos después, el `while(1)` principal ejecuta `task_actuator_update()`, la máquina de estados ve la bandera levantada, obedece el comando, enciende el LED y vuelve a poner el `flag = false` a la espera de la siguiente orden.
