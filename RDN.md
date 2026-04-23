# RDN — Requisitos de Desenvolvimento da Aplicação

> Documento baseado no simulador interativo `simulador.html`.  
> Referência de firmware: `robocamv2/` (ESP-IDF, ESP32).  
> Empresa: **Little Engenharia**

---

## 1. Visão Geral

Aplicativo Flutter genérico para comunicação BLE com dispositivos ESP32 da Zenitum Engenharia.  
O app identifica o tipo do dispositivo pelo nome BLE e carrega o plugin correspondente (Central Flap, RoboCam, RC Car).

### 1.1 Tipos de Dispositivo Suportados

| Tipo (plugin) | Prefixo BLE | Ícone | Cor |
|---|---|---|---|
| `central_flap` | `Central Flap_` | 🎛 | Azul |
| `robocam` | `RoboCam_` | 📷 | Laranja |
| `rc_car` | `RC Car_` | 🚗 | Roxo |
| `unknown` | — | ❓ | Cinza (não clicável) |

### 1.2 Perfis de Usuário

| Perfil | Acesso |
|---|---|
| **Usuário Comum** | Todas as telas exceto "Testes Dev" |
| **Desenvolvedor** | Todas as telas, incluindo "Testes Dev" por dispositivo |

---

## 2. Arquitetura

```
lib/
├── main.dart
├── config/
│   └── app_config.dart          # UUIDs BLE, timeouts, credenciais demo
├── models/
│   ├── ble_device.dart          # {id, name, type, rssi, icon, displayType}
│   └── app_session.dart         # {email, role, isGuest}
├── services/
│   ├── ble_service.dart         # scan, connect, discover, write, notify
│   ├── auth_service.dart        # login, sessão, SharedPreferences
│   └── device_registry.dart     # persistência de dispositivos (SharedPrefs)
├── screens/
│   ├── splash_screen.dart
│   ├── login_screen.dart
│   ├── home_screen.dart
│   ├── connecting_screen.dart
│   ├── device_home_screen.dart
│   ├── device_config_screen.dart
│   └── dev_tests_screen.dart
├── plugins/
│   ├── robocam/
│   │   └── robocam_ctrl_screen.dart
│   └── central_flap/
│       └── central_flap_ctrl_screen.dart
└── widgets/
    ├── device_card.dart
    ├── cfg_section.dart
    └── test_button.dart
```

### 1.3 Pacotes Flutter Necessários

```yaml
dependencies:
  flutter_blue_plus: ^1.31.0   # BLE (scan, connect, write, notify)
  shared_preferences: ^2.2.0   # persistência de sessão e registro
  provider: ^6.1.0             # gerenciamento de estado
  local_auth: ^2.1.8           # TODO: biometria
  camera: ^0.10.5              # TODO: stream de câmera (RoboCam)
  permission_handler: ^11.0.0  # permissões BLE + câmera + localização
```

---

## 3. BLE — Protocolo de Comunicação

### 3.1 UUIDs

| Nome | UUID | Tipo |
|---|---|---|
| Serviço Principal | `0000EE00-0000-1000-8000-00805f9b34fb` | Service |
| Característica TX (app→ESP) | `0000EE01-0000-1000-8000-00805f9b34fb` | Write Without Response |
| Característica RX (ESP→app) | `0000EE02-0000-1000-8000-00805f9b34fb` | Notify |

### 3.2 Procedimento de Conexão (4 etapas)

```
1. device.connect()                          timeout: 10 s
2. discoverServices()                        busca serviço 0x00EE
3. setNotifyValue(rxChar, true)              CCCD write [0x01, 0x00] + delay 200 ms
4a. Sem token salvo  → exibe botão "Sincronizar" → envia SYNC_REQUEST
4b. Com token salvo  → envia SYNC_VERIFY:<TOKEN>
    Sucesso           → botão "Controlar Dispositivo" → CtrlScreen (RoboCam ou Central Flap)
```

### 3.3 Comandos por Característica TX

