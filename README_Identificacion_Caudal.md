# Identificación experimental del subproceso de caudal

> **Ubicación sugerida en GitHub:** `caudal/README.md`

## Objetivo del subproceso

Este subproyecto caracteriza experimentalmente la relación entre el PWM aplicado a la bomba JT-500 y el caudal medido por el sensor ZJ-S201. El ESP32 genera la señal PWM, cuenta los pulsos del caudalímetro, ejecuta la secuencia de ensayo, supervisa las condiciones de seguridad y transmite los datos por el puerto serial.

La identificación se realiza alrededor de un punto de operación reproducible, con la válvula manual V-102 fija en **15/16 de apertura**. Se aplican escalones positivos y negativos de PWM para estimar la ganancia, la constante de tiempo y el retardo del subproceso. El modelo obtenido se utiliza posteriormente como planta del lazo interno del control en cascada de presión y caudal.

## Relación con la metodología de referencia

El repositorio del profesor caracteriza un motor mediante la relación PWM–RPM. En este proyecto se conserva la misma lógica experimental, pero se adapta de la siguiente manera:

| Elemento metodológico | Aplicación en este proyecto |
|---|---|
| Proceso | Bomba JT-500 y línea hidráulica de 1/2 pulgada |
| Variable manipulada | PWM de la bomba, de 0 a 255 cuentas |
| Variable medida | Caudal filtrado, en L/min |
| Sensor | Caudalímetro ZJ-S201 |
| Actuador | Bomba JT-500 accionada mediante L298N |
| Prueba experimental | Escalones alrededor de PWM 189 |
| Punto de operación | V-102 en 15/16; caudal cercano a 2,8 L/min |
| Datos exportados | Registros CSV copiados desde el monitor serial |
| Modelo aproximado | Primer orden con retardo, FOPDT |

## Descripción funcional

```mermaid
flowchart LR
    A["ESP32: PWM"] --> B["L298N"]
    B --> C["Bomba JT-500"]
    C --> D["ZJ-S201"]
    D --> E["Caudal Q"]
    E -. "pulsos" .-> A
```

La bomba toma agua del depósito TK-101 y la impulsa a través del ZJ-S201. La posición de V-102, el nivel disponible en el depósito, el diámetro de las líneas y la fuente de alimentación influyen en la relación PWM–caudal; por ello deben mantenerse sin cambios durante las repeticiones.

## Componentes utilizados

- ESP32 Dev Module.
- Bomba JT-500 de 12 VCC.
- Driver L298N, canal A para la bomba.
- Caudalímetro ZJ-S201.
- Fuente CA/CC de 12 V y 3 A.
- Válvula manual V-102.
- Electroválvula auxiliar conectada al canal B del L298N.
- Mangueras y tuberías de 1/2 pulgada.
- Depósito TK-101 y columna piezométrica con rebose.
- Cable USB para programar y supervisar el ESP32.

## Advertencias eléctricas e hidráulicas

- No alimentar la bomba desde el ESP32.
- Alimentar la bomba y el L298N desde la fuente de 12 V.
- Mantener tierra común entre ESP32, L298N, sensores y fuente.
- Retirar el jumper de `ENA` si el módulo L298N lo incluye y se desea controlar la bomba por PWM.
- No introducir en un GPIO una señal superior a 3,3 V sin acondicionamiento.
- Mantener la señal del ZJ-S201 acondicionada para una entrada segura del ESP32.
- No hacer funcionar la bomba en seco.
- Verificar que la succión esté cebada y que TK-101 tenga agua suficiente.
- Mantener libre el rebose de la columna.
- No modificar V-102 durante una corrida de identificación de caudal.
- Detener la prueba ante pérdida prolongada de pulsos, fuga, obstrucción o nivel inseguro.
- La parada por software no sustituye la desconexión física de emergencia.

## Tabla de conexiones

| Dispositivo | Terminal | Conexión |
|---|---|---|
| L298N, bomba | `ENA` | GPIO 25 |
| L298N, bomba | `IN1` | GPIO 26 |
| L298N, bomba | `IN2` | GPIO 27 |
| ZJ-S201 | Señal de pulsos | GPIO 33 mediante acondicionamiento |
| HC-SR04 | `TRIG` | GPIO 18 |
| HC-SR04 | `ECHO` | GPIO 19 mediante adaptación a 3,3 V |
| L298N, electroválvula | `ENB` | GPIO 32 |
| L298N, electroválvula | `IN3` | GPIO 14 |
| L298N, electroválvula | `IN4` | GPIO 13 |
| Fuente | `+12 V` | Entrada de potencia del L298N |
| Fuente y ESP32 | `GND` | Tierra común |

## Calibración del ZJ-S201

La constante volumétrica adoptada es:

```text
453,33 pulsos/L
```

Para un intervalo de adquisición `Δt`, el caudal se calcula mediante:

```text
Q = (N / (453,33 · Δt)) · 60
```

donde:

- `Q` es el caudal en L/min;
- `N` es el número de pulsos contado;
- `Δt` es el intervalo de medición en segundos.

