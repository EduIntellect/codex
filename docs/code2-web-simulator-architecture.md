# Arquitectura conceptual: simulador web CODE-2 en un único HTML

## 1) Modelo de CPU en JavaScript

### Estado base de la CPU
- **RF (registro de propósito general)**: arreglo tipado (`Int16Array` o `Uint16Array`) de tamaño fijo (por ejemplo, 8 o 16 registros) para simplificar operaciones y asegurar semántica de palabra de 16 bits.
- **RT (registro temporal)**: entero de 16 bits usado como latch de datos intermedios entre etapas (`decode/execute/writeback`).
- **IR (instruction register)**: objeto con dos vistas:
  - cruda: palabra original de memoria (si se quiere mostrar codificación binaria/hex),
  - decodificada: estructura normalizada (`op`, operandos, metadatos de salto).
- **PC (program counter)**: entero sin signo de 16 bits (o limitado al tamaño de memoria), siempre apuntando a la siguiente instrucción a buscar.
- **SP (stack pointer)**: entero de 16 bits independiente del PC; aunque CODE-2 mínimo no incluya push/pop, se mantiene visible para docencia y extensibilidad.
- **Flags**: objeto plano `{ C, S, Z, P, V }` con valores booleanos, actualizado únicamente por operaciones ALU relevantes (y no por todas las instrucciones).
- **Estado HALT**: bandera `halted: boolean` en el estado global; cuando `true`, bloquea `step/run` hasta `reset`.

### Memoria principal
- Modelo lineal direccionable por palabra (16 bits).
- Tamaño sugerido para prácticas: **256 o 1024 posiciones** (suficiente para ejercicios de aula y renderización manejable en UI).
- Estructura: `Uint16Array(memSize)` para representar contenido bruto; para depuración, se puede mantener un mapa auxiliar de metadatos (`source line`, etiquetas resueltas, breakpoint).

### Estado global recomendado
- Objeto `cpuState` único con subárboles: `registers`, `flags`, `memory`, `pipelineStage`, `halted`, `cycleCount`, `lastError`.
- Este estado central facilita un flujo determinista tipo “reducer + render”.

## 2) Modelo de instrucciones CODE-2

### Representación interna normalizada
- Usar objetos homogéneos del estilo:
  - `{ op, rd, rs, rt, imm, addr, label, mode }`
- Solo algunos campos aplican según instrucción; los no usados quedan en `null`.

### Convención de operandos
- **Registros**: índices enteros (`rd`, `rs`, `rt`) hacia RF.
- **Inmediatos (`imm`)**: entero de 8 o 16 bits según instrucción, con política explícita de sign/zero extension.
- **Direcciones (`addr`)**:
  - absolutas para `BR`, `BZ`, `BS` en versión docente simple, o
  - relativas a PC si se desea introducir desplazamientos.
- **Etiquetas (`label`)**: solo durante ensamblado; en ejecución todo salto se resuelve a dirección numérica.

### Semántica mínima por instrucción
- **LLI**: reemplaza la parte baja del registro destino con inmediato bajo.
- **LHI**: reemplaza la parte alta del registro destino con inmediato alto.
- **ADDS**: suma con interpretación con signo; actualiza flags relevantes.
- **SUBS**: resta con signo; actualiza flags relevantes.
- **LD**: lee memoria en dirección efectiva y carga en registro.
- **ST**: escribe contenido de registro en memoria.
- **SHL**: desplazamiento lógico a izquierda (×2 en enteros no saturados).
- **SHR**: desplazamiento a derecha (en diseño docente conviene fijar si lógico o aritmético y mantenerlo consistente).
- **NAND**: negación de AND bit a bit de operandos.
- **BZ**: salto si `Z == 1`.
- **BS**: salto si `S == 1`.
- **BR**: salto incondicional.
- **HALT**: activa `halted` y detiene el ciclo.

## 3) Ciclo de ejecución

### Visión por etapas
1. **Fetch**
   - Leer memoria en `PC`.
   - Copiar palabra a `IR.raw`.
   - (Opcional didáctico) guardar `MAR/MDR` virtuales como variables internas para visualización.

2. **Decode**
   - Traducir `IR.raw` (o instrucción textual ya ensamblada) a `IR.decoded`.
   - Determinar tipo: ALU, memoria, salto, control.
   - Extraer operandos y preparar señales de control internas (`willWriteReg`, `willWriteMem`, `willBranch`).

