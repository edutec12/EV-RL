# EV-RL

Codigo base para resolver el problema de gestion autonoma de carga de vehiculos
electricos del documento `EriacFinal_corregido.docx`.

El script implementa:

- Restriccion agregada de potencia `sum(p_n,h) <= Pmax`.
- Evolucion de SoC `s_n,h+1 = min(1, s_n,h + p_n,h / C)`.
- Penalizaciones por costo energetico, pico de potencia, SoC bajo,
  SoC no homogeneo y SoC insuficiente al desconectar.
- Comparacion contra `early`, `late` y una referencia `MPC` precio-aware.
- Separacion entre `mpc_oracle` (informacion real, benchmark ideal opcional) y
  `mpc_forecast` (pronostico imperfecto y discretizacion mas gruesa, caso
  operativo).
- Incertidumbre explicita en SoC observado, SoC previsto, hora real de llegada,
  hora real de salida y errores de pronostico de llegada/salida.
- Entrenamiento DDPG para acciones continuas de potencia.
- Warm-start opcional mediante behavior cloning desde `MPC`.
- Safety shield de deadline para evitar politicas RL que incumplan el SoC
  objetivo por falta de potencia futura.
- Resultados reproducibles por semilla, metadatos, CSVs y figuras a 300 dpi.

## Instalacion

```bash
cd /Users/eduardosalazar/Documents/EV-RL
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

## Ejecutar escenarios

Escenario con 4 EV, ventana 9:00-19:00:

```bash
python ev_charging_ddpg.py --n-evs 4 --episodes 1500
```

Escenario con 8 EV:

```bash
python ev_charging_ddpg.py --n-evs 8 --episodes 1500
```

Ambos escenarios:

```bash
python ev_charging_ddpg.py --run-all --episodes 1500
```

Corrida recomendada para resultados publicables rapidos:

```bash
python ev_charging_ddpg.py \
  --publication-run \
  --seeds 7,11,19 \
  --episodes 600 \
  --warmup-policy mpc \
  --bc-epochs 8 \
  --bc-episodes 150 \
  --alpha-low 4 \
  --alpha-dep 250 \
  --alpha-hom 2 \
  --lambda-peak 0.015 \
  --target-soc 0.90 \
  --output-dir outputs_publication
```

Corrida recomendada para comparar RL contra MPC operativo con incertidumbre:

```bash
python ev_charging_ddpg.py \
  --publication-run \
  --seeds 7,11,19 \
  --episodes 80 \
  --warmup-policy mpc \
  --warmup-steps 160 \
  --bc-epochs 30 \
  --bc-episodes 100 \
  --alpha-low 4 \
  --alpha-dep 250 \
  --alpha-hom 2 \
  --lambda-peak 0.015 \
  --mpc-chunk-kwh 0.5 \
  --mpc-forecast-error 0.35 \
  --mpc-baseline-chunk-kwh 8.0 \
  --noise-start 0.15 \
  --noise-end 0.01 \
  --actor-lr 0.00001 \
  --critic-lr 0.0003 \
  --output-dir outputs_publication_robust
