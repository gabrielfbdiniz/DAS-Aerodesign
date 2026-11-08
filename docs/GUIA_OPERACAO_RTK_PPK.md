# Guia de Operação e Arquitetura: GNSS RTK e PPK de Alta Precisão (ZED-F9P)
**Projeto:** AeroDesign — DAS V2 (Data Acquisition System)  
**Módulo GNSS:** u-blox ZED-F9P-02B Multibanda (L1/L2)  
**Autor:** Equipe AeroDesign  

---

## 1. Visão Geral: Por que o RTK precisa de uma Estação de Solo?

O módulo GNSS de alta precisão **u-blox ZED-F9P** embarcado na placa DAS V2 opera nas frequências **L1 ($1575.42\text{ MHz}$)** e **L2 ($1227.60\text{ MHz}$)**.

Contudo, **se o avião voar utilizando apenas a placa DAS V2 a bordo, a precisão será métrica ($\sim 1.5\text{ m} \text{ a } 2.5\text{ m}$)**.

### A Física do Problema:
O sinal emitido pelos satélites GNSS (GPS, GLONASS, Galileo, BeiDou) percorre mais de 20.000 km no espaço e sofre distorções severas ao atravessar a **ionosfera** (plasma ionizado) e a **troposfera** (vapor de água e pressão atmosférica), além de pequenos erros de efemérides (órbita) e relógio atômico dos satélites.

Um receptor GNSS isolado no avião não tem como discernir se uma variação na medição de tempo decorreu de um deslocamento real da aeronave ou de um atraso atmosférico momentâneo.

---

## 2. Como Funciona a Precisão Centimétrica (Geodésia Diferencial)

Para atingir a precisão de **$1\text{ cm} \text{ a } 2\text{ cm}$**, a técnica RTK (*Real-Time Kinematic*) utiliza dois receptores em simultâneo:

```mermaid
flowchart TD
    SATS["Satélites GNSS (L1 / L2)"] -->|Sinais com Erro Atmosférico| BASE["Estação de Solo (Base Fixa)"]
    SATS -->|Mesmos Sinais com Erro| ROVER["DAS V2 no Avião (Rover)"]
    
    BASE -->|Calcula Erros Instantâneos| CORR["Mensagens RTCM 3.x\n(1005, 1074, 1084, 1094, 1124, 1230)"]
    CORR -->|Enlace de Rádio RF (Telemetria)| ROVER
    
    ROVER -->|Subtrai Erros + Solução da Fase da Portadora| SOL["RTK Fixed: Precisão de 1 cm!"]
```

1. **A Base (Estação de Solo):** Permanece estática em um tripé no solo. Como sua posição não varia, qualquer flutuação nas medições é puramente ruído atmosférico e erro de satélite.
2. **Mensagens RTCM 3.x:** A base calcula os desvios e gera pacotes diferenciais compactos padronizados pelo comitê internacional de radiocomunicação marítima/terrestre (**RTCM 3.x**).
3. **O Rover (DAS V2 no Avião):** Recebe o fluxo RTCM em tempo real via rádio e resolve a ambiguidade da **fase da onda portadora (*Carrier Phase*)**, atingindo o estado **RTK Fixed** ($1\text{ cm} - 2\text{ cm}$).

---

## 3. Reaproveitamento do Hardware DAS V2 como Estação de Solo

A equipe **não precisa projetar nem comprar uma placa de solo dedicada**. A própria placa **DAS V2** pode ser montada e utilizada como a Estação Base de Solo da equipe.

### Por que a mesma placa funciona?
O circuito integrado **u-blox ZED-F9P** não possui variantes de hardware para "Base" ou "Rover". O chip é idêntico; a função de Base é habilitada puramente por software/configuração de registradores (`CFG-TMODE-*`).

### Como operar o DAS V2 como Base na Pista:
1. **Alimentação:** Conectar um cabo USB-C (conector `J2`) a um notebook da equipe na barraca de solo ou a um simples *powerbank* USB de 5V.
2. **Antena:** Rosquear uma antena ativa GNSS multibanda (L1/L2 com plano de terra) no conector SMA `J8` da placa de solo e fixá-la em um tripé estável.
3. **Configuração via u-center (Software oficial u-blox):**
   * Abrir o software **u-center** no notebook conectado via USB.
   * Acessar `View -> Configuration View -> TMODE3 (Time Mode 3)`.
   * Selecionar **Survey-In**:
     * *Minimum Duration:* $180\text{ s}$ (3 minutos).
     * *Position Accuracy Limit:* $2.0\text{ m}$.
   * Habilitar as mensagens de saída RTCM3 na porta UART conectada ao rádio:
     * `1005` (Coordenadas da antena da base) a cada 5s.
     * `1074` (GPS MSM4) a 1 Hz.
     * `1084` (GLONASS MSM4) a 1 Hz.
     * `1094` (Galileo MSM4) a 1 Hz.
     * `1124` (BeiDou MSM4) a 1 Hz.
     * `1230` (GLONASS Code-Phase Biases) a cada 5s.
4. **Transmissão:** Conectar o rádio de telemetria de solo (SiK Telemetry 915 MHz, Holybro ou LoRa) no conector `J6` ou `J9` do DAS V2. O rádio transmitirá os pacotes RTCM para o ar automaticamente.

---

## 4. RTK em Tempo Real vs. PPK Pós-Voo

A equipe tem à disposição duas metodologias complementares de alta precisão:

