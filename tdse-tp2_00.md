Codificar diagramas de estado (también conocidos como Máquinas de Estados Finitos o FSM, por sus siglas en inglés) en C es un tema excelente para un trabajo práctico. Es un patrón de diseño fundamental, especialmente en sistemas embebidos, videojuegos y control de hardware.

Aquí tienes una estructura completa que puedes usar como base para tu Trabajo Práctico, enfocada en un ejemplo clásico y fácil de entender: Un Semáforo.

# Trabajo Práctico: Codificación de Diagramas de Estado en C

## 1. Objetivos del Trabajo
* Comprender el concepto de Máquina de Estados Finitos (FSM).
* Aprender a traducir un diagrama de estados visual a código estructurado en C.
* Utilizar estructuras de control fundamentales (`enum`, `switch-case`) para el manejo de estados y transiciones.

## 2. Marco Teórico
Un diagrama de estados describe el comportamiento de un sistema que puede estar en uno de varios estados en un momento dado. El sistema cambia de un estado a otro a través de transiciones, las cuales son desencadenadas por eventos o condiciones específicas (como el paso del tiempo o la pulsación de un botón).

Para este trabajo, modelaremos un sistema con tres estados básicos:
* **VERDE:** Los vehículos pueden avanzar.
* **AMARILLO:** Precaución, el semáforo está por cambiar.
* **ROJO:** Los vehículos deben detenerse.

## 3. Diseño del Sistema (Máquina de Estados)
La transición entre estos estados dependerá de un temporizador (simulado en el código).
* **Estado inicial:** VERDE.
* De VERDE transiciona a AMARILLO tras N segundos.
* De AMARILLO transiciona a ROJO tras M segundos.
* De ROJO transiciona nuevamente a VERDE tras X segundos.

## 4. Implementación en C
La forma más común y eficiente de implementar esto en C es utilizando un tipo enumerado (`enum`) para definir los estados y una estructura `switch-case` dentro de un bucle para manejar las transiciones.

```c
#include <stdio.h>
#include <unistd.h> // Para la función sleep()

// 1. Definición de los estados usando enum
typedef enum {
    ESTADO_VERDE,
    ESTADO_AMARILLO,
    ESTADO_ROJO
} EstadoSemaforo;

int main() {
    // 2. Inicialización del estado
    EstadoSemaforo estadoActual = ESTADO_VERDE;
    
    printf("Iniciando simulación del semáforo...\n\n");

    // Bucle infinito de la máquina de estados
    for(int i = 0; i < 6; i++) { // Limitado a 6 ciclos para el ejemplo
        
        // 3. Lógica de estados y transiciones (Switch-Case)
        switch (estadoActual) {
            
            case ESTADO_VERDE:
                printf("[VERDE]    Los autos avanzan. (Esperando 3 segundos...)\n");
                sleep(3); // Simulamos el tiempo que pasa
                estadoActual = ESTADO_AMARILLO; // Transición al siguiente estado
                break;
                
            case ESTADO_AMARILLO:
                printf("[AMARILLO] Precaución.        (Esperando 1 segundo...)\n");
                sleep(1);
                estadoActual = ESTADO_ROJO; // Transición al siguiente estado
                break;
                
            case ESTADO_ROJO:
                printf("[ROJO]     Autos detenidos.   (Esperando 3 segundos...)\n");
                sleep(3);
                estadoActual = ESTADO_VERDE; // Transición al estado inicial
                break;
                
            default:
                printf("Error: Estado desconocido.\n");
                estadoActual = ESTADO_VERDE; // Recuperación de error
                break;
        }
    }
    
    printf("\nSimulación finalizada.\n");
    return 0;
}

```   

## 5. Consignas Propuestas para el Alumno
Para que el trabajo práctico sea interactivo, puedes pedir a los alumnos (o a ti mismo) que amplíen este código base con los siguientes desafíos:
* Agregar un botón de peatón: Incorporar un evento externo (entrada por consola usando kbhit() o leyendo un caracter) que, si se presiona mientras está en VERDE, acelere el paso a AMARILLO.
* Modo Nocturno: Agregar un estado extra (ESTADO_INTERMITENTE) donde el semáforo solo titile en amarillo indefinidamente, activado por un comando específico.
* Múltiples Semáforos: Modularizar el código utilizando struct para poder manejar dos semáforos de una intersección al mismo tiempo.
