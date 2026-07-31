# Final Capstone for EEL 4775 Real-Time Systems

Theme: Hardware Security

In my app, the button on GPIO 18 is treated like a tamper sensor. When the button is pressed, it triggers an interrupt. The ISR is the top half. It does the quick interrupt work, then wakes the bottom-half tasks. The bottom-half tasks do the slower work like printing the latency.

## Wokwi Link

https://wokwi.com/projects/468065854653295617

## Load Setting

```c
#define WITH_LOAD 0   /* changed to 1 for the loaded run */
```

  `WITH_LOAD = 0` — idle baseline. This is the simple test with no extra background load.
  `WITH_LOAD = 1` — loaded test. App 2's four background tasks run on Core 1.

The bottom-half tasks are priority 12.

The App 2 load tasks are:

  Task A: priority 15
  Task B: priority 10
  Task C: priority 5
  Task D: priority 2

Task A is the only load task with a higher priority than the bottom-half tasks. Task B, C, and D are lower priority, so they should not cut in front of the bottom-half tasks once the bottom-half tasks are ready.

## Capturing Latency with Wokwi Logic Analyzer

The logic analyzer was connected as:

  Channel 0 to GPIO 18
  Channel 1 to GPIO 19
  GND to GND

Where GPIO 18 is the button input and GPIO 19 is the ISR pulse output.

The VCD screenshots show the button signal changing and GPIO 19 pulsing when the ISR runs.

## Latency Results

I pressed the button at least 50 times for each run.

### Idle Run

`WITH_LOAD = 0`

```text
[notif] tamper event #50  latency=30 us (max=30)
[sem] tamper event #50  latency=669 us (max=2309)
```

Idle results:

  Notification max latency: `30 us`
  Semaphore max latency: `2309 us`

### Loaded Run

`WITH_LOAD = 1`

```text
[sem] tamper event #50  latency=416 us (max=10369)
[notif] tamper event #50  latency=638 us (max=17706)
```

Loaded results:

  Notification max latency: `17706 us`
  Semaphore max latency: `10369 us`

The loaded run also printed watchdog warnings from `load_b`. I still used the run because it reached 50 button presses and gave max latency values.

## Engineering Analysis

### 1. What's in my ISR? What's NOT?

The ISR is as follows:

 `static void IRAM_ATTR button_isr(void *arg)`
  This is the ISR. `IRAM_ATTR` keeps it in RAM, which helps keep interrupt timing more consistent.

 `int64_t now = esp_timer_get_time();`
  This saves the time when the ISR starts.

 `if (now - last_edge_us < DEBOUNCE_US) return;`
  This ignores button bounces so one press does not count as a bunch of presses.

 `last_edge_us = now;`
  This saves the last accepted button press time.

 `gpio_set_level(ISR_PULSE_GPIO, 1);`
  This turns GPIO 19 on so the logic analyzer can see when the ISR runs.

 `isr_entry_time_us = now;`
  This saves the ISR time for the latency calculation.

 `presses_observed++;`
  This counts the accepted button press.

 `BaseType_t higher_woken = pdFALSE;`
  This is used by the FreeRTOS ISR-safe wake functions.

 `xSemaphoreGiveFromISR(btn_sem, &higher_woken);`
  This wakes the semaphore bottom-half task.

 `vTaskNotifyGiveFromISR(task_notif_handle, &higher_woken);`
  This wakes the notification bottom-half task.

 `gpio_set_level(ISR_PULSE_GPIO, 0);`
  This turns GPIO 19 back off.

  `portYIELD_FROM_ISR(higher_woken);`
  This lets FreeRTOS switch to a woken bottom-half task after the ISR if needed.

What is not in the ISR:

No `printf()`, `ESP_LOGI()`, delays, or blocking functions.

The ISR is kept short on purpose. The bottom-half tasks do the logging later.

### 2. Binary Semaphore vs Direct Task Notification

My measured numbers were:

Idle:

Notification max latency: `30 us`
Semaphore max latency: `2309 us`

Loaded:

Notification max latency: `17706 us`
Semaphore max latency: `10369 us`

In idle mode, the notification path was faster.

In loaded mode, the semaphore path had the better worst-case max latency.

For my final design, I would use the semaphore path because the loaded worst-case number was lower in my Wokwi run.

I am not trying to fully explain the inside details of semaphore vs notification yet. I am just using the measured numbers from my Wokwi runs.

### 3. Latency Under Load

Notification path:

```text
17706 / 30 = 590.2
```

The notification max latency increased by about `590x`.

Semaphore path:

```text
10369 / 2309 = 4.5
```

The semaphore max latency increased by about `4.5x`.

So yes, latency got worse under load.

The reason is the priority setup. The bottom-half tasks are priority 12. Load Task A is priority 15, so Task A can run before the bottom-half tasks and delay them.

Load Tasks B, C, and D are priorities 10, 5, and 2. They are lower than priority 12, so they should not preempt the bottom-half tasks once the bottom-half tasks are ready.

So based on the priority setup, Task A is the load task that can cause the worst-case delay.

### 4. Induced Failure

For the failure test, I used `WITH_LOAD = 0`.

I commented out:

```c
portYIELD_FROM_ISR(higher_woken);
```

Prediction:

I expected the button press to still signal the bottom-half task, but the bottom-half task might not run right away.

Observed result:

```text
Notification max = 31 us
Semaphore max = 2309 us
```

This looked almost the same as my normal idle run.

So my prediction did not really show up strongly in Wokwi. The task still woke up, and the latency numbers stayed close to the idle test.

## Concurrency Diagram

```text
GPIO 18 button
tamper input
      |
      v
+-----------------------------+
| button_isr                  |
| top half / interrupt        |
| not a task                  |
|                             |
| checks debounce             |
| saves ISR time              |
| pulses GPIO 19              |
| wakes bottom-half tasks     |
+-----------------------------+
        |                 |
        | semaphore       | notification
        v                 v
+----------------+   +----------------+
| btn_sem task   |   | btn_notif task |
| priority 12    |   | priority 12    |
| waits for ISR  |   | waits for ISR  |
| prints latency |   | prints latency |
+----------------+   +----------------+

When WITH_LOAD = 1, the App 2 load tasks also run on Core 1:

load_a: priority 15
load_b: priority 10
load_c: priority 5
load_d: priority 2


```

## Screenshot Evidence

The ZIP includes:

```text
screenshots/idleTerminal.png
screenshots/idleVCD.png
screenshots/idleVCDzoomed.png
screenshots/loadedTerminal.png
screenshots/loadedVCD.png
screenshots/loadedVCDzoomed.png
screenshots/failureTerminal.png
```

## AI Disclosure

I used ChatGPT to help organize and properly format this README for easier reading. 
