# Betaflight - Versión Custom para UAV FPV Militar

![Betaflight](https://raw.githubusercontent.com/betaflight/.github/main/profile/images/bf_logo.svg#gh-light-mode-only)

[![Licencia: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Betaflight](https://img.shields.io/badge/Firmware-Betaflight%204.4.2%20Custom-blue)](https://github.com/betaflight/betaflight)

**Autor:** Luis Olmos ([@Luchinol](https://github.com/Luchinol))

---

## 📋 Acerca de Este Repositorio

Este repositorio contiene una versión modificada de **Betaflight 4.4.2** desarrollada como parte del proyecto de diseño preliminar de un **UAV FPV Militar de bajo costo** para la Fuerza Aérea de Chile (FACh).

### Estructura del Repositorio

```
Betaflight_Github/
├── README.md                          # Este archivo
├── MEMORIA LUIS OLMOS.pdf             # Documento técnico completo
├── MEMORIA LUIS OLMOS.docx            # Versión editable de la memoria
└── betaflight/                        # Firmware Betaflight 4.4.2 Custom
    ├── README.md                      # Documentación de Betaflight (español)
    └── src/                           # Código fuente con modificaciones
```

### Sobre Betaflight

Betaflight es un software de controlador de vuelo (firmware) de código abierto utilizado para volar aeronaves multi-rotor y de ala fija. Se enfoca en el rendimiento de vuelo, la adición de características de vanguardia y un amplio soporte de targets.

**Documentación completa de Betaflight:** Ver [betaflight/README.md](betaflight/README.md)

**Sitio oficial:** https://betaflight.com
**Documentación:** https://betaflight.com/docs/wiki
**Discord:** https://discord.gg/n4E6ak4u3c

---

## ⚠️ Modificaciones Custom - Proyecto UAV FPV Militar FACh

### Contexto del Proyecto

**Institución:** Academia Politécnica Militar del Ejército de Chile (ACAPOMIL)
**Cliente:** Fuerza Aérea de Chile - Comandos de Aviación e Infantería de Aviación
**Objetivo:** Diseño preliminar de un UAV FPV de bajo costo basado en componentes COTS

### Modificaciones Implementadas

Esta versión de Betaflight incluye las siguientes mejoras custom para aplicaciones militares:

#### 1. GPS Rescue Enhanced con Dead Reckoning
**Archivos modificados:**
- [`src/main/flight/gps_rescue_multirotor.c`](betaflight/src/main/flight/gps_rescue_multirotor.c) (43 KB)
- [`src/main/flight/gps_rescue_multirotor.h`](betaflight/src/main/flight/gps_rescue_multirotor.h)
- [`src/main/io/gps.c`](betaflight/src/main/io/gps.c)

**Mejoras:**
- Navegación con proyección vectorial y horizonte virtual extendido (1.5s lookahead)
- Dead reckoning (navegación inercial) durante pérdida GPS transitoria (<30s)
- Compensación activa de viento mediante feedforward
- Precisión de aterrizaje mejorada: ±2m (vs ±5m estándar Betaflight)
- Aterrizaje en tres fases con ajuste fino de posición

**Resultados validados:**
- Tasa de éxito: 100% en 45+ activaciones
- Funcionamiento validado hasta 4.680 msnm a -15°C
- Dead reckoning activado exitosamente en 3 ocasiones

#### 2. Position Hold 6DOF Mejorado
**Archivos modificados:**
- [`src/main/flight/pos_hold_multirotor.c`](betaflight/src/main/flight/pos_hold_multirotor.c)
- [`src/main/flight/position.c`](betaflight/src/main/flight/position.c)

**Mejoras:**
- Control cascada: Posición (50Hz) → Velocidad (100Hz) → Actitud (8kHz)
- Parámetros PID optimizados empíricamente
- Compensación activa de viento

**Resultados validados:**
- Deriva horizontal promedio: ±1,8m (viento <10 km/h)
- Deriva horizontal máxima: ±3,2m (viento 25 km/h ráfagas)
- Deriva altitud: ±0,5m

#### 3. Filtrado Digital Avanzado
**Archivos modificados:**
- [`src/main/flight/dyn_notch_filter.c`](betaflight/src/main/flight/dyn_notch_filter.c)
- [`src/main/flight/pid.c`](betaflight/src/main/flight/pid.c)

**Mejoras:**
- Cadena de filtros optimizada para motores XING2 2207 1855KV
- Dynamic Notch Filters tracking 150-300 Hz automáticamente
- Lowpass Butterworth 2º orden con cutoffs adaptativos por perfil
- D-term filtering agresivo para reducir noise amplification

**Resultados validados:**
- Latencia total agregada: <50 μs (medido con analizador lógico)
- CPU utilization hover: 45-55% (STM32F405 @ 168MHz)
- CPU utilization acro: 70-80%

#### 4. Failsafe Multinivel
**Archivos modificados:**
- [`src/main/flight/failsafe.c`](betaflight/src/main/flight/failsafe.c)
- [`src/main/fc/rc_modes.c`](betaflight/src/main/fc/rc_modes.c)

**Mejoras:**
- Sistema de seguridad en cascada con 4 niveles
- Rollback automático si señal se recupera <2s después de activación
- Integración con GPS Rescue Enhanced y dead reckoning
- Detección térmica ESC (>100°C) y batería crítica (<3.3V/celda)

---

## 🛠️ Arquitectura del Firmware Custom

### Flujo de Control Modificado

```
┌──────────────────────────────────────────────────────────────┐
│                      HARDWARE LAYER                          │
│  ICM-42688P │ u-blox M10 │ ESC 4-in-1 │ ELRS 900MHz │ W25Q  │
└──────┬─────────┬───────────┬──────────┬───────────┬─────────┘
       │SPI 8kHz │UART 25Hz  │DShot 8kHz│UART 500Hz │SPI      │
┌──────▼─────────▼───────────▼──────────▼───────────▼─────────┐
│                     SENSOR LAYER                             │
│  gyro.c │ gps.c (⚠️CUSTOM) │ battery.c │ esc_sensor.c       │
└──────┬─────────┬───────────┬──────────────────────┬─────────┘
       │         │           │                      │
┌──────▼─────────▼───────────▼──────────────────────▼─────────┐
│                  ESTIMATION LAYER                            │
│  imu.c - Sensor Fusion                                      │
│  position.c - GPS + Dead Reckoning (⚠️CUSTOM)               │
└──────┬──────────────────────────────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────────────┐
│                   CONTROL LAYER                              │
│  pid.c (⚠️CUSTOM) │ pos_hold_*.c (⚠️CUSTOM) │ gps_rescue  │
│  8 kHz            │ 50 Hz outer              │ (⚠️CUSTOM)  │
└─────────┼────────────────┼───────────────┼─────────────────┘
          │                │               │
┌─────────▼────────────────▼───────────────▼─────────────────┐
│  mixer.c - Motor mixing (quad-X)                            │
│  Output: 4x motor DShot600 @ 8kHz                           │
└─────────────────────────────────────────────────────────────┘
```

### Task Scheduler

```c
TASK(PID_LOOP)          // 8000 Hz - Control PID + motor output
TASK(GYRO)              // 8000 Hz - Gyro sampling (SPI + DMA)
TASK(GPS)               // 25 Hz   - GPS processing + navigation (⚠️CUSTOM)
TASK(FAILSAFE)          // 100 Hz  - Failsafe multinivel (⚠️CUSTOM)
TASK(BATTERY_VOLTAGE)   // 100 Hz  - ADC voltage monitoring
TASK(RC)                // 500 Hz  - RC input (CRSF/ELRS)
TASK(OSD)               // 50 Hz   - OSD rendering
TASK(BLACKBOX)          // 2000 Hz - Flight recorder
```

---

## 📊 Especificaciones del Sistema UAV

### Hardware (Target: SpeedyBee F405 V4)

| Componente | Modelo/Especificación | Función |
|------------|----------------------|---------|
| **FC + ESC** | SpeedyBee F405 V4 | STM32F405, ESC 4-in-1 50A integrado |
| **Motores** | IFlight XING2 2207 1855KV | 1200g empuje/motor @ 6S |
| **Hélices** | Gemfan 5149 tri-blade | Balance thrust/efficiency |
| **GPS** | u-blox M10 Dual-Band | L1+L5, precisión <1m, 25Hz update |
| **RC RX** | ExpressLRS 900MHz 2W | Alcance >10km, latencia <10ms |
| **VTX** | Rush Tank 5.8GHz | 25-1600mW ajustable |
| **Cámara** | Caddx Ratel 2 | 1200TVL, FOV 160° |
| **Batería** | 6S1P Li-ion 4200-8000mAh | 93-177 Wh |

### Especificaciones de Rendimiento

| Parámetro | Valor Validado | Margen vs Requerido |
|-----------|----------------|---------------------|
| **Masa total** | 768,64 g | +74,9% (+1,231 kg capacidad) |
| **Relación T/W** | 8,77:1 | +150,6% (vs 3,5:1 mín) |
| **Velocidad máxima** | 108 km/h | +54,3% (vs 70 km/h mín) |
| **Autonomía base** | 18,5 min | -7,5% (marginal) |
| **Autonomía extendida** | 28,8 min | +44% |
| **Alcance control ELRS** | >10 km | +233% (vs 3 km mín) |
| **Alcance video** | 4,1 km @ 1600mW | +105% (vs 2 km mín) |
| **Latencia RC** | 15-25 ms | +50-67% (vs 50 ms máx) |
| **GPS Rescue precisión** | ±2,0 m promedio | +33% (vs ±3 m máx) |
| **Position Hold deriva** | ±1,8 m promedio | +40% (vs ±3 m máx) |

### Validación en Condiciones Extremas

**Prueba Crítica: Calama, Chile**
- ✅ Altitud: 4.680 msnm
- ✅ Temperatura: -15°C
- ✅ Presión atmosférica: ~570 hPa (57% nivel del mar)
- ✅ GPS Rescue: 3/3 activaciones exitosas
- ✅ Operación nominal mantenida

---

## 🔧 Compilación y Flasheo

### Prerrequisitos

```bash
# Toolchain ARM GCC
sudo apt install gcc-arm-none-eabi

# Build tools
sudo apt install make git
```

### Compilar

```bash
# Clonar este repositorio
git clone https://github.com/Luchinol/betaflight.git
cd betaflight

# Compilar para SpeedyBee F405 V4
make TARGET=SPEEDYBEEF405V4

# Output generado en:
# obj/SPEEDYBEEF405V4.hex  (para flasheo)
# obj/SPEEDYBEEF405V4.bin
# obj/SPEEDYBEEF405V4.elf  (símbolos debug)
```

### Flashear

**Método 1: Betaflight Configurator (Recomendado)**
1. Conectar FC vía USB
2. Firmware Flasher → Load Firmware [Local]
3. Seleccionar `obj/SPEEDYBEEF405V4.hex`
4. ☑️ Full Chip Erase
5. Flash Firmware

**Método 2: DFU (Línea de comandos)**
```bash
make TARGET=SPEEDYBEEF405V4 flash
```

---

## ⚙️ Configuración Recomendada

### Perfiles PID

```c
// Perfil 1: Vigilancia (Estable, video limpio)
profile 0
set p_pitch = 40
set i_pitch = 60
set d_pitch = 35
set f_pitch = 80

// Perfil 2: Persecución (Agresivo, maniobras)
profile 1
set p_pitch = 65
set i_pitch = 90
set d_pitch = 55
set f_pitch = 120

// Perfil 3: GPS Hold (Balanceado, loiter)
profile 2
set p_pitch = 50
set i_pitch = 75
set d_pitch = 42
set f_pitch = 95
```

### GPS Rescue Custom

```c
set gps_rescue_angle = 30                  // Ángulo inclinación máx (grados)
set gps_rescue_initial_alt = 30            // Altitud RTH (metros)
set gps_rescue_descent_dist = 50           // Inicio descenso (metros)
set gps_rescue_ground_speed = 1500         // Velocidad crucero (cm/s)
set gps_rescue_min_sats = 5                // Mín sats (< activa dead reckoning)

// PID throttle (control altitud)
set gps_rescue_throttle_p = 150
set gps_rescue_throttle_i = 20
set gps_rescue_throttle_d = 50

// PID velocidad (control navegación)
set gps_rescue_velocity_p = 80
set gps_rescue_velocity_i = 40
set gps_rescue_velocity_d = 15
```

Para configuración CLI completa, ver **Anexo B** en la memoria técnica.

---

## 📖 Documentación Completa

### Memoria Técnica del Proyecto

**Documento principal:** [MEMORIA LUIS OLMOS.pdf](MEMORIA%20LUIS%20OLMOS.pdf)

**Contenidos:**
1. Contexto y Justificación
2. Objetivos del Proyecto
3. Metodología de Ingeniería de Sistemas (Modelo V INCOSE)
4. Arquitectura del Sistema (Matriz N², FAST diagrams)
5. Especificaciones Técnicas (RTM completa: 31 RO, 37 RT)
6. **Firmware y Diseño Lógico** ← Detalles técnicos de las modificaciones
7. Resultados de Validación Experimental (45+ vuelos)
8. Conclusiones y Recomendaciones

### Archivos Clave del Código

**Navegación Autónoma:**
- [`gps_rescue_multirotor.c`](betaflight/src/main/flight/gps_rescue_multirotor.c) - GPS Rescue Enhanced (43 KB)
- [`pos_hold_multirotor.c`](betaflight/src/main/flight/pos_hold_multirotor.c) - Position Hold 6DOF
- [`position.c`](betaflight/src/main/flight/position.c) - Estimación posición

**Control de Vuelo:**
- [`pid.c`](betaflight/src/main/flight/pid.c) - PID controller (67 KB, 8kHz)
- [`imu.c`](betaflight/src/main/flight/imu.c) - Sensor fusion
- [`failsafe.c`](betaflight/src/main/flight/failsafe.c) - Failsafe multinivel

**Sensores:**
- [`gps.c`](betaflight/src/main/io/gps.c) - Driver GPS u-blox M10
- [`gyro.c`](betaflight/src/main/sensors/gyro.c) - Giroscopio (8kHz)
- [`battery.c`](betaflight/src/main/sensors/battery.c) - Monitoreo batería

**Target:**
- [`target.h`](betaflight/src/main/target/SPEEDYBEEF405V4/target.h) - Pin definitions
- [`Makefile`](betaflight/Makefile) - Build system

---

## 🎓 Metodología de Ingeniería de Sistemas

### Modelo V INCOSE Aplicado

```
DESCOMPOSICIÓN                    INTEGRACIÓN Y VERIFICACIÓN
┌──────────────────┐             ┌──────────────────────┐
│ Necesidades      │◄────────────┤ Validación           │
│ Stakeholders     │             │ Operacional (45+ vls)│
└────────┬─────────┘             └──────────▲───────────┘
         │                                  │
┌────────▼─────────┐             ┌──────────┴───────────┐
│ 31 Requerimientos│◄────────────┤ Verificación         │
│ Operacionales    │             │ Sistema (RTM 100%)   │
└────────┬─────────┘             └──────────▲───────────┘
         │                                  │
┌────────▼─────────┐             ┌──────────┴───────────┐
│ 37 Requerimientos│◄────────────┤ Integración          │
│ Técnicos         │             │ Subsistemas          │
└────────┬─────────┘             └──────────▲───────────┘
         │                                  │
┌────────▼─────────┐             ┌──────────┴───────────┐
│ Diseño Físico    │◄────────────┤ Verificación         │
│ + Firmware Custom│             │ Componentes COTS     │
└──────────────────┘             └──────────────────────┘
```

### Herramientas Aplicadas

- **QFD (Quality Function Deployment):** Transformación RO → RT
- **FAST (Function Analysis System Technique):** Descomposición funcional
- **Matriz N²:** Mapeo interfaces subsistemas
- **AHP (Proceso Analítico Jerárquico):** Selección componentes COTS
- **RTM (Requirement Traceability Matrix):** Trazabilidad bidireccional 100%

### Resultados

✅ **100% cumplimiento:** 31/31 RO + 37/37 RT satisfechos
✅ **45+ vuelos** de validación documentados
✅ **Operación validada:** 4.680 msnm, -15°C
✅ **Reducción costos:** 93,6-99,4% vs UAV militares tradicionales

---

## 📄 Licencia

Este proyecto se distribuye bajo **GNU General Public License v3.0 (GPL-3.0)**, la misma licencia que Betaflight.

Cualquier modificación o distribución debe cumplir con los términos de esta licencia.

### Reconocimientos

- **Betaflight Development Team:** Por el firmware de código abierto base
- **ExpressLRS Team:** Por el protocolo de comunicaciones de largo alcance
- **Comunidad FPV:** Por el desarrollo continuo de componentes COTS
- **FACh - Comandos de Aviación:** Por la colaboración y feedback operacional
- **ACAPOMIL:** Por el apoyo institucional al proyecto

---

## 📧 Contacto

**Autor:** Luis Olmos
**GitHub:** [@Luchinol](https://github.com/Luchinol)
**Repositorio:** https://github.com/Luchinol/betaflight

---

**Última actualización:** Noviembre 2025
**Estado:** Diseño Preliminar Completado - Validación Experimental Exitosa
**Próximos pasos:** Integración operacional con Comandos de Aviación FACh
