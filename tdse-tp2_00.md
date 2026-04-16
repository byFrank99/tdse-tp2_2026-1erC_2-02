En ingeniería, la implementación de una Máquina de Estados Finitos (FSM) se define teóricamente por la tupla (S,Σ,δ,s0​,F). Al trasladar esto a lenguaje C, el conjunto S representará los estados, Σ los eventos, y la función δ(Sactual​,Σevento​) será la lógica de transición. Basado en los patrones de diseño de software embebido (como los documentados por Miro Samek para arquitecturas reactivas), existen dos enfoques principales y rigurosos para codificar esto.
Metodologías de Codificación en C
1. Método de Switch-Case Anidados

Este es el diseño clásico y secuencial, ideal para diagramas de complejidad baja o moderada. Todo el ruteo se resuelve en tiempo de ejecución mediante saltos condicionales.

    Definición de Dominio: Se utilizan enumeraciones (enum) tanto para el conjunto de estados S como para el conjunto de eventos Σ. Esto elimina los "números mágicos" y asegura tipado fuerte.

    Variable de Estado: Se inicializa una variable Sactual​ con el valor del estado inicial s0​.

    Estructura Principal: Se implementa una función o tarea que contiene un switch primario que evalúa Sactual​.

    Lógica de Transición: Dentro de cada case correspondiente a un estado, se evalúa el evento entrante (con otro switch o sentencias if). Si el evento dispara una transición válida, se ejecutan las acciones (de salida, transición y entrada) y se actualiza Sactual​=Ssiguiente​.

2. Método de Punteros a Funciones (Tabla de Transiciones)

Para sistemas complejos, los anidamientos múltiples son difíciles de mantener. Este método mapea la función de transición δ de forma matricial, logrando un tiempo de despacho de eventos constante de O(1).

    Definición de Dominio: Se declaran los mismos enum para estados y eventos.

    Funciones de Acción: Se encapsula el comportamiento de cada estado (o de cada transición) en funciones individuales e independientes con la misma firma, por ejemplo: void Estado_Idle(void).

    Matriz de Despacho: Se construye un arreglo bidimensional estático de punteros a funciones de tamaño S×Σ.

    Motor de FSM: La lógica central se reduce a indexar la matriz. El sistema toma la variable Sactual​ y el índice del evento Σevento​, y ejecuta la función apuntada en esa coordenada: Tabla_FSM[S_{actual}][\Sigma_{evento}]().

Suposiciones iniciales: Asumo que el diagrama ya está modelado teóricamente, que ya está definido si el sistema obedece a una arquitectura de Mealy (las salidas dependen de Sactual​ y Σevento​) o de Moore (las salidas solo dependen de Sactual​), y que el código correrá en un entorno de ejecución estándar o sobre un microcontrolador (sin un RTOS de por medio todavía).

Para no inventar la estructura de tu sistema ni el código, necesito los parámetros exactos de tu diseño.
