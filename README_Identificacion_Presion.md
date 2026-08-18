# Identificación experimental del subproceso de presión

> **Ubicación sugerida en GitHub:** `presion/README.md`

## Objetivo del subproceso

Este subproyecto caracteriza experimentalmente la dinámica de presión manométrica de la línea hidráulica. La presión se obtiene a partir de la altura de agua en una columna piezométrica abierta, medida con un sensor ultrasónico HC-SR04.

Durante esta identificación, el PI interno de caudal permanece activo y el controlador externo de presión se mantiene desactivado. La entrada experimental es la referencia de caudal y la salida es la presión filtrada. Esta separación permite obtener el modelo de la planta lenta que posteriormente utiliza el lazo externo del control en cascada.

## Relación con la metodología de referencia

La metodología PWM–RPM del repositorio del profesor se adapta a una relación referencia de caudal–presión:

| Elemento metodológico | Aplicación en este proyecto |
|---|---|
| Proceso | Línea de descarga y columna piezométrica |
| Variable manipulada | Referencia del lazo interno de caudal |
| Variable medida | Altura de agua y presión manométrica |
| Sensor | HC-SR04 instalado sobre la columna |
| Actuador | JT-500 mediante el PI interno de caudal |
| Prueba experimental | Escalón de 2,8 a 3,1 L/min |
| Punto de operación | V-102 en 15/16; presión cercana a 4,3 kPa |
| Perturbación posterior | Cambio manual de la apertura de V-102 |
| Datos exportados | Registros CSV copiados desde el monitor serial |
| Modelo aproximado | Primer orden con retardo, FOPDT |

## Descripción funcional

```mermaid
flowchart LR
    A["Referencia de caudal"] --> B["PI interno"]
    B --> C["Bomba y línea"]
    C --> D["Columna piezométrica"]
    D --> E["HC-SR04"]
    E --> F["Altura y presión"]
```

La columna está conectada mediante una derivación en T a la descarga y permanece abierta a la atmósfera. Por esta razón, la altura de agua representa la presión manométrica del punto de conexión mediante la relación hidrostática.

## Variables del ensayo

| Tipo | Variable |
|---|---|
| Variable manipulada | Referencia de caudal `Qsp`, en L/min |
| Variable intermedia | Caudal regulado por el PI interno, en L/min |
| Variable medida | Distancia ultrasónica, en cm |
| Variable calculada | Altura de agua `h`, en cm |
| Variable de salida | Presión manométrica `P`, en kPa |
| Sensor principal | HC-SR04 |
| Actuador | Bomba JT-500 mediante L298N |
| Condición fija | V-102 en 15/16 durante la identificación |
| Perturbación del proceso | Apertura manual de V-102 durante la validación del control |

## Componentes utilizados

- ESP32 Dev Module.
- Bomba JT-500 de 12 VCC.
- Driver L298N.
- Caudalímetro ZJ-S201 para cerrar el lazo interno.
- Sensor ultrasónico HC-SR04.
- Columna piezométrica de aproximadamente 90 cm de altura y 4 pulgadas de diámetro.
- Rebose ubicado aproximadamente a 80 cm.
- Válvula manual V-102.
- Electroválvula auxiliar para operación y alivio.
- Depósito TK-101.
- Fuente CA/CC de 12 V y 3 A.
- Mangueras y tuberías de 1/2 pulgada.

## Advertencias eléctricas e hidráulicas

- No alimentar la bomba desde el ESP32.
- Alimentar el L298N y la bomba desde la fuente de 12 V.
- Mantener tierra común entre ESP32, L298N y sensores.
- Adaptar `ECHO` del HC-SR04 antes de conectarlo al GPIO 19 del ESP32.
- Mantener la señal del ZJ-S201 acondicionada para 3,3 V.
- Instalar el HC-SR04 perpendicular a la superficie del agua.
- Mantener la columna abierta a la atmósfera.
- No obstruir el rebose ni el retorno a TK-101.
- No iniciar la bomba con la línea seca o sin agua suficiente en TK-101.
- No mover V-102 durante la identificación de presión.
- Detener la prueba si la altura alcanza el límite de seguridad o si se pierde la medición ultrasónica.
- La parada automática no sustituye la desconexión física de emergencia.

## Tabla de conexiones

