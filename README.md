[![Open in Visual Studio Code](https://classroom.github.com/assets/open-in-vscode-2e0aaae1b6195c2367325f4f02e2d04e9abb55f0b24a779b69b11b9e10269abc.svg)](https://classroom.github.com/online_ide?assignment_repo_id=24057923&assignment_repo_type=AssignmentRepo)
# Lab06 - Visualización usando pantalla LCD 16x2

# Integrantes
**Lady Lorena Cardenas Poveda**  
Carrera: Ingeniería Eléctrica  
Correo: llcardenasp@unal.edu.co  

**Samuel Tovar Vásquez**  
Carrera: Ingeniería Electrónica  
Correo: satovarv@unal.edu.co  


# Informe

Indice:

## Índice

1. [Introducción y Objetivos](#1-introducción-y-objetivos)
2. [Descripción de la Arquitectura de Hardware](#2-descripción-de-la-arquitectura-de-hardware)
3. [Diagrama de la Máquina de Estados Finitos (FSM)](#3-diagrama-de-la-máquina-de-estados-finitos-fsm)
4. [Implementación en Hardware](#4-implementación-en-hardware)
5. [Evidencias de Funcionamiento](#5-evidencias-de-funcionamiento)
6. [Conclusiones](#6-conclusiones)

## 1. Introducción y Objetivos
Este laboratorio introduce el diseño de hardware mediante Máquinas de Estados Finitos (FSM) utilizando Verilog HDL para interactuar con una pantalla LCD alfanumérica de 16x2 en modo de 8 bits. 

El proyecto se divide en dos fases:
1. **Parte 1 (Estática):** Inicialización y escritura de cadenas de texto fijas leídas desde un bloque de memoria ROM basado en un archivo externo (`path_to_txt.txt`).
2. **Parte 2 (Dinámica):** Modificación de la FSM y acoplamiento de lógica combinacional para interceptar la transmisión de datos e inyectar valores numéricos dinámicos en tiempo real provenientes de los interruptores individuales (*switches*) de la FPGA Altera Cyclone IV.

---

## 2. Descripción de la Arquitectura de Hardware

La solución implementada está contenida en el módulo `LCD1602_controller` y se compone de tres bloques estructurales descritos en hardware:

### A. Divisor de Frecuencia
Dado que la FPGA opera a una frecuencia elevada (50 MHz) en comparación con los tiempos de respuesta requeridos por el estándar HD44780 de la LCD, se diseñó un contador parametrizado (`COUNT_MAX = 800000`) para generar una señal síncrona lenta de `clk_16ms`. Esta señal dicta el periodo de refresco de la máquina de estados y se conecta de forma directa a la salida física de habilitación (`enable`) de la pantalla.

### B. Unidad de Decodificación ASCII
Para cumplir con los requerimientos dinámicos, el módulo captura un bus de entrada de 6 bits correspondiente a los interruptores (`input [5:0] sw`). El bus se segmenta en dos grupos de 3 bits:
* `sw[5:3]` para la medición de la Batería 1 (rango entero de 0 a 7).
* `sw[2:0]` para la medición de la Batería 2 (rango entero de 0 a 7).

A nivel combinacional, se realiza una conversión sumando el offset hexadecimal del carácter cero en la tabla ASCII (`8'h30`):
* `assign decodificador_L1 = 8'h30 + sw[5:3];`
* `assign decodificador_L2 = 8'h30 + sw[2:0];`

### C. Máquina de Estados Finitos (FSM)
La lógica de control se rige por una FSM síncrona al flanco de subida de `clk_16ms` con 5 estados operativos:

1. **`IDLE` (3'b000):** Estado de reposo. Permanece retenido hasta recibir la bandera lógica `ready_i = 1`.
2. **`CONFIG_CMD1` (3'b001):** Rutina secuencial de inicialización. Envía consecutivamente 4 comandos almacenados en `config_mem`: modo de bus a 8 bits/2 líneas (`8'h38`), desplazamiento del cursor a la derecha (`8'h06`), encendido de pantalla sin cursor (`8'h0C`), y limpieza general del display (`8'h01`). Durante este estado, la señal de control mantiene `rs = 0` (Modo Comando).
3. **`WR_STATIC_TEXT_1L` (3'b010):** Modo escritura de datos (`rs = 1`). Envía de forma secuencial los primeros 16 bytes direccionados en `static_data_mem`. Al alcanzar la posición de memoria indexada por `data_counter == 4'd12`, el flujo directo de la memoria ROM se intercepta para inyectar el bus síncrono `decodificador_L1`, imprimiendo el estado dinámico de la primera batería.
4. **`CONFIG_CMD2` (3'b011):** Transición síncrona de control (`rs = 0`). Transmite el comando de salto de línea `START_2LINE` (`8'hC0`) para reubicar la dirección de la DDRAM interna de la pantalla al inicio de la segunda línea física.
5. **`WR_STATIC_TEXT_2L` (3'b100):** Envía secuencialmente los 16 caracteres de la segunda línea. Al igual que en la primera línea, el flujo es interceptado cuando `data_counter == 4'd12` para mapear el bus dinámico `decodificador_L2` modificado por los interruptores de hardware. Una vez completado, el sistema retorna a `IDLE`.

---

## 3. Diagrama de la Máquina de Estados Finitos (FSM)

El siguiente modelo describe de forma abstracta las transiciones lógicas implementadas en el bloque multiplexor del controlador:

## 4. Implementación y Resultados en Hardware

Para la carga exitosa del sistema en la tarjeta de desarrollo **Altera Cyclone IV**, se ejecutó la siguiente metodología de configuración en el entorno Quartus Prime:

1. **Estructura del Proyecto:** El diseño estructural fue compilado vinculando la entidad de nivel superior `LCD1602_controller.v` junto al archivo de inicialización de caracteres estáticos `path_to_txt.txt` cargado verticalmente con los bytes en formato hexadecimal provistos para la visualización de texto base.
2. **Asignación de Pines y Restricciones:** Se enrutaron los puertos del bus de datos (`data[7:0]`) y las señales de sincronismo (`rs`, `rw`, `enable`) de acuerdo al mapeo de los conectores del *header* LCD de la placa de desarrollo.
3. **Liberación del Pin Dual 101:** Para habilitar el pin físico de la FPGA asignado a la línea de datos del módulo de visualización, se ingresó al menú *Assignments -> Device -> Device and Pin Options -> Dual-Purpose Pins* modificando el estado por defecto del pin de propósito dual `nCEO` a la configuración de operación normal **"Use as regular I/O"**.

### Comportamiento Dinámico Verificado:
Al energizar la FPGA, la pantalla se inicializa limpiamente cargando el texto plano estático:
* **Fila 1:** `Bateria 1: X`
* **Fila 2:** `Bateria 2: Y`

Donde `X` e `Y` varían instantáneamente de `0` a `7` al modificar el código binario de los interruptores físicos `sw[5:3]` y `sw[2:0]` respectivamente, validando la integridad del multiplexor y del decodificador combinacional ASCII integrados directamente en el flujo secuencial de la FSM.

## 5. Evidencias de Funcionamiento
A continuación, se presentan las pruebas fotográficas de la implementación física operando en la tarjeta de desarrollo:

<img width="1200" height="1600" alt="image" src="https://github.com/user-attachments/assets/e164a881-1103-4f86-a759-e3a944b2dd10" />

<img width="1200" height="1600" alt="image" src="https://github.com/user-attachments/assets/a9a0ad0f-19bb-46c6-b3a4-dc5ab20a5e92" />


Inicialización de la Máquina de Estados
Durante los primeros ciclos de reloj, la FSM envía los comandos de configuración, limpia la pantalla y procede a escribir exitosamente el texto estático correspondiente a la primera fila, demostrando el correcto funcionamiento del estado WR_STATIC_TEXT_1L.

Ciclo Completo de Escritura
Tras enviar el comando de salto de línea (0xC0) en el estado CONFIG_CMD2, la FSM transiciona al estado WR_STATIC_TEXT_2L y completa la visualización de la memoria. La pantalla despliega los mensajes "Bateria 1" y "Bateria 2" de manera estable y continua.

## 6. Conclusiones

* **Control Secuencial mediante FSM:** La implementación de una Máquina de Estados Finitos (FSM) en Verilog demostró ser la arquitectura de *hardware* ideal para el manejo de la pantalla LCD. Dado que el lenguaje de descripción de hardware es inherentemente concurrente, la FSM permitió imponer un flujo secuencial riguroso, garantizando que los comandos de inicialización y los datos de los caracteres se enviaran en el orden estricto que exige el controlador de la pantalla.
* **Sincronización y Dominios de Reloj:** Se comprobó la importancia crítica de adaptar las velocidades de procesamiento entre distintos dispositivos físicos. La frecuencia nativa de la FPGA (50 MHz) es demasiado alta para los tiempos de respuesta del display. El diseño del divisor de frecuencia para generar la señal `clk_16ms` fue fundamental para evitar la saturación del bus de datos y garantizar una visualización estable.
* **Visualización Dinámica y Decodificación ASCII:** Se logró modificar con éxito un flujo de datos estático proveniente de una memoria ROM mediante la inyección en tiempo real de variables dinámicas. La implementación de lógica combinacional para sumar el *offset* hexadecimal (`8'h30`) a las entradas binarias de los interruptores evidenció cómo el *hardware* puede traducir y acondicionar señales físicas directas al estándar ASCII de manera instantánea.
* **Consideraciones Físicas del Hardware:** El proceso de síntesis e implementación resaltó que el diseño en FPGA trasciende el código Verilog. La necesidad de reconfigurar pines de propósito dual (como el pin 101, `nCEO`) en Quartus Prime subraya la importancia de conocer a fondo la arquitectura física y las restricciones de la tarjeta de desarrollo específica (Altera Cyclone IV) para evitar fallos de compilación y colisiones eléctricas.

## 7. Referencias

* *Guía de Laboratorio 06 – Parte 1 y 2.*


