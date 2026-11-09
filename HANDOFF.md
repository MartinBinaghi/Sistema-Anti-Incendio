# Handoff - Sistema Anti-Incendio

## Equipo y herramientas
- PLC con HMI integrada Kinco MK043E-27DT
- PLC: KincoBuilder -> `MK043E-27DT_empty.kpr` (programa en `MK043E-27DT_empty/MAIN_MAIN.ilp`, variables locales en `MAIN_MAIN.ilv`)
- HMI: Kinco DTools -> `sistema-anti-incendio/sistema-anti-incendio.dpj`
- Guía de diseño: `ladder_fire_system.md` (ojo: su escala de AI 819-4095 y sus tiempos están mal para este equipo, ver abajo)
- Accesos directos en la carpeta padre (`..\`): `KincoBuilder V8.5En.lnk`, `Kinco DTools V4.5.lnk`, `KDManager V4.5.lnk`
- Instalación: `C:\Program Files (x86)\Kinco\KincoBuilderEn\` (ayuda `KincoBuilder_V1.1.5.0En.chm`, manual `HMI-PLC User Manual.pdf`)
- Computer use quedó activado en la app de Claude: la próxima sesión puede abrir KincoBuilder y DTools.
- **DTools se abre como admin** (manifest `highestAvailable`) y así computer use no puede hacer clic. Para manejarlo, abrirlo desde PowerShell con `$env:__COMPAT_LAYER='RunAsInvoker'; Start-Process "C:\Kinco\Kinco DTools V4.5\Kinco DTools.exe" -ArgumentList '"<ruta>\sistema-anti-incendio.dpj"'`. No usar `open_application`: lanza otra copia elevada y abre el actualizador.
- Las pantallas `.whe` están cifradas (`KINCOEN`): solo se editan desde DTools.

## Estado
- Ladder: 50 networks, 201 instrucciones. Compila en KincoBuilder sin errores (ver abajo).
- HMI: 4 pantallas en DTools (ver abajo). Compila con 0 warnings y 0 errores.

## Mapa de direcciones PLC <-> HMI (todo en %V, retentivo)
| Variable | Dirección | Tipo |
|---|---|---|
| HMI_Temp_Zone1 / Zone2 / Max / Avg (°C) | VD8 / VD12 / VD16 / VD20 | REAL |
| HMI_Threshold_Warning / Critical (°C) | VD24 / VD28 | REAL |
| HMI_Delay_Alarm / Delay_Sprinkler (s) | VW32 / VW34 | INT |
| HMI_System_Status (0 apagado, 1 normal, 2 advertencia, 3 crítico, 4 falla sensor) | VW36 | INT |
| HMI_Event_Counter / HMI_System_Hours | VW38 / VW40 | INT |
| HMI_Fire_Zone1, Fire_Zone2, Alarm_Active, Sprinklers_Active, Silence_Active, Test_Mode | V50.0 ... V50.5 | BOOL |
| HMI_Fault_Zone1, HMI_Fault_Zone2 (falla de sensor) | V50.6, V50.7 | BOOL |
| HMI_BTN_Reset, Silence_Alarm, Test_System, Manual_Zone1, Manual_Zone2 | V51.0 ... V51.4 | BOOL |
| HMI_Warning (copia de Alarm_Warning, para el registro de eventos) | V52.0 | BOOL |
| Temperaturas en décimas (copia DINT) | VD0 / VD4 | DINT |

Son **variables globales** (Global Variable). Las REAL van como `%VR8`...`%VR28`, que es la misma memoria que `VD8`...`VD28`; la HMI las lee como `VD` float. KincoBuilder toma la tabla global del `.kpr`, no del `.kgv` (el `.kgv` es solo una copia que regenera). Para cargar muchas a la vez: clic derecho en la tabla > Import from .csv (formato `Symbol,Address,Data Type,Comment,`). En el `.ilp` las referencias a globales se guardan por dirección (`%V51.0`) y el editor muestra el nombre.

## Pantallas HMI (DTools)
- **0: Frame0 (principal):** título + estado (`VW36`, texto con color; 4 = FALLA SENSOR en violeta), temperatura de cada zona (`VD8`/`VD12`) con lámpara de fuego (`V50.0`/`V50.1`, parpadea en rojo), máx/prom (`VD16`/`VD20`), indicadores de sirena (`Q0.0`), rociadores, silencio y test (`V50.3`-`V50.5`). Botones momentáneos RESET/SILENCIAR/TEST (`V51.0`-`V51.2`), CONFIG (pantalla 10, pide clave), EVENTOS (11) y CURVA (12).
- **10: Configuracion:** entradas de umbral advertencia/crítico (`VD24`/`VD28`, 0-100 °C) y retardos (`VW32`/`VW34`, 0-300 s), contadores eventos/horas (`VW38`/`VW40`), botones con retención MANUAL Z1/Z2 (`V51.3`/`V51.4`) y VOLVER. **Protegida con nivel de seguridad 1**: el botón CONFIG muestra la ventana de login; al salir vuelve a nivel 0. **La clave del nivel 1 es la de fábrica (888888): cambiarla en HMI Attribute > Security Levels Setting.**
- **11: Eventos:** Event Display con N°, fecha (M/D), hora y texto. Barra de scroll. Los eventos están en *Event Information Logon* (barra de herramientas, 4.º ícono de la segunda fila): fuego Z1/Z2 (`V50.0`/`V50.1`), falla sensor Z1/Z2 (`V50.6`/`V50.7`), advertencia (`V52.0`), rociadores (`V50.3`), silencio (`V50.4`), prueba (`V50.5`), reset pantalla (`V51.0`), parada de emergencia (`I0.5`), reset pulsador (`I0.0`). Todos se disparan en OFF→ON.
- **12: Tendencia:** Trend Curve de 2 canales float (`VR8` y `VR12`, contiguos), 0-100 °C, una muestra cada 5 s x 120 puntos = últimos 10 minutos. Zona 1 azul, zona 2 naranja, con leyenda. No usa Datalogger (la curva se borra al apagar).
- La simulación offline (KHSimulator) no abrió la pantalla con DTools sin admin. No se probó.
- La fecha del registro sale M/D (formato de DTools); no hay opción D/M en el componente.
- `temp/` y `~backup/` son carpetas de trabajo de DTools: están en `.gitignore`.

## Prueba en KincoBuilder (2026-11-04)
1. OK: Compile All -> 0 errores, 0 warnings, 173 instrucciones.
2. OK: MAIN en LD carga el `.ilp` editado (networks 0-46). Se ven bien la 0 (`%SM0.1`), la 33 (sirena con dos `OR(`), la 45 (`R_TRIG` de minutos) y la 46.
3. Al compilar, KincoBuilder guardó el proyecto: `.obj` ya tiene la copia nueva, `.kgv` se reordenó y quitó T3/T6/T7.
4. OK en el **simulador offline de KincoBuilder** (botón "Start offline simulator" de la barra, o Debug > Start Simulator). Con `I0.2` = 1 desde el panel I/O del simulador y `%AIW0` = 16000 forzado desde la Status Chart (Debug > Force > Force All): `VD0` = 750 (75,0 °C), `V50.0` = 1, `VW36` = 3 (crítico), y después del retardo sirena `Q0.0` = 1 y rociador `Q0.2` = 1. La Status Chart no tiene formato float: para ver temperaturas usar `VD0`/`VD4` (décimas).
5. Antes, con las variables HMI como locales en `%V`, el simulador avisaba "memory allocation for the local variables does not suit the current CPU". Al pasarlas a globales el aviso ya no aparece y la prueba del punto 4 da lo mismo. **No usar nunca "Reallocate Local Variables"**.

## Qué se cambió en el ladder (resumen)
- **Temperaturas (networks 1-7):** leen `%AIW0`/`%AIW2` con `(raw-4000)/16`, que da décimas de °C. El límite de 0 a 100 °C se aplica en DINT y recién después se pasa a REAL. Para la HMI se copian a %VD0/%VD4 en décimas.
- **Primer scan (network 0):** advertencia 50 °C, crítico 70 °C, retardo de alarma 3 s, retardo de rociadores 5 s.
- **Advertencia:** se activa con `HMI_Temp_Max >= HMI_Threshold_Warning` (una sola network).
- **Sirena:** una sola network que combina:
  - continua por T4, o por `Alarm_Latched` cuando la alarma crítica ya no está (para que el retardo sirva),
  - intermitente con `%SM0.3` en advertencia.
- **Baliza:** parpadea con `%SM0.3`. Ya no se usan T6/T7.
- **Contador de eventos:** suma con el flanco de `Alarm_Critical`. El test no suma aparte.
- **Timers:**
  - Test: T2 en 10000 (10 s).
  - Silencio: T8 en 12000 (120 s). T3 ya no se usa.
  - Retardos: s x 100 (T4/T5 cuentan en 10 ms).
- **Horas:** se cuentan los flancos de `%SM0.6` en `System_Minutes` (%LW70, variable nueva) y cada 60 se suma una hora.
- **Estado del sistema:** los estados 2 y 1 tienen `ANDN Alarm_Latched` para no pisar el 3.
- **Falla de sensor (networks 7-8, 2026-11-09):** con `%AIW0`/`%AIW2` < 3600 (cable cortado, < 3,6 mA) o > 20500 (corto, > 20,5 mA) se activa `V50.6`/`V50.7` y la temperatura de esa zona se fuerza a 0 (no hay falso fuego). Network 45: estado 4 si hay falla y no hay alarma crítica. Probado en el simulador: sin sensores → estado 4; 12000/8000 → 50,0 °C y estado 2; zona 1 en 21000 → falla Z1 y estado 4; falla Z1 + fuego Z2 (16000) → estado 3 (el fuego tiene prioridad).

## Datos del equipo (de la ayuda de KincoBuilder y del manual del MK)
- AI: %AIW0 y %AIW2. En 4-20 mA el valor va de 4000 a 20000 (mA x 1000).
- Timers: T0-T3 cuentan en 1 ms, T4-T19 en 10 ms y T20-T255 en 100 ms. El máximo es 32767 cuentas.
- SM0.0 está siempre en ON, SM0.1 solo en el primer scan, SM0.3 es un reloj de 1 s, SM0.4 de 2 s, SM0.5 de 4 s y SM0.6 de 60 s.
- IL: `ADD/SUB/MUL/DIV IN, OUT` hacen `OUT = OUT op IN`. Las comparaciones (`GE`, `LT`...) dejan en CR el AND con el resultado.
- Área L (variables locales): no es retentiva y no conviene accederla directo. La HMI no la ve.
- Área V: 4 KB. Es retentiva de VB0 a VB1907.

## Pendientes
1. Descargar PLC + HMI al MK043E y probar todo junto (ver arriba, punto 4, y que los botones e indicadores respondan).
2. Los umbrales y retardos que se cambian desde la HMI se pisan en cada arranque (network 0 los vuelve a 50/70/3/5). Si tienen que quedar guardados, sacar esa inicialización o hacerla solo si el valor es 0.
3. Confirmar en la configuración de hardware que AI0 y AI1 estén en 4-20 mA.
4. Cambiar la clave de CONFIG (hoy 888888, la de fábrica).
5. `AI_Temp_Sensor1/2` (%MD0/%MD4) ya no se usan: el programa lee %AIW0/%AIW2 directo.
6. Rociador manual: `HMI_BTN_Manual_Zone1/2` solo actúa si `T5` terminó (es decir, con alarma crítica). Si el manual tiene que funcionar siempre, cambiar las networks 36/37.