#### Autenticação / Sessão
| Comando | Descrição |
|---|---|
| `SYNC_REQUEST` | Solicita vinculação (1ª vez) |
| `SYNC_VERIFY:<TOKEN>` | Reautenticação com token armazenado |
| `SYNC_RESET` | Desvincula dispositivo, apaga token |

#### RoboCam
| Comando | Descrição |
|---|---|
| `Pan:<valor>` | Move câmera horizontal (−180 a +180°) |
| `Tilt:<valor>` | Move câmera vertical (−90 a +90°) |
| `GoHome` | Centraliza Pan=0 e Tilt=0 |
| `Motor:LEFT` | Move trilho para esquerda (enquanto pressionado) |
| `Motor:RIGHT` | Move trilho para direita (enquanto pressionado) |
| `Motor:STOP` | Para motor linear |
| `CAM:CAPTURE` | Dispara captura de foto |

#### Central Flap
| Comando | Descrição |
|---|---|
| `Flap Up` | Sobe |
| `Flap Down` | Desce |
| `Flap Left` | Esquerda |
| `Flap Right` | Direita |
| `Flap Stop` | Para |

#### Configurações (todos os dispositivos)
| Comando | Seção NVS |
|---|---|
| `CFG:WIFI:<json>` | ssid_01/02, pass_01/02, static_ip_on, ip, gw, mask |
| `CFG:MQTT01:<json>` | broker, port, user, pass, pub, sub, tsend |
| `CFG:MQTT02:<json>` | idem, broker 2 |
| `CFG:OTA:<json>` | server_url, timer, enable |
| `CFG:MOTOR:<json>` | max_speed, acceleration, microsteps, wheel_diam (RoboCam) |
| `CFG:MOTOR_FLAP:<json>` | speed_pct (0–100 %) — Motor de Giro DC (Central Flap) |
| `CFG:MOTOR_ABERTURA:<json>` | current_limit_a (A) — Motor de Abertura, proteção contra travamento (Central Flap) |
| `CFG:RF:<json>` | rf_enable, open/close/right/left/stop codes |
| `CFG:IR:<json>` | ir_enable, open/close/right/left/stop codes |
| `CFG:BUZZER:<on/off>` | buzzer_enable |
| `CFG:CAMERA:<json>` | ip, port, user, pass, proto, auth, path, channel (RoboCam) |

#### Testes Dev
| Comando | Dispositivo | Descrição |
|---|---|---|
| `TEST:MOTOR:FWD:50` | RoboCam | Avançar 50 mm |
| `TEST:MOTOR:BWD:50` | RoboCam | Recuar 50 mm |
| `TEST:MOTOR:HOME` | RoboCam | Homing (busca endstop MIN) |
| `TEST:MOTOR:STOP` | RoboCam | Para motor |
| `TEST:ENDSTOP:MIN` | RoboCam | Ler fim de curso MIN |
| `TEST:ENDSTOP:MAX` | RoboCam | Ler fim de curso MAX |
| `TEST:PAN:+90` | RoboCam | Pan +90° |
| `TEST:PAN:-90` | RoboCam | Pan −90° |
| `TEST:TILT:+45` | RoboCam | Tilt +45° |
| `TEST:TILT:-45` | RoboCam | Tilt −45° |
| `TEST:PANTILT:CENTER` | RoboCam | Centraliza pan/tilt |
| `TEST:AHT20` | RoboCam | Ler temperatura + umidade |
| `TEST:BMP280` | RoboCam | Ler pressão + altitude |
| `TEST:MPU:READ` | RoboCam | Ler acelerômetro |
| `TEST:MPU:CALIB` | RoboCam | Calibrar MPU9250 |
| `TEST:MPU:MOTION` | RoboCam | Detectar movimento |
| `TEST:FLAP:OPEN` | Flap | Girar motor → abrir |
| `TEST:FLAP:CLOSE` | Flap | Girar motor → fechar |
| `TEST:FLAP:RIGHT` | Flap | Girar direita |
| `TEST:FLAP:LEFT` | Flap | Girar esquerda |
| `TEST:FLAP:STOP` | Flap | Para motor DC |
| `TEST:FLAP:CURRENT` | Flap | Leitura de corrente (mA) |
| `TEST:FC:<dir>` | Flap | Ler endstop: open/close/right/left/up/down |
| `TEST:RF:<cmd>` | Flap | Aguardar código RF: open/close/right/left/stop |
| `TEST:IR:<cmd>` | Flap | Aguardar código IR: open/close/right/left/stop |
| `TEST:HEAP` | Todos | Heap livre (KB) |
| `TEST:MQTT` | Todos | Publicar telemetria |
| `TEST:LED` | Todos | Piscar LED de status |
| `TEST:REBOOT` | Todos | esp_restart() |

