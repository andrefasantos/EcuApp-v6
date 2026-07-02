# ECU Control

App Android nativo (Kotlin) para se inscrever nos topicos MQTT da Thingable e
enviar comandos de bloqueio / desbloqueio das ECUs.

## O que faz (v1.3)

Tela inicial de **cards de maquinas** com **menu lateral** (drawer):

- **Maquinas** (home): cada ECU e um card minimalista com modelo, empresa,
  fabricante, SN, IMEI e a **ultima comunicacao** da ECU. Tocar no card abre o
  **dashboard** do dispositivo; o menu ⋮ edita/exclui; o botao abre
  Bloquear / Desbloquear / Resetar.
- **Dashboard do dispositivo**: telemetria parseada dos payloads /evt —
  versao FW (`vs`), sinal (`rssi`), temperatura (`temp`), MOP, rele (`RLS`),
  DI1/DI2/DI3, AI1..AI4 e localizacao (lat/lng) com **quando o GPS foi recebido
  pela ultima vez** (esses dados nao vem em todo frame). Tem tambem os botoes
  de bloqueio/desbloqueio/reset.
- **Cadastros**: gerencia listas reutilizaveis de fabricantes, modelos e
  empresas. Ao criar/editar um card, esses campos viram listas suspensas — da
  para escolher um existente ou digitar um novo (que e adicionado ao catalogo).
- **Registro (payloads)**: historico ao vivo com **duas buscas simultaneas**
  (topico E payload) e controles da **base gravada** (contador, exportar CSV,
  limpar com confirmacao).
- **Comando manual** e **Configuracao** (broker/topicos/conectar).

Cada maquina tem **payloads de bloqueio/desbloqueio proprios** (padrao
`open_relay` / `close_relay`; invertivel para torres de iluminacao). O **reset**
usa o mesmo topico com `{"type":"management","command":"reboot"}`.

O **IMEI** pode ser lido pelo **QR code** da ECU (formato
`IMEI:...SN:...`). Todos os payloads recebidos sao **gravados em banco local**
(SQLite) para consulta/exportacao posterior.

A **ultima comunicacao** so avanca com telemetria espontanea da ECU: conta
apenas frames no topico `/evt` que carregam `ts` (exclui ecos/respostas de
comando). Em **segundo plano**, o app roda enquanto estiver aberto (inclusive em
recentes) e **para ao ser fechado** (removido dos recentes) para poupar bateria;
ao reabrir, reconecta se estava conectado.


## Antes de usar: ajuste o formato de comando

O topico de comando e os payloads de bloqueio/desbloqueio vem com
**placeholders** e sao editaveis dentro do app (persistem entre sessoes):

- Topico de comando: `Thingable/aurabrasil/ECU/{imei}/cmd`  ← ajuste o sufixo real
- Payload de BLOQUEIO: `{"cmd":"block"}`  ← substitua pelo formato real da Thingable
- Payload de DESBLOQUEIO: `{"cmd":"unblock"}`  ← idem

O `{imei}` no topico e substituido pelo campo "IMEI da ECU alvo" na tela.
Assim voce nao precisa recompilar quando confirmar o formato correto.

## Compilar SEM privilegios de admin

O Gradle wrapper ja acompanha o projeto (`gradlew`, `gradlew.bat` e
`gradle-wrapper.jar`), entao nenhuma dessas opcoes exige instalar o
Android Studio nem ter admin.

### Opcao 1 (recomendada): GitHub Actions — build na nuvem

Nao instala nada na sua maquina. O APK e compilado nos servidores do GitHub.

1. Crie um repositorio no GitHub (pode ser privado).
2. Suba o conteudo desta pasta. Pela interface web: `Add file > Upload files`,
   arraste tudo (inclusive a pasta `.github`) e commit. Ou via git:
   ```
   git init && git add . && git commit -m "app"
   git branch -M main
   git remote add origin https://github.com/SEU_USUARIO/EcuControl.git
   git push -u origin main
   ```
3. O workflow `.github/workflows/build.yml` roda sozinho a cada push
   (ou manualmente na aba **Actions > build-apk > Run workflow**).
4. Ao terminar (uns 2-4 min), abra a execucao e baixe o artefato
   **EcuControl-debug-apk** — dentro esta o `app-debug.apk`.
5. Transfira o APK para o celular, habilite "instalar apps de fontes
   desconhecidas" e instale.

### Opcao 2: linha de comando, tudo em pasta do usuario (sem admin)

Nao precisa de admin — tudo fica em pastas suas.

1. JDK 17: baixe o Temurin 17 em formato **.zip** (nao o instalador .msi) do
   site da Adoptium, extraia para uma pasta sua e aponte
   `JAVA_HOME` para ela.
2. Android SDK command-line tools: baixe o zip "Command line tools only" do
   site do Android, extraia para ex. `C:\Users\voce\android-sdk\cmdline-tools\latest`.
   Defina `ANDROID_HOME=C:\Users\voce\android-sdk` e rode:
   ```
   sdkmanager "platform-tools" "platforms;android-34" "build-tools;34.0.0"
   ```
3. Na pasta do projeto:
   ```
   gradlew.bat assembleDebug        (Windows)
   ./gradlew assembleDebug          (Linux/Mac)
   ```
4. O APK sai em `app/build/outputs/apk/debug/app-debug.apk`.

### Opcao 3: Android Studio portatil (sem instalar)

Se sua politica permitir rodar executaveis extraidos: baixe o Android Studio
em **.zip** (nao o instalador .exe), extraia para uma pasta do usuario e rode
`bin\studio64.exe`. Ele instala o SDK em `%LOCALAPPDATA%`, sem admin. Alguns
ambientes corporativos bloqueiam isso (AppLocker) — se for o caso, use a opcao 1.

Requisitos gerais: JDK 17, Android SDK 34, minSdk 26 (Android 8.0).

## Observacoes

- `usesCleartextTraffic=true` esta ativo porque a 1883 e sem criptografia.
  Se migrar para TLS, marque "Usar TLS/SSL" e considere restringir o cleartext.
- A conexao vive enquanto o app esta aberto. Para operacao 24x7 em background,
  a evolucao natural e mover o cliente MQTT para um Foreground Service.
- A logica *safe-to-block* fica na propria ECU (via CAN). O app apenas emite o
  pedido; a decisao de efetivar o bloqueio permanece no dispositivo.

## Estrutura

```
app/src/main/java/br/com/andre/ecucontrol/
  MainActivity.kt      menu lateral (DrawerLayout + NavigationView)
  MachinesFragment.kt  cards (add/editar/excluir/rele/reset + QR)
  MachineAdapter.kt    card minimalista
  DashboardFragment.kt dashboard de telemetria por dispositivo
  Machine.kt           modelo da maquina/ECU (+ payloads e reset)
  MachineStore.kt      persistencia da lista (JSON) + LiveData
  CadastrosFragment.kt catalogos de fabricante/modelo/empresa
  CatalogStore.kt      persistencia dos catalogos + LiveData
  ConfigFragment.kt    tela Configuracao
  CommandFragment.kt   tela Comando manual
  LogFragment.kt       tela Registro (busca dupla + base)
  LogAdapter.kt        lista do historico
  PayloadDb.kt         banco SQLite de todos os payloads + export CSV
  MqttController.kt    estado + historico + telemetria + ultima comunicacao
  MqttService.kt       servico em primeiro plano (para ao fechar o app)
  MqttManager.kt       cliente Paho
  Prefs.kt             configuracao do broker
```
