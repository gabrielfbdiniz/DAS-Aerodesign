# Relatório de Auditoria Elétrica e Validação de Esquemático: DAS V2
**Projeto:** AeroDesign — DAS V2 (Data Acquisition System)  
**Data:** 08 de Novembro de 2026  
**Status do Esquemático:** ⚠️ **NÃO LIBERADO PARA FABRICAÇÃO IMEDIATA SEM AS CORREÇÕES CRÍTICAS**

---

## 1. Veredito Executivo

As conexões hierárquicas entre os blocos na folha raiz (`DAS_V2.kicad_sch`) foram completadas com sucesso e mapeadas corretamente (linhas SPI, I2C, UARTs com DMA, barramento CAN, alimentações +3.3V, +5.0V e GND). 

Contudo, **não é possível garantir que a placa funcionará com o esquemático do jeito que está hoje**. A auditoria detalhada contra os datasheets dos circuitos integrados revelou **4 falhas elétricas críticas** no nível de componentes/silício que impediriam a inicialização ou causariam queima/instabilidade em bancada:

1. **Circuito de Proteção contra Inversão de Polaridade (Q1 + D1):** O diodo Zener de 10V (D1) está ligado em série para o terra. Em baterias **LiPo 2S (7.4 V – 8.4 V)**, o diodo nunca conduz, $V_{GS} = 0\text{ V}$ e o MOSFET **nunca liga**. Em baterias **LiPo 3S (11.1 V)**, $V_{GS}$ fica em apenas $-1.0\text{ V}$, mantendo o MOSFET na região linear, gerando queda de tensão e aquecimento severo.
2. **Instabilidade do LDO 3.3V (U2 - LM1117MP-3.3):** O regulador linear LM1117 exige capacitores de saída com ESR entre $0.1\,\Omega$ e $1.0\,\Omega$. Capacitores puramente cerâmicos MLCC (como C7 de $10\,\mu\text{F}$ e C8 de $100\text{ nF}$) provocam **oscilação de alta frequência na linha de 3.3V**, repetindo a falha observada na V1.
3. **Conflito de Inicialização do MCU (Pino 45 - PB8 / BOOT0 & CAN RX):** O pino 45 do STM32G431CBT6 acumula as funções de `PB8-BOOT0` e `FDCAN1_RX`. Em repouso (recessive), o transceptor CAN SN65HVD230 (U6) força a linha RX ativamente em **3.3 V**. Ao ligar a placa, o MCU detecta BOOT0 em nível alto e **fica preso no bootloader de fábrica** (em vez de rodar o código da aeronave), além do botão SW2 provocar risco de curto-circuito caso pressionado com dado em trânsito no barramento CAN.
4. **Capacitância Parasita do Diodo TVS de RF (D7):** D7 está definido genericamente como `"D_TVS"` sem part number. Se for montado um diodo TVS comum ($30 - 100\text{ pF}$), o sinal de RF do GNSS ($1.575\text{ GHz}$) será completamente curto-circuitado para o terra. É mandatório o uso de um TVS de ultra-baixa capacitância ($C_j < 0.5\text{ pF}$).
5. **Transição para o Layout 2D (PCB):** **85 dos 97 componentes** do projeto ainda não possuem *footprints* atribuídos.

---

## 2. Diagnóstico Detalhado por Bloco e Confronto com Datasheets