---

## 4. Fluxo de Telas

```
Splash (2 s)
  └─► Login
        ├─► [credenciais válidas / visitante]
        └─► Home (scan BLE contínuo)
              └─► [toca em dispositivo]
                    └─► Connecting (4 etapas BLE)
                          └─► [plugin ctrl direto]
                                ├─► RoboCam Ctrl  ──┬─► Device Config
                                └─► Central Flap Ctrl ┤  └─► [voltar → ctrl]
                                                     └─► Dev Tests ← [só Desenvolvedor]
                                                           └─► [voltar → ctrl]

⚠️  Device Home Screen existe mas está fora do fluxo principal.
    Acessível apenas via sidebar do simulador (ferramenta de dev).
```

---

## 5. Telas — Especificação Detalhada

### 5.1 Splash Screen
- Duração: ~2 s com barra de progresso animada
- Logo + nome do app + "Zenitum Engenharia"
- **Sempre** navega para LoginScreen (sem bypass de sessão aqui)
- TODO: personalizar logo e nome final

### 5.2 Login Screen
- Campos: e-mail + senha (com toggle de visibilidade)
- **Seletor de Perfil**: 👤 Usuário Comum / 🔧 Desenvolvedor
- Botão "Entrar como Visitante" (acesso direto sem credenciais, mas com perfil selecionado)
- Credenciais demo: `admin@littleeng.com` / `1234`
- Erro exibido inline (sem dialog)
- Sessão salva em `SharedPreferences`: `{email, role, isGuest}`
- TODO: Firebase Auth ou backend real
- TODO: biometria (`local_auth`)
- TODO: fluxo "Esqueci senha"

### 5.3 Home Screen
- AppBar com saudação + botão logout
- **Card "Dispositivo Conectado"** (visível se há conexão ativa): nome, tipo, botão "Controlar"
- **Lista de Scan BLE** atualizada a cada 3 s:
  - RSSI varia ±4 dBm por ciclo
  - Identificação automática pelo prefixo do nome
  - Chip colorido por tipo
  - Dispositivos `unknown` mostrados mas não clicáveis
  - Barras de sinal (3 níveis): ≥−65 / −65 a −80 / <−80
- Toque no card → ConnectingScreen

### 5.4 Connecting Screen
- 4 etapas com ícones animados:
  1. 📶 Conectando ao dispositivo
  2. 🔍 Descobrindo serviços BLE
  3. 🔗 Inscrevendo notificações (CCCD)
  4. 🔐 Autenticação / Sincronização
- Timeout por etapa: 10 s com mensagem de erro + botão "Tentar Novamente"
- Após sucesso: botão "Controlar Dispositivo" → navega direto para a tela de controle do plugin (`RoboCamCtrlScreen` ou `CentralFlapCtrlScreen`)
- Token salvo por endereço MAC: `prefs.setString('token_$mac', token)`

### 5.5 Device Home Screen _(fora do fluxo principal)_
> Esta tela foi removida do fluxo de navegação. Após conectar, o app navega diretamente para a tela de controle do plugin. A Device Home permanece acessível somente como ferramenta de diagnóstico interno (sidebar do simulador).

- AppBar com nome + status online (ponto verde) + botão desconectar (×)
- **Banner**: ícone grande do dispositivo, nome, chip de tipo
- **Telemetria rápida** (apenas RoboCam): Posição (m), Pan (°), Temperatura (°C)
- **3 cards de ação**: Controlar, Configurar, Testar (Desenvolvedor)
- Botão voltar → HomeScreen

