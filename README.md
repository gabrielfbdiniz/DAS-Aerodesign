# DAS V2 — Sistema de Aquisição de Dados de Voo
**Equipe de AeroDesign — Temporada 2025/2026**  
*Placa de Aviônica Integrada, Telemetria e Posicionamento GNSS RTK Centimétrico*

[![KiCad](https://img.shields.io/badge/EDA-KiCad%2010.0-blue.svg)](https://kicad.org)
[![MCU](https://img.shields.io/badge/MCU-STM32G431CBT6%20(170MHz%20Cortex--M4F)-red.svg)](https://www.st.com)
[![GNSS](https://img.shields.io/badge/GNSS-u--blox%20ZED--F9P%20RTK-brightgreen.svg)](https://www.u-blox.com)
[![Status](https://img.shields.io/badge/Status-Validação%20de%20Esquemático-orange.svg)]()

---

## 1. Visão Geral do Projeto

O **DAS V2 (Data Acquisition System V2)** é a nova geração da aviônica de bordo da aeronave da equipe de AeroDesign. 

Projetada a partir do zero para superar as limitações térmicas e de estabilidade da versão anterior (V1), a placa unifica em um único PCB de 4 camadas:
* Processamento matemático de alto desempenho com FPU de ponto flutuante em hardware;
* Posicionamento geodésico centimétrico ($1\text{ cm} - 2\text{ cm}$) via RTK/PPK multibanda;
* Fusão de dados inerciais de 9 graus de liberdade (IMU);
* Telemetria do trem de força elétrico via barramento CAN ligado ao VESC;
* Armazenamento em alta velocidade em cartão MicroSD via SPI com DMA;
* Alimentação chaveada de alta eficiência suportando baterias LiPo 2S a 4S ($7.4\text{ V} - 16.8\text{ V}$) sem aquecimento.

---

## 2. Especificações Técnicas

| Subsistema | Componente / Chip | Características Principais |
| :--- | :--- | :--- |
| **Microcontrolador** | **STM32G431CBT6** (LQFP-48) | ARM Cortex-M4F @ 170 MHz com FPU, 128 kB Flash, 32 kB SRAM, FDCAN nativo. |
| **GNSS RTK** | **u-blox ZED-F9P-02B** | Receptor multibanda L1/L2 (GPS, GLONASS, Galileo, BeiDou), RTK centimétrico e logs brutos para PPK. |
| **Linha RF GNSS** | Conector SMA Fêmea 50 $\Omega$ | Bias-T ativo onboard (alimentação da antena via `VCC_RF`), filtro choke 47 nH e proteção ESD ultrabaixa capacitância. |
| **IMU (9-DOF)** | **Bosch BNO055** | Acelerômetro, Giroscópio e Magnetômetro com processador Cortex-M0 interno para fusão e saída em quatérnions via I2C (`0x28`). |
| **Expansão Pitot** | Conector JST-PH 2.0 mm (4 pinos) | Barramento I2C dedicado com pull-ups de $2.2\text{ k}\Omega$ para acoplamento do sensor de pressão diferencial externo. |
| **Telemetria VESC** | **SN65HVD230** + TVS **NUP2105L** | Transceptor CAN 3.3V com terminação split $120\,\Omega$ configurável e conector padrão VESC JST-PH 4 pinos. |
| **Armazenamento** | Slot MicroSD (Push-Push) | Interface SPI de alta velocidade com DMA (>1.5 MB/s de escrita sustentada). |
| **Telemetria RF** | Conector JST-PH 2.0 mm (4 pinos) | UART dedicada conectada a rádio modem de longo alcance (ex.: SiK 915 MHz, Holybro ou LoRa). |
| **Interface USB** | Conector USB-C 16 Pinos | Proteção ESD dedicada **USBLC6-2SC6**, resistores CC de $5.1\text{ k}\Omega$ e proteção de sobretensão para o GNSS. |
| **Alimentação** | **TPS563200** (Buck 5.0V @ 2A) | Eficiência $>90\%$, entrada 7.4V a 16.8V (LiPo 2S–4S), entrada XT30 e proteção P-MOSFET contra inversão de polaridade. |

---

## 3. Arquitetura Estrutural do Repositório

O projeto adota design modular e hierárquico no KiCad 10:

```
DAS_V2/
├── .gitignore                      # Regras de exclusão de arquivos temporários e de backup
├── README.md                       # Documentação principal do projeto
├── DAS_V2.kicad_pro                # Arquivo principal do projeto KiCad 10
├── DAS_V2.kicad_sch                # Folha Raiz (Top-Level) com barramentos interfolhas
├── 01_Power_Supply.kicad_sch       # Módulo 1: Entrada XT30, Proteção P-FET, Buck 5V e LDO 3.3V
├── 02_MCU_Core.kicad_sch           # Módulo 2: STM32G431, USB-C, SWD, HSE 16MHz, Reset & Status
├── 03_Sensors.kicad_sch            # Módulo 3: IMU Bosch BNO055 e Conector Pitot I2C
├── 04_GNSS_RTK.kicad_sch           # Módulo 4: u-blox ZED-F9P, Bias-T, Linha RF 50R e Porta RTCM
├── 05_Storage_Telemetry.kicad_sch  # Módulo 5: MicroSD Card (SPI), Transceptor CAN e Telemetria RF
├── DAS_V2.kicad_pcb                # Layout da Placa de Circuito Impresso (Stackup 4 camadas)
├── fp-lib-table                    # Tabela de bibliotecas de footprints locais
├── sym-lib-table                   # Tabela de bibliotecas de símbolos locais
├── DAS_V2.pretty/                  # Diretório de footprints customizados (Logo AeroDesign)
└── docs/                           # Documentação Técnica Aprofundada
    ├── GUIA_OPERACAO_RTK_PPK.md    # Guia Completo: Operação RTK, Base de Solo, NTRIP e PPK
    ├── RELATORIO_AUDITORIA_ESQUEMATICO_DAS_V2.md # Auditoria Forense Completa contra Datasheets dos CIs
    └── PLANO_DE_ACAO.md            # Diagnóstico forense da V1 e diretrizes do redesenho V2
```

---

## 4. Documentação Técnica em Destaque

* 🛰️ **[Guia de Operação GNSS RTK & PPK](docs/GUIA_OPERACAO_RTK_PPK.md):**
  Explica o funcionamento do ZED-F9P, por que o receptor sozinho no avião não atinge precisão centimétrica sem base, como **reaproveitar uma segunda placa DAS V2 idêntica como Estação de Solo em tripé**, como usar a rede gratuita NTRIP/IBGE (RBMC) e o passo a passo de pós-processamento com o **RTKLIB** (PPK).
* 🔍 **[Relatório de Auditoria Elétrica do Esquemático](docs/RELATORIO_AUDITORIA_ESQUEMATICO_DAS_V2.md):**
  Auditoria pino a pino de todos os CIs, apontando as correções críticas de hardware (Zener do P-MOSFET, estabilidade do LDO com cerâmica, bootloader do STM32 vs. CAN RX e capacitância do TVS de RF).
* 📋 **[Plano de Ação e Diagnóstico da V1](docs/PLANO_DE_ACAO.md):**
  Histórico forense das falhas da versão anterior e justificativas de engenharia para cada escolha de arquitetura da V2.

---

## 5. Como Abrir e Compilar o Projeto

1. Certifique-se de ter o **KiCad 10.0** ou superior instalado.
2. Clone o repositório:
   ```bash
   git clone <URL_DO_REPOSITORIO>
   cd "DAS V2"
   ```
3. Abra o arquivo de projeto:
   * Abra o KiCad e selecione `DAS_V2.kicad_pro`.
   * Clique em `Esquemático (Eeschema)` para visualizar as folhas hierárquicas.
   * Clique em `Placa (Pcbnew)` para visualizar o layout 2D/3D.

---

## 6. Padronização Mecânica e Conectores

Todos os chicotes externos da aeronave foram padronizados para garantir segurança contra vibração, leveza e facilidade de manutenção em campo:

| Conector | Função | Pinos | Padrão |
| :--- | :--- | :---: | :--- |
| **J1** | Entrada de Bateria Principal | 2 | **XT30** Macho/Fêmea polarizado |
| **J2** | Interface de Programação / Dados | 16 | **USB-C** 2.0 (UFP) |
| **J3** | Debug / Gravação Firmware | 5 | SWD Header (3.3V, SWDIO, SWCLK, GND, NRST) |
| **J4** | Sensor de Pressão (Tubo de Pitot) | 4 | **JST-PH 2.0 mm** (3.3V, GND, SCL, SDA) |
| **J6** | Telemetria RF de Longo Alcance | 4 | **JST-PH 2.0 mm** (5.0V, GND, TX, RX) |
| **J7** | Telemetria VESC do Motor | 4 | **JST-PH 2.0 mm** (NC, CAN_H, CAN_L, GND) |
| **J8** | Antena GNSS Multibanda | Coaxial | **SMA Fêmea** Borda / Vertical 50 $\Omega$ |
| **J9** | Entrada de Correção RTCM | 4 | **JST-PH 2.0 mm** (3.3V, TX, RX, GND) |

---
*Desenvolvido com excelência técnica pela Equipe AeroDesign.*