```mermaid
flowchart TD
    subgraph S1["01 - Fonte de Alimentação"]
        XT30["Bateria XT30 (7.4V - 16.8V)"] --> Q1_PROT["P-MOSFET AO3401A\n(Falha no Zener D1)"]
        Q1_PROT --> TPS["Buck TPS563200\n(+5.0V @ 2A) [OK]"]
        TPS --> LDO["LDO LM1117-3.3\n(Risco Oscilação ESR)"]
        LDO --> RAIL33["Barramento +3.3V Limpo"]
        TPS --> RAIL5["Barramento +5.0V"]
    end

    subgraph S2["02 - Processamento (STM32G431)"]
        RAIL33 --> MCU["STM32G431CBT6"]
        MCU --> PB8_CONFLICT["Pino 45: PB8/BOOT0\n(Conflito com CAN RX)"]
    end

    subgraph S3["04 - GNSS RTK (ZED-F9P)"]
        RAIL33 --> ZED["u-blox ZED-F9P"]
        VBUS_SENSE["VBUS 5V"] --> DIVIDER["Divisor 10k/20k (3.3V) [OK]"] --> V_USB["Pino 38: V_USB"]
        SMA["Antena SMA 50R"] --> D7_TVS["TVS D7 (Exige <0.5pF)"] --> ZED
    end

    subgraph S4["05 - Telemetria & CAN"]
        RAIL33 --> CAN_TRANS["SN65HVD230 (CAN)"]
        CAN_TRANS -->|Linha RXD (Idle = 3.3V)| PB8_CONFLICT
    end
```

---

### Bloco 1: Alimentação e Proteções (`01_Power_Supply.kicad_sch`)

#### 1. Circuito de Proteção P-MOSFET (AO3401A + D1 + R1 + R2)
* **Ligação atual no esquemático:**
  * R1 ($100\text{ k}\Omega$) entre Source e Gate.
  * R2 ($10\text{ k}\Omega$) entre Gate e o Cátodo do Zener D1 (BZX84C10).
  * Ânodo de D1 ligado ao GND.
* **Análise Física e do Datasheet do AO3401A:**
  * Com bateria 2S LiPo ($V_{BAT} = 7.4\text{ V} - 8.4\text{ V}$): O diodo Zener tem tensão de ruptura nominal de $10\text{ V}$. Abaixo de $10\text{ V}$, a corrente reversa é nula ($I_R < 1\,\mu\text{A}$). Como não há corrente circulando por R1 e R2 para o terra, a tensão no Gate permanece idêntica à do Source ($V_{GS} = 0\text{ V}$). **O MOSFET fica 100% desligado**. Toda a corrente da placa passaria pelo diodo de corpo (*body diode*), provocando uma queda de $0.8\text{ V}$ e dissipação de calor inaceitável.
  * Com bateria 3S LiPo ($V_{BAT} = 11.1\text{ V}$): O Zener limita o cátodo a $10.0\text{ V}$. A queda sobre o ramo resistivo é de apenas $1.1\text{ V}$. A tensão Gate-Source resultante é:
    $$V_{GS} = -1.1\text{ V} \times \frac{100\text{ k}\Omega}{110\text{ k}\Omega} = -1.0\text{ V}$$
    O limite de condução ($V_{th}$) do AO3401A varia entre $-0.9\text{ V}$ e $-1.3\text{ V}$. Em $-1.0\text{ V}$, o canal está no início da condução, apresentando alta resistência interna ($R_{DS(on)} > 1\,\Omega$), aquecendo severamente o componente em SOT-23 e derrubando a tensão de entrada do Buck.
* **Solução Canônica:**
  * Ligar o Zener D1 diretamente **em paralelo com R1** (entre Source e Gate), com o **Cátodo no Source** e o **Ânodo no Gate**.
  * Ligar o resistor R2 ($10\text{ k}\Omega$ ou $47\text{ k}\Omega$) diretamente **do Gate para o GND**.
  * Dessa forma:
    * Em 7.4V (2S): Gate vai a 0V, $V_{GS} = -7.4\text{ V}$ (MOSFET satura totalmente com $R_{DS(on)} \approx 30\text{ m}\Omega$).
    * Em 16.8V (4S): O Zener grampeia $V_{GS}$ em exatamente $-10\text{ V}$, protegendo a isolação de óxido do Gate ($V_{GS(max)} = \pm 12\text{ V}$).