### 5.6 Device Config Screen
- AppBar com chip de tipo do dispositivo
- Botão voltar → tela de controle do plugin (RoboCam Ctrl ou Central Flap Ctrl)
- Seções expansíveis por dispositivo:

#### Seções Comuns (todos os dispositivos)
**📶 Rede WiFi**
- SSID Rede 1 + Senha
- SSID Rede 2 + Senha (opcional)
- Toggle "IP Estático" → expande campos: IP, Gateway, Máscara de sub-rede
- Salvar → `CFG:WIFI:<json>`

**📡 MQTT**
- Broker, Porta, Usuário, Senha
- Tópico Publicação, Tópico Subscrição
- Intervalo de envio (s)
- (Suporte a 2 brokers: seções MQTT 01 e MQTT 02)
- Salvar → `CFG:MQTT01:<json>` / `CFG:MQTT02:<json>`

**🔄 OTA**
- URL do servidor
- Intervalo de verificação (min)
- Toggle "OTA Automático"
- Salvar → `CFG:OTA:<json>`

#### Seções Exclusivas — RoboCam
**⚙️ Motor Linear (A4988 / NEMA17)**
- Velocidade máxima (mm/s) — padrão: 100
- Aceleração (mm/s²) — padrão: 500
- Microsteps: select 1 / 1/2 / 1/4 / 1/8 / 1/16 (padrão: 1/16 = 3200 passos/rev)
- Diâmetro da roda de tração (mm) — padrão: 23.80
- Salvar → `CFG:MOTOR:<json>`

**📷 Câmera IP**
- Endereço IP da câmera na rede local — padrão: `192.168.3.129`
- Porta HTTP — padrão: `80`
- Usuário / Senha
- Protocolo: `MJPEG (HTTP)` | `RTSP` | `Snapshot CGI`
- Autenticação: `Digest Auth` | `Basic Auth` | `Sem autenticação`
- Path do stream — padrão: `/cgi-bin/mjpg/video.cgi`
- Canal / subtype (RTSP) — padrão: `channel=1&subtype=0`
- Salvar → `CFG:CAMERA:<json>`

#### Seções Exclusivas — Central Flap
**📻 Comunicação RF**
- Toggle "RF Habilitado"
- Tabela de códigos: Abrir / Fechar / Direita / Esquerda / Stop
  - Botão **Gravar**: envia `RF_WRITE:<cmd>`, aguarda sinal no receptor
  - Botão **Deletar**: envia `RF_DEL:<cmd>`
- Salvar → `CFG:RF:<json>`

**💡 Comunicação IR**
- Idem RF, protocolo NEC
- Salvar → `CFG:IR:<json>`

**⚙️ Motor de Giro (DC)**
- Velocidade (%) — range 0–100 %, step 5, padrão: 50 %
- Salvar → `CFG:MOTOR_FLAP:<json>`

**⚡ Motor de Abertura — Proteção contra Travamento**
- O firmware monitora a corrente do motor de abertura e interrompe o movimento ao ultrapassar o limite configurado
- Corrente Limite (A) — range 0–10 A, step 0.1, padrão: 0.8 A
- Salvar → `CFG:MOTOR_ABERTURA:<json>`

**🔔 Buzzer**
- Toggle "Buzzer Habilitado"
- Salvar → `CFG:BUZZER:<on/off>`

### 5.7 RoboCam Ctrl Screen
- **AppBar**: botão `←` (voltar → HomeScreen) + nome do dispositivo + status online
  - Canto superior direito (ícones minimalistas):
    - ⚙ → DeviceConfigScreen
    - 🔧 → DevTestsScreen (**visível somente para perfil Desenvolvedor**)
    - ✕ → desconectar
- **Preview de câmera** (topo):
  - Área 16:9 (stream MJPEG via HTTP usando config de câmera — TODO integração real)
  - Badge "⦿ AO VIVO" piscando
  - Crosshair central
  - Info: resolução e FPS
- **Slider Pan** (−180° a +180°) → envia `Pan:<val>`
- **Slider Tilt** (−90° a +90°) → envia `Tilt:<val>`
- **Botões de movimento linear**:
  - `←` (segurar → `Motor:LEFT`, soltar → `Motor:STOP`)
  - `⏹ STOP` (central, vermelho)
  - `→` (segurar → `Motor:RIGHT`, soltar → `Motor:STOP`)
