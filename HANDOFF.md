# Handoff - Sistema Anti-Incendio

## Equipo y herramientas
- PLC con HMI integrada Kinco MK043E-27DT
- PLC: KincoBuilder -> `MK043E-27DT_empty.kpr` (programa en `MK043E-27DT_empty/MAIN_MAIN.ilp`)
- HMI: Kinco DTools -> `sistema-anti-incendio/sistema-anti-incendio.dpj`
- Guía de diseño: `ladder_fire_system.md`

## Estado
- Ladder: los 48 rungs de la guía están implementados, más los rungs 49-50 (temperaturas `HMI_Temp_Zone*Global` en %VD0/%VD4 para la HMI).
- HMI: hay un proyecto creado, pero falta hacer las pantallas (punto trabado).

## Pendientes en el ladder
1. Networks 0-8 empiezan con `LD FALSE`: nunca se ejecutan (no hay conversión de temperatura, ni límites, ni máximo/promedio).
2. Networks 19-20: la advertencia usa `HMI_Threshold_Critical` en vez de `HMI_Threshold_Warning`, y las dos escriben en `Alarm_Warning` (bobina doble). Unir con OR.
3. Networks 34-35: bobina doble en `%Q0.0` (sirena); la 35 pisa a la 34. Unir con OR.
4. Networks 13 y 22: `ADD 1, HMI_Event_Counter` suma en cada scan. Hace falta un flanco.
5. Networks 47-48: T8 nunca se resetea, así que después de la primera hora suma en cada scan.
6. Confirmar la base de tiempo de los timers: T7 250, T8 3600 y `Delay_*_Time x1000` no coinciden con los demás (que suponen una base de 10 ms).
7. Umbrales sin valor inicial (0.0): con temperaturas en 0, el sistema detecta fuego apenas se activa.
8. `AI_Temp_Sensor1/2` están en %MD0/%MD4 (memoria, no entrada analógica). Revisar de dónde viene el valor del sensor.
9. Restos en las Networks 14 y 26 (`MOVE Tx, %VWxx`, `ST %BR0.0`).