3. **Execute**
   - **ALU**: calcular resultado para `ADDS/SUBS/SHL/SHR/NAND`.
   - **Memoria**:
     - `LD`: calcular dirección efectiva y leer.
     - `ST`: calcular dirección efectiva y preparar dato a escribir.
   - **Saltos**:
     - evaluar condición (`BZ`, `BS`) o siempre verdadero (`BR`),
     - decidir `nextPC`.

4. **Writeback**
   - Escribir resultado en registro destino para ALU/LD/LLI/LHI.
   - Escribir memoria para ST.
   - Actualizar flags según política:
     - típicamente en operaciones ALU (y opcionalmente SHL/SHR),
     - no en saltos ni en HALT.

5. **Actualización de PC**
   - Si hay salto tomado: `PC = target`.
   - Si no hay salto: `PC = PC + 1`.
   - En `HALT`: no avanzar más ciclos automáticos.

### Política de flags recomendada
- **Z**: resultado igual a cero.
- **S**: bit de signo del resultado.
- **C**: acarreo/borrow en suma/resta (definir con precisión para SUBS).
- **V**: overflow aritmético con signo.
- **P**: paridad del resultado (por ejemplo, paridad par del número de bits a 1).

## 4) Arquitectura de la interfaz (frontend)

### Layout general de 3 paneles
- **Panel izquierdo: editor + controles**
  - `textarea` para programa CODE-2.
  - Botones: **Ensamblar**, **Paso**, **Ejecutar**, **Reset**.
  - Área de mensajes: errores de ensamblado, estado de ejecución, instrucción actual.

- **Panel central: estado CPU**
  - Tabla/tiles de RF (R0..Rn).
  - Bloque destacado para RT, IR, PC, SP.
  - Indicadores visuales de flags (`C S Z P V`) tipo LED/chip activo-inactivo.
  - Línea de etapa actual (`FETCH`, `DECODE`, etc.) y contador de ciclos.

- **Panel derecho: memoria**
  - Tabla paginada o ventana desplazable con columnas: `Dir`, `Hex`, `Dec`, opcional `ASCII`.
  - Resaltado de celda apuntada por `PC` y de última dirección accedida por `LD/ST`.

### Interacción recomendada
- `Ensamblar`:
  - parsea texto, resuelve etiquetas, carga memoria de programa, hace reset parcial.
- `Paso`:
  - ejecuta exactamente un ciclo completo de instrucción (o subpaso si se quiere modo microetapas).
- `Ejecutar`:
  - bucle temporizado (`setInterval` / `requestAnimationFrame`) con control de velocidad.
- `Reset`:
  - inicializa CPU, flags, PC/SP, memoria de datos (y opcionalmente conserva programa).

## 5) Empaquetado en un único HTML

### Organización interna del archivo
- `<style>`
  - CSS autocontenido para grid de 3 paneles, tablas y estado visual de flags.
  - Evitar dependencias externas (frameworks, fuentes CDN) para máxima portabilidad.

- `<script>`
  - Módulos lógicos separados por secciones internas:
    1. constantes/configuración (`memSize`, número de registros),
    2. modelo CPU + utilidades de palabra/flags,
    3. ensamblador CODE-2 (lexer simple + parser por líneas + resolución de labels),
    4. motor de ejecución (`stepInstruction`, `run`, `halt`),
    5. render/UI binding (`renderCPU`, `renderMemory`, `renderMessages`),
    6. controladores de eventos de botones.

### Requisitos para incrustar en iframe (Google Sites u otros)
- Archivo autocontenido sin imports externos para evitar bloqueos por CSP o recursos mixtos.
- Uso exclusivo de APIs web estándar del navegador (sin backend, sin `fetch` obligatorio).
- Evitar `window.top` y operaciones cross-origin.
- Diseñar con contenedor responsive para ajustarse a tamaños de iframe variables.
- Incluir estilos con `box-sizing: border-box` y manejo de overflow para que la tabla de memoria no rompa el layout embebido.

### Extensibilidad docente prevista
- Añadir modo “micropaso” por etapa (`fetch/decode/execute/writeback/pc-update`).
- Breakpoints por dirección.
- Carga/descarga de programas con `localStorage` o archivo local, sin salir del modelo 100 % cliente.
