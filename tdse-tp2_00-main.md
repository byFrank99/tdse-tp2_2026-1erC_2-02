Aquí tienes un análisis detallado del funcionamiento del código fuente proporcionado para el microcontrolador STM32, así como la evolución cronológica de las variables y componentes relacionados con el reloj del sistema.

# Análisis de Archivos Fuente y Evolución de Relojes

## Análisis de los Archivos Fuente

### 1. `startup_stm32f103rbtx.s` (Archivo de Arranque)
Este archivo está escrito en lenguaje ensamblador y es lo primero que se ejecuta cuando el microcontrolador recibe energía o es reiniciado.

* **Tabla de Vectores (`g_pfnVectors`):** Define las direcciones de memoria de todas las rutinas de servicio de interrupción (ISRs) y excepciones del sistema (como `Reset_Handler`, `SysTick_Handler`, etc.).
* **`Reset_Handler`:** Es el punto de entrada principal tras un reinicio.
  * Llama a la función `SystemInit` (típicamente definida en el archivo `system_stm32f1xx.c` provisto por CMSIS) para realizar una configuración de reloj básica a muy bajo nivel.
  * Copia los valores iniciales de las variables globales/estáticas desde la memoria Flash a la memoria RAM (inicializando la sección `.data`).
  * Llena con ceros la memoria asignada a las variables globales no inicializadas (sección `.bss`).
  * Llama a la inicialización de la librería estándar de C (`__libc_init_array`) y finalmente salta a la función `main()` de tu código C.

### 2. `main.c` (Programa Principal)
Contiene la lógica de inicialización general de alto nivel y el bucle infinito de la aplicación.

* **Inicialización:** Comienza llamando a `HAL_Init()` para resetear periféricos y configurar la base de tiempo (`SysTick`).
* **Configuración del Reloj (`SystemClock_Config`):** Configura los osciladores y multiplicadores (PLL) para que el núcleo funcione a una velocidad específica (**64 MHz** en este caso).
* **Periféricos:** Llama a `MX_GPIO_Init()` y `MX_USART2_UART_Init()` para configurar los pines de entrada/salida (como el LED en `LD2_Pin` y el botón en `B1_Pin`) y la comunicación serial.
* **Aplicación:** Llama a `app_init()` para preparar la lógica de usuario y luego entra en un bucle infinito `while (1)` donde se ejecuta cíclicamente `app_update()`.

### 3. `stm32f1xx_it.c` (Rutinas de Interrupción)
Contiene las funciones que el hardware invoca automáticamente cuando ocurre un evento específico (interrupciones o fallos de hardware).

* **Excepciones de Sistema:** Contiene manejadores (por ahora como bucles infinitos) para errores graves como `HardFault_Handler` o `MemManage_Handler`.
* **`SysTick_Handler(void)`:** Esta es una de las funciones más críticas. Se dispara periódicamente (generalmente cada 1 ms) y llama a `HAL_IncTick()`, lo que incrementa un contador global utilizado por la capa HAL para llevar la cuenta del tiempo (usado en funciones como `HAL_Delay`).
* **`EXTI15_10_IRQHandler(void)`:** Atiende las interrupciones externas de los pines 10 al 15. En este caso, reenvía el control a la capa HAL (`HAL_GPIO_EXTI_IRQHandler`) para procesar el evento del botón conectado al pin `B1_Pin`.

---

## Evolución de `SysTick` y `SystemCoreClock`

`SystemCoreClock` es una variable global (definida típicamente en CMSIS) que almacena la frecuencia actual del núcleo en Hercios (Hz). El `SysTick` es un temporizador de hardware de 24 bits integrado en el núcleo Cortex-M3 que genera interrupciones periódicas.

Aquí está la evolución cronológica desde el reinicio hasta el bucle principal:

### 1. Arranque en `Reset_Handler` (`startup_stm32f103rbtx.s`)
* El microcontrolador arranca usando su oscilador interno por defecto (HSI), que en la familia STM32F1 funciona a **8 MHz**.
* Se ejecuta la función externa `SystemInit`. En este momento, la variable `SystemCoreClock` se inicializa a su valor por defecto de **8,000,000 Hz** (8 MHz).
* El temporizador `SysTick` se encuentra apagado o inactivo durante esta fase temprana de ensamblador.

### 2. Entrada a `main()` y llamada a `HAL_Init()`
* La ejecución llega a C. Se llama a `HAL_Init()`.
* `HAL_Init()` configura el `SysTick` para que genere una interrupción cada **1 milisegundo** basándose en el valor actual de `SystemCoreClock` (8 MHz).
* A partir de este instante, el temporizador empieza a contar y dispara la interrupción `SysTick_Handler` (en `stm32f1xx_it.c`) por primera vez de forma regular.

### 3. Llamada a `SystemClock_Config()` en `main.c`
* Esta función reprograma completamente la velocidad del sistema. Configura el oscilador interno HSI como base, lo divide a la mitad (4 MHz) y lo pasa por el bucle de bloqueo de fase (PLL) con un multiplicador de x16 (`RCC_PLL_MUL16`). 
* **4 MHz * 16 = 64 MHz**.
* Al llamar internamente a `HAL_RCC_ClockConfig()`, la capa HAL actualiza de manera automática la variable global `SystemCoreClock` a **64,000,000 Hz**.
* Inmediatamente después de aplicar esta nueva velocidad, la función de la HAL recalcula y actualiza el registro de recarga del `SysTick`. Como el reloj va mucho más rápido ahora, el temporizador debe contar un número mayor de ciclos antes de dispararse para asegurar que el tiempo entre interrupciones se mantenga exactamente en 1 milisegundo.

### 4. Bucle principal `while(1)`
* `SystemCoreClock` se estabiliza en **64 MHz** y ya no sufre más modificaciones.
* El `SysTick` sigue funcionando continuamente en segundo plano. Mientras la aplicación entra y sale de `app_update()`, la CPU es interrumpida brevemente cada 1 ms para saltar a `SysTick_Handler()`, incrementar la base de tiempo de la librería HAL y retornar al código principal, permitiendo la gestión precisa del tiempo durante la ejecución de la aplicación.
