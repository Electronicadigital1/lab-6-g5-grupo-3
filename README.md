[![Open in Visual Studio Code](https://classroom.github.com/assets/open-in-vscode-2e0aaae1b6195c2367325f4f02e2d04e9abb55f0b24a779b69b11b9e10269abc.svg)](https://classroom.github.com/online_ide?assignment_repo_id=24057923&assignment_repo_type=AssignmentRepo)
# Lab04 - Visualización usando pantalla LCD 16x2

# Integrantes
**Lady Lorena Cardenas Poveda**  
Carrera: Ingeniería Eléctrica  
Correo: llcardenasp@unal.edu.co  

**Samuel Tovar Vásquez**  
Carrera: Ingeniería Electrónica  
Correo: satovarv@unal.edu.co  


# Informe

Indice:

1. [Diseño implementado](#diseño-implementado)
2. [Simulaciones](#simulaciones)
3. [Implementación](#implementación)
4. [Conclusiones](#conclusiones)
5. [Referencias](#referencias)

## Diseño implementado:

El sistema se armó de forma modular en Verilog para que el flujo de datos fuera limpio: Entrada de datos (Teclado), luego el procesamiento (Validar la clave) y lograr la acción (Mover el motor).  Para que todo funcionara bien en la práctica y evitar problemas con los rebotes de los botones o los cambios de velocidad en los relojes, dividimos el código en 5 bloques principales conectados dentro de un módulo central (top_sistema.v):

### Diagramas

 <img width="1200" height="1600" alt="image" src="https://github.com/Electronicadigital1/lab-6-g5-grupo-3/blob/main/bloques.jpeg" />
  
### Descripción de los Módulos

1. **Divisor de Frecuencia (`divisor_frecuencia.v`):** Reduce el reloj principal de la FPGA (50 MHz) a una señal mucho más lenta de 1 kHz para el escaneo del teclado. Esto evita que el circuito lea las pulsaciones tan rápido que confunda el ruido eléctrico con teclas presionadas.
2. **Escáner de Teclado (`escaner_teclado.v`):** Revisa las filas y columnas de forma secuencial para detectar qué tecla fue presionada. Implementa un contador de filtrado (*debounce*) para asegurar que la pulsación sea real y no un falso contacto.
3. **Control de Clave (`control_clave.v`):** Es el bloque de control principal. Almacena los dígitos introducidos en un registro que desplaza los datos a la izquierda, cuenta que se ingresen 4 dígitos y los compara con la contraseña secreta parametrizada (`16'h1234`).
4. **Módulo PWM (`pwm_servo.v`):** Genera la señal de control para el servomotor con un periodo de 20 ms. Si la puerta está configurada como abierta, envía un pulso de 2 ms; si está cerrada, envía un pulso de 1 ms.
5. **Top del Sistema (`top_sistema.v`):** Módulo de más alto nivel que se encarga de interconectar todos los bloques anteriores y asignar las señales a los pines físicos de la placa de desarrollo (LEDs, servomotor y líneas del teclado).

---

## 2. Implementación y Ajustes Críticos

Al realizar las pruebas en el hardware real, se identificaron varios desfases entre el comportamiento ideal del simulador y los componentes físicos, los cuales se solucionaron mediante las siguientes modificaciones en el código:

### A. Ajuste del Reloj y Tiempo de Rebote del Teclado
El teclado matricial físico demostró ser sumamente sensible. Inicialmente, con el tiempo estándar de 10 ms, una sola pulsación suave registraba el mismo dígito múltiples veces consecutivas debido a las vibraciones mecánicas internas de los contactos.
* **Solución:** Se incrementó el parámetro `DEBOUNCE_TIME` a **60 ms**. Como el contador ahora requería alcanzar un valor mayor, se expandió el registro `contador_debounce` a **8 bits** en `escaner_teclado.v` para evitar desbordamientos numéricos. Con este cambio, el teclado lee exactamente un dígito por cada pulsación real.

### B. Uso de Máquinas de Estados Finitos (FSM)
Tanto para el escaneo de pines como para la lógica de validación de la contraseña, se utilizaron estructuras de **máquinas de estados finitos**. Por ejemplo, el control de la contraseña transita de forma ordenada por los estados `IDLE` (espera), `INPUT` (captura de dígitos), `VERIFY` (comparación de contraseñas), y finalmente bifurca hacia `OPEN` (apertura) o `ERROR` (bloqueo).
* **Utilidad del diseño con FSM:** Dividir el comportamiento en etapas lógicas bien definidas aporta grandes ventajas en hardware:
   **Seguridad e predictibilidad:** El sistema tiene un comportamiento totalmente controlado; por ejemplo, el servomotor tiene prohibido activarse de la nada, ya que la FSM obliga al circuito a pasar estrictamente por el estado `VERIFY` antes de poder pisar el estado `OPEN`.


  stateDiagram-v2
    [*] --> ESCANEO
    ESCANEO --> VALIDAR_PRESION : Si alguna columna baja a 0
    VALIDAR_PRESION --> ESCANEO : Si fue un falso contacto
    VALIDAR_PRESION --> ESPERAR_LIBERACION : Si la tecla sigue hundida tras 60ms
    ESPERAR_LIBERACION --> VALIDAR_LIBERACION : Si todas las columnas vuelven a 1
    VALIDAR_LIBERACION --> ESPERAR_LIBERACION : Si se volvió a presionar antes de tiempo
    VALIDAR_LIBERACION --> ESCANEO : Si se mantiene libre tras 60ms (Manda Tecla Válida)


  stateDiagram-v2
    [*] --> IDLE
    IDLE --> INPUT : Se detecta una nueva tecla
    INPUT --> INPUT : Guarda dígito y corre registro (Mínimo 4 veces)
    INPUT --> VERIFY : Al completar los 4 dígitos
    VERIFY --> OPEN : Contraseña coincide (1234)
    VERIFY --> ERROR : Contraseña incorrecta
    OPEN --> IDLE : Se presiona '*' (Cierra puerta y limpia)
    ERROR --> IDLE : Se presiona '*' (Quita LED de error)

### C. Sincronización entre Dominios de Reloj
El escáner del teclado trabaja con el reloj lento (1 kHz), mientras que la FSM de control procesa los datos con el reloj rápido de la FPGA (50 MHz). Conectar la señal de validación directamente generaba lecturas inestables o pérdida de pulsaciones.
* **Solución:** Se añadieron biestables de sincronización de doble etapa (`key_valid_sync1` y `key_valid_sync2`) en el módulo de control para estabilizar la señal de entrada al dominio de 50 MHz. Adicionalmente, se creó un detector de flancos (`nueva_tecla`) que captura el pulso de manera limpia en un único ciclo de reloj.

### D. Inversión de la Lógica del Reset Físico
El botón de reset integrado en nuestra placa de desarrollo opera con lógica inversa (mantiene un '1' lógico por defecto y cae a '0' únicamente al ser presionado). Esto provocaba que el sistema se iniciara bloqueado en un estado de reinicio constante.
* **Solución:** En el módulo `top_sistema.v` se implementó una línea de inversión lógica: `wire rst_interno = ~rst;`. De esta forma, el sistema arranca con normalidad y solo se reinicia al presionar físicamente el pulsador.

---

## 3. Conclusiones

* **Eficiencia del Registro de Desplazamiento:** La técnica de concatenación de bits (`pass_in <= {pass_in[11:0], tecla};`) demostró ser una solución óptima en hardware para almacenar datos secuenciales en formato BCD, evitando el uso de estructuras de memoria o direccionamientos complejos.
* **Robustez mediante FSM:** Las máquinas de estados finitos son la herramienta fundamental para el diseño de sistemas de control seguros. Permitieron establecer un flujo riguroso de eventos, garantizando que el actuador físico jamás responda si no se cumple previamente con la validación de la clave.
* **Diseño orientado al entorno físico:** El desarrollo en FPGA requiere prever las imperfecciones mecánicas de los periféricos. El manejo de tiempos de rebote (*debounce*) y la sincronización de señales asíncronas son pasos obligatorios para trasladar con éxito un diseño del simulador al mundo real.
* **Integración Digital-Mecánica:** La combinación de procesamiento digital con modulación por ancho de pulso (PWM) permitió controlar un elemento mecánico real (servomotor) a partir de un flujo de datos numéricos, asentando las bases para el diseño de cerraduras electrónicas y sistemas de acceso cotidianos.

---

## 4. Estructura del Repositorio

* `/` (`top_sistema.v`): Módulo de jerarquía superior que interconecta todo el sistema.
* `/` (`control_clave.v`): FSM de control, almacenamiento por desplazamiento y comparador de contraseña.
* `/` (`escaner_teclado.v`): FSM de escaneo matricial de filas/columnas y filtro de rebotes.
* `/` (`pwm_servo.v`): Generador de señal PWM para la posición del servomotor a 20 ms de periodo.
* `/` (`divisor_frecuencia.v`): Divisor de reloj para reducir la frecuencia de 50 MHz a 1 kHz.

---

## 5. Referencias

* *Guía de Laboratorio 05 – Parte 2: Registro de Desplazamiento y Sistema de Contraseña con Control de Servo.*
* Mano, M. M., & Ciletti, M. D. (2013). *Digital Design: With an Introduction to the Verilog HDL*. Pearson.
"""

with open("README.md", "w", encoding="utf-8") as f:
    f.write(readme_content)

print("Archivo README.md generado correctamente.")


Tu archivo adaptado para GitHub está listo. He generado un archivo de texto con extensión .md formateado con bloques de código, diagramas en texto estructurado (monospace), listas organizadas y las convenciones correctas de Markdown que se usan en los repositorios de GitHub.

Tu archivo listo para descargar

README
MD
Aquí tienes una vista previa de cómo quedó estructurado el código Markdown para que lo puedas copiar y pegar directamente en tu archivo README.md de tu repositorio si lo prefieres:

Markdown
# Lab 05 – Parte 2: Registro de Desplazamiento y Sistema de Contraseña con Control de Servo

Este repositorio contiene la implementación en Verilog de un sistema de seguridad de acceso mediante un teclado matricial, un registro de desplazamiento para capturar una contraseña de 4 dígitos, una máquina de estados finitos (FSM) para la validación y un servomotor controlado por PWM como actuador de apertura.

---

## 1. Diseño Implementado

El sistema fue diseñado siguiendo una arquitectura modular en Verilog para mantener un flujo de datos limpio y ordenado:  
**Entrada de datos (Teclado) → Procesamiento (Validar la clave) → Acción (Mover el motor).**

Para asegurar un comportamiento estable en la tarjeta física (evitando problemas con los rebotes mecánicos de los botones y los cambios de velocidad en los relojes), dividimos el código en 5 bloques principales conectados jerárquicamente dentro de un módulo central (`top_sistema.v`):
 <img width="1200" height="1600" alt="image" src="" />
  

### Descripción de los Módulos

1. **Divisor de Frecuencia (`divisor_frecuencia.v`):** Reduce el reloj principal de la FPGA (50 MHz) a una señal mucho más lenta de 1 kHz para el escaneo del teclado. Esto evita que el circuito lea las pulsaciones tan rápido que confunda el ruido eléctrico con teclas presionadas.
2. **Escáner de Teclado (`escaner_teclado.v`):** Revisa las filas y columnas de forma secuencial para detectar qué tecla fue presionada. Implementa un contador de filtrado (*debounce*) para asegurar que la pulsación sea real y no un falso contacto.
3. **Control de Clave (`control_clave.v`):** Es el bloque de control principal. Almacena los dígitos introducidos en un registro que desplaza los datos a la izquierda, cuenta que se ingresen 4 dígitos y los compara con la contraseña secreta parametrizada (`16'h1234`).
4. **Módulo PWM (`pwm_servo.v`):** Genera la señal de control para el servomotor con un periodo de 20 ms. Si la puerta está configurada como abierta, envía un pulso de 2 ms; si está cerrada, envía un pulso de 1 ms.
5. **Top del Sistema (`top_sistema.v`):** Módulo de más alto nivel que se encarga de interconectar todos los bloques anteriores y asignar las señales a los pines físicos de la placa de desarrollo (LEDs, servomotor y líneas del teclado).

---

## 2. Implementación y Ajustes Críticos

Al realizar las pruebas en el hardware real, se identificaron varios desfases entre el comportamientos esperados los cuales se solucionaron mediante las siguientes modificaciones en el código:

### A. Ajuste del Reloj y Tiempo de Rebote del Teclado
El teclado matricial físico demostró ser sumamente sensible. Inicialmente, con el tiempo estándar de 10 ms, una sola pulsación suave registraba el mismo dígito múltiples veces consecutivas.
* **Solución:** Se incrementó el parámetro `DEBOUNCE_TIME` a **60 ms**. Como el contador ahora requería alcanzar un valor mayor, se expandió el registro `contador_debounce` a **8 bits** en `escaner_teclado.v` para evitar desbordamientos numéricos. Con este cambio, el teclado lee exactamente un dígito por cada pulsación real.

### B. Uso de Máquinas de Estados Finitos (FSM)
Tanto para el escaneo de pines como para la lógica de validación de la contraseña, se utilizaron estructuras de **máquinas de estados finitos**. Por ejemplo, el control de la contraseña transita de forma ordenada por los estados `IDLE` (espera), `INPUT` (captura de dígitos), `VERIFY` (comparación de contraseñas), y finalmente hacia `OPEN` (apertura) o `ERROR` (bloqueo).
* **Utilidad del diseño con FSM:** Dividir el comportamiento en etapas lógicas bien definidas aporta grandes ventajas en hardware:
 **Seguridad y predictibilidad:** El sistema tiene un comportamiento totalmente controlado; por ejemplo, el servomotor tiene prohibido activarse de la nada, ya que la FSM obliga al circuito a pasar estrictamente por el estado `VERIFY` antes de poder pisar el estado `OPEN`.
  

### C. Sincronización entre Dominios de Reloj
El escáner del teclado trabaja con el reloj lento (1 kHz), mientras que la FSM de control procesa los datos con el reloj rápido de la FPGA (50 MHz). Conectar la señal de validación directamente generaba lecturas inestables o pérdida de pulsaciones.
* **Solución:** Se añadieron biestables de sincronización de doble etapa (`key_valid_sync1` y `key_valid_sync2`) en el módulo de control para estabilizar la señal de entrada al dominio de 50 MHz. Adicionalmente, se creó un detector de flancos (`nueva_tecla`) que captura el pulso de manera limpia en un único ciclo de reloj.


---

## 3. Conclusiones

* **Eficiencia del Registro de Desplazamiento:** La técnica de concatenación de bits (`pass_in <= {pass_in[11:0], tecla};`) demostró ser una solución óptima en hardware para almacenar datos secuenciales en formato BCD, evitando el uso de estructuras de memoria o direccionamientos complejos.
* **Robustez mediante FSM:** Las máquinas de estados finitos son la herramienta fundamental para el diseño de sistemas de control seguros. Permitieron establecer un flujo riguroso de eventos, garantizando que el actuador físico jamás responda si no se cumple previamente con la validación de la clave.
* **Diseño orientado al entorno físico:** El desarrollo en FPGA requiere prever las imperfecciones mecánicas de los periféricos. El manejo de tiempos de rebote (*debounce*) y la sincronización de señales asíncronas son pasos obligatorios para trasladar con éxito un diseño del simulador al mundo real.
* **Integración Digital-Mecánica:** La combinación de procesamiento digital con modulación por ancho de pulso (PWM) permitió controlar un elemento mecánico real (servomotor) a partir de un flujo de datos numéricos, asentando las bases para el diseño de cerraduras electrónicas y sistemas de acceso cotidianos.

---

## 4. Estructura del Repositorio

* `/` (`top_sistema.v`): Módulo de jerarquía superior que interconecta todo el sistema.
* `/` (`control_clave.v`): FSM de control, almacenamiento por desplazamiento y comparador de contraseña.
* `/` (`escaner_teclado.v`): FSM de escaneo matricial de filas/columnas y filtro de rebotes.
* `/` (`pwm_servo.v`): Generador de señal PWM para la posición del servomotor a 20 ms de periodo.
* `/` (`divisor_frecuencia.v`): Divisor de reloj para reducir la frecuencia de 50 MHz a 1 kHz.

---

## 5. Referencias

* *Guía de Laboratorio 05 – Parte 2: Registro de Desplazamiento y Sistema de Contraseña con Control de Servo.*
* Mano, M. M., & Ciletti, M. D. (2013). *Digital Design: With an Introduction to the Verilog HDL*. Pearson.