| Dispositivo | Terminal | Conexión |
|---|---|---|
| HC-SR04 | `TRIG` | GPIO 18 |
| HC-SR04 | `ECHO` | GPIO 19 mediante adaptación a 3,3 V |
| ZJ-S201 | Señal de pulsos | GPIO 33 mediante acondicionamiento |
| L298N, bomba | `ENA` | GPIO 25 |
| L298N, bomba | `IN1` | GPIO 26 |
| L298N, bomba | `IN2` | GPIO 27 |
| L298N, electroválvula | `ENB` | GPIO 32 |
| L298N, electroválvula | `IN3` | GPIO 14 |
| L298N, electroválvula | `IN4` | GPIO 13 |
| Fuente | `+12 V` | Entrada de potencia del L298N |
| Fuente y ESP32 | `GND` | Tierra común |

## Calibración del HC-SR04

La distancia `d` medida desde la parte superior se convierte en altura mediante:

```text
h = 106,442 - 1,068448 · d
```

donde `d` y `h` se expresan en centímetros.

La presión se calcula con:

```text
P = 0,0980665 · h
```

donde `P` se expresa en kPa.

La conversión representa la relación hidrostática `P = ρgh` para agua. La calibración geométrica debe revisarse si se cambia la posición del sensor, la altura de la columna o el punto de referencia.

## Tratamiento de la señal ultrasónica

El firmware utiliza:

- 7 lecturas ultrasónicas por ciclo;
- mediana espacial para rechazar ecos atípicos;
- ventana temporal de 5 valores;
- confirmación de cambios coherentes;
- filtro adicional de altura;
- indicador de calidad de la medición.

El período de registro es aproximadamente **0,5 s**. Debe mantenerse igual al comparar los datos experimentales con el modelo.

## Control interno de caudal usado durante la prueba

El PI interno permanece activo con los parámetros:

```text
Kp,Q = 8,50
Ki,Q = 4,25 s⁻¹
```

La bomba trabaja con PWM de 8 bits a 5 kHz. El firmware limita la actuación al intervalo definido para operación segura y aplica restricciones al cambio máximo de PWM por muestra.

## Archivos del subproceso

```text
presion/
├── README.md
├── firmware/
│   └── 09_Identificacion_Lazo_Abierto_Presion.ino
├── data/
│   ├── Identificacion_Lazo_Abierto_Presion_R1.txt
│   └── Identificacion_Lazo_Abierto_Presion_R2.txt
├── images/
│   ├── montaje_columna.jpg
│   ├── respuesta_escalon_presion.png
│   └── comparacion_R1_R2_presion.png
└── analysis/
    └── analisis_presion.m
```

## Configuración principal del firmware

| Parámetro | Valor |
|---|---:|
| Referencia nominal | 2,80 L/min |
| Referencia del escalón | 3,10 L/min |
| Preapertura | 1 s |
| Acondicionamiento | 180 s |
| Base inicial | 30 s |
| Escalón de presión | 180 s |
| Retorno nominal | 180 s |
| Drenaje | 15 s |
| Duración aproximada total | 9 min 46 s |
| Altura permitida para iniciar | 28 a 33 cm |
| Altura de advertencia | 55 cm |
| Altura de apagado | 60 cm |

Los valores corresponden al firmware definitivo de identificación de presión. Si se utiliza una revisión anterior, se deben registrar sus tiempos y límites en la metadata de la corrida.

## Preparación del ensayo

1. Revisar fuente, conexiones, sensores y continuidad de tierra.
2. Llenar TK-101 y purgar el aire de las líneas.
3. Confirmar el retorno separado y el rebose libre.
4. Colocar V-102 exactamente en **15/16 de apertura**.
5. Abrir la trayectoria de flujo y ajustar la altura inicial entre 28 y 33 cm.
6. Confirmar caudal aproximadamente cero antes del inicio automático.
7. Abrir el monitor serial a 115200 baudios.
8. Rearmar el sistema y reiniciar datos.

## Comandos seriales

| Comando | Función |
|---|---|
| `R` | Rearmar el sistema si las condiciones son seguras |
| `C` | Reiniciar control, filtros y volúmenes |
| `A` | Abrir la electroválvula auxiliar durante la espera |
| `F` | Cerrar la electroválvula auxiliar durante la espera |
| `E` | Iniciar la secuencia automática |
| `P` | Parada manual con alivio temporal |

Secuencia normal:

```text
R
C
E
```

## Secuencia de identificación

1. El firmware abre la trayectoria durante la preapertura.
2. Activa el PI interno y acondiciona el sistema a 2,8 L/min.
3. Registra una base inicial estable.
4. Aplica el escalón de referencia de 2,8 a 3,1 L/min.
5. Mantiene el escalón hasta observar la dinámica lenta de presión.
6. Retorna la referencia a 2,8 L/min.
7. Mantiene el retorno para evaluar recuperación e histéresis dinámica.
8. Apaga la bomba, alivia la línea y cierra la válvula auxiliar.

Marcas esperadas en el registro:

```text
# INICIO_IDENTIFICACION_ABIERTA_PRESION
# V102_DEBE_ESTAR_FIJA_EN_15_16
# PI_INTERNO_CAUDAL_ACTIVO
# CONTROL_EXTERNO_PRESION_DESACTIVADO
# ACONDICIONAMIENTO: SP_CAUDAL=2.800_Lmin
# BASE_INICIAL_PRESION: SP_CAUDAL=2.800_Lmin
# ESCALON_PRESION: SP_CAUDAL_2.800_A_3.100_Lmin
# RETORNO_NOMINAL: SP_CAUDAL_3.100_A_2.800_Lmin
# FIN_IDENTIFICACION_PRESION: BOMBA_APAGADA_Y_VALVULA_CERRADA
```

## Datos registrados

La cabecera del archivo contiene:

```text
tiempo_s,ensayo_s,fase,referencia_caudal_Lmin,valvula_abierta,
pwm,pulsos,frecuencia_Hz,caudal_Lmin,caudal_filtrado_Lmin,
error_caudal_Lmin,integral_PWM,volumen_impulsado_L,
volumen_retorno_L,distancia_raw_cm,altura_raw_cm,
altura_mediana_cm,altura_filtrada_cm,presion_raw_kPa,
presion_filtrada_kPa,calidad_HC,estado_valvula,estado
```

Las líneas que comienzan con `#` identifican las etapas y deben conservarse con los datos originales.

## Resultados experimentales resumidos

Promedios de fase obtenidos de las corridas disponibles:

| Corrida | Base inicial | Escalón a 3,1 L/min | Retorno a 2,8 L/min |
|---|---:|---:|---:|
| R1 | 4,050 kPa | 4,475 kPa | 4,397 kPa |
| R2 | 4,270 kPa | 4,467 kPa | 4,350 kPa |

En R2, la base inicial presentó aproximadamente **43,55 cm** y **4,27 kPa**. El caudal filtrado medio fue 2,797 L/min en la base y 3,099 L/min durante el escalón, lo que confirma que el PI interno siguió adecuadamente las referencias.

La diferencia entre R1 y R2 se debe considerar al estimar incertidumbre, deriva del nivel inicial y repetibilidad. Para el modelo consolidado se debe emplear el tramo con acondicionamiento suficiente y condiciones iniciales mejor controladas.

## Modelo dinámico identificado

El modelo consolidado es:

```text
              0,679 · e^(-5,5s)
GP(s) = -----------------------------
                  35,5s + 1
```

con:

```text
GP(s) = ΔP(s) / ΔQsp(s)
```

| Parámetro | Valor | Unidad |
|---|---:|---|
| Ganancia `KP` | 0,679 | kPa/(L/min) |
| Constante de tiempo `τP` | 35,5 | s |
| Retardo `LP` | 5,5 | s |

La dinámica de presión es considerablemente más lenta que la del caudal, lo que justifica utilizar el caudal como lazo interno y la presión como lazo externo.

## Interpretación de resultados

### Retardo

El retardo incluye el transporte hidráulico, la acumulación en la columna, el filtrado ultrasónico y la respuesta del lazo interno de caudal.

### No linealidad

La relación caudal–presión depende de la apertura de V-102 y de las pérdidas de carga. El modelo es local y no representa todas las aperturas ni todo el rango de la bomba.

### Histéresis y recuperación

La presión durante el retorno no coincide inmediatamente con la base inicial porque la columna almacena volumen y presenta una dinámica lenta. La comparación debe realizarse una vez transcurrido el tiempo suficiente de asentamiento.

### Saturación y seguridad

La altura física de la columna, el rebose, los límites del PWM y los umbrales de apagado restringen el rango de operación. El modelo no debe utilizarse para ordenar condiciones superiores a los límites seguros.

