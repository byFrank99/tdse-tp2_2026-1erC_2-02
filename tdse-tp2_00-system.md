from weasyprint import HTML

# Contenido Markdown para el archivo
markdown_content = """# Análisis Técnico: Módulo de Actuador (TDSE TP2)

Este documento detalla el funcionamiento técnico del módulo del actuador, basado en los archivos `task_actuator_attribute.h`, `task_actuator.c` y `task_actuator_interface.c`.

## 1. Funcionamiento General del Módulo

El módulo implementa una **Tarea de Actuador** bajo un modelo de **Máquina de Estados Finita (FSM)**. Su propósito es controlar un periférico (en este caso, un LED) respondiendo a eventos externos enviados generalmente por una tarea de "Sistema".

* **task_actuator_attribute.h**: Define las estructuras de configuración (`cfg`) y de datos dinámicos (`dta`). El actuador tiene dos estados principales: inactivo (`IDLE`) y activo (`ACTIVE`).
* **task_actuator_interface.c**: Provee el mecanismo de comunicación. La función `put_event_task_actuator` permite que otras tareas "escriban" un evento y activen un aviso (*flag*) para que el actuador reaccione.
* **task_actuator.c**: Contiene la lógica de ejecución. En cada ciclo, verifica si hay eventos pendientes o si el tiempo de actividad (*tick*) ha expirado para apagar el actuador.

## 2. Evolución de Variables
### index, tick, state, event y flag

Asumiendo que el sistema arranca y recibe la orden de activar el LED (`ID_LED_A`), la evolución es la siguiente:

| Momento / Ejecución | index | tick [ms] | state | event | flag | Acción de Hardware |
| :--- | :---: | :---: | :--- | :--- | :---: | :--- |
| **Inicio (init)** | 0 | 0 | ST_LED_IDLE | EV_LED_IDLE | false | LED OFF |
| **Update (espera)** | 0 | 0 | ST_LED_IDLE | EV_LED_IDLE | false | - |
| **Llega Evento** | - | 0 | ST_LED_IDLE | EV_LED_ACTIVE | true | (Desde Interfaz) |
| **Update N** | 0 | 0 | ST_LED_ACTIVE | EV_LED_ACTIVE | false | **LED ON** |
| **Update N+1** | 0 | 1 | ST_LED_ACTIVE | EV_LED_ACTIVE | false | - |
| **Update N+X** | 0 | X | ST_LED_ACTIVE | EV_LED_ACTIVE | false | - |
| **Expiración (tick≥max)** | 0 | 0 | ST_LED_IDLE | EV_LED_ACTIVE | false | **LED OFF** |

> **Nota sobre la unidad de medida:** El `tick` se mide en **milisegundos (ms)**. Esto se debe a que la función `task_actuator_update` es llamada por el planificador de `app.c` sincronizada con el SysTick (configurado a 1ms).

## 3. Comportamiento de `void task_actuator_statechart(uint32_t index)`

Esta función es el motor lógico del actuador. En cada llamada, realiza lo siguiente:

1.  **Sincronización:** Obtiene los punteros a la configuración y a los datos dinámicos del actuador correspondiente al `index` (ej. `ID_LED_A`).
2.  **Evaluación de Estado:**
    * **En ST_LED_IDLE:** Vigila el `flag`. Si es `true` y el evento es `EV_LED_ACTIVE`, inicia la activación. Limpia el `flag`, pone el pin GPIO en estado `led_on`, cambia el estado a `ACTIVE` y resetea el `tick` a 0.
    * **En ST_LED_ACTIVE:** Incrementa el contador `tick` en cada ciclo. Compara este valor con `tick_max` (tiempo de encendido configurado). Al cumplirse el tiempo, apaga el pin (`led_off`) y regresa al estado `IDLE`.

## 4. Evolución de la Comunicación (Interfaz)

Las variables `identifier`, `event` y `flag` dentro de `task_actuator_interface.c` evolucionan cuando una tarea externa (como `task_system`) interactúa con el actuador.

### Secuencia de eventos:

1.  **Llamada a `put_event_task_actuator(EV_LED_ACTIVE, ID_LED_A)`:**
    * `identifier`: Toma el valor `ID_LED_A` (índice 0).
    * `task_actuator_dta_list[0].event`: Se actualiza inmediatamente a `EV_LED_ACTIVE`.
    * `task_actuator_dta_list[0].flag`: Se establece en `true`. Esto actúa como una "interrupción lógica" para que la FSM detecte la nueva petición.

2.  **Consumo en `task_actuator_update`:**
    * En la siguiente ejecución del loop principal, `task_actuator_statechart(0)` detecta el `flag == true`.
    * Ejecuta la acción (encender LED) y restablece el `flag` a `false`.
    * *Nota:* Si se recibe un evento mientras el LED ya está activo, el flag vuelve a `true`, pero la lógica actual de `ST_LED_ACTIVE` no lo re-evalúa hasta terminar el ciclo actual (a menos que se implemente re-disparo).

---

## Resumen de Variables Clave

* **index / identifier:** Identificador único que permite manejar múltiples actuadores con la misma lógica.
* **flag:** Semáforo lógico que indica "mensaje nuevo pendiente".
* **tick_max:** Define la duración del pulso de activación.
"""

with open("tdse-tp2_00-system.md", "w", encoding="utf-8") as f:
    f.write(markdown_content)