#### 2. Regulador Linear 3.3V (U2 - LM1117MP-3.3)
* **Conexão atual:** Entrada ligada aos +5.0V do Buck, saída ligada a C7 ($10\,\mu\text{F}$ cerâmico) e C8 ($100\text{ nF}$ cerâmico).
* **Conflito com o Datasheet (TI / AMS):** O LM1117 é uma arquitetura bipolar antiga que necessita de um capacitor com ESR entre $0.1\,\Omega$ e $1.0\,\Omega$ para manter a margem de fase do amplificador de erro. Capacitores cerâmicos MLCC modernos possuem ESR $< 0.01\,\Omega$, o que empurra o polo da malha de controle para a região de ganho positivo, causando oscilação auto-sustentada de centenas de kHz.
* **Solução:**
  * *Opção Recomendada:* Substituir U2 por um LDO CMOS de alta rejeição de ruído (PSRR) e compatível com cerâmica sem ESR, conforme previsto no plano inicial: **AP2112K-3.3** (SOT-23-5, 600mA) ou **TLV75533P** (SOT-23-5, 500mA).
  * *Opção Alternativa (mantendo SOT-223):* Adicionar um capacitor de tântalo de $10\,\mu\text{F}$ (Case A/B SMD) ou incluir um resistor de amortecimento de $0.33\,\Omega$ em série com C7.

#### 3. Estágio Buck 5.0V (U1 - TPS563200) — Aprovado ✅
* Frequência fixa: $580\text{ kHz}$.
* Indutor L1: $4.7\,\mu\text{H}$ com footprint robusto `L_APV_APH0630` ($6.3\times 6.3\text{ mm}$, saturação $> 4\text{ A}$).
* Divisor de realimentação: $R_3 = 54.9\text{ k}\Omega$, $R_4 = 10.0\text{ k}\Omega$.
  $$V_{OUT} = 0.765\text{ V} \times \left(1 + \frac{54.9\text{ k}\Omega}{10.0\text{ k}\Omega}\right) = 4.965\text{ V} \approx 5.0\text{ V}$$
* Bootstrap: C9 ($100\text{ nF}$) entre VBST (pino 6) e SW (pino 2). Perfeito.

#### 4. Pino VBAT na Folha Raiz
* O conector XT30 (J1) já está fisicamente dentro da folha `01_Power_Supply.kicad_sch`. O pino hierárquico `VBAT` na folha raiz está flutuando, gerando erro no ERC.
* *Solução:* Excluir o pino hierárquico caso a bateria só alimente esta folha, OU (altamente recomendado para aviônica) adicionar um divisor resistivo (ex: $100\text{ k}\Omega / 10\text{ k}\Omega$) para encaminhar uma linha `VBAT_SENSE` a um canal ADC do STM32 (ex: PA1).

---

### Bloco 2: Microcontrolador (`02_MCU_Core.kicad_sch`)

#### 1. Conflito do Pino 45 (`PB8-BOOT0` e `FDCAN1_RX`)
* **Situação:** O STM32G431 no encapsulamento LQFP-48 só possui o periférico FDCAN1 mapeável nos pinos PA11/PA12 ou PB8/PB9. Como PA11/PA12 estão alocados para a USB-C nativa, a CAN foi obrigatoriamente alocada em PB8/PB9.
* **O Problema:** No STM32G4, PB8 é compartilhado fisicamente com a função de `BOOT0`. O CI CAN (SN65HVD230) em estado recessivo (barramento livre) força seu pino R (RXD) ativamente para **3.3 V**.
* Se a placa for ligada com os Option Bytes originais de fábrica (`nSWBOOT0 = 1`), o STM32 lerá PB8 em nível alto durante a borda de subida do reset e **entrará no Bootloader DFU/UART interno, sem executar o firmware de voo**.
* **Procedimento Obrigatório de Gravação:**
  Ao gravar a placa pela primeira vez via SWD (STM32CubeProgrammer), configurar:
  $$\text{FLASH\_OPTR} \rightarrow \text{nSWBOOT0} = 0 \quad (\text{Software BOOT0 ativo})$$
  $$\text{FLASH\_OPTR} \rightarrow \text{nBOOT0} = 1 \quad (\text{Boot padrão da Flash } 0x08000000)$$
  Isso desativa completamente a amostragem física de PB8 no reset, permitindo que a CAN funcione sem afetar o boot.
