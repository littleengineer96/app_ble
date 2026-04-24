# Protocolo BLE — app_ble × robocamv2

> Versão do protocolo: **2**  
> Formato: JSON sobre GATT (Write Without Response + Notify)

---

## 1. Visão Geral

Toda comunicação entre o app Flutter e o ESP32 usa um **envelope JSON uniforme**.  
O mesmo formato serve para comandos, respostas e eventos assíncronos.

```
App ──(Write TX)──► ESP32   →  comando
App ◄──(Notify RX)── ESP32  →  resposta ou evento
```

### UUIDs

| Role | UUID |
|---|---|
| Serviço principal | `0000EE00-0000-1000-8000-00805f9b34fb` |
| TX — app → ESP32 (Write) | `0000EE01-0000-1000-8000-00805f9b34fb` |
| RX — ESP32 → app (Notify) | `0000EE02-0000-1000-8000-00805f9b34fb` |

---

## 2. Envelope de Mensagem

### 2.1 Comando (App → ESP32)

```json
{
  "cmd": "namespace.action",
  "seq": 42,
  "data": { }
}
```

| Campo | Tipo | Descrição |
|---|---|---|
| `cmd` | string | Identificador `namespace.action` |
| `seq` | int | Número sequencial da requisição (≥ 1) |
| `data` | object | Parâmetros do comando. Omitir se vazio |

### 2.2 Resposta (ESP32 → App)

**Sucesso:**
```json
{ "seq": 42, "ok": true, "data": { } }
```

**Erro:**
```json
{ "seq": 42, "ok": false, "err": "CODIGO_ERRO", "msg": "Descrição legível" }
```

| Campo | Tipo | Descrição |
|---|---|---|
| `seq` | int | Mesmo `seq` do comando que originou a resposta |
| `ok` | bool | `true` = sucesso, `false` = erro |
| `data` | object | Dados retornados (apenas quando `ok: true`) |
| `err` | string | Código de erro em SNAKE_UPPER_CASE |
| `msg` | string | Mensagem de erro legível (debug) |

### 2.3 Evento Assíncrono (ESP32 → App)

O ESP32 pode enviar eventos sem que o app tenha feito um comando.  
Identificados por `seq: 0`.

```json
{ "seq": 0, "evt": "namespace.evento", "data": { } }
```

---

## 3. Fluxo de Sessão

```
1. App conecta ao dispositivo BLE
2. App subscreve notificações (CCCD)
3. App envia sync.request  OU  sync.verify (se já tem token)
4. App envia sys.info  → descobre capacidades do device
5. App renderiza UI com base nas capabilities recebidas
6. App envia comandos normais
```

### Exemplo — primeira conexão

```json
// → App envia
{ "cmd": "sync.request", "seq": 1 }

// ← ESP32 responde
{ "seq": 1, "ok": true, "data": { "token": "A3F9C1D2E4B8761F" } }
```

### Exemplo — reconexão com token salvo

```json
// → App envia
{ "cmd": "sync.verify", "seq": 1, "data": { "token": "A3F9C1D2E4B8761F" } }

// ← ESP32 responde (sucesso)
{ "seq": 1, "ok": true }

// ← ESP32 responde (token inválido)
{ "seq": 1, "ok": false, "err": "INVALID_TOKEN", "msg": "Token não reconhecido" }
```

### Exemplo — descoberta de capacidades

```json
// → App envia (pode ser antes ou depois do sync)
{ "cmd": "sys.info", "seq": 2 }

// ← ESP32 responde (RoboCam)
{
  "seq": 2, "ok": true,
  "data": {
    "type": "robocam",
    "fw": "2.4.1",
    "proto": 2,
    "caps": [
      "sync.request", "sync.verify", "sync.reset",
      "sys.info", "sys.reboot",
      "cfg.get", "cfg.set", "test.run",
      "motor.move", "motor.home", "motor.stop", "motor.jog",
      "pantilt.set", "pantilt.home",
      "cam.capture"
    ]
  }
}
```

---

## 4. Catálogo de Comandos

### Autenticação / Sessão

