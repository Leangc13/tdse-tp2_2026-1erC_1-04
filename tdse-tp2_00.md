¡Hola! Veo que estás empezando a trabajar con la guía del **Trabajo Práctico 2 (TA134)** sobre la implementación de **Máquinas de Estado (FSM)** en lenguaje C para sistemas embebidos. Este es uno de los conceptos más importantes para lograr que un microcontrolador ejecute múltiples tareas de forma ordenada y no bloqueante.

Para ir calentando motores, aquí te dejo un resumen rápido de la estructura clásica que solemos usar en C para codificar diagramas de estado:

### Estructura Básica: El método `switch-case`

La forma más común, legible y directa de traducir un diagrama de estados a código C (especialmente en la etapa universitaria) es utilizando un tipo enumerado (`enum`) para definir los estados y una estructura `switch-case` para manejar la lógica y las transiciones.

**1. Definición de Estados:**
Se utiliza un `enum` para listar todos los estados posibles del sistema. Esto hace que el código sea mucho más fácil de leer.

```c
typedef enum {
    ESTADO_REPOSO,
    ESTADO_ENCENDIDO,
    ESTADO_ALARMA,
    // ... otros estados
} Estado_t;

// Variable global o estática que guarda el estado actual
Estado_t estado_actual = ESTADO_REPOSO; 
```

**2. Función de Actualización (La Máquina de Estados):**
Esta función se llama periódicamente (por ejemplo, dentro del `while(1)` principal del `main` o en la interrupción de un Timer). Revisa en qué estado estamos, ejecuta las acciones de ese estado (lógica de tipo Moore) y evalúa si hay que saltar a otro estado (lógica de tipo Mealy o evaluación de eventos).

```c
void actualizar_fsm(void) {
    switch (estado_actual) {
        
        case ESTADO_REPOSO:
            // 1. Acciones del estado
            apagar_leds();
            
            // 2. Evaluación de transiciones (Eventos)
            if (boton_presionado() == true) {
                estado_actual = ESTADO_ENCENDIDO; // Transición
            }
            break;

        case ESTADO_ENCENDIDO:
            // 1. Acciones del estado
            encender_led_verde();
            
            // 2. Evaluación de transiciones
            if (temperatura_critica() == true) {
                estado_actual = ESTADO_ALARMA;
            } else if (boton_presionado() == true) {
                estado_actual = ESTADO_REPOSO;
            }
            break;

        case ESTADO_ALARMA:
            // 1. Acciones del estado
            titilar_led_rojo();
            activar_buzzer();
            
            // 2. Evaluación de transiciones
            if (boton_reset_presionado() == true) {
                estado_actual = ESTADO_REPOSO;
            }
            break;
            
        default:
            // Manejo de errores: si la máquina cae en un estado no válido
            estado_actual = ESTADO_REPOSO;
            break;
    }
}
```

---

¿De qué trata el primer ejercicio de tu guía o qué diagrama de estados específico necesitas traducir a código? Puedes pegarme el enunciado o describirme el diagrama y lo vamos armando.
