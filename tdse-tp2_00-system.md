Con base en los archivos proporcionados, el sistema implementa la "Tarea del Sistema" (el cerebro intermedio o controlador) y la interfaz de comunicación hacia la "Tarea del Actuador" (quien controla un LED, motor, etc.). 

A continuación, presento el análisis detallado del funcionamiento y la evolución de las variables.

### 1. Análisis y Funcionamiento General
* **`task_system_attribute.h` / `task_actuator_attribute.h`**: Definen los estados, eventos y estructuras de datos para el sistema y el actuador. El sistema maneja `EV_SYS_IDLE` / `EV_SYS_ACTIVE`, y el actuador maneja `EV_LED_IDLE` / `EV_LED_ACTIVE`.
* **`task_system_interface.c`**: Es la entrada de datos al Sistema. Implementa una **cola de eventos (FIFO)** donde otras tareas (como la del sensor) dejan mensajes (`put_event`).
* **`task_system.c`**: Es el cerebro. Periódicamente revisa la cola de eventos, extrae los mensajes y evalúa su Máquina de Estados. Dependiendo del evento y de su estado actual, decide enviar una orden al actuador.
* **`task_actuator_interface.c`**: Es la salida de datos hacia el Actuador. En lugar de usar una cola compleja, utiliza un método de acceso directo (buzón o variable compartida) donde el Sistema le sobrescribe el evento al actuador y levanta una bandera (`flag = true`) para avisarle.

---

### 2. Evolución de variables del Sistema (`task_system.c`)

* **`index` / `NORMAL`**:
  En este código específico, la tarea del sistema no usa un bucle `for` tradicional con un `index`. En su lugar, usa `g_task_system_mode` (cuyo valor suele ser `NORMAL`, equivalente a un índice `0`). 
* **`task_system_dta_list[index].tick`**:
  * **Unidad:** Milisegundos (mS) (derivado del SysTick global).
  * **Evolución:** En este bloque de lógica orientada a eventos puros (Event-Triggered), la variable `tick` no tiene un rol activo ni evoluciona dentro del `task_system_normal_statechart()`. Se inicializa en `0` en el `init` y no se modifica, ya que las transiciones dependen de eventos y no de retardos (timeouts).
* **`task_system_dta_list[index].event`**:
  * **Evolución:** Arranca con un valor por defecto. Durante el `update`, si `any_event_task_system()` indica que hay un mensaje en la cola, esta variable se sobrescribe con el valor devuelto por `get_event_task_system()` (ej. `EV_SYS_ACTIVE` o `EV_SYS_IDLE`).
* **`task_system_dta_list[index].flag`**:
  * **Evolución:** Arranca en `false`. Cuando se extrae un evento nuevo de la cola, pasa a `true`. Luego, cuando la máquina de estados consume ese evento para hacer una transición de estado, la bandera vuelve a `false` (indicando que el evento ya fue procesado y no debe procesarse dos veces).
* **`task_system_dta_list[index].state`**:
  * **Evolución:** Comienza en `ST_SYS_IDLE`. Si llega el evento de activación y la bandera está en `true`, pasa a `ST_SYS_ACTIVE`. Al llegar el evento de reposo, vuelve a `ST_SYS_IDLE`.

---

### 3. Comportamiento de `task_system_normal_statechart()`
Esta función contiene la máquina de estados del sistema:
1. **Lectura de la cola:** Primero, consulta si hay eventos pendientes (`any_event_task_system()`). Si los hay, lee el evento, lo guarda y pone el `flag` en `true`.
2. **Evaluación (Switch-Case):**
   * **Caso `ST_SYS_IDLE` (Reposo):** Si hay un evento sin procesar (`flag == true`) y resulta ser `EV_SYS_ACTIVE` (ej. apretaron el botón), hace tres cosas: baja la bandera (`flag = false`), llama a `put_event_task_actuator()` para ordenarle al LED que se encienda (`EV_LED_ACTIVE`), y cambia su propio estado a `ST_SYS_ACTIVE`.
   * **Caso `ST_SYS_ACTIVE` (Activo):** Si `flag == true` y el evento es `EV_SYS_IDLE` (ej. soltaron el botón), baja la bandera (`flag = false`), ordena al actuador apagarse (`EV_LED_IDLE`), y regresa su estado a `ST_SYS_IDLE`.

---

### 4. Evolución de la Cola de Eventos (`event_task_system_queue`)

Estas variables en `task_system_interface.c` manejan la recepción de mensajes:
* **`i` (Variable local):** Solo se usa durante `init_event_task_system()` para recorrer las posiciones `0` hasta `QUEUE_LENGTH - 1` y limpiar el arreglo inicializando todo en `EMPTY`. Luego deja de existir/usarse.
* **`count` (Cantidad):** Arranca en `0`. Aumenta (`+1`) cada vez que otra tarea inserta un mensaje (`put_event`). Disminuye (`-1`) cada vez que el sistema consume un mensaje (`get_event`).
* **`head` (Cabeza/Escritura):** Arranca en `0`. Es modificada por la tarea que *envía* (el sensor). Avanza de a uno cada vez que entra un mensaje. Si llega al final del arreglo (`QUEUE_LENGTH`), vuelve a `0` (comportamiento circular).
* **`tail` (Cola/Lectura):** Arranca en `0`. Es modificada por la *recepción* (el sistema). Avanza de a uno cada vez que `get_event_task_system()` extrae un mensaje para dárselo al statechart. Si llega al final, también vuelve a `0`.
* **`queue[i]` (Arreglo):** Comienza con todos sus elementos en `EMPTY`. Sus posiciones se van sobrescribiendo con eventos (`EV_SYS_ACTIVE` o `EV_SYS_IDLE`) en el índice apuntado por `head`, y son leídas desde el índice apuntado por `tail`.

---

### 5. Evolución de variables del Actuador (`task_actuator_interface.c`)

Cuando el sistema llama a `put_event_task_actuator(event, identifier)`, se altera directamente la memoria del actuador:
* **`identifier`:** Es el índice que indica qué actuador vamos a modificar. Por ejemplo, al pasarle `ID_LED_A` (que vale `0`), el código buscará la posición `task_actuator_dta_list[0]`.
* **`task_actuator_dta_list[identifier].event`:** Se sobrescribe con la nueva orden que mandó el sistema (ej. `EV_LED_ACTIVE`).
* **`task_actuator_dta_list[identifier].flag`:** Se fuerza inmediatamente a `true`. 

**Diferencia clave:** A diferencia de la cola compleja del Sistema, el Actuador usa un "Buzón de 1 solo mensaje". Al poner `flag = true`, el sistema le está gritando al actuador: *"¡Tienes una nueva orden!"*. En la próxima pasada del bucle principal donde se llame al *update* del actuador, este verá la bandera levantada, leerá el evento, ejecutará la acción (prender el LED) y bajará su propia bandera a `false`.