| Comando | Auth? | Parâmetros | Retorno |
|---|---|---|---|
| `sync.request` | Não | — | `token` |
| `sync.verify` | Não | `token` | — |
| `sync.reset` | Sim | — | — |
| `sync.status` | Não | — | `is_synced`, `mac` |
| `sys.info` | Não | — | `type`, `fw`, `proto`, `caps[]` |
| `sys.reboot` | Sim | — | — |

### Configuração (todos os devices)

| Comando | Auth? | Parâmetros | Retorno |
|---|---|---|---|
| `cfg.get` | Sim | `section` | campos da seção |
| `cfg.set` | Sim | `section` + campos | — |

Seções disponíveis: `wifi`, `mqtt01`, `mqtt02`, `ota`  
Seções por device: `motor`, `camera` (RoboCam) / `motor_flap`, `motor_abertura`, `rf`, `ir`, `buzzer` (Central Flap)

### Testes de desenvolvimento (todos os devices)

| Comando | Auth? | Parâmetros |
|---|---|---|
| `test.run` | Sim | `id` (ex: `"heap"`, `"mqtt"`, `"led"`, `"reboot"`) |

### RoboCam

| Comando | Auth? | Parâmetros |
|---|---|---|
| `motor.move` | Sim | `distance_mm` (float) |
| `motor.home` | Sim | — |
| `motor.stop` | Sim | — |
| `motor.jog` | Sim | `dir`: `"left"` / `"right"` / `"stop"` |
| `pantilt.set` | Sim | `pan` (float, −180/+180), `tilt` (float, −90/+90) |
| `pantilt.home` | Sim | — |
| `cam.capture` | Sim | — |

### Central Flap

| Comando | Auth? | Parâmetros |
|---|---|---|
| `flap.move` | Sim | `dir`: `"up"` / `"down"` / `"left"` / `"right"` / `"stop"` |

---

## 5. Eventos Assíncronos (ESP32 → App)

| Evento | Quando | `data` |
|---|---|---|
| `motor.done` | Movimento linear concluído | `pos_mm` |
| `motor.fault` | Endstop atingido / erro | `reason` |
| `telemetry` | Periódico (intervalo configurável) | `pos_mm`, `pan`, `tilt`, `temp_c`, `heap_kb` |
| `motion_detected` | MPU9250 detectou vibração | `accel` |

### Exemplo

```json
{ "seq": 0, "evt": "motor.done",  "data": { "pos_mm": 250.0 } }
{ "seq": 0, "evt": "motor.fault", "data": { "reason": "ENDSTOP_MAX" } }
{ "seq": 0, "evt": "telemetry",   "data": { "pos_mm": 250.0, "pan": 45.0, "tilt": -10.0, "temp_c": 28.3, "heap_kb": 142 } }
```

---

## 6. Códigos de Erro

| Código | Significado |
|---|---|
| `PARSE_ERROR` | JSON malformado |
| `UNKNOWN_CMD` | Comando não existe neste build |
| `NOT_AUTHED` | Requer autenticação prévia |
| `INVALID_TOKEN` | Token não reconhecido |
| `MISSING_PARAM` | Campo obrigatório ausente em `data` |
| `INVALID_PARAM` | Valor fora do range permitido |
| `ENDSTOP_MIN` / `ENDSTOP_MAX` | Fim de curso atingido |
| `MOTOR_BUSY` | Motor em movimento, comando ignorado |
| `NVS_ERROR` | Falha ao ler/gravar configuração |

---

## 7. Adicionando Novos Comandos

### 7.1 No firmware (C)

Vamos usar como exemplo um novo comando `led.set` que controla o brilho do LED de status.

**Passo 1 — Criar o handler** no arquivo do subsistema correto:

```c
// main/hardware/gpio/gpio_ble_handlers.c  (ou no device específico)

esp_err_t handle_led_set(cJSON *data, cJSON *out) {
    // 1. Ler parâmetros de 'data'
    cJSON *item = cJSON_GetObjectItem(data, "brightness");
    if (!item) {
        return ESP_ERR_INVALID_ARG;  // dispatcher enviará MISSING_PARAM
    }

    int brightness = (int)cJSON_GetNumberValue(item);
    if (brightness < 0 || brightness > 100) {
        return ESP_ERR_INVALID_ARG;  // dispatcher enviará INVALID_PARAM
    }

    // 2. Executar ação
    led_set_brightness(brightness);

    // 3. Preencher retorno em 'out' (opcional)
    cJSON_AddNumberToObject(out, "brightness", brightness);

    return ESP_OK;
}
```

