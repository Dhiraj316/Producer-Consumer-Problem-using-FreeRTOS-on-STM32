# Producer-Consumer Problem using FreeRTOS on STM32 (LED Demo)

A mini project that implements the classic **Producer-Consumer problem** on an STM32 microcontroller using **FreeRTOS (CMSIS-RTOS V2)** in **STM32CubeIDE**. No serial terminal is needed: the behaviour of the system is shown entirely with **LEDs**.

## Overview

In the Producer-Consumer problem, one task (the **producer**) generates data and places it in a shared bounded buffer, while another task (the **consumer**) removes and processes that data. The two must be synchronised so that:

- the producer does not add data to a **full** buffer,
- the consumer does not read from an **empty** buffer,
- shared resources are not accessed by both tasks at the same time.

This project solves it using a FreeRTOS **Queue** (the buffer) and a **Mutex** (to protect the shared LEDs).

---

## How It Works

```
 +-----------+      +-----------------+      +-----------+
 | Producer  | ---> |  Queue (size 3) | ---> | Consumer  |
 |   Task    |      |  [ 1 | 2 | 3 ]  |      |   Task    |
 +-----------+      +-----------------+      +-----------+
       |                                           |
       +------------- Mutex (ledMutex) ------------+
                           |
                    LED Output Block
```

| LED | Meaning |
|-----|---------|
| Green (`LED_PROD`) | Producer added an item to the queue (1 blink) |
| Blue/Yellow (`LED_CONS`) | Consumer took an item from the queue. Blinks as many times as the item's value (1 to 3) |
| Red (`LED_FULL`) | Queue is full, producer could not add (2 blinks) |

- The **producer** is fast (300 ms) and generates the values 1, 2, 3, 1, 2, 3, ...
- The **consumer** is slow (1500 ms), so the queue gradually fills up and the red LED shows the full condition.
- The **mutex** ensures only one task controls the LEDs at a time, so only one LED is active at any moment.

---

## Hardware Required

- STM32 Nucleo board (e.g. NUCLEO-F401RE / F446RE / any STM32 with FreeRTOS support)
- 3 x LEDs (green, blue/yellow, red)
- 3 x 330 ohm resistors
- Breadboard and jumper wires
- USB cable (for ST-LINK programming)

## Pin Connections

> The pins below are examples. Change them to match your board.

| Signal | STM32 Pin | User Label |
|--------|-----------|------------|
| Producer LED (Green) | PB0 | `LED_PROD` |
| Consumer LED (Blue)  | PB1 | `LED_CONS` |
| Queue Full LED (Red) | PB2 | `LED_FULL` |

Wiring for each LED:

```
GPIO Pin ---> 330 ohm resistor ---> LED anode (+) ... LED cathode (-) ---> GND
```

---

## FreeRTOS Configuration

| Item | Setting |
|------|---------|
| Interface | CMSIS_V2 |
| SYS Timebase Source | TIM6 (not SysTick, which is used by FreeRTOS) |
| ProducerTask | osPriorityNormal, stack 256 words, entry `StartProducer` |
| ConsumerTask | osPriorityNormal, stack 256 words, entry `StartConsumer` |
| Queue `dataQueue` | Size 3, item type `uint32_t` |
| Mutex `ledMutex` | Default attributes |

---

## Setup Guide (STM32CubeIDE)

1. **Create project:** File > New > STM32 Project, select your board/MCU, name it `ProducerConsumer_LED`, language **C**.
2. **Set timebase:** System Core > SYS > Timebase Source = **TIM6**.
3. **Configure GPIO:** Set PB0, PB1, PB2 as `GPIO_Output` and add the user labels `LED_PROD`, `LED_CONS`, `LED_FULL`.
4. **Enable FreeRTOS:** Middleware > FREERTOS > Interface = **CMSIS_V2**.
5. **Add tasks, queue and mutex** as shown in the configuration table above.
6. **Generate code:** Save the `.ioc` file (Ctrl+S) and click **Yes** to generate code.
7. **Add the code** from the section below into `main.c` (or `freertos.c`) inside the `USER CODE` blocks.
8. **Build** (hammer icon) and **Run/Debug** to flash the board.

---

## Source Code

### Helper function

```c
/* Blink one LED 'times' times while holding the mutex */
void BlinkLED(GPIO_TypeDef *port, uint16_t pin, uint32_t times)
{
    osMutexAcquire(ledMutexHandle, osWaitForever);   // take the LED resource
    for (uint32_t i = 0; i < times; i++)
    {
        HAL_GPIO_WritePin(port, pin, GPIO_PIN_SET);
        osDelay(200);
        HAL_GPIO_WritePin(port, pin, GPIO_PIN_RESET);
        osDelay(200);
    }
    osMutexRelease(ledMutexHandle);                  // free it for the other task
}
```

### Producer task

```c
void StartProducer(void *argument)
{
    uint32_t data = 0;
    for(;;)
    {
        data = (data % 3) + 1;     // produces 1, 2, 3, 1, 2, 3 ...

        if (osMessageQueuePut(dataQueueHandle, &data, 0, 0) == osOK)
        {
            BlinkLED(LED_PROD_GPIO_Port, LED_PROD_Pin, 1);   // item produced
        }
        else
        {
            BlinkLED(LED_FULL_GPIO_Port, LED_FULL_Pin, 2);   // queue full
        }

        osDelay(300);   // fast production rate
    }
}
```

### Consumer task

```c
void StartConsumer(void *argument)
{
    uint32_t received;
    for(;;)
    {
        // Blocks until the queue has data
        if (osMessageQueueGet(dataQueueHandle, &received, NULL, osWaitForever) == osOK)
        {
            BlinkLED(LED_CONS_GPIO_Port, LED_CONS_Pin, received);  // blink = item value
        }
        osDelay(1500);  // slow consumption rate
    }
}
```

> Make sure `BlinkLED` is declared after the RTOS handles (or add a prototype near the top of the file).

---

## Expected Output

1. The **green** LED blinks once each time an item is produced.
2. The **blue** LED blinks 1, 2 or 3 times when an item is consumed, matching the data value.
3. Because the producer is faster, the queue fills up and the **red** LED blinks twice, indicating a full buffer.
4. Only one LED is on at any time, which demonstrates mutual exclusion through the mutex.

---

## Experiments

- Make the consumer faster than the producer: the queue stays empty and the consumer remains blocked.
- Change the queue size to 1 or 5 and observe how often the red LED appears.
- Change task priorities and see which task gets the mutex first.
- Add a second producer task to show multiple producers sharing one queue.

---

## Project Structure

```
ProducerConsumer_LED/
|-- Core/
|   |-- Inc/
|   |-- Src/
|   |   |-- main.c          # Tasks, queue, mutex and LED logic
|   |   |-- freertos.c
|-- Drivers/                # STM32 HAL drivers
|-- Middlewares/            # FreeRTOS source
|-- ProducerConsumer_LED.ioc
|-- README.md
```
### I uploaded the zip file of this project in case there is any issue. just extract the zip file and import in the stmcube IDE for the reference.