```

Interpretacion recomendada para el articulo:

- `mpc_oracle` no es una politica operativa justa para "ganarle"; es una cota
  superior diagnostica con SoC y tiempos reales.
- `mpc_forecast` representa un MPC operativo con error de pronostico y
  discretizacion limitada.
- La afirmacion defendible es: RL se aproxima al MPC perfecto y supera al MPC
  operativo bajo incertidumbre, manteniendo el SoC objetivo.

## Competencia bajo incertidumbre de SoC y horarios

Para eliminar el supuesto de MPC perfecto, usa incertidumbre real y pronosticada
en perfiles de EV:

```bash
python ev_charging_ddpg.py \
  --publication-run \
  --seeds 7,11,19 \
  --episodes 160 \
  --warmup-policy deadline \
  --warmup-steps 180 \
  --bc-epochs 0 \
  --eval-episodes 10 \
  --checkpoint-eval-episodes 5 \
  --randomize-profiles \
  --arrival-std-hours 1.0 \
  --departure-std-hours 2.0 \
  --forecast-arrival-std-hours 0.5 \
  --forecast-departure-std-hours 1.5 \
  --soc-observation-std 0.03 \
  --soc-forecast-std 0.10 \
  --controller-departure-margin-hours 1.0 \
  --mpc-forecast-error 0.2 \
  --mpc-chunk-kwh 1.0 \
  --mpc-baseline-chunk-kwh 4.0 \
  --alpha-low 4 \
  --alpha-dep 800 \
  --alpha-hom 2 \
  --lambda-peak 0.015 \
  --noise-start 0.4 \
  --noise-end 0.03 \
  --actor-lr 0.00004 \
  --critic-lr 0.0005 \
  --output-dir outputs_uncertain_competition_multiseed
```

Con esa corrida multi-semilla, RL supera al MPC basado en pronosticos:

- 4 EV: `RL = 104.94` frente a `MPC forecast = 130.48`.
- 8 EV: `RL = 279.13` frente a `MPC forecast = 346.49`.

Los resultados agregados quedan en
`outputs_uncertain_competition_multiseed/publication_results_aggregate.csv`.

## Variante RL orientada a pico

Si el criterio de comparacion exige que RL tambien reduzca el pico frente al
MPC con pronosticos, activa un limite operativo de pico en la accion RL:

```bash
# 4 EV
python ev_charging_ddpg.py ... --n-evs 4 --rl-peak-cap-ratio 0.75

# 8 EV
python ev_charging_ddpg.py ... --n-evs 8 --rl-peak-cap-ratio 0.65
```

Con la corrida `outputs_uncertain_peak_competition`, los promedios multi-semilla
son:

- 4 EV:
  - `mpc_forecast`: objetivo `130.48`, pico `15.69 kW`.
  - `rl_peak`: objetivo `106.94`, pico `15.62 kW`.
- 8 EV:
  - `mpc_forecast`: objetivo `346.49`, pico `28.21 kW`.
  - `rl_peak`: objetivo `307.49`, pico `28.00 kW`.

Esta variante debe describirse como RL con restriccion operacional de pico, no
como el mismo controlador RL sin limite.

Para una prueba corta:

```bash
python ev_charging_ddpg.py \
  --n-evs 4 \
  --episodes 200 \
  --warmup-policy mpc \
  --bc-epochs 5 \
  --alpha-low 4 \
  --alpha-dep 250 \
  --lambda-peak 0.015 \
  --output-dir outputs_quick_publishable
```

Si quieres forzar que el agente busque 100 % de SoC al salir, como se describe
en la seccion de resultados, usa:

```bash
python ev_charging_ddpg.py --run-all --episodes 1500 --target-soc 1.0
```

Los resultados se guardan en `outputs/`:

- `scenario_<N>_summary.csv`: comparacion de costos, pico y SoC final.
- `scenario_<N>_<policy>_rollout.csv`: potencias y SoC por hora.
- `scenario_<N>_training_history.csv`: evolucion del entrenamiento RL.
- `scenario_<N>_behavior_cloning_history.csv`: perdida de warm-start.
- `scenario_<N>_metadata.json`: parametros usados en la corrida.
- `scenario_<N>_power.png`, `scenario_<N>_rl_soc.png`,
  `scenario_<N>_costs.png`, `scenario_<N>_training.png`: graficas para el
  articulo.

En `--publication-run` tambien se generan:

- `publication_results_raw.csv`: resultados por semilla.
- `publication_results_aggregate.csv`: medias y desviaciones estandar.
- `publication_summary_<N>ev.png`: resumen grafico por escenario.

## Prueba rapida sin PyTorch

Para validar solo el entorno y las politicas `early/late`:

```bash
python ev_charging_ddpg.py --baseline-only --no-plots --n-evs 4
```