El intervalo de registro usado en las pruebas es aproximadamente **0,5 s**. La calibración debe verificarse comparando el volumen calculado por pulsos con un volumen real medido en un recipiente graduado.

## Configuración del ESP32

- Entorno: Arduino IDE.
- Placa: `ESP32 Dev Module`.
- Resolución PWM: 8 bits.
- Frecuencia PWM: 5 kHz.
- Velocidad del monitor serial: 115200 baudios.
- Intervalo de muestreo: 500 ms.
- Entrada principal: PWM aplicado al pin `ENA` del L298N.
- Adquisición principal: interrupción de pulsos en GPIO 33.

## Archivos del subproceso

```text
caudal/
├── README.md
├── firmware/
│   └── Identificacion_Lazo_Abierto_Caudal.ino
├── data/
│   ├── Identificacion_Lazo_Abierto_Caudal_R1.txt
│   └── Identificacion_Lazo_Abierto_Caudal_R2.txt
├── images/
│   ├── montaje_caudal.jpg
│   ├── respuesta_escalon_caudal.png
│   └── comparacion_R1_R2_caudal.png
└── analysis/
    └── analisis_caudal.m
```

El nombre `Identificacion_Lazo_Abierto_Caudal.ino` es el nombre recomendado al organizar el firmware definitivo en GitHub.

## Preparación del ensayo

1. Revisar las conexiones y confirmar la tierra común.
2. Llenar TK-101 y purgar el aire de la línea.
3. Comprobar que la bomba esté cebada.
4. Ubicar V-102 exactamente en **15/16 de apertura**.
5. Confirmar que la descarga, el retorno y el rebose estén libres.
6. Abrir la trayectoria de flujo antes de energizar la bomba.
7. Abrir el monitor serial a 115200 baudios.
8. Confirmar que el caudal sea aproximadamente cero antes de iniciar.
9. Reiniciar el sistema y los acumuladores de datos.

## Secuencia de identificación

Las dos repeticiones R1 y R2 usan la misma secuencia:

| Etapa | Entrada aplicada | Duración aproximada | Propósito |
|---|---:|---:|---|
| Preapertura | PWM 0 | 1 s | Abrir la trayectoria antes del bombeo |
| Acondicionamiento | PWM 189 | 40 s | Llevar el sistema al punto nominal |
| Base inicial | PWM 189 | 10 s | Estimar el régimen inicial |
| Escalón positivo | PWM 189 → 209 | 15 s | Medir la respuesta ante `ΔPWM = +20` |
| Regreso a base | PWM 209 → 189 | 15 s | Comprobar recuperación |
| Escalón negativo | PWM 189 → 169 | 15 s | Medir la respuesta ante `ΔPWM = -20` |
| Base final | PWM 169 → 189 | 10 s | Comprobar repetibilidad y deriva |
| Drenaje | PWM 0 | 10 s | Llevar el montaje a condición segura |

En el monitor serial se deben conservar las marcas que comienzan con `#`, porque identifican cada transición experimental.

Marcas principales:

```text
# INICIO_IDENTIFICACION_ABIERTA_CAUDAL
# V102_DEBE_ESTAR_FIJA_EN_15_16
# PI_CAUDAL_DESACTIVADO
# BASE_INICIAL: PWM=189
# ESCALON_POSITIVO: PWM_189_A_209
# REGRESO_BASE: PWM_209_A_189
# ESCALON_NEGATIVO: PWM_189_A_169
# REGRESO_FINAL: PWM_169_A_189
# FIN_IDENTIFICACION: BOMBA_APAGADA_Y_VALVULA_CERRADA
```

## Comandos seriales

La secuencia habitual es:

```text
R
C
E
```

- `R`: rearma el sistema si las condiciones son seguras.
- `C`: reinicia contadores, filtros y volúmenes.
- `E`: inicia el ensayo automático.
- `P`: ejecuta la parada manual y el alivio previsto por el firmware.

Si el firmware muestra una secuencia distinta al iniciar, se debe seguir la cabecera impresa por esa versión.

## Datos registrados

Los archivos de ensayo incluyen, según la versión del firmware:

- tiempo absoluto y tiempo de ensayo;
- fase experimental;
- estado de la válvula auxiliar;
- PWM aplicado;
- pulsos contados;
- frecuencia en Hz;
- caudal instantáneo en L/min;
- caudal filtrado en L/min;
- volumen acumulado;
- distancia, altura y presión auxiliares;
- calidad de la medición;
- estado de seguridad.

Los datos numéricos se almacenan como filas separadas por comas. Las líneas que empiezan con `#` son anotaciones y no deben eliminarse antes de conservar una copia del archivo original.

## Resultados experimentales resumidos

Promedios de fase calculados a partir de los registros disponibles:

| Corrida | Base inicial, PWM 189 | Escalón alto, PWM 209 | Escalón bajo, PWM 169 |
|---|---:|---:|---:|
| R1 | 2,754 L/min | 3,326 L/min | 2,321 L/min |
| R2 | 2,793 L/min | 3,337 L/min | 2,335 L/min |

