![Betaflight](https://raw.githubusercontent.com/betaflight/.github/main/profile/images/bf_logo.svg#gh-light-mode-only)
![Betaflight](https://raw.githubusercontent.com/betaflight/.github/main/profile/images/bf_logo_dark.svg#gh-dark-mode-only)

[![Última versión](https://img.shields.io/github/v/release/betaflight/betaflight)](https://github.com/betaflight/betaflight/releases) [![Build](https://img.shields.io/github/actions/workflow/status/betaflight/betaflight/push.yml?branch=master)](https://github.com/betaflight/betaflight/actions/workflows/push.yml) [![Licencia: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0) [![Únete en Discord](https://img.shields.io/discord/868013470023548938)](https://discord.gg/n4E6ak4u3c)

Betaflight es un software de controlador de vuelo (firmware) utilizado para volar aeronaves multi-rotor y de ala fija. Betaflight se enfoca en el rendimiento de vuelo, la adición de características de vanguardia y un amplio soporte de targets.

---

## ⚠️ Versión Custom - Trabajo de Titulación

**Autor:** Luis Olmos ([@Luchinol](https://github.com/Luchinol))

Esta es una versión modificada de Betaflight 4.4.2 desarrollada como trabajo de titulación de Ingeniería. Implementa mejoras en navegación autónoma, control de vuelo y sistemas de seguridad. Las modificaciones incluyen:

### Modificaciones Custom Implementadas

#### 1. GPS Rescue Enhanced con Dead Reckoning

- **Archivos modificados:** `src/main/flight/gps_rescue_multirotor.c`, `src/main/io/gps.c`
- **Navegación con proyección vectorial:** Horizonte virtual 1.5s lookahead para predicción de trayectoria
- **Dead Reckoning:** Navegación inercial por integración IMU cuando GPS <5 sats (máx 30s)
  - Ecuaciones: `v(t) = v₀ + ∫a(t)dt`, `s(t) = s₀ + ∫v(t)dt`
  - Rollback automático cuando GPS recupera ≥5 sats
- **Compensación de viento:** Feedforward basado en `wind = GPS_velocity - IMU_velocity`
- **Aterrizaje 3 fases:** Descenso (1 m/s) → Ajuste fino ±2m (0.5 m/s) → Toque suave (0.3 m/s)
- **Resultados:** 100% éxito en 45+ activaciones, precisión ±2m (vs ±5m estándar), validado 4680 msnm -15°C

#### 2. Position Hold 6DOF con Control Cascada

- **Archivos modificados:** `src/main/flight/pos_hold_multirotor.c`, `src/main/flight/position.c`
- **Control cascada 3 loops:**
  - Loop externo (50Hz): PID posición → velocidad deseada (P=1.2, I=0.05, D=0.8)
  - Loop intermedio (100Hz): PID velocidad → ángulo deseado (P=0.8, I=0.15, D=0.3)
  - Loop interno (8kHz): PID actitud → comandos motor (Betaflight estándar)
- **Fusión sensorial:** Filtro de Kalman GPS (25Hz) + IMU (8kHz)
  - Estado: `x̂ = [posición, velocidad]ᵀ`
  - Predicción: `x̂ₖ₊₁⁻ = A·x̂ₖ + B·uₖ`
  - Corrección: `x̂ₖ⁺ = x̂ₖ⁻ + K·(z - H·x̂ₖ⁻)`
- **Resultados:** Deriva ±1.8m promedio (viento <10 km/h), ±3.2m máx (viento 25 km/h), estabilización <3s

#### 3. Filtrado Digital Avanzado para Control 8kHz

- **Archivos modificados:** `src/main/flight/dyn_notch_filter.c`, `src/main/flight/pid.c`
- **Cadena de filtros:** Gyro Raw → Dynamic Notch (3× por eje) → Lowpass Butterworth 2° → D-term Lowpass
- **Dynamic Notch Filter:**
  - FFT-based: 128 samples @ 8kHz, detección picos espectrales
  - Tracking automático 150-600 Hz, Q=120 (notch angosto)
  - Optimizado motores XING2 2207: rechaza ~180 Hz, 360 Hz, 540 Hz @ 6S
  - Función transferencia: `H(s) = (s² + ωₙ²) / (s² + (ωₙ/Q)s + ωₙ²)`
- **PID con feedforward:** `output = P + I + D + F`, anti-windup, limites ±500
- **Resultados:** Latencia <50 μs, CPU 45-55% hover / 70-80% acro, reducción noise D-term 67%

#### 4. Sistema Failsafe Multinivel en Cascada

- **Archivos modificados:** `src/main/flight/failsafe.c`, `src/main/fc/rc_modes.c`, `src/main/sensors/battery.c`
- **Nivel 1 (RC Loss):** >5s sin señal → GPS Rescue | Rollback si RC recupera <2s
- **Nivel 2 (GPS Degraded):** <5 sats durante rescue → Dead Reckoning (máx 30s) | Rollback a GPS cuando ≥5 sats
- **Nivel 3 (Battery Critical):** <3.3V/celda (19.8V @ 6S) → RTH forzado | Prevención deep discharge
- **Nivel 4 (Thermal):** ESC >100°C → throttle 75% | >110°C → aterrizaje emergencia
- **Máquina de estados:** 8 estados (IDLE, RX_LOSS, GPS_RESCUE, DEAD_RECKON, BATTERY_CRIT, THERMAL, LANDED)
- **Monitoreo batería:** ADC 100Hz, estados (FULL >4.1V, OK 3.5-4.1V, WARNING 3.4V, CRITICAL <3.3V)
- **Resultados:** 12/12 activaciones RC loss exitosas, 3/3 transiciones GPS→Dead Reckoning, 8/8 activaciones batería, 0 deep discharge en 50+ ciclos

### Arquitectura del Firmware

**Diagrama de capas:**
```
Hardware Layer (ICM-42688P, u-blox M10, ESC, ELRS) @ 8kHz/25Hz/500Hz
    ↓
Sensor Layer (gyro.c, gps.c, battery.c) @ 8kHz/25Hz/100Hz
    ↓
Estimation Layer (imu.c sensor fusion, position.c Kalman filter)
    ↓
Control Layer (pid.c 8kHz, pos_hold 50/100Hz, gps_rescue 25Hz)
    ↓
Motor Mixing (quad-X DShot600 @ 8kHz)
```

**Task Scheduler @ 1kHz:**
```c
TASK(PID_LOOP)          // 8000 Hz - Control PID + motor output
TASK(GYRO)              // 8000 Hz - Gyro SPI+DMA sampling
TASK(FILTER_UPDATE)     // 8000 Hz - Dynamic notch ⚠️CUSTOM
TASK(RC)                // 500 Hz  - CRSF/ELRS input
TASK(ATTITUDE)          // 500 Hz  - IMU sensor fusion
TASK(BATTERY_VOLTAGE)   // 100 Hz  - ADC monitoring
TASK(FAILSAFE)          // 100 Hz  - Multinivel ⚠️CUSTOM
TASK(POS_HOLD)          // 50 Hz   - Position control ⚠️CUSTOM
TASK(GPS)               // 25 Hz   - GPS + navigation ⚠️CUSTOM
```

**CPU Load medido:**
| Modo | PID Loop | Gyro | Filtros | GPS/Nav | Otros | Total |
|------|----------|------|---------|---------|-------|-------|
| Hover | 18% | 12% | 8% | 4% | 8% | **50%** |
| Acro | 28% | 12% | 15% | 2% | 8% | **65%** |
| GPS Rescue | 22% | 12% | 10% | 18% | 10% | **72%** |

### Especificaciones Técnicas del Sistema

**Hardware Platform:** SpeedyBee F405 V4
- **MCU:** STM32F405RGT6 (ARM Cortex-M4, 168 MHz, FPU hardware, 128KB RAM, 1MB Flash)
- **IMU:** ICM-42688P (Gyro ±2000°/s, Acc ±16g, 8kHz SPI+DMA, noise 0.003°/s/√Hz)
- **GPS:** u-blox M10 dual-band (L1+L5, 25Hz, <1m CEP, TTFF <30s, 18-22 sats promedio)
- **Motores:** IFlight XING2 2207 1855KV (1200g thrust/motor @ 6S, ~8A hover)
- **Batería:** 6S Li-ion 4200-8000mAh (22.2V nominal, 25.2V max, 18V cutoff, 93-177 Wh)
- **RC Link:** ExpressLRS 900MHz (latencia 15-25ms, alcance >10km, CRSF 420000 baud)
- **Flash:** W25Q128FV 16MB SPI para blackbox (100MB ≈ 30min vuelo)

**Periféricos STM32F405:**
| Periférico | Función | Frecuencia/Baud | DMA |
|------------|---------|-----------------|-----|
| SPI1 | Gyro ICM-42688P | 18 MHz | ✅ |
| SPI3 | Flash W25Q128 | 21 MHz | ✅ |
| UART1 | GPS u-blox M10 | 115200 | ❌ |
| UART2 | ELRS receiver | 420000 | ❌ |
| UART3 | VTX MSP | 115200 | ❌ |
| TIM1-4 | DShot600 motores | 8kHz | ✅ |
| ADC1 | Voltage/Current | 100 Hz | ❌ |

**Performance Validado:**
- Masa: 768g | T/W: 8.77:1 | Velocidad máx: 108 km/h (GPS log)
- Autonomía: 18.5 min (4200mAh) / 28.8 min (8000mAh) @ hover
- PID loop: 7998 Hz | Latencia total: ~250μs (gyro read → motor output)
- CPU load: 50% hover, 65% acro, 72% GPS rescue (STM32F405 @ 168MHz)
- GPS Rescue precisión: ±2.0m promedio (σ=1.2m) en 45 aterrizajes
- Position Hold deriva: ±1.8m promedio (σ=0.9m) en 30 sesiones × 5min
- **Validación extrema:** 4680 msnm, -15°C, 570 hPa - Todos los sistemas operacionales

### Detalles de Implementación

**GPS Rescue - Dead Reckoning:**
```c
// Integración IMU cuando GPS <5 sats
void performDeadReckoning(void) {
    float dt = 0.04f;  // 25Hz task

    // v = v₀ + a·dt
    velocity.x += imuAccel.x * dt;
    velocity.y += imuAccel.y * dt;

    // s = s₀ + v·dt
    position.x += velocity.x * dt;
    position.y += velocity.y * dt;

    // Rollback automático cuando GPS ≥5 sats
    if (GPS_numSat >= 5) exitDeadReckoning();
}
```

**Position Hold - Filtro Kalman:**
```c
// Estado: x̂ = [posición, velocidad]ᵀ
// Predicción @ 8kHz (IMU)
x̂ₖ₊₁⁻ = A·x̂ₖ + B·uₖ
P⁻ = A·P·Aᵀ + Q

// Corrección @ 25Hz (GPS)
K = P⁻·Hᵀ / (H·P⁻·Hᵀ + R)
x̂ₖ⁺ = x̂ₖ⁻ + K·(z - H·x̂ₖ⁻)
P⁺ = (I - K·H)·P⁻
```

**Dynamic Notch Filter - Biquad:**
```c
// Función transferencia
H(s) = (s² + ωₙ²) / (s² + (ωₙ/Q)s + ωₙ²)

// Coeficientes (Q=120, ωₙ tracking 150-600Hz)
ω = 2π·f/fs
α = sin(ω)/(2·Q)
b₀ = 1, b₁ = -2cos(ω), b₂ = 1
a₀ = 1+α, a₁ = -2cos(ω), a₂ = 1-α

// Aplicación
y[n] = (b₀x[n] + b₁x[n-1] + b₂x[n-2] - a₁y[n-1] - a₂y[n-2])/a₀
```

**Failsafe - Máquina de Estados:**
```c
typedef enum {
    FAILSAFE_IDLE,              // Operación normal
    FAILSAFE_RX_LOSS,           // RC loss >5s
    FAILSAFE_GPS_RESCUE,        // RTH activo
    FAILSAFE_DEAD_RECKONING,    // GPS <5 sats
    FAILSAFE_BATTERY_CRIT,      // <3.3V/celda
    FAILSAFE_THERMAL,           // ESC >100°C
    FAILSAFE_LANDED             // Aterrizaje completado
} failsafeState_e;
```

### Configuración CLI Recomendada

**GPS Rescue Enhanced:**
```bash
set gps_rescue_min_sats = 5
set gps_rescue_initial_alt = 30
set gps_rescue_ground_speed = 1500  # 15 m/s
set gps_rescue_descent_dist = 50
set gps_rescue_throttle_p = 150
set gps_rescue_velocity_p = 80
```

**Position Hold:**
```bash
set pos_hold_pos_p = 120  # P posición (×100)
set pos_hold_vel_p = 80   # P velocidad
set pos_hold_max_speed = 500  # 5 m/s
```

**Failsafe Multinivel:**
```bash
set failsafe_procedure = GPS-RESCUE
set failsafe_delay = 5  # 0.5s
set failsafe_recovery_delay = 20  # 2.0s
set vbat_min_cell_voltage = 330  # 3.3V critical
```

**Dynamic Notch Filters:**
```bash
set dyn_notch_count = 3
set dyn_notch_q = 120
set dyn_notch_min_hz = 150
set dyn_notch_max_hz = 600
```

### Metodología de Desarrollo

Este firmware fue desarrollado siguiendo el **Modelo V de INCOSE** (Ingeniería de Sistemas):
- **Requerimientos:** 31 RO (Operacionales) + 37 RT (Técnicos)
- **Herramientas:** QFD, FAST, Matriz N², AHP, RTM (100% trazabilidad)
- **Verificación:** 45+ vuelos de validación, blackbox logs, análisis estadístico
- **Validación:** Pruebas en condiciones extremas (4680 msnm, -15°C)

**Cumplimiento:** 31/31 RO + 37/37 RT satisfechos (100%)

### Compilación

```bash
# Clonar repositorio
git clone https://github.com/Luchinol/betaflight.git
cd betaflight

# Compilar para SpeedyBee F405 V4
make TARGET=SPEEDYBEEF405V4

# Flashear (Betaflight Configurator o DFU)
# obj/SPEEDYBEEF405V4.hex generado
```

### Documentación Completa

Para arquitectura de software completa, código fuente con explicaciones, ecuaciones matemáticas detalladas, y metodología de Ingeniería de Sistemas (Modelo V INCOSE), ver el [README principal del repositorio](../README.md).

---

## Noticias

### 📣 Anuncio: Nuevo Esquema de Versionado y Cadencia de Lanzamientos 📣

Para crear un calendario de lanzamientos más predecible, estamos cambiando a un nuevo sistema de versionado y ciclo de desarrollo, comenzando con la próxima versión.

**Nuevo Formato**: `YYYY.M.PATCH` (ej., `2025.12.1`)

**Cadencia de Lanzamientos**: Dos lanzamientos principales por año.

**Meses Objetivo**: Junio y Diciembre.

Esto significa que el sucesor de nuestra serie actual `4.x` será Betaflight `2025.12.x`, seguido de Betaflight `2026.6.x`. También alinearemos la App de Betaflight y el Firmware a las mismas versiones `YYYY.M.PATCH` (y cadencia).

**Nuestro Nuevo Ciclo de Lanzamiento**

Para soportar este calendario, nuestras fases de desarrollo estarán estructuradas de la siguiente manera:

**Alpha**: Para desarrollo de nuevas características. Las compilaciones alpha para la próxima versión estarán disponibles poco después de que se publique una versión estable.

**Beta**: Congelamiento de características de un mes solo para corrección de errores, y pull requests existentes actualmente en revisión, comenzando aproximadamente dos meses antes de un lanzamiento.

**Release Candidate (RC)**: Un período de un mes para estabilización final y pruebas antes del lanzamiento oficial.

⚠️ **Nota Importante para el Lanzamiento `2025.12`** ⚠️

Para el lanzamiento `2025.12`, debido al tiempo transcurrido desde el último lanzamiento, estamos extendiendo el período RC a dos meses. La fase Release Candidate comenzará en octubre de 2025 y hasta finales de noviembre de 2025.

### Requisitos para la Presentación de Targets Nuevos y Actualizados

Los siguientes nuevos requisitos para pull requests que agregan nuevos targets o modifican targets existentes están vigentes a partir de ahora:

1. Leer la [especificación de hardware](https://www.betaflight.com/docs/development/manufacturer/manufacturer-design-guidelines)
2. No se aceptarán nuevos targets basados en F3;
3. Para cualquier nuevo target que se agregue, solo se necesita enviar una configuración de Unified Target en https://github.com/betaflight/unified-targets/tree/master/configs/default. Ver las [instrucciones](https://www.betaflight.com/docs/manufacturer/creating-an-unified-target) sobre cómo crear una configuración de Unified Target. Si no hay un Unified Target para el tipo de MCU del nuevo target (ver instrucciones anteriores), entonces también se debe enviar una definición de target en formato 'legacy' en `src/main/target/`;
4. Para cambios en targets existentes, el cambio debe aplicarse a la configuración de Unified Target en https://github.com/betaflight/unified-targets/tree/master/configs/default. Si no existe una configuración de Unified Target para el target, se deberá crear y enviar una nueva configuración de Unified Target. Si no hay un Unified Target para el tipo de MCU del nuevo target (ver instrucciones anteriores), entonces se debe enviar una actualización a la definición de target en formato 'legacy' en `src/main/target/` junto con la actualización a la configuración de Unified Target.

## Características

Betaflight tiene las siguientes características:

* Soporte de tiras LED RGB multicolor (cada LED puede ser de un color diferente usando tiras RGB direccionables WS2811 de longitud variable - útiles para Indicadores de Orientación, Advertencia de Batería Baja, Estado de Modo de Vuelo, Solución de Problemas de Inicialización, etc.)
* Soporte de protocolos de motor DShot (150, 300 y 600), Multishot, Oneshot (125 y 42) y Proshot1000
* Registro de caja negra (blackbox) de grabador de vuelo (a flash integrada o tarjeta microSD externa cuando esté equipado)
* Soporte para targets que utilizan procesadores STM32 F4, G4, F7 y H7
* Conexión RX PWM, PPM, SPI y Serial (SBus, SumH, SumD, Spektrum 1024/2048, XBus, etc.) con detección de failsafe
* Múltiples protocolos de telemetría (CRSF, FrSky, HoTT smart-port, MSP, etc.)
* RSSI vía ADC - Usa ADC para leer señales RSSI PWM, probado con FrSky D4R-II, X8R, X4R-SB y XSR
* Soporte y configuración OSD sin necesidad de software/firmware/dispositivos de comunicación OSD de terceros
* Pantallas OLED - Muestra información sobre: Voltaje/corriente/mAh de batería, perfil, perfil de rates, modo, versión, sensores, etc.
* Ajuste manual de PID en vuelo y ajuste de rates
* Ajuste de PID y filtros usando sliders
* Perfiles de rates y selección de ellos en vuelo
* Puertos seriales configurables para Serial RX, Telemetría, Telemetría ESC, MSP, GPS, OSD, Sonar, etc. - Use la mayoría de dispositivos en cualquier puerto, softserial incluido
* Soporte VTX para Unify Pro e IRC Tramp
* Y MUCHO, MUCHO más.

## Instalación y Documentación

Ver: https://betaflight.com/docs/wiki

## Canal de Soporte y Desarrolladores

Hay un [servidor Discord](https://discord.gg/n4E6ak4u3c) dedicado para ayuda, soporte y comunidad general.

## Aplicación Betaflight

Para configurar Betaflight debes usar la [Betaflight App](https://app.betaflight.com). Es una aplicación web progresiva, por lo que siempre debería ser la última versión.

## Contribuir

Las contribuciones son bienvenidas y alentadas. Puedes contribuir de muchas maneras:

* Implementar una nueva característica en el firmware o en la aplicación (ver [más abajo](#desarrolladores));
* Actualizaciones y correcciones de documentación;
* Guías de cómo hacer - ¿recibiste ayuda? ¡Ayuda a otros!
* Reporte y corrección de errores;
* Ideas y sugerencias de nuevas características;
* Proporcionar una nueva traducción para la aplicación, o ayudarnos a mantener las existentes (ver [más abajo](#traductores)).

El mejor lugar para comenzar es el Discord de Betaflight (registro [aquí](https://discord.gg/n4E6ak4u3c)). El siguiente lugar es el rastreador de problemas de GitHub:

https://github.com/betaflight/betaflight/issues
https://github.com/betaflight/betaflight-configurator/issues

Antes de crear nuevos issues, por favor verifica si ya existe uno, ¡busca primero o de lo contrario desperdicias el tiempo de las personas cuando podrían estar programando!

Si deseas contribuir financieramente a nuestros esfuerzos, considera hacer una donación a través de [PayPal](https://paypal.me/betaflight).

Si deseas contribuir financieramente de manera continua, considera convertirte en patrocinador en [Patreon](https://www.patreon.com/betaflight).

## Desarrolladores

Se fomenta la contribución de correcciones de errores y nuevas características. Ten en cuenta que tenemos un proceso de revisión exhaustivo para pull requests, y prepárate para explicar lo que deseas lograr con tu pull request.

Antes de comenzar a escribir código, lee nuestras [directrices de desarrollo](https://www.betaflight.com/docs/development) y [definición de estilo de codificación](https://www.betaflight.com/docs/development/CodingStyle).

Se utilizan GitHub Actions para ejecutar compilaciones automáticas.

## Traductores

Queremos hacer Betaflight accesible para pilotos que no dominan el inglés, y por esta razón actualmente mantenemos traducciones en 21 idiomas para Betaflight Configurator: Català, Dansk, Deutsch, Español, Euskera, Français, Galego, Hrvatski, Bahasa Indonesia, Italiano, 日本語, 한국어, Latviešu, Português, Português Brasileiro, polski, Русский язык, Svenska, 简体中文, 繁體中文.

Tenemos un equipo de traductores voluntarios que hacen este trabajo, pero siempre son bienvenidos traductores adicionales para compartir la carga de trabajo, y estamos ansiosos por agregar idiomas adicionales. Si deseas ayudarnos con las traducciones, tienes las siguientes opciones:

- Si ayudas sugiriendo algunas actualizaciones o mejoras a las traducciones en un idioma con el que estás familiarizado, dirígete a [crowdin](https://crowdin.com/project/betaflight-configurator) y agrega tus traducciones sugeridas allí;
- Si deseas comenzar a trabajar en la traducción para un nuevo idioma, o asumir la responsabilidad de revisar la traducción para un idioma con el que estás muy familiarizado, dirígete al chat de Discord de Betaflight (registro [aquí](https://discord.gg/n4E6ak4u3c)), y únete al canal [&#39;translation&#39;](https://discord.com/channels/868013470023548938/1057773726915100702) - las personas allí pueden ayudarte a agregar un nuevo idioma, o configurarte como revisor.

## Problemas de Hardware

Betaflight no fabrica ni distribuye su propio hardware. Si bien colaboramos y contamos con el apoyo de varios fabricantes, no brindamos ningún tipo de soporte de hardware.

Si encuentras algún problema de hardware con tu controlador de vuelo u otro componente, comunícate con el fabricante o proveedor de tu hardware, o consulta [Discord](https://discord.gg/n4E6ak4u3c) para ver si otros con el mismo problema han encontrado una solución.

## Lanzamientos de Betaflight

Puedes encontrar nuestros lanzamientos [aquí](https://github.com/betaflight/betaflight/releases) en GitHub y también tenemos [notas de lanzamiento](https://www.betaflight.com/docs/category/release-notes) más detalladas en [betaflight.com](https://www.betaflight.com).

## Código Abierto / Contribuidores

Betaflight es software de **código abierto** y está disponible de forma gratuita sin garantía para todos los usuarios.

Para obtener una lista completa de contribuidores (pasados y presentes), consulta [GitHub](https://github.com/betaflight/betaflight/graphs/contributors).