* **Botão SW2:** O botão táctil SW2 está ligado diretamente entre +3.3V e a linha `BOOT0_CAN_RX` (pino 4 do CI CAN). Se um operador pressionar SW2 enquanto o barramento CAN estiver transmitindo um bit dominante (0V), a saída push-pull do SN65HVD230 será curto-circuitada diretamente para os 3.3V da alimentação, arriscando queimar a porta lógica do transceptor.
  * *Correção:* Inserir um resistor limitador de $1\text{ k}\Omega$ em série com SW2 ou mover o botão de usuário para um pino GPIO livre (como PB0, PB1 ou PC14).

#### 2. USB-C e Proteção ESD (U4 - USBLC6-2SC6) — Aprovado ✅
* CC1 e CC2 com resistores de $5.1\text{ k}\Omega$ para terra (negociação correta em modo Device/UFP).
* As linhas USB_CONN_DM e USB_CONN_DP passam pelo chip de proteção ESD USBLC6-2SC6 antes de chegarem aos pinos PA11 e PA12 do MCU.
* Capacitância de entrada ultrabaixa ($C_{line} = 3.5\text{ pF}$), perfeitamente transparente para USB 2.0 Full Speed (12 Mbps).

#### 3. Ajustes de Valores Discretos
* **R12 (Resistor do LED de Status em PC13):** O valor está como `"R"`. Deve ser corrigido para **$1\text{ k}\Omega$** (ou $470\,\Omega$).
* **Cristal Y1 (HSE):** O símbolo está com o valor `"Crystal_GND24"`. O valor real de projeto deve ser fixado em **$16\text{ MHz}$** (ou $8\text{ MHz}$), com capacitores de carga de $15\text{ pF}$ (C10/C11) já posicionados corretamente.

---

### Bloco 3: Sensores Inerciais (`03_Sensors.kicad_sch`)

#### 1. Bosch BNO055 (IMU 9-DOF) — 100% Validado com o Datasheet ✅
A pinagem e configuração de hardware do BNO055 seguem rigorosamente as recomendações da Bosch:
* **Protocolo:** Pinos 5 (PS1) e 6 (PS0) conectados ao GND $\rightarrow$ Seleciona interface **I2C Standard/Fast**.
* **Endereço I2C:** Pino 17 (COM3) aterrado $\rightarrow$ Define endereço padrão **`0x28`** (evitando conflito com outros periféricos).
* **Barramento I2C:** Pino 19 (COM1) em SCL, Pino 20 (COM0) em SDA, com resistores de pull-up dedicados R13 e R14 de $2.2\text{ k}\Omega$ para +3.3V.
* **Clock Interno:** Pino 27 (XIN32) aterrado e Pino 26 (XOUT32) flutuante $\rightarrow$ Habilita o oscilador interno de precisão.
* **Alimentação & Reset:** VDD (pino 3) e VDDIO (pino 28) em +3.3V com 4 capacitores de desacoplamento ($10\,\mu\text{F} + 3\times 100\text{ nF}$). Pino 9 (CAP) com capacitor de $100\text{ nF}$ para o terra. Pino 11 (nRESET) com pull-up de $10\text{ k}\Omega$ (R15). Pino 4 (nBOOT_LOAD) em 3.3V.

---

### Bloco 4: GNSS RTK ZED-F9P (`04_GNSS_RTK.kicad_sch`)