- **Botões Tilt step**: ↑ / 🏠 Home / ↓
- **Botão 📷 Tirar Foto**: envia `CAM:CAPTURE`, efeito de flash, contador de fotos na sessão
- Último comando exibido em badge inferior

### 5.8 Central Flap Ctrl Screen
- **AppBar**: botão `←` (voltar → HomeScreen) + nome do dispositivo + status online
  - Canto superior direito (ícones minimalistas):
    - ⚙ → DeviceConfigScreen
    - 🔧 → DevTestsScreen (**visível somente para perfil Desenvolvedor**)
    - ✕ → desconectar
- **D-Pad direcional** (5 botões):
  - ↑ Sobe / ↓ Desce / ← Esquerda / → Direita
  - ⏹ STOP (circular, vermelho, central)
- Último comando exibido abaixo do D-Pad
- Botão "📶 WiFi Config" → DeviceConfigScreen
- Botão "🔄 Desparecer" → envia `SYNC_RESET`, apaga token, volta para Home

### 5.9 Dev Tests Screen
- Visível somente para perfil **Desenvolvedor**
- Nome e tipo do dispositivo exibidos no AppBar
- Botão voltar → tela de controle do plugin (RoboCam Ctrl ou Central Flap Ctrl)
- Seções e botões de teste conforme tipo de dispositivo:

#### RoboCam
- ⚙️ Motor A4988/NEMA17: Avançar 50mm, Recuar 50mm, Homing, STOP, Endstop MIN, Endstop MAX
- 🎥 Câmera Pan/Tilt (28BYJ-48): Pan +90°, Pan −90°, Tilt +45°, Tilt −45°, Centralizar
- 🌡️ Sensores: AHT20 (temp/umidade), BMP280 (pressão/altitude)
- 📐 IMU MPU9250: Ler acelerômetro, Calibrar, Detectar movimento
- 🖥️ Sistema: Heap livre, MQTT, LED status, Reboot

#### Central Flap
- ⚙️ Motor DC: Girar Abrir, Girar Fechar, Direita, Esquerda, STOP, Leitura de Corrente
- 🔌 Fim de Curso: Aberto, Fechado, Direita, Esquerda, Sobe, Desce
- 📻 RF: Receber Abrir, Fechar, Direita, Esquerda, Stop
- 💡 IR: Receber Abrir, Fechar, Direita, Esquerda, Stop
- 🖥️ Sistema: Heap livre, MQTT, LED status, Reboot

#### Estado visual dos botões de teste
- **Normal** → cinza claro
- **Rodando** → amarelo + spinner
- **OK** → verde + ✓
- **Falha** → vermelho + ✗
- Barra de resultado abaixo de cada seção com saída textual da resposta BLE NOTIFY

---

## 6. Gerenciamento de Estado

```dart
// Estado global via Provider
class AppState extends ChangeNotifier {
  // Auth
  AppSession? session;          // {email, role, isGuest}

  // BLE
  BluetoothDevice? connectedDevice;
  String? connectedDeviceType;  // 'robocam' | 'central_flap' | 'rc_car'
  bool isConnected = false;

  // RoboCam
  double pan = 0;
  double tilt = 0;
  int photoCount = 0;

  // Sessão de dispositivos
  List<BleDevice> deviceRegistry = [];   // persistido em SharedPrefs
  List<ScanResult> scanResults = [];     // atualizado a cada 3 s
}
```

---

## 7. Persistência (SharedPreferences)

| Chave | Tipo | Descrição |
|---|---|---|
| `ble_session` | JSON String | `{email, role, isGuest}` |
| `ble_device_registry` | JSON String | Lista de dispositivos cadastrados |
| `ble_token_<MAC>` | String | Token de autenticação por dispositivo |

---

## 8. Permissões Necessárias

### Android (`AndroidManifest.xml`)
```xml
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.CAMERA" />
```