### Ruido y ecos atípicos

El HC-SR04 puede producir ecos falsos por inclinación, espuma, salpicaduras o paredes de la columna. La mediana, la ventana temporal y la confirmación de cambios reducen estas alteraciones, pero no sustituyen una instalación perpendicular y estable.

## Perturbación manual mediante V-102

La electroválvula auxiliar no produjo una variación de presión suficiente para validar claramente el rechazo de perturbaciones. Por autorización del profesor, la perturbación física se aplica modificando manualmente V-102 y retornándola después a su marca nominal de 15/16.

Este cambio no forma parte de la identificación abierta de presión. Se utiliza posteriormente para validar el control en cascada. El instante de apertura y el instante de retorno deben registrarse para compararlos con la simulación.

## Uso del ESP32 como plataforma de adquisición y supervisión

El ESP32 cumple las funciones de:

- generar el PWM para la bomba;
- ejecutar el PI interno de caudal;
- adquirir los pulsos del ZJ-S201;
- medir la distancia con el HC-SR04;
- calcular altura y presión;
- filtrar señales;
- ejecutar la secuencia automática;
- supervisar límites de altura y pérdida de flujo;
- anotar eventos y fases;
- exportar los registros por Serial.

La visualización y el análisis se realizan mediante el monitor serial, MATLAB, Simulink o una hoja de cálculo. La interfaz web del ejemplo del profesor no es obligatoria para conservar la metodología de identificación.

## Relación con el control en cascada

El modelo de presión se utiliza como planta del lazo externo. Los parámetros iniciales del PI externo son:

```text
Kp,P = 2,05
Ki,P = 0,0577 s⁻¹
```

La salida del PI externo es una referencia para el lazo interno de caudal. El lazo interno actúa sobre la bomba y debe responder más rápido que el lazo externo.

## Relación con Aspen HYSYS

| Elemento físico | Elemento equivalente en HYSYS |
|---|---|
| JT-500 | `Pump` |
| Mangueras y tuberías | `Pipe Segment` |
| V-102 | `Valve` |
| ZJ-S201 | Medición de flujo |
| Columna + HC-SR04 | Medición de presión o columna equivalente |
| TK-101 | `Tank` o reservorio |

Los parámetros experimentales se comparan con las ganancias y tiempos de respuesta del modelo de HYSYS. El software no reproduce directamente los pulsos ni el PWM; utiliza magnitudes equivalentes como capacidad de la bomba, caudal, apertura de válvula y presión.

## Evidencias que deben incorporarse

- Fotografía de la columna completa.
- Detalle del HC-SR04 perpendicular al agua.
- Evidencia de la derivación en T y del rebose.
- Evidencia de V-102 en 15/16.
- Capturas del monitor serial de R1 y R2.
- Gráfica de referencia y caudal medido.
- Gráfica de altura y presión contra tiempo.
- Comparación entre datos experimentales y modelo FOPDT.
- Captura del equivalente desarrollado en Aspen HYSYS.

## Reproducibilidad desde cero

1. Descargar o clonar el repositorio.
2. Abrir `09_Identificacion_Lazo_Abierto_Presion.ino` en Arduino IDE.
3. Seleccionar `ESP32 Dev Module` y el puerto correspondiente.
4. Revisar pines, calibraciones, tiempos y límites de seguridad.
5. Compilar y cargar el firmware.
6. Preparar el circuito hidráulico y colocar V-102 en 15/16.
7. Ajustar la altura inicial dentro del intervalo permitido.
8. Ejecutar la corrida R1 y guardar el registro completo.
9. Drenar o acondicionar el sistema hasta reproducir la condición inicial.
10. Ejecutar R2 sin cambiar el montaje.
11. Conservar los TXT originales y trabajar sobre copias.
12. Generar las gráficas, estimar el modelo y comparar las repeticiones.

## Referencia metodológica

- [ESP32 TT Motor Characterizer](https://github.com/garciamsu/esp32-tt-motor-characterizer)

La referencia se utiliza como guía metodológica y documental. Este README describe exclusivamente el subproceso hidráulico de presión del proyecto integrador.

## Autor

**Julio Enrique Delgado**  
Ingeniería Electrónica — Sistemas de Control II  
Universidad Nacional Experimental del Táchira
