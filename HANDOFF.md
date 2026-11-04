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
- Ladder: 47 networks. Compila en KincoBuilder sin errores (ver abajo).
- HMI: 2 pantallas hechas en DTools (ver abajo). Compila con 0 warnings y 0 errores.

## Mapa de direcciones PLC <-> HMI (todo en %V, retentivo)
| Variable | Dirección | Tipo |
|---|---|---|
| HMI_Temp_Zone1 / Zone2 / Max / Avg (°C) | VD8 / VD12 / VD16 / VD20 | REAL |
| HMI_Threshold_Warning / Critical (°C) | VD24 / VD28 | REAL |
| HMI_Delay_Alarm / Delay_Sprinkler (s) | VW32 / VW34 | INT |
| HMI_System_Status (0 apagado, 1 normal, 2 advertencia, 3 crítico) | VW36 | INT |
| HMI_Event_Counter / HMI_System_Hours | VW38 / VW40 | INT |
| HMI_Fire_Zone1, Fire_Zone2, Alarm_Active, Sprinklers_Active, Silence_Active, Test_Mode | V50.0 ... V50.5 | BOOL |
| HMI_BTN_Reset, Silence_Alarm, Test_System, Manual_Zone1, Manual_Zone2 | V51.0 ... V51.4 | BOOL |
| Temperaturas en décimas (copia DINT) | VD0 / VD4 | DINT |

Son **variables globales** (Global Variable). Las REAL van como `%VR8`...`%VR28`, que es la misma memoria que `VD8`...`VD28`; la HMI las lee como `VD` float. KincoBuilder toma la tabla global del `.kpr`, no del `.kgv` (el `.kgv` es solo una copia que regenera). Para cargar muchas a la vez: clic derecho en la tabla > Import from .csv (formato `Symbol,Address,Data Type,Comment,`). En el `.ilp` las referencias a globales se guardan por dirección (`%V51.0`) y el editor muestra el nombre.

## Pantallas HMI (DTools)
- **0: Frame0 (principal):** título + estado (`VW36`, texto con color), temperatura de cada zona (`VD8`/`VD12`) con lámpara de fuego (`V50.0`/`V50.1`, parpadea en rojo), máx/prom (`VD16`/`VD20`), indicadores de sirena (`Q0.0`), rociadores, silencio y test (`V50.3`-`V50.5`). Botones momentáneos RESET/SILENCIAR/TEST (`V51.0`-`V51.2`) y CONFIG (va a la pantalla 10).
- **10: Configuracion:** entradas de umbral advertencia/crítico (`VD24`/`VD28`, 0-100 °C) y retardos (`VW32`/`VW34`, 0-300 s), contadores eventos/horas (`VW38`/`VW40`), botones con retención MANUAL Z1/Z2 (`V51.3`/`V51.4`) y VOLVER.
- La simulación offline (KHSimulator) no abrió la pantalla con DTools sin admin. No se probó.

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
4. Sensor cortado: con 0 mA la temperatura queda en 0 °C y no se avisa ninguna falla.
5. `AI_Temp_Sensor1/2` (%MD0/%MD4) ya no se usan: el programa lee %AIW0/%AIW2 directo.
6. Rociador manual: `HMI_BTN_Manual_Zone1/2` solo actúa si `T5` terminó (es decir, con alarma crítica). Si el manual tiene que funcionar siempre, cambiar las networks 36/37.
