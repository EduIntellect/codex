# Diseño conceptual del simulador web **asmbasic**

## 1) Modelo de CPU en JavaScript

### Estado global de CPU
Definir una estructura de estado única (por ejemplo `cpuState`) con estos campos lógicos:

- **RF (Register File)**: banco de 8 registros de propósito general `R0..R7`, con ancho fijo (recomendado: 16 bits para docencia).
- **RT (Registro Temporal)**: registro auxiliar para operaciones internas del ciclo.
- **IR (Instruction Register)**: instrucción actual ya capturada de memoria, en formato interno parseado.
- **PC (Program Counter)**: índice entero de la próxima instrucción en memoria.
- **Flags**: objeto con bits booleanos `C, S, Z, P, V`.
- **Memoria principal única (Von Neumann)**: un arreglo lineal indexado por dirección, compartido para instrucciones y datos.
- **Estado de ejecución**: flags de control tipo `halted`, `running`, `currentPhase`, `cycleCount`.

### Representación recomendada por componente
- **RF**: arreglo numérico de longitud fija 8 (`rf[0]..rf[7]`).
- **RT**: escalar numérico.
- **IR**: objeto de instrucción normalizada (ver sección 2).
- **PC**: entero sin signo acotado al rango de memoria.
- **Flags**: objeto `{ C:false, S:false, Z:false, P:false, V:false }`.
- **Memoria**:
  - Opción didáctica simple: arreglo de celdas tipadas por contenido (`{ kind: 'instr'|'data', value: ... }`).
  - Opción eficiente: dos vistas (lista de instrucciones parseadas + RAM de datos), pero visualmente presentadas como único espacio Von Neumann.

### Ausencia explícita de pila y subrutinas
- **No se define SP** en el estado de CPU.
- **No se reserva segmento de pila** en memoria.
- **No existen CALL/RET/PUSH/POP** en parser, ISA ni UI.
- La semántica de control de flujo queda restringida a `BZ`, `BN`, `BR` y `HALT`.

---

## 2) Modelo de instrucciones de asmbasic

### Formato interno de instrucción
Normalizar cada instrucción parseada a un objeto homogéneo:

- `op`: mnemónico canónico (`LDI`, `ADD`, `BZ`, etc.).
- `rd`: índice de registro destino (cuando aplique).
- `rs`: índice de registro fuente (cuando aplique).
- `imm`: inmediato numérico (para `LDI`).
- `addr`: dirección absoluta de memoria (para `LD`, `ST`, saltos).
- `text`: representación original opcional para trazabilidad UI.
- `line`: número de línea fuente para mensajes de error/resaltado.

### Gramática mínima conceptual
- Registros: `R0..R7`.
- Inmediatos: decimal por defecto; opcionalmente binario (`0b...`) y hexadecimal (`0x...`) para reforzar representación binaria.
- Direcciones: absolutas entre corchetes en memoria de datos (`LD`, `ST`) y absolutas para salto (`BZ`, `BN`, `BR`).

### Mapeo de operandos por instrucción
- **Registro + inmediato**: `LDI Rn, imm`.
- **Registro + registro**: `MOV`, `ADD`, `SUB`, `AND`, `OR`, `XOR`, `CMP`.
- **Unario sobre registro**: `NOT`, `SHL`, `SHR`, `CMPZ`.
- **Acceso memoria**: `LD Rn, [addr]`, `ST Rn, [addr]`.
- **Control flujo**: `BZ addr`, `BN addr`, `BR addr`, `HALT`.

### Política de anchura y saturación
Para consistencia didáctica:
- Ancho fijo de palabra (p. ej. 16 bits).
- Todas las operaciones aritmético-lógicas se enmascaran al ancho.
- `S` se toma del bit más significativo; `Z` de resultado cero; `P` de paridad del resultado; `C` y `V` según reglas de suma/resta/desplazamiento.

---

## 3) Ciclo de ejecución (fetch → decode → execute → writeback → PC update)

### Fase 1: Fetch
- Leer celda de memoria en dirección `PC`.
- Copiar contenido a `IR`.
- Validar que sea instrucción ejecutable (si no, excepción didáctica).

### Fase 2: Decode
- Interpretar `IR.op` y determinar micro-acciones requeridas.
- Resolver índices de registros, inmediatos y direcciones ya parseadas.
- Preparar operandos en `RT` cuando convenga mostrar transferencia intermedia.