### iOS (`Info.plist`)
```xml
NSBluetoothAlwaysUsageDescription
NSLocationWhenInUseUsageDescription
NSCameraUsageDescription
```

---

## 9. Referência de Firmware (robocamv2)

### 9.1 Chaves NVS — WiFi
| Chave NVS | Descrição |
|---|---|
| `ssid_01` / `ssid_02` | SSIDs cadastrados |
| `pass_01` / `pass_02` | Senhas |
| `static_ip_on` | IP estático: `"1"` ou `"0"` |
| `static_ip` / `static_gw` / `static_mask` | Configuração IP estático |

### 9.2 Chaves NVS — MQTT
| Chave NVS | Descrição |
|---|---|
| `mqtt_broker01` / `mqtt_broker02` | Endereço dos brokers |
| `mqtt_port01` / `mqtt_port02` | Portas (padrão: 1883) |
| `mqtt_user01` / `mqtt_user02` | Usuários |
| `mqtt_pass01` / `mqtt_pass02` | Senhas |
| `mqtt_pub01` / `mqtt_pub02` | Tópicos de publicação |
| `mqtt_sub01` / `mqtt_sub02` | Tópicos de subscrição |
| `mqtt_tsend01` / `mqtt_tsend02` | Intervalo de envio (s) |

### 9.3 Chaves NVS — OTA
| Chave NVS | Descrição |
|---|---|
| `ota_server` | URL do servidor |
| `ota_timer` | Intervalo de verificação (min) |
| `ota_enable` | `"1"` ou `"0"` |

### 9.4 Motor Linear — RoboCam (A4988 + NEMA17)
- `steps_per_rev` = 200 (full step)
- Microstep padrão: `SIXTEENTH` → 3200 passos/rev
- `pulley_circumference` = π × `wheel_diam` (padrão: 23.80 mm)
- `max_speed`: mm/s; `acceleration`: mm/s²
- Endstop MIN: `_PIN_END_COURSE_1`; Endstop MAX: `_PIN_END_COURSE_2`

### 9.5 Pan/Tilt — RoboCam (28BYJ-48)
- Motor de passo unipolar, 4 pinos de bobina (pan + tilt)
- Redução: 64:1 → 2048 passos/rev

### 9.6 Motor DC — Central Flap

#### Motor de Giro (rotação do flap)
- Controle por PWM
- Velocidade configurável em porcentagem: 0–100 % (padrão: 50 %)
- O firmware mapeia internamente a % para o duty cycle PWM

#### Motor de Abertura (translação/abertura física)
- Monitoramento de corrente via sensor analógico (ADC)
- Corrente limite configurável em Amperes para detecção de travamento (padrão: 0.8 A)
- Ao ultrapassar o limite o firmware interrompe o movimento imediatamente
- Chave NVS: `motor_abertura_ilim` (float, A)

---

## 10. TODO — Funcionalidades Pendentes

- [ ] Integrar com Firebase Auth ou backend da Little Engenharia
- [ ] Biometria no login (`local_auth`)
- [ ] Fluxo "Esqueci senha"
- [ ] Stream de câmera no RoboCam — integrar URL montada a partir das configurações de câmera (IP, porta, path, auth) com widget MJPEG
- [ ] Snapshot sob demanda com salvar na galeria (`image_gallery_saver`) usando Digest Auth ou Basic Auth conforme configurado
- [x] ~~Tela de configuração da câmera IP~~ — implementado (IP, porta, usuário, senha, protocolo, auth, path, canal)
- [ ] Feedback háptico nos botões de controle
- [ ] Receber posição real do motor via BLE NOTIFY
- [ ] Receber ângulos reais de pan/tilt via BLE NOTIFY
- [ ] Receber telemetria (temperatura, corrente) em tempo real
- [ ] Validação de campos antes de enviar configurações via BLE
- [ ] Log de testes Dev com timestamp exportável
- [ ] Timeout + retry automático em cada etapa da conexão
- [ ] Suporte a OTA pelo próprio app (download + envio via BLE)
- [ ] Modo offline: cache de última configuração conhecida
- [ ] Push notifications quando dispositivo fica online/offline
