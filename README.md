![Betaflight](https://raw.githubusercontent.com/betaflight/.github/main/profile/images/bf_logo.svg#gh-light-mode-only)
![Betaflight](https://raw.githubusercontent.com/betaflight/.github/main/profile/images/bf_logo_dark.svg#gh-dark-mode-only)

[![Última versión](https://img.shields.io/github/v/release/betaflight/betaflight)](https://github.com/betaflight/betaflight/releases) [![Build](https://img.shields.io/github/actions/workflow/status/betaflight/betaflight/push.yml?branch=master)](https://github.com/betaflight/betaflight/actions/workflows/push.yml) [![Licencia: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0) [![Únete en Discord](https://img.shields.io/discord/868013470023548938)](https://discord.gg/n4E6ak4u3c)

Betaflight es un software de controlador de vuelo (firmware) utilizado para volar aeronaves multi-rotor y de ala fija. Betaflight se enfoca en el rendimiento de vuelo, la adición de características de vanguardia y un amplio soporte de targets.

---

## ⚠️ Versión Custom - Proyecto UAV FPV Militar

**Autor:** Luis Olmos ([@Luchinol](https://github.com/Luchinol))

Esta es una versión modificada de Betaflight 4.4.2 con mejoras específicas para aplicaciones militares de la Fuerza Aérea de Chile (FACh). Las modificaciones incluyen:

### Modificaciones Custom Implementadas

#### 1. GPS Rescue Enhanced con Dead Reckoning

- **Archivos modificados:** `src/main/flight/gps_rescue_multirotor.c`, `src/main/io/gps.c`
- Navegación con proyección vectorial y horizonte virtual
- Dead reckoning (navegación inercial) durante pérdida GPS transitoria (<30s)
- Compensación activa de viento mediante feedforward
- Precisión de aterrizaje mejorada: ±2m (vs ±5m estándar)
- Tasa de éxito validada: 100% en 45+ activaciones

#### 2. Position Hold 6DOF Mejorado

- **Archivos modificados:** `src/main/flight/pos_hold_multirotor.c`, `src/main/flight/position.c`
- Control cascada: Posición (50Hz) → Velocidad (100Hz) → Actitud (8kHz)
- Deriva validada: ±1,8m en viento 12 m/s
- Parámetros PID optimizados empíricamente

#### 3. Filtrado Digital Avanzado

- **Archivos modificados:** `src/main/flight/dyn_notch_filter.c`, `src/main/flight/pid.c`
- Cadena de filtros optimizada para motores XING2 2207
- Dynamic Notch Filters tracking 150-300 Hz
- Latencia total agregada: <50 μs

#### 4. Failsafe Multinivel

- **Archivos modificados:** `src/main/flight/failsafe.c`, `src/main/fc/rc_modes.c`
- Sistema de seguridad en cascada con rollback
- Integración con GPS Rescue Enhanced
- Detección térmica y de batería crítica

### Documentación Completa del Proyecto

Para documentación técnica completa del proyecto UAV FPV Militar, ver el [README principal del repositorio](../README.md).

---

## Calendario de Lanzamientos

| Fecha      | Versión | Etapa             | Estado     |
| ---------- | -------- | ----------------- | ---------- |
| 01-10-2025 | 2025.12  | Beta              | Completado |
| 01-10-2025 | 2025.12  | Release Candidate | En curso   |
| 01-12-2025 | 2025.12  | Release           | Pendiente  |
| 01-04-2026 | 2026.6   | Beta              |            |
| 01-05-2026 | 2026.6   | Release Candidate |            |
| 01-06-2026 | 2026.6   | Release           |            |
| 01-10-2026 | 2026.12  | Beta              |            |
| 01-11-2026 | 2026.12  | Release Candidate |            |
| 01-12-2026 | 2026.12  | Release           |            |

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