### Fase 3: Execute
- Ejecutar la ALU o la lógica de control:
  - ALU (`ADD`, `SUB`, `AND`, `OR`, `XOR`, `NOT`, `SHL`, `SHR`, `CMP`, `CMPZ`).
  - Memoria (`LD`, `ST`).
  - Control de flujo (`BZ`, `BN`, `BR`, `HALT`).
- Calcular flags candidatas en esta fase.

### Fase 4: Writeback
- Escribir resultado a destino si aplica (`rf[rd]`, memoria en `ST`).
- En instrucciones de comparación (`CMP`, `CMPZ`) actualizar flags sin escribir registro destino.

### Fase 5: Actualización de PC
- Regla base: `PC = PC + 1`.
- Excepciones de salto:
  - `BZ addr`: si `Z==1`, `PC = addr`; si no, secuencial.
  - `BN addr`: si `S==1`, `PC = addr`; si no, secuencial.
  - `BR addr`: siempre `PC = addr`.
  - `HALT`: no avanza; estado `halted=true`.

### Rol de flags en control de flujo
- `BZ` depende exclusivamente de `Z`.
- `BN` depende exclusivamente de `S`.
- `BR` ignora flags.
- `HALT` detiene ejecución sin evaluación de flags.

---

## 4) Arquitectura de la interfaz (frontend)

## Distribución en 3 paneles
1. **Panel izquierdo: Editor asmbasic**
   - Área de texto con numeración de líneas.
   - Botones: `Cargar`, `Ensamblar/Validar`, `Run`, `Step`, `Reset`.
   - Resaltado de línea actual (`PC`) y línea con error de sintaxis.

2. **Panel central: Estado CPU**
   - Bloque de registros RF (`R0..R7`) en tabla.
   - Bloque de registros especiales: `RT`, `IR`, `PC`.
   - Bloque de flags `C,S,Z,P,V` con indicadores visuales (0/1).
   - Sección de fase actual del ciclo (fetch/decode/execute/writeback/pc-update).

3. **Panel derecho: Memoria**
   - Vista tabular dirección/valor/tipo (`instr` o `data`).
   - Marcadores visuales:
     - Celda apuntada por `PC`.
     - Última lectura/escritura.
   - Opcional: filtros (rango direcciones, solo instrucciones, solo datos).

### Comportamiento UI recomendado
- Modo **paso a paso por fase** (micro-step) y **paso por instrucción**.
- Trazas breves por instrucción ejecutada.
- Validación estática antes de ejecutar para prevenir estados inválidos.

---

## 5) Empaquetado en un único HTML

### Organización interna del archivo
- `<head>`:
  - Metadatos y `<style>` completo (layout, tablas, colores de estado).
- `<body>`:
  - Contenedor principal con grid/flex de 3 paneles.
  - Plantillas mínimas de componentes (si se desea con `<template>`).
- `<script>` (embebido):
  - Módulo 1: constantes ISA y utilidades (parseo numérico, masking, flags).
  - Módulo 2: ensamblador/parser (texto → instrucciones internas).
  - Módulo 3: núcleo CPU (máquina de estados por fases).
  - Módulo 4: renderizado UI y binding de eventos.
  - Módulo 5: presets de programas docentes.

### Requisitos para embebido en iframe (Google Sites u otros)
- No depender de backend ni rutas relativas externas críticas.
- Evitar `import` ES modules remotos y peticiones de red obligatorias.
- Usar recursos inline (CSS/JS y, si aplica, SVG embebido).
- Diseñar layout responsive al tamaño del iframe (`width:100%`, alturas adaptables).
- Evitar APIs bloqueadas en entornos embebidos (popups, descargas automáticas).
- Si se usa `localStorage`, contemplar fallback por restricciones del host.
- Mantener políticas de seguridad compatibles: sin `eval`, sin inyección HTML no sanitizada.

---

## Criterios docentes de calidad (para la implementación posterior)

- **Observabilidad total del ciclo**: cada fase visible y comprensible.
- **Determinismo**: misma entrada, misma traza.
- **Mensajes pedagógicos**: errores de sintaxis y ejecución con línea y causa.
- **Simplicidad estructural**: arquitectura clara, sin pila y sin subrutinas.
- **Foco curricular**: prioridad en aritmética, lógica binaria, flags y saltos condicionales.