| Característica | RTK (Real-Time Kinematic) | PPK (Post-Processed Kinematic) |
| :--- | :--- | :--- |
| **Momento do Cálculo** | Durante o voo (em tempo real) | Após o voo (no computador) |
| **Necessidade de Rádio RF** | **Sim** (fluxo contínuo de dados RTCM) | **Não** (apenas gravação local) |
| **Sensibilidade a Perda de Link** | Se o rádio cair, degrada para Float/Autônomo | **Imune** (não depende de link de rádio) |
| **Onde a Posição é Gravada** | Direto no log de voo | Gerada via software RTKLIB no PC |
| **Precisão Típica** | $1\text{ cm} - 2\text{ cm}$ | **Sub-centimétrica ($< 1\text{ cm}$)** |

### Por que um GPS Comum NÃO Consegue Fazer PPK?
Muitos membros da equipe perguntam se um GPS automotivo ou de drone comum (ex: NEO-6M, NEO-7M, NEO-M8N padrão) poderia ser usado para fazer PPK:
* **A Resposta é NÃO.**
* Um GPS comum mede apenas o tempo de trânsito dos pulsos digitais (*Pseudorange* do código C/A). Esse pulso tem um comprimento de quase **300 metros**, limitando a resolução a alguns metros.
* Para fazer PPK ou RTK, o receptor precisa medir frações da **onda portadora de rádio (*Carrier Phase*)** na frequência de $1.575\text{ GHz}$ (comprimento de onda de apenas **$19\text{ centímetros}$**).
* Receptores convencionais de baixo custo **bloqueiam ou não disponibilizam essas medições de onda** em seu firmware.
* O **ZED-F9P** exporta os pacotes brutos geodésicos necessários:
  * `UBX-RXM-RAWX`: Medições brutas de pseudodistância, fase da portadora (*carrier phase*), medição Doppler e relação sinal-ruído ($C/N_0$) de cada satélite.
  * `UBX-RXM-SFRBX`: Efemérides e dados de navegação transmitidos pelos satélites.

---

## 5. Como Executar o PPK Pós-Voo com o DAS V2

Se a equipe optar pelo PPK (ou se o rádio de telemetria sofrer interferência durante uma rodada da competição), o procedimento para obter os gráficos centimétricos para o relatório de voo da SAE é:

1. **Configuração no Avião:**
   * Garantir que o firmware do STM32G431 salve os pacotes `RAWX` e `SFRBX` do ZED-F9P no cartão **MicroSD** (via barramento SPI da folha `05_Storage_Telemetry.kicad_sch`).
2. **Obtenção dos Dados da Base:**
   * **Opção 1 (Base da Equipe):** A placa DAS V2 de solo gravou simultaneamente em seu próprio MicroSD o arquivo de observação estática da base.
   * **Opção 2 (Rede do IBGE - RBMC):** Baixar o arquivo estático público formato RINEX da estação oficial do IBGE mais próxima (em São José dos Campos/DCTA, há estação ativa da RBMC).
3. **Processamento no RTKLIB (Software Aberto Geodésico):**
   * Abrir o programa **`rtkpost.exe`** do pacote RTKLIB.
   * Inserir o log do avião (Rover: `.ubx` do MicroSD).
   * Inserir o log da base (Base: `.ubx` ou `.obs` RINEX).
   * Inserir o arquivo de efemérides (`.nav` ou `.sfrbx`).
   * Configurar:
     * *Positioning Mode:* **Kinematic**
     * *Frequencies:* **L1 + L2**
     * *Filter Type:* **Combined (Forward + Backward)**
   * Clicar em **`Execute`**.
4. **Resultado:** O RTKLIB gera uma planilha e trajetória `.pos` e `.kml` (Google Earth) com a altitude geométrica e trajetória milimétrica exata da corrida de decolagem, voo e toque na pista.

---

## 6. Diagrama de Integração de Conectores no DAS V2

```
                       [ Conector SMA J8 ]
                                │
                        (Antena GNSS 50R)
                                │
┌───────────────────────────────▼───────────────────────────────┐
│                     PLACA DAS V2 (AVIÃO)                      │
│                                                               │
│   [u-blox ZED-F9P] ──UART1──> [STM32G431] ──SPI──> [MicroSD]  │
│          ▲ (UART2)                 │                          │
│          │                         │                          │
└──────────┼─────────────────────────┼──────────────────────────┘
           │                         │
     [Conector J9]             [Conector J6]
    (Entrada RTCM)          (Telemetria Geral)
           ▲                         ▲
           │                         │
   ┌───────┴─────────────────────────┴───────┐
   │        Rádio Receptor Telemetria        │
   │               (ex: SiK 915MHz)          │
   └───────────────────▲─────────────────────┘
                       │
                       │ Link RF 915 MHz (RTCM 3.x)
                       │
   ┌───────────────────▼─────────────────────┐
   │        Rádio Transmissor Solo           │
   │               (ex: SiK 915MHz)          │
   └───────────────────▲─────────────────────┘
                       │
┌──────────────────────┴────────────────────────────────────────┐
│                   PLACA DAS V2 (BASE SOLO)                    │
│                                                               │
│       [u-blox ZED-F9P em modo Base] ──SMA──> (Tripé Solo)     │
│                 │                                             │
│            [USB-C J2] ───> Notebook / Powerbank               │
└───────────────────────────────────────────────────────────────┘
```

---
*Documento homologado para o subsistema de aviônica do projeto AeroDesign.*
