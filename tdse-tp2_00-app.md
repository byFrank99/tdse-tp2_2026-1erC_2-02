Analizar el código fuente de un sistema embebido "bare-metal" es fundamental para comprender cómo se gestionan los recursos y los tiempos de ejecución a bajo nivel. A continuación, te explico en detalle el funcionamiento del sistema y la evolución de sus variables clave, basándome en los archivos proporcionados.

### Funcionamiento General del Sistema

El código implementa un **Sistema Disparado por Eventos (ETS - Event-Triggered System)** de tipo *bare-metal* (sin sistema operativo). Su funcionamiento se basa en un bucle principal que espera "eventos de tiempo" para ejecutar una lista de tareas predefinidas y medir sus tiempos de ejecución.

1.  **Inicialización (`app.c`, `systick.c`, `dwt.h`):** La función `app_init()` configura el sistema. Imprime información de inicio a través del *logger*, inicializa contadores de la aplicación, activa el contador de ciclos del procesador (basado en la unidad DWT de ARM Cortex) y ejecuta la función de inicialización de cada tarea (Sensor, Sistema, Actuador).
2.  **Generación de Eventos (`systick.c`, `app.c`):** El sistema utiliza el temporizador del sistema (SysTick) para generar interrupciones periódicas. Cada vez que ocurre esta interrupción, se ejecuta `HAL_SYSTICK_Callback()`, la cual incrementa un contador de "tics" (`g_app_tick_cnt`).
3.  **Ejecución y Monitoreo (`app.c`):** La función `app_update()` se ejecuta continuamente en el bucle principal. Cuando detecta que el contador de tics es mayor a 0, consume un tic y ejecuta secuencialmente todas las tareas (`task_update`). 
    * Antes de cada tarea, reinicia el contador de ciclos del DWT.
    * Al terminar, calcula cuánto tiempo (en microsegundos) tomó ejecutar esa tarea para mantener estadísticas de rendimiento (Mejor y Peor caso).
    * Al finalizar todas las tareas, pone al procesador en modo de bajo consumo (`WFI` - Wait For Interrupt) hasta el próximo evento.
4.  **Sistema de Registro (`logger.c`, `logger.h`):** Permite imprimir mensajes formateados. Protege la escritura desactivando temporalmente las interrupciones y utiliza *semihosting* (imprimir a través del depurador usando la consola del entorno de desarrollo) si está configurado.

---

### Evolución de las Variables

A continuación, se detalla la evolución y unidad de medida de cada variable solicitada desde la ejecución de `app_init()` y durante sucesivas iteraciones de `app_update()`.

* **`g_app_tick_cnt`**
    * **Unidad:** Tics (eventos de tiempo, típicamente milisegundos dependiendo del reloj del sistema).
    * **Evolución:** Se inicializa en `0` al final de `app_init()`. Se incrementa en `1` de forma asíncrona dentro de la interrupción `HAL_SYSTICK_Callback()`. En `app_update()`, si es mayor a `0`, se decrementa en `1` para indicar que el evento de tiempo va a ser procesado.
* **`g_app_runtime_us`**
    * **Unidad:** Microsegundos ($\mu s$).
    * **Evolución:** Comienza sin inicialización explícita, pero dentro de `app_update()` se reinicia a `0` al inicio de cada ciclo de ejecución de tareas. Luego, acumula el tiempo de ejecución de cada tarea sumando consecutivamente el valor de `task_dta_list[index].LET`. Representa el tiempo total que tomó ejecutar todas las tareas en ese tic.
* **`index`**
    * **Unidad:** Adimensional (índice de iteración).
    * **Evolución:** Toma valores enteros desde `0` hasta `TASK_QTY - 1` (en este caso, 0, 1 y 2 para las tres tareas). Se reinicia a `0` al comienzo de los bucles `for` tanto en `app_init()` como en `app_update()`.
* **`task_dta_list[index].NOE` (Number of Executions)**
    * **Unidad:** Adimensional (cantidad).
    * **Evolución:** Se inicializa en `0` en `app_init()`. En `app_update()`, se incrementa en `1` cada vez que se ejecuta la tarea correspondiente.
* **`task_dta_list[index].LET` (Last Execution Time)**
    * **Unidad:** Microsegundos ($\mu s$).
    * **Evolución:** Se inicializa en `0` en `app_init()`. En `app_update()`, justo después de ejecutar la tarea, se actualiza llamando a `cycle_counter_get_time_us()`, sobreescribiendo su valor con el tiempo que tardó la última ejecución.
* **`task_dta_list[index].BCET` (Best-Case Execution Time)**
    * **Unidad:** Microsegundos ($\mu s$).
    * **Evolución:** Se inicializa artificialmente alto, en `1000`, durante `app_init()`. En `app_update()`, evalúa si el `LET` actual es *menor* que el `BCET` guardado; si es así, el `BCET` adopta el valor del `LET`. Representa el tiempo mínimo histórico registrado para la tarea.
* **`task_dta_list[index].WCET` (Worst-Case Execution Time)**
    * **Unidad:** Microsegundos ($\mu s$).
    * **Evolución:** Se inicializa en `0` durante `app_init()`. En `app_update()`, evalúa si el `LET` actual es *mayor* que el `WCET` guardado; si es así, el `WCET` adopta el valor del `LET`. Representa el tiempo máximo histórico registrado para la tarea.

---

### Impacto de usar `LOGGER_INFO()` en los Tiempos de Ejecución

El uso de la macro `LOGGER_INFO()` dentro de una tarea tendrá un **impacto negativo severo y perjudicial** sobre las variables de medición de tiempo.

1.  **Mecanismo de Bloqueo y *Semihosting*:** `LOGGER_INFO` ejecuta internamente un bloque de código que deshabilita las interrupciones (`__asm("CPSID i")`), llama a `snprintf` para formatear una cadena de caracteres, y luego llama a `logger_log_print_` que utiliza `printf` (basado en *semihosting*). El *semihosting* detiene el procesador para comunicarse con el depurador de hardware y enviar los datos a la consola del PC.
2.  **Impacto en `task_dta_list[index].WCET`:** Esta operación de entrada/salida es extremadamente lenta en ciclos de reloj (puede tardar milisegundos). Como el contador DWT sigue contando o evalúa la diferencia de tiempo, el `LET` de esa iteración se disparará. Dado que la lógica del código actualiza el `WCET` cuando el `LET` es mayor, el `WCET` absorberá este enorme pico de tiempo. **Resultado:** El peor caso quedará inflado permanentemente, perdiendo su utilidad como métrica del rendimiento de la lógica real de la tarea.
3.  **Impacto en `g_app_runtime_us`:** Dado que esta variable es la suma de los `LET` de todas las tareas en una iteración específica, el valor de `g_app_runtime_us` experimentará un pico masivo durante el ciclo de ejecución en el que se llamó al *logger*, afectando temporalmente la medición de la carga del procesador.