Las dos corridas reproducen la dirección y la magnitud general de la respuesta. La diferencia entre los incrementos positivo y negativo muestra que la relación PWM–caudal no debe considerarse perfectamente lineal fuera del entorno identificado.

## Modelo dinámico identificado

El modelo consolidado alrededor de PWM 189 y V-102 en 15/16 es:

```text
              0,02584 · e^(-0,40s)
GQ(s) = --------------------------------
                    0,65s + 1
```

con:

```text
GQ(s) = ΔQ(s) / ΔPWM(s)
```

| Parámetro | Valor | Unidad |
|---|---:|---|
| Ganancia `KQ` | 0,02584 | (L/min)/cuenta PWM |
| Constante de tiempo `τQ` | 0,65 | s |
| Retardo `LQ` | 0,40 | s |

Este modelo es válido localmente para el montaje construido, la fuente de 12 V, las líneas de 1/2 pulgada y V-102 en 15/16. No debe extrapolarse directamente a otra bomba, tubería, nivel del depósito o apertura de válvula.

## Interpretación de resultados

### Zona muerta

Debe analizarse con ensayos adicionales desde reposo o con una rampa de PWM. Los escalones alrededor de PWM 189 identifican la dinámica local, pero no determinan por sí solos el PWM mínimo de arranque.

### No linealidad

La ganancia observada puede variar entre el escalón positivo y el negativo por pérdidas hidráulicas, fricción, presión de descarga y resolución del caudalímetro.

### Saturación

El control está limitado físicamente por el rango PWM de 0 a 255 y, durante la operación regulada, por los límites establecidos en el firmware. No se debe usar el modelo local para predecir el caudal cerca de los extremos sin nuevas pruebas.

### Retardo y filtrado

El retardo incluye la dinámica real de la bomba, el transporte hidráulico, la ventana de conteo de pulsos y el filtrado digital. Por ello debe conservarse el mismo período de muestreo al validar el modelo.

### Repetibilidad

R1 y R2 deben compararse usando los mismos intervalos, el mismo punto inicial y la misma apertura de V-102. Cualquier intervención manual debe registrarse.

## Uso del ESP32 como plataforma de adquisición y supervisión

En este proyecto el ESP32 integra:

- generación de la señal PWM;
- adquisición de pulsos del ZJ-S201;
- lectura auxiliar del HC-SR04;
- filtrado y cálculo de variables físicas;
- ejecución automática de etapas;
- protección por altura o pérdida de flujo;
- anotación de eventos;
- exportación serial para MATLAB, Python o una hoja de cálculo.

No se implementa la interfaz web del repositorio de referencia. La visualización se realiza mediante el monitor serial y las gráficas generadas posteriormente, lo cual conserva la lógica central de identificación experimental solicitada.

## Relación con el control en cascada y Aspen HYSYS

El modelo de caudal representa la planta rápida del lazo interno. En el control en cascada se empleó inicialmente un PI con:

```text
Kp,Q = 8,50
Ki,Q = 4,25 s⁻¹
```

En Aspen HYSYS, la bomba se representa mediante un bloque `Pump`, las líneas mediante `Pipe Segment`, V-102 mediante `Valve` y el caudal mediante la medición de la corriente líquida. El PWM no se introduce como una señal eléctrica directa; se representa por una variación equivalente de velocidad o capacidad de la bomba.

## Evidencias que deben incorporarse

- Fotografía general del montaje de caudal.
- Detalle de la JT-500, el ZJ-S201 y V-102.
- Evidencia de V-102 en 15/16.
- Captura del monitor serial de R1 y R2.
- Gráfica PWM y caudal contra tiempo.
- Comparación entre datos experimentales y modelo FOPDT.
- Tabla con los parámetros estimados.

## Reproducibilidad desde cero

1. Descargar o clonar el repositorio.
2. Abrir el firmware en Arduino IDE.
3. Seleccionar `ESP32 Dev Module` y el puerto correspondiente.
4. Verificar las constantes de calibración y los pines.
5. Compilar y cargar el firmware.
6. Preparar hidráulicamente el sistema.
7. Colocar V-102 en 15/16.
8. Ejecutar la secuencia R1 y guardar el registro completo.
9. Retornar el sistema a una condición segura.
10. Repetir el procedimiento para R2.
11. Procesar los datos sin modificar los archivos originales.
12. Estimar el modelo y comparar la respuesta calculada con ambas corridas.

## Referencia metodológica

- [ESP32 TT Motor Characterizer](https://github.com/garciamsu/esp32-tt-motor-characterizer)

La referencia se utiliza como guía de organización y metodología. Los componentes, variables, condiciones experimentales y modelos de este README corresponden al prototipo hidráulico del proyecto integrador.

## Autor

**Julio Enrique Delgado**  
Ingeniería Electrónica — Sistemas de Control II  
Universidad Nacional Experimental del Táchira
