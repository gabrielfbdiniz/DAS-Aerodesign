# Firmware — DAS V2 (`STM32G431CBT6`)

Diretório destinado ao firmware embarcado da placa **DAS V2** (ARM Cortex-M4F @ 170 MHz).

## Mapeamento de Periféricos (Hardware V2)

| Periférico | Pinos MCU | Função |
| :--- | :--- | :--- |
| **I2C1** | `PB6` (SCL), `PB7` (SDA) | Sensores *onboard*: IMU **BNO055** (`0x28`) + Barômetro **MS5611** (`0x77`) |
| **I2C3** | `PA8` (SCL), `PB5` (SDA) | Multiplexador **PCA9548APW** (`0x70`) — Canais `0..3` (`J4`, `J12`, `J13`, `J14`) |
| **USART1** | `PA9` (TX), `PA10` (RX) | GNSS RTK **u-blox ZED-F9P** (protocolo UBX + passthrough USB CDC) |
| **USART2** | `PA2` (TX), `PA3` (RX) | Rádio de Telemetria RF (`J6`) |
| **USART3** | `PB10` (TX), `PB11` (RX) | Interface Pixhawk / MAVLink (`J11`) |
| **FDCAN1** | `PB8` (RX), `PB9` (TX) | Barramento CAN (`SN65HVD230`) — VESC (`J7`) e Expansão (`J10`) |
| **SPI1** | `PA4` (CS), `PA5` (SCK), `PA6` (MISO), `PA7` (MOSI) | Cartão MicroSD (`J5`) com DMA |
| **USB** | `PA11` (DM), `PA12` (DP) | USB-C CDC (diagnóstico, CLI e ponte *u-center*) |
| **TIM (PPS)** | `PA0` | Pulso `TIMEPULSE` (`GPS_PPS`) do ZED-F9P |
| **GPIO** | `PC13` | LED de status do MCU |