#### 1. Circuito de Sensoriamento de VBUS (`V_USB`)
* Divisor de tensão formado por R5 ($10\text{ k}\Omega$) e R6 ($20\text{ k}\Omega$) na Folha 1:
  $$V_{USB\_SENSE} = 5.0\text{ V} \times \frac{20\text{ k}\Omega}{10\text{ k}\Omega + 20\text{ k}\Omega} = 3.333\text{ V}$$
  O valor está perfeitamente seguro e abaixo do limite absoluto da u-blox ($3.6\text{ V}$), resolvendo de vez o problema que destruiu os módulos na V1.
* **Correção no ramo de filtragem na Folha 4:**
  * No esquemático atual, R21 ($100\text{ k}\Omega$) e C35 ($100\text{ nF}$) estão desenhados **em série** para o terra. Como o capacitor bloqueia corrente contínua, R21 não descarrega a linha.
  * *Correção:* Manter R6 ($20\text{ k}\Omega$) na Folha 1 como o pull-down principal e colocar C35 **diretamente entre o pino 38 e o terra** (em paralelo), eliminando R21 ou posicionando-o em paralelo para atuar como pull-down redundante.

#### 2. Diodo TVS de RF (D7)
* O pino 2 (`RF_IN`) opera nas frequências L1 ($1575.42\text{ MHz}$) e L2 ($1227.60\text{ MHz}$).
* O símbolo D7 está marcado como `"D_TVS"` genérico.
* **Requisito Crítico:** O componente montado **deve obrigatoriamente possuir capacitância parasita $C_j < 0.5\text{ pF}$** (exemplos homologados: `ESD9R3.3ST5G`, `PESD5V0X1BCSF` ou `TPD1E10B06`). Qualquer diodo TVS convencional ($15 - 50\text{ pF}$) agirá como um curto-circuito para o sinal dos satélites.

#### 3. Circuito Bias-T da Antena Ativa — Aprovado ✅
* Pino 7 (`VCC_RF`) alimenta a antena ativa através de R22 ($10\,\Omega$ para limitação de sobrecorrente em caso de curto no cabo SMA), filtrado por C36 ($100\text{ nF}$) e acoplado por L2 ($47\text{ nH}$ RF choke).

#### 4. Pinagem e Linhas de Controle do ZED-F9P — Aprovado ✅
* `D_SEL` (pino 47) com pull-up de $10\text{ k}\Omega$ (R29) $\rightarrow$ Habilita portas UART e I2C.
* `~SAFEBOOT` (pino 50) e `~RESET` (pino 49) com resistores de pull-up de $10\text{ k}\Omega$ (R27/R28) $\rightarrow$ Operação nominal garantida.
* LEDs de status: `TIMEPULSE` (PPS) em D5 (via R25 de 1k) e `RTK_STAT` em D6 (via R26 de 1k).
* Conector J9 dedicado para correção RTCM via UART2.

---

### Bloco 5: Gravação e Telemetria (`05_Storage_Telemetry.kicad_sch`)

#### 1. Conector MicroSD em Modo SPI — Aprovado ✅
* `SD_CS` em PA4, `SD_CLK` em PA5, `SD_DAT0` (MISO) em PA6 e `SD_CMD` (MOSI) em PA7.
* Resistores de pull-up em CS ($10\text{ k}\Omega$), MOSI ($47\text{ k}\Omega$) e MISO ($47\text{ k}\Omega$), garantindo barramento limpo e sem oscilações em alta velocidade.
* Desacoplamento local com C27 ($100\text{ nF}$) e C28 ($10\,\mu\text{F}$).