**Passo 2 — Declarar o handler** no header:

```c
// main/hardware/gpio/gpio_ble_handlers.h

esp_err_t handle_led_set(cJSON *data, cJSON *out);
```

**Passo 3 — Registrar na tabela**, no bloco `#if` correto:

```c
// main/application/ble/ble_dispatcher.c

static const ble_cmd_entry_t cmd_table[] = {
    // ... outros comandos comuns ...

    // Disponível para todos os devices:
    { "led.set", handle_led_set, true },

    // Ou restrito a um device específico:
    // #if CONFIG_DEVICE_TYPE == _APP_DEVICE_ROBOCAM
    //     { "led.set", handle_led_set, true },
    // #endif

    { NULL, NULL, false }
};
```

Pronto. O comando aparece automaticamente na lista de `caps` retornada por `sys.info`.

---

### 7.2 No app Flutter

**Passo 1 — Enviar o comando** via `BleService`:

```dart
// lib/services/ble_service.dart  (método já existente, só usar)

await bleService.sendCommand(
  cmd: 'led.set',
  data: { 'brightness': 75 },
);
```

**Passo 2 — Tratar a resposta** (se precisar do retorno):

```dart
final response = await bleService.sendCommand(
  cmd: 'led.set',
  data: { 'brightness': 75 },
);

if (response['ok'] == true) {
  final brightness = response['data']['brightness'];
  print('LED ajustado para $brightness%');
} else {
  print('Erro: ${response['err']}');
}
```

**Passo 3 — Renderizar o controle condicionalmente** (se for feature exclusiva de device):

```dart
// Em qualquer Screen, após sys.info já ter sido processado:

if (context.read<AppState>().capabilities.contains('led.set'))
  LedBrightnessSlider(
    onChanged: (value) => bleService.sendCommand(
      cmd: 'led.set',
      data: { 'brightness': value },
    ),
  ),
```

---

### 7.3 Ouvir eventos assíncronos no app

Se o novo subsistema também emite eventos (ex: `led.changed` quando alguém muda o LED via botão físico):

```dart
// lib/services/ble_service.dart  (no stream de notificações)

bleService.eventStream
  .where((evt) => evt['evt'] == 'led.changed')
  .listen((evt) {
    final brightness = evt['data']['brightness'];
    context.read<AppState>().updateLedBrightness(brightness);
  });
```

---

## 8. Fragmentação de Payloads Grandes

O MTU efetivo do BLE é ~244 bytes. Payloads maiores (ex: `cfg.set` com muitos campos) devem ser fragmentados pelo app:

```json
// Fragmento 1/2
{ "cmd": "cfg.set", "seq": 10, "chunk": 1, "total": 2,
  "data": { "section": "wifi", "ssid_01": "MinhaRede", "pass_01": "senha123" } }

// Fragmento 2/2
{ "cmd": "cfg.set", "seq": 10, "chunk": 2, "total": 2,
  "data": { "ssid_02": "RedeBackup", "pass_02": "outrasenha" } }
```

O ESP32 acumula os chunks pelo `seq` e despacha apenas quando `chunk == total`.  
Mensagens sem o campo `chunk` são tratadas como payload único (`chunk: 1, total: 1`).

---

## 9. Referência Rápida

```
Novo comando para TODOS os devices
└── Handler em arquivo do subsistema
└── Linha no bloco comum da cmd_table
└── Nada mais — aparece automaticamente em sys.info/caps

Novo comando para UM device
└── Handler em arquivo do device (com ou sem #if)
└── Linha dentro do bloco  #if CONFIG_DEVICE_TYPE == _APP_DEVICE_XYZ
└── App verifica caps.contains('meu.cmd') antes de mostrar o widget

Novo evento assíncrono
└── ESP32 chama  ble_send_event("meu.evt", payload_json)  quando ocorre
└── App escuta  bleService.eventStream.where((e) => e['evt'] == 'meu.evt')
```
