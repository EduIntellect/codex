# Título y autoría

- **Título:** “CODE2 Web: Simulador de Arquitectura Educativa para Enseñanza de Fundamentos de Computadores”.
- **Autor:** GPT-5.2-Codex (OpenAI).
- **Titular:** GPT-5.2-Codex (OpenAI).

## Descripción breve del programa

**Qué es:** un simulador web de la arquitectura CODE-2 orientado a la docencia universitaria.

**Qué hace:** permite introducir programas en ensamblador CODE-2, ensamblarlos, ejecutarlos paso a paso o en modo continuo, y visualizar el estado de registros, memoria y flags durante la ejecución.

## Lenguaje de programación y entorno operativo

### Lenguajes

- **HTML5** para la estructura de la interfaz.
- **CSS3** para estilos.
- **JavaScript (ECMAScript)** para la lógica del simulador.

### Entorno

- Navegadores modernos: **Chrome, Firefox y Edge**.
- Sistemas operativos: **Windows, macOS y Linux**.
- Integración como recurso estático en **Google Sites** (mediante URL pública/iframe).

## Listado de ficheros

- **`code2_simulator.html`**: página principal con layout de 3 paneles, controles y área de mensajes, incluyendo también la lógica del simulador en un único archivo.
- **`simulator_core.js`**: núcleo de simulación de la arquitectura CODE-2 (CPU, ALU, ensamblador y ciclo de ejecución).
- **`ui_controller.js`**: gestión del DOM (render de registros, memoria, mensajes y eventos de botones).
- **`code2_sites.html`** (opcional): versión autocontenida específica para incrustación en Sites.

> Nota: en el estado actual del repositorio se dispone de una versión autocontenida en `code2_simulator.html`.

## Modelo de CPU y semántica de instrucciones

- **Banco de registros:** `RF[0..7]`.
- **Registros de control/estado:** `PC`, `SP`, `RT`, `IR`.
- **Flags:** `C`, `S`, `Z`, `P`, `V`.
  - `C` (carry/borrow): acarreo en suma y préstamo en resta.
  - `S` (signo): bit más significativo del resultado (16 bits).
  - `Z` (cero): activada cuando el resultado es `0`.
  - `P` (paridad): `1` si hay paridad par en el resultado de 16 bits.
  - `V` (overflow): desbordamiento aritmético en operaciones con signo.
- **Memoria:** lineal de `256` posiciones (`0..255`) para instrucciones y datos.

### Juego de instrucciones soportado

- **`LLI Rd, imm`**: carga inmediata en el byte bajo de `Rd`.
- **`LHI Rd, imm`**: carga inmediata en el byte alto de `Rd`.
- **`ADDS Rd, Rs`**: suma con semántica de signo (`Rd = Rd + Rs`) y actualización de flags.
- **`SUBS Rd, Rs`**: resta con semántica de signo (`Rd = Rd - Rs`) y actualización de flags.
- **`NAND Rd, Rs`**: operación lógica NAND bit a bit (`Rd = ~(Rd & Rs)`).
- **`SHL Rd`**: desplazamiento lógico a la izquierda de `Rd`.
- **`SHR Rd`**: desplazamiento lógico a la derecha de `Rd`.
- **`LD Rd, addr`**: carga en `Rd` el dato almacenado en memoria `addr`.
- **`ST Rs, addr`**: almacena en memoria `addr` el contenido de `Rs`.
- **`BZ target`**: salto a `target` si `Z == 1`.
- **`BS target`**: salto a `target` si `S == 1`.
- **`BR target`**: salto incondicional a `target`.
- **`HALT`**: detiene la ejecución.

## Diagrama de flujo (simplificado)

```mermaid
flowchart TD
    A[Entrada: texto ensamblador CODE-2] --> B[Ensamblado]
    B --> C[Carga en memoria 0..255]
    C --> D{Bucle step}
    D --> E[Fetch: IR ← M[PC]]
    E --> F[Decode: op + operandos]
    F --> G[Execute: ALU / LOAD / STORE / BRANCH]
    G --> H[Update: PC, flags y memoria]
    H --> I[Render UI: registros, memoria, flags, mensajes]
    I --> J{HALT o error?}
    J -- No --> D
    J -- Sí --> K[Fin de ejecución]
```
