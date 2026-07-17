# UART (Verilog)

A UART transmitter and receiver written in Verilog. Standard async framing - 1 start bit, 8 data bits (LSB first), 1 stop bit, no parity.

## What it does

- TX takes an 8-bit byte and shifts it out serially
- RX watches the serial line, detects a start bit, and reconstructs the byte
- Baud rate and clock frequency are parameterized in both modules
- TX: `tx_active` high while sending, `tx_done` pulses when finished
- RX: `rx_active` high while receiving, `rx_done` pulses when a byte is ready

## Files

| File           | Description                                              |
| -------------- | --------------------------------------------------------- |
| `uart_tx.v`    | UART transmitter                                           |
| `tb_uart_tx.v` | Testbench for TX - sends `8'hA5` and checks the output    |
| `uart_rx.v`    | UART receiver                                              |
| `tb_uart_rx.v` | Testbench for RX - drives `8'hA5` in and checks it decodes correctly |

## How it works

### TX

4-state FSM: `IDLE -> START -> DATA -> STOP -> IDLE`. A counter (`clk_count`) tracks cycles within the current bit and switches state once it hits `CLK_PER_BIT - 1`, where `CLK_PER_BIT = CLK_FREQ / BAUD_RATE`. `bit_index` walks the byte LSB first, one bit per baud period.

### RX

Same 4-state FSM idea, but running the other direction. The line idles high, so a falling edge means a start bit might be coming. Instead of trusting it right away, RX waits till the middle of that bit and checks again - if it's still low, it's a real start bit, if not it was just noise and RX goes back to idle. From there it samples the middle of each following bit (safest point, away from the edges) and shifts the byte in one bit at a time. Also runs `rx_in` through two flip-flops first since it's coming from outside the chip and isn't synced to our clock.

## Simulating

Used Icarus Verilog + GTKWave.

TX:
```
iverilog -o uart_tx_sim uart_tx.v tb_uart_tx.v
vvp uart_tx_sim
```

RX:
```
iverilog -o uart_rx_sim uart_rx.v tb_uart_rx.v
vvp uart_rx_sim
```

Both testbenches dump a `.vcd` you can open in GTKWave:
```
gtkwave uart_tx.vcd
gtkwave uart_rx.vcd
```

## Test cases

Both testbenches use `8'hA5` (`1010 0101`).

TX output on `tx_out`:
```
START(0) -> D0(1) -> D1(0) -> D2(1) -> D3(0) -> D4(0) -> D5(1) -> D6(0) -> D7(1) -> STOP(1)
```

RX is fed the same pattern on `rx_in` and should decode back to `rx_data = 8'hA5` with `rx_done` pulsing once.

Note: sim uses 10 MHz / 1 Mbps (10 cycles per bit) so waveforms don't take forever to look through. Real values (e.g. 50 MHz / 115200) just mean more cycles per bit, same logic either way.

## Waveforms

### TX
![UART TX Waveform](uart_tx_wave.png)

`tx_out` going through START -> D0 to D7 -> STOP while sending `8'hA5`. `tx_active` stays high for the whole transfer, `tx_done` pulses right at the end.

### RX
![UART RX Waveform](uart_rx_wave.png)

`rx_in` driven with the same `8'hA5` pattern, `rx_active` goes high once the start bit is detected and stays high through the byte, `rx_data` settles to `A5` and `rx_done` pulses once the stop bit is sampled.

## Possible next steps

- Add parity bit support
- Wrap TX/RX into a single top-level UART core with FIFO buffers
- Loopback test - connect tx_out to rx_in directly and verify end to end
