# Growatt Monitoring

Stack auto-hospedada de monitoramento de inversor solar Growatt, contornando totalmente a nuvem do fabricante. Coleta dados via MQTT do stick ShineWiFi-X (com firmware [OpenInverterGateway](https://github.com/OpenInverterGateway/OpenInverterGateway) substituindo o original), armazena no InfluxDB com retenção infinita, visualiza no Grafana e envia alertas críticos via Telegram. Tudo roda em containers no TrueNAS SCALE.

## Arquitetura

```
[Inversor Growatt]
       ↓ USB Modbus
[Stick ShineWiFi-X com OpenInverterGateway]
       ↓ MQTT (energy/solar)
[Mosquitto] ← TrueNAS:1883
       ↓ subscribe
[Telegraf]
       ↓ write
[InfluxDB v2] (bucket: solar, retenção infinita)
       ↓ query
[Grafana] (dashboards live)
       ↓ alert rule trigger
[Telegram bot]

Acesso remoto via Tailscale (MagicDNS: http://truenas:3000)
```

## Pré-requisitos

> **Fora do escopo deste README**: instalação e configuração inicial do TrueNAS SCALE. Assume-se que você já tem um TrueNAS SCALE 24.10+ rodando, com pool ZFS criado e acesso administrativo. Para começar do zero com TrueNAS, consulte a [documentação oficial](https://www.truenas.com/docs/scale/).

O que você precisa antes de começar:

- **TrueNAS SCALE 24.10+** (Electric Eel ou superior) com Docker disponível
- **SSH habilitado** no TrueNAS, com usuário admin que tenha permissão de rodar `docker`
- **Pool ZFS** já criado (este README usa `<POOL>` como placeholder — substitua pelo nome real do seu pool)
- **Stick Growatt ShineWiFi-X** (com USB-Serial Exar XR21V1410 ou CH340)
- **Adaptador USB-to-TTL 3.3V** (para o flash inicial do firmware)
- **Conexão direta entre stick e inversor** funcionando

## Visão geral das etapas

1. Substituir firmware do stick por OpenInverterGateway
2. Subir stack Docker no TrueNAS (Mosquitto + InfluxDB + Telegraf + Grafana)
3. Configurar stick para publicar no Mosquitto
4. Importar dashboard no Grafana
5. Criar alertas via Telegram
6. (Opcional) Configurar acesso remoto via Tailscale

---

## Etapa 1: Flash do firmware OpenInverterGateway

O stick original da Growatt comunica via portas TCP 5279/5280 com o servidor da Growatt. Se sua rede bloqueia essas portas, ou se o stick está bricado por OTA mal feito, a única saída é substituir o firmware.

### 1.1 Hardware necessário

- Adaptador USB-to-TTL **3.3V** (FTDI, CH340 ou CP2102)
- Fios dupont fêmea
- Pinout do stick (CN1):
  - Pino 2: GND
  - Pino 3: RX0
  - Pino 4: TX0
  - Pino 6: GPIO0
  - Pino 7: GND

### 1.2 Conexões

```
Adaptador USB-TTL    Stick CN1
─────────────────    ─────────
GND          ──────  Pino 2 (GND)
TX           ──────  Pino 3 (RX0)
RX           ──────  Pino 4 (TX0)
                     Pino 6 (GPIO0) ──jumper──> Pino 7 (GND)
```

Alimente o stick via USB normal (não use o 3.3V do adaptador junto com USB).

O jumper GPIO0–GND é o que coloca o ESP8266 em modo flash.

### 1.3 Instalar esptool e clonar OpenInverterGateway

```bash
# esptool via uv (ou pip install esptool)
uv tool install esptool

# Clona o projeto
git clone https://github.com/OpenInverterGateway/OpenInverterGateway.git
cd OpenInverterGateway
```

### 1.4 Patch necessário para autodetecção (importante!)

A autodetecção do tipo de stick falha em sticks reais com inversor em estado de erro (timeout de 250ms é apertado demais). Edite `SRC/ShineWiFi-ModBus/Growatt.cpp`, função `Growatt::begin()`, substituindo o bloco da auto-detecção por:

```cpp
#if SIMULATE_INVERTER == 1
  _eDevice = SIMULATE_DEVICE;
#else
  // Skip auto-detection: hardware is known to be ShineWiFi-X (USB, 115200 baud).
  Serial.begin(115200);
  Modbus.begin(1, serial);
  Modbus.setResponseTimeout(2000);
  _eDevice = ShineWiFi_X;
#endif
```

### 1.5 Compilar e gravar

```bash
# Copia template de configuração
cp SRC/ShineWiFi-ModBus/Config.h.example SRC/ShineWiFi-ModBus/Config.h

# Habilita feature de leitura/escrita arbitrária de Modbus (importante para diagnóstico)
sed -i 's/#define ENABLE_MODBUS_COMMUNICATION 0/#define ENABLE_MODBUS_COMMUNICATION 1/' \
  SRC/ShineWiFi-ModBus/Config.h

# Instala PlatformIO
uv tool install platformio

# Compila
pio run -e ShineWifiX
```

Com o stick ligado e jumper GPIO0–GND no lugar:

```bash
# Backup do firmware atual (importante!)
uv run esptool --port /dev/ttyUSB0 read_flash 0x0 0x400000 shinewifix_backup.bin

# Apaga e grava novo firmware
uv run esptool --port /dev/ttyUSB0 erase-flash
pio run -e ShineWifiX -t upload --upload-port /dev/ttyUSB0
```

Remova o jumper, desconecte e reconecte o stick. Procure a rede WiFi `GrowattConfig` (senha: `growsolar`).

### 1.6 Configuração inicial do stick

Conecte ao WiFi `GrowattConfig`, abra `http://192.168.4.1`, vá em **Configure WiFi**:

- Selecione sua rede WiFi de casa, digite a senha
- Em **Setup**, configure:
  - **hostname**: `growatt-stick`
  - **ip / netmask / gateway / dns**: IP estático na sua LAN (recomendado, evita perder o stick depois)
  - **mqtt server**: IP do TrueNAS (vazio por enquanto, vamos preencher depois da Etapa 2)
  - **mqtt port**: `1883`
  - **mqtt topic**: `energy/solar`
  - **mqtt user**: `growatt`
  - **mqtt password**: (defina uma senha — vai ser a mesma usada na Etapa 2)
- Save

---

## Etapa 2: Stack Docker no TrueNAS

### 2.1 Criar dataset

> **Nota**: Os comandos abaixo usam `<POOL>` como placeholder pro nome do seu pool ZFS. Substitua pelo nome real (ex: `tank`, `data`, `pool0` — o que aparece em `sudo zpool list -H -o name`).

Na UI do TrueNAS:

1. **Datasets** → seu pool `<POOL>` → **Add Dataset**
2. Nome: `apps`
3. Dentro de `apps`, criar outro: `solar`
4. **Edit Permissions** no dataset `solar`:
   - Owner: seu usuário
   - Group: `builtin_users`
   - Mode: `775` ou `777`

### 2.2 Copiar arquivos para o TrueNAS

Do seu notebook:

```bash
git clone https://github.com/wfoschiera/growatt-monitoring.git
cd growatt-monitoring

# Cria seu .env baseado no template
cp .env.example .env

# Edita com valores reais
$EDITOR .env

# Gera token InfluxDB
openssl rand -hex 32  # cole no INFLUXDB_ADMIN_TOKEN

# Sincroniza para TrueNAS
rsync -av --exclude='.git/' ./ admin@<IP_TRUENAS>:/mnt/<POOL>/apps/solar/
```

### 2.3 Preparar diretórios e senhas no TrueNAS

Via SSH no TrueNAS:

```bash
cd /mnt/<POOL>/apps/solar

# Restringe leitura do .env
chmod 600 .env

# Cria diretórios de runtime
mkdir -p mosquitto/data mosquitto/log influxdb/data influxdb/config grafana/data
chmod 777 mosquitto/data mosquitto/log influxdb/data influxdb/config grafana/data

# Carrega variáveis e gera passwords do Mosquitto
set -a; source .env; set +a
touch mosquitto/config/passwords
chmod 600 mosquitto/config/passwords

# Importante: o user dentro do container Mosquitto é UID 1883
sudo chown 1883:1883 mosquitto/config/passwords mosquitto/data mosquitto/log

docker run --rm -v "$PWD/mosquitto/config:/mosquitto/config" \
  eclipse-mosquitto:2 \
  mosquitto_passwd -b /mosquitto/config/passwords \
  "$MQTT_GROWATT_USER" "$MQTT_GROWATT_PASSWORD"

docker run --rm -v "$PWD/mosquitto/config:/mosquitto/config" \
  eclipse-mosquitto:2 \
  mosquitto_passwd -b /mosquitto/config/passwords \
  "$MQTT_TELEGRAF_USER" "$MQTT_TELEGRAF_PASSWORD"
```

### 2.4 Subir os containers

```bash
docker compose up -d
docker compose ps    # confirma que os 4 estão Up
docker compose logs --tail=20    # confirma sem erros
```

### 2.5 Smoke tests

```bash
# InfluxDB
curl http://localhost:8086/health

# Grafana
curl http://localhost:3000/api/health

# Mosquitto pub/sub roundtrip
docker exec -it solar-mosquitto sh -c \
  'mosquitto_sub -h localhost -t test -u telegraf -P "<senha>" -v' &
docker exec -it solar-mosquitto sh -c \
  'mosquitto_pub -h localhost -t test -m "ping" -u telegraf -P "<senha>"'
```

---

## Etapa 3: Conectar o stick ao broker

Volte na UI do stick (`http://growatt-stick.local` ou pelo IP estático que você definiu).

Acesse `http://<IP_DO_STICK>/startAp` para forçar modo AP, então:

1. Conecte ao WiFi `GrowattConfig`
2. Abra `http://192.168.4.1` → **Setup**
3. Atualize **mqtt server** para o IP do TrueNAS
4. Save

O stick reboota e começa a publicar em `energy/solar`. Verifique:

```bash
docker exec -it solar-mosquitto sh -c \
  'mosquitto_sub -h localhost -t "energy/solar" -u telegraf -P "<senha>" -v'
```

Você deve ver JSON chegando a cada ~5 segundos.

---

## Etapa 4: Grafana

### 4.1 Datasource

`http://<IP_TRUENAS>:3000` → login admin / `<GRAFANA_ADMIN_PASSWORD do .env>`

1. **Connections** → **Data sources** → **Add new data source** → **InfluxDB**
2. Configure:
   - **Name**: `InfluxDB`
   - **Query Language**: `Flux`
   - **URL**: `http://influxdb:8086` (rede interna do compose)
   - **Organization**: `home`
   - **Token**: o `INFLUXDB_ADMIN_TOKEN` do `.env`
   - **Default Bucket**: `solar`
3. **Save & Test**

### 4.2 Dashboard

1. **Dashboards** → **New** → **Import**
2. Faça upload de `grafana/dashboards/solar-overview.json` (ou cole o conteúdo)
3. Quando solicitado, escolha o datasource `InfluxDB` recém-criado
4. **Import**

Dashboard pronto com 11 painéis: geração atual, energia hoje/total, status, potência ao longo do tempo, temperatura, potência DC por string, tensão DC por string, tensão AC com thresholds ANEEL, frequência AC, geração diária últimos 30 dias.

---

## Etapa 5: Alertas Telegram

### 5.1 Bot do Telegram

1. No Telegram, busque `@BotFather` → `/newbot` → siga o wizard → guarde o token
2. Inicie conversa com seu novo bot (envie `/start`)
3. Busque `@userinfobot` → `/start` → guarde seu chat ID

### 5.2 Contact point no Grafana

1. **Alerting** → **Contact points** → **+ Add contact point**
2. Integration: **Telegram**
3. Bot API Token + Chat ID
4. **Test** (deve receber mensagem) → **Save**
5. **Notification policies** → editar default policy → trocar contact point para o Telegram

### 5.3 Mute timing para horário noturno

1. **Alerting** → **Notification policies** → tab **Mute timings**
2. **+ Add mute timing**:
   - Name: `Fora de horas de sol`
   - Time ranges: `17:00-23:59` e `00:00-08:59`
   - Location: `America/Sao_Paulo`

### 5.4 Alertas

Crie via **Alerting** → **Alert rules** → **+ New alert rule**:

| Alerta | Query | Condição | Pending | Mute timing |
|--------|-------|----------|---------|-------------|
| Temperatura > 70°C | `... \|> filter(fn: (r) => r._field == "TemperatureInsideIPM") \|> last()` | last > 70 | 5m | — |
| Tensão AC fora de 198-235V | `... \|> filter(fn: (r) => r._field == "L1ThreePhaseGridVoltage") \|> mean()` | outside [198, 235] | 10m | — |
| Stick offline | `... \|> filter(fn: (r) => r._field == "Uptime") \|> last()` | (no data triggers) | 5m | — |
| Geração zerada de dia | `... \|> filter(fn: (r) => r._field == "OutputPower") \|> mean()` | mean < 50 | 15m | `Fora de horas de sol` |

Para o alerta de stick offline, na seção "Configure no data and error handling", defina **Alert state if no data** = `Alerting`.

---

## Etapa 6 (opcional): Acesso remoto via Tailscale

### 6.1 Conta + auth key

1. [tailscale.com](https://tailscale.com) → criar conta (free plan suficiente)
2. [admin/settings/keys](https://login.tailscale.com/admin/settings/keys) → **Generate auth key**:
   - Pre-approved: ligado
   - One-time: ligado

### 6.2 App no TrueNAS

1. **Apps** → **Discover Apps** → buscar **Tailscale** → **Install**
2. Configure:
   - Auth Key: cole o token gerado
   - Hostname: `truenas`
   - Accept DNS: ligado
   - **Host Network: ligado** (crítico para acessar serviços do host via tailnet)
3. **Install**

### 6.3 Cliente no celular/notebook

- Celular: app **Tailscale** (Play Store / App Store) → login
- Linux: `curl -fsSL https://tailscale.com/install.sh | sh && sudo tailscale up`
- Mac/Windows: download em [tailscale.com/download](https://tailscale.com/download)

### 6.4 Acesso

Com Tailscale ligado nos dois dispositivos, em qualquer rede:

```
http://truenas:3000   # Grafana
http://truenas:8086   # InfluxDB UI
http://truenas         # TrueNAS UI
ssh admin@truenas      # SSH
```

---

## Estrutura do repositório

```
.
├── docker-compose.yml          # Stack: mosquitto, influxdb, telegraf, grafana
├── .env                        # Segredos (gitignored)
├── .env.example                # Template versionável
├── mosquitto/
│   └── config/
│       ├── mosquitto.conf      # Config do broker (auth obrigatória)
│       └── passwords           # Hashes mosquitto_passwd (gitignored)
├── telegraf/
│   └── telegraf.conf           # Bridge MQTT -> InfluxDB
└── grafana/
    └── dashboards/
        └── solar-overview.json # Dashboard com 11 painéis
```

## Customização

### Adicionar painéis no Grafana

Edite via UI, depois exporte: **Dashboard settings → JSON Model → copy → cole em `grafana/dashboards/solar-overview.json`** e commite. Versionar o JSON garante que mudanças não se percam se o dataset do Grafana for resetado.

### Mudar bucket / retenção

Edite `.env`:
- `INFLUXDB_BUCKET`: nome do bucket
- `INFLUXDB_RETENTION`: `0` = infinito; ou ex: `30d`, `1y`

Se mudar depois do primeiro `up`, as variáveis `DOCKER_INFLUXDB_INIT_*` são ignoradas (só rodam na inicialização). Para mudar bucket existente, use a UI do InfluxDB.

### Adicionar campos do inversor

A lista completa de campos disponíveis depende da versão do protocolo Modbus do seu inversor (1.20, 1.24, 3.05, etc.). Veja `Growatt124.cpp` no repositório do OpenInverterGateway para a lista de seu protocolo.

## Troubleshooting

### Mosquitto reinicia em loop com `Unable to open pwfile`

Permissão do `passwords`. Dentro do container o user é UID 1883:

```bash
sudo chown 1883:1883 mosquitto/config/passwords
sudo chmod 600 mosquitto/config/passwords
docker compose restart mosquitto
```

### Telegraf reinicia com `lookup mosquitto on 127.0.0.11:53: no such host`

Mosquitto não está saudável. Telegraf se cura sozinho assim que Mosquitto subir.

### Stick mostra "Error: Unknown Shine Stick" nos logs

Você esqueceu o patch da seção 1.4 (autodetecção). Recompile e reflashe.

### Painel "Energia Total" igual a "Energia Hoje"

Pode acontecer se você fez recentemente um write no registrador 235 (correção de PE Abnormal) — algumas firmwares de inversor resetam contadores cumulativos junto. Confirma comparando o valor de amanhã: se Total = Today + Hoje (geração de hoje), foi reset pontual.

### Dashboard mostra "No data" em algum painel

Field name pode não existir no seu protocolo. Abra o **Data Explorer** do InfluxDB UI, veja quais fields têm dados, e edite a query do painel.

## Créditos

- [OpenInverterGateway](https://github.com/OpenInverterGateway/OpenInverterGateway) — firmware substituto que tornou tudo isso possível
- Inspirado em projetos da comunidade Growatt no Reddit, GitHub e fóruns alemães (photovoltaikforum.com)