#### 2. Transceptor CAN (SN65HVD230) — Aprovado ✅
* Pino 8 (Rs) aterrado $\rightarrow$ Modo de alta velocidade (até 1 Mbps).
* Proteção contra transientes via diodo TVS duplo dedicado **NUP2105L** (D4) em SOT-23.
* Terminação resistiva de $120\,\Omega$ em topologia *Split Termination* ($2\times 60\,\Omega$ R19/R20 + capacitor de modo comum C26 de $4.7\text{ nF}$), com jumpers JP1 e JP2 para isolar a terminação se o DAS for colocado no meio do chicote da aeronave.
* Conector J7 padrão VESC (Pino 1 livre, Pino 2 CAN_H, Pino 3 CAN_L, Pino 4 GND).

---

## 3. Matriz de Correções Mandatórias antes do Roteamento 2D

| Ref | Subsistema | Problema Atual | Ação Corretiva Exata | Gravidade |
| :--- | :--- | :--- | :--- | :--- |
| **Q1 / D1 / R2** | Fonte (Folha 1) | Zener em série para o terra; MOSFET não liga em 2S e opera em linear em 3S. | Conectar D1 em paralelo com R1 (Cátodo no Source, Ânodo no Gate); ligar R2 do Gate para o GND. | 🔴 **Crítico** |
| **U2 / C7** | LDO 3.3V (Folha 1) | LM1117 oscila com capacitores cerâmicos puros de saída (baixo ESR). | Substituir por **AP2112K-3.3** (SOT-23-5) ou usar capacitor de tântalo de $10\,\mu\text{F}$ em C7. | 🔴 **Crítico** |
| **PB8 / SW2** | MCU (Folha 2) | Transceptor CAN mantém PB8 em 3.3V (bootloader loop); SW2 pode dar curto na saída de U6. | Configurar `nSWBOOT0=0` nos Option Bytes do MCU; colocar resistor de $1\text{ k}\Omega$ em série com SW2. | 🔴 **Crítico** |
| **D7** | GPS RF (Folha 4) | Símbolo `"D_TVS"` genérico na linha de 1.5 GHz da antena GNSS. | Especificar part number com $C_j < 0.5\text{ pF}$ (ex.: `ESD9R3.3ST5G` / `PESD5V0X1BCSF`). | 🔴 **Crítico** |
| **R21 / C35** | GPS V_USB (Folha 4) | R21 e C35 em série para o terra bloqueiam corrente DC. | Ligar C35 diretamente do pino 38 para o GND (desacoplamento RF); manter R6 como pull-down. | 🟡 **Moderado** |
| **R12 / Y1** | MCU (Folha 2) | R12 com valor `"R"` e Y1 sem frequência definida. | Alterar R12 para **$1\text{ k}\Omega$** e Y1 para **$16\text{ MHz}$**. | 🟡 **Moderado** |
| **Passivos** | Geral (Todas) | 85 componentes não possuem footprint atribuído. | Atribuir footprints padronizados (0603 para R/C, JST-PH para conectores, XT30 para bateria). | 🔴 **Bloqueante 2D** |

---

## 4. Próximos Passos para Avançar ao 2D da Placa

1. **Ajuste dos 4 pontos de circuito** no esquemático (Fonte P-FET, LDO, SW2 e R21/C35).
2. **Atribuição em lote de Footprints:**
   * Resistores e Capacitores discretos: `Resistor_SMD:R_0603_1608Metric_Pad0.98x0.95mm_HandSolder` e `Capacitor_SMD:C_0603_1608Metric_Pad1.08x0.95mm_HandSolder`.
   * Conectores de chicote externo: `Connector_JST:JST_PH_S4B-PH-K_1x04_P2.00mm_Horizontal` (ou vertical conforme mecânica).
   * Conector de força principal: `Connector:XT30PW-M` (ou padrão XT30 90°/180°).
   * Conector SMA RF: `Connector_Coaxial:SMA_Amphenol_132134_Vertical` (ou borda Edge-Mount).
3. **Execução Final do KiCad ERC:** Certificar 0 erros e 0 avisos elétricos.
4. **Atualização da PCB (`Update PCB from Schematic` / F8):** Carregar todos os componentes e *nets* no editor de layout 2D.
