# 📖 Dicionário de Dados – Simulador de Telemetria EV

Cada linha gerada pelo simulador corresponde a um evento de telemetria de um veículo elétrico (EV).  
O formato de saída é **JSON** (pode ser convertido para CSV/Parquet).

## Raiz

| Campo             | Tipo      | Obrigatório | Formato / Unidade         | Descrição |
|-------------------|-----------|-------------|---------------------------|-----------|
| **schema_version** | string   | Sim         | Texto (ex.: `"1.0.0"`)  | Versão do schema da mensagem para garantir compatibilidade futura. |
| **vehicle_id**     | string   | Sim         | `EV-` + 5 dígitos         | Identificador único do veículo na frota. |
| **timestamp**      | datetime | Sim         | ISO 8601 UTC (`YYYY-MM-DDThh:mm:ss.ssssss+00:00`) | Momento exato de geração do evento de telemetria. |
| **metrics**        | objeto   | Sim         | Estrutura JSON aninhada   | Grupo de métricas contínuas (sinais vitais e sensores do veículo). |
| **events**         | objeto   | Sim         | Estrutura JSON aninhada   | Grupo de eventos discretos (falhas e alertas do veículo). |


## Metrics + Events

| Campo              | Tipo           | Unidade / Formato      | Obrigatório | Domínio / Valores válidos | Descrição |
|--------------------|----------------|------------------------|-------------|---------------------------|-----------|
| **soc_pct**        | float          | %                      | Sim         | 0 – 100                   | State of Charge: nível de carga da bateria. |
| **pack_voltage_v** | float          | Volts (V)              | Sim         | 200 – 900                 | Tensão do pack de baterias. |
| **pack_current_a** | float          | Amperes (A)            | Sim         | -2000 – 2000              | Corrente elétrica do pack (negativa em carga, positiva em descarga). |
| **power_kw**       | float          | Quilowatts (kW)        | Sim         | -300 – 300                | Potência instantânea consumida ou fornecida. |
| **battery_temp_c** | float          | Graus Celsius (°C)     | Sim         | -30 – 90                  | Temperatura média do pack de baterias. |
| **motor_temp_c**   | float          | Graus Celsius (°C)     | Sim         | -30 – 150                 | Temperatura do motor elétrico. |
| **coolant_temp_c** | float          | Graus Celsius (°C)     | Sim         | -30 – 120                 | Temperatura do fluido de arrefecimento. |
| **tyre_pressure_kpa** | array[float] | Quilopascal (kPa)      | Sim         | 120 – 260                 | Pressão dos quatro pneus (ordem: dianteiro-esq., dianteiro-dir., traseiro-esq., traseiro-dir.). |
| **speed_kph**      | float          | km/h                   | Sim         | 0 – 250                   | Velocidade atual do veículo. |
| **odometer_km**    | float          | km                     | Sim         | 0 – 2.000.000             | Quilometragem acumulada do veículo (odômetro). |
| **latitude**       | float          | graus decimais         | Sim         | -90 – 90                  | Latitude aproximada (GPS). |
| **longitude**      | float          | graus decimais         | Sim         | -180 – 180                | Longitude aproximada (GPS). |
| **heading_deg**    | float          | graus (0–360)          | Sim         | 0 – 360                   | Direção/rumo de movimento em relação ao norte. |
| **ambient_temp_c** | float          | Graus Celsius (°C)     | Sim         | -50 – 80                  | Temperatura ambiente externa. |
| **is_charging**    | boolean        | true / false           | Sim         | `true` ou `false`         | Indica se o veículo está em processo de recarga. |
| **charge_power_kw**| float          | Quilowatts (kW)        | Sim         | 0 – 400                   | Potência de carga aplicada (0 se não estiver carregando). |
| **health_score**   | float          | Escala 0–1             | Sim         | 0 – 1                     | Índice de saúde da bateria (1=ótima, 0.5=degradada). |
| **fault_code**     | string / null  | Texto ou `null`        | Não         | `null` (sem falha) ou:<br>• `BMS_WARN_TEMP`<br>• `TIRE_PRESS_LOW`<br>• `COOLANT_LEVEL_LOW`<br>• `BRAKE_PAD_WEAR` | Código de falha ativa do veículo (eventos discretos). |
