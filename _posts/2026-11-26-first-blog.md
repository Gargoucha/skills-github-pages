---
title: "First Blog"
date: 2026-11-26
---

# Why Your STM32 UART Drops Bytes Under FreeRTOS (And How to Fix It)

When bridging an STM32 microcontroller to a Linux host (like a Raspberry Pi or Jetson) over UART, everything works fine in a bare-metal `while(1)` loop—until you move to **FreeRTOS** and suddenly start losing bytes at 115200+ baud.

---

### The Root Cause: Polling vs. Task Scheduling

In a bare-metal loop, the CPU checks the UART peripheral continuously. Under an RTOS, higher-priority tasks or critical sections delay your UART task. Since most STM32 UART data registers only hold **1 byte** at a time, a single missed read triggers an **Overrun Error (`ORE`)** and drops incoming data.

---

### The Fix: DMA + Idle Line Interrupt

Instead of polling byte-by-byte, let hardware handle the incoming stream:

1. **Enable UART RX DMA** in Circular mode so incoming bytes transfer straight into a RAM ring buffer without CPU intervention.
2. **Enable the UART Idle Line Interrupt (`IDLE`)** to trigger an ISR only when a packet finishes arriving.
3. **Notify a FreeRTOS task** from the ISR using `xTaskNotifyFromISR()` to parse the complete frame cleanly.

```c
void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t Size) {
    if (huart->Instance == USART2) {
        BaseType_t xHigherPriorityTaskWoken = pdFALSE;
        rx_length = Size;
        vTaskNotifyGiveFromISR(uartRxTaskHandle, &xHigherPriorityTaskWoken);
        portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
    }
}
