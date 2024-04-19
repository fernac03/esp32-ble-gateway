# esp32-ble-gateway
Gateway WiFi para BLE (Bluetooth Low Energy) no ESP32 usando uma versão modificada do [protocolo WebSocket] da [Noble](https://github.com/abandonware/noble)(https://github.com/abandonware/noble) /blob/master/ws-slave.js). As modificações consistem em uma camada de autenticação adicional na conexão e algumas cargas extras aqui e ali. Ele foi projetado para ser usado com [ttlock-sdk-js](https://github.com/kind3r/ttlock-sdk-js) pelo menos até eu encontrar tempo para documentar a API e implementar uma ligação Noble separada. Observe que os fluxos de dados são via **websocket não criptografado padrão**, pois o ESP mal consegue lidar com os requisitos de memória para BLE, WiFi e WebSocket ao mesmo tempo e, além disso, o gateway deve estar em sua LAN e o tráfego BLE pode ser facilmente detectado pelo ar, então não faz sentido criptografar todas as comunicações neste momento.  

> Nota: isso não tem nada a ver com o gateway oficial TTLock G2, é basicamente apenas um proxy GATT sobre WiFi.


## O que funciona por enquanto
- Inicialização WiFi com configuração estilo AP via página HTTPS
- Comunicação Websocket e autenticação AES 128 CBC
- Iniciar/parar varredura BLE
- Descubra dispositivos
- Leia as características
- Escrever características
- Assine a característica

## Como instalar

Este breve guia explica como instalar o gateway e configurar o [complemento TTLock Home Assistant](https://github.com/kind3r/hass-addons) para que você possa interligá-lo com um bloqueio TTLock. Você mesmo precisa compilar e fazer upload do binário, não existe uma versão pré-compilada, mas o processo deve ser bastante fácil, mesmo para iniciantes.
### Requisitos

1. Extensão [Visual Studio Code (VSCode)](https://code.visualstudio.com/) e [PlatformIO](https://platformio.org/).
2. Um clone deste repositório
3. Uma **placa ESP32-WROVER** funcional ou esp32-wroom-32
4. Algum tipo de bloqueio TTLock emparelhado com uma instalação funcional do Home Assistant com [complemento TTLock Home Assistant](https://github.com/fernac03/hass-addons)
5. 
### Preparando o ESP32

Abra o repositório clonado no VSCode e o PlatformIO deverá instalar automaticamente todas as dependências necessárias (isso levará alguns minutos, dependendo do seu computador e da velocidade da Internet, seja paciente e deixe *resolver*). Você precisa modificar `sdkconfig.h` localizado em `.platformio/packages/framework-arduinoespressif32/tools/sdk/include/config` e alterar `CONFIG_ARDUINO_LOOP_STACK_SIZE` para `10240`. Isso ocorre porque a geração do certificado HTTPS ocupa mais espaço de pilha.

> No momento o projeto está configurado para funcionar apenas em **placas ESP32-WROVER**. Se você tiver uma placa diferente, precisará editar o arquivo `platformio.ini` e criar sua própria configuração de ambiente. No momento em que este artigo foi escrito, o código ocupava cerca de 1,5 MB, então estou usando o esquema de partição `min_spiffs.csv` para poder fazer OTA no futuro.

Conecte seu ESP32 ao PC, vá ao menu PlatformIO (a cabeça alienígena na barra de ferramentas esquerda do VSCode, onde você tem arquivos, pesquisa, plugins etc.) e em **Project Tasks** escolha **env:esp-wrover** -> **Plataforma** -> **Carregar Imager do Sistema de Arquivos**. Isso irá ‘formatar’ o armazenamento e fazer upload da UI da web.

Em seguida, você precisa construir e fazer upload do código principal. Em **Tarefas do Projeto** escolha **env:esp-wrover** -> **Geral** -> **Carregar e Monitorar**. Isso deve iniciar o processo de construção e assim que terminar o resultado compilado será carregado no ESP32.

Assim que o upload terminar, você deverá começar a ver alguns resultados de depuração, incluindo o status do AP WiFi e o status de geração do certificado HTTPS (isso levará algum tempo, então seja paciente). Após a conclusão da inicialização, você pode se conectar ao AP do ESP denominado **ESP32GW** com a senha **87654321** e acessar [https://esp32gw.local](https://esp32gw.local). O navegador reclamará do certificado autoassinado, mas você pode ignorar e continuar. O nome de usuário e a senha padrão são **admin/admin**. Configure suas credenciais wifi e copie a **Chave AES** que você precisa configurar no **complemento TTLock Home Assistant**.

Depois de salvar a nova configuração, o ESP irá reiniciar, conectar-se ao seu WiFi e gerar seu **endereço IP** na porta serial (ele também irá gerar um novo certificado HTTPS se você alterar seu nome). Também estará acessível via `esp32gw.local` (ou o novo nome que você deu) via MDNS se este serviço estiver funcionando em sua rede. Você ainda pode fazer alterações na configuração acessando seu endereço IP no navegador.

### Configurando HA

Depois de ter o ESP executando o software de gateway, vá para as opções de configuração do complemento **TTLock Home Assistant** e adicione o seguinte:

```yaml
portal: nobre
gateway_host: IP_ADDRESS_OF_YOUR_ESP
porta_gateway: 8080
chave_gateway: AES_KEY_FROM_ESP_CONFIG
usuário_gateway: administrador
gateway_pass: administrador
```

> O **usuário e a senha estão codificados** para admin/admin no momento, assim como a porta. Você só precisará atualizar o **endereço IP do gateway ESP** e a **chave AES que você gerou**.

Para obter informações extras de depuração, você pode adicionar a opção `gateway_debug: true` para registrar todas as comunicações de e para o gateway no Home Assistant.

Se tudo foi feito corretamente você agora poderá usar o addon usando o dispositivo ESP32 como gateway BLE.

## Pendência

- verifique se múltiplas conexões com múltiplos dispositivos são possíveis (`BLEDevice::createClient` parece armazenar apenas 1 `BLEClient`, mas poderíamos simplesmente criar o cliente nós mesmos)
- Filtragem UUID de serviço para verificação e permissão/proibição de duplicatas
- Tempo limite para conexões não autenticadas
- Investigue o wifi instável (às vezes ele se conecta, mas não há tráfego; tente executar ping no gw durante a configuração)
- Otimize a fragmentação da memória

### Pensamentos aleatórios

- a descoberta de dispositivos é sempre enviada a todos os clientes autenticados
- forneça a cada dispositivo um ID exclusivo (peripheralUuid) e armazene o ID, o endereço e o tipo de endereço em um mapa, conforme necessário para a conexão
- a conexão é feita com base em periféricoUuid traduzido para endereço e digite no noble_api
- sempre pare a digitalização antes de conectar a um dispositivo
- apenas um cliente pode se conectar a um dispositivo por vez, portanto associe o websocket à conexão e à limpeza ao desconectar
