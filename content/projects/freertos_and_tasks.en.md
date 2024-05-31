+++
title =  "Fun with FreeRTOS and sync primitives"
date = 2024-04-04
+++

So, I'm currently building a tiny little thing called minuteman.
It's an esp-idf based alarm clock, complete with a 7 segment display. 
It's in its early stages. I want to use a rotary encoder as input, 
so I can modify the alarm time, but lets go one step at a time.

The basic test code for the rotary encoder looks like this:

```c
// headers and defines

typedef struct {
    char digits[9];
    max7219_t* display;
} minuteman_t;

static max7219_t display;
static minuteman_t device;

void render_display(minuteman_t* dev){
    max7219_clear(dev->display);
    max7219_draw_text_7seg(dev->display, 0, dev->digits);
}

void minuteman_init(minuteman_t* dev, max7219_t* display){
    dev->display = display;
    sprintf(dev->digits, "00000000");
}

void encoder_test(void *arg)
{
    rotary_encoder_event_t e;
    int32_t val = 0;

    ESP_LOGI(TAG, "Initial value: %" PRIi32, val);
    while (1)
    {
        xQueueReceive(event_queue, &e, portMAX_DELAY);

        switch (e.type)
        {
            case RE_ET_BTN_PRESSED:
                ESP_LOGI(TAG, "Button pressed");
                break;
            case RE_ET_BTN_RELEASED:
                ESP_LOGI(TAG, "Button released");
                break;
            case RE_ET_BTN_CLICKED:
                ESP_LOGI(TAG, "Button clicked");
                break;
            case RE_ET_BTN_LONG_PRESSED:
                ESP_LOGI(TAG, "Looooong pressed button");
                break;
            case RE_ET_CHANGED:
                val += e.diff;
                sprintf(device.digits, "%08d", val);
                render_display(&device);
                break;
            default:
                break;
        }
    }
}

void app_main(void)
{
    ESP_ERROR_CHECK(nvs_flash_init()); 
    display_init(&display);
    encoder_init(&encoder);
    minuteman_init(&device, &display);
    xTaskCreate(encoder_test, TAG, configMINIMAL_STACK_SIZE * 8, NULL, 5, NULL);
    vTaskDelay(pdMS_TO_TICKS(30000));
    for(;;);
}

```

So far so good. It works.
But I want to also add a timer ticker task and several other goodies that might
want to make use of the `&device`, so I need a mutex for my structure and a render task that gets woken up via
an event queue of some sort. I want some minimal task sync manipulation code to make sure everything works as I expect
it to work.

## The same, but async'ed 

```c
// headers and defines

typedef struct {
    char digits[9];
    max7219_t* display;
    SemaphoreHandle_t mutex;
} minuteman_t;

static minuteman_t device;
static TaskHandle_t render_task = NULL;

esp_err_t render_display(minuteman_t* dev){
    if (xSemaphoreTake(dev->mutex, 0) != pdTRUE){
        return ESP_ERR_INVALID_STATE;
    }
    ESP_ERROR_CHECK(max7219_clear(dev->display));
    ESP_ERROR_CHECK(max7219_draw_text_7seg(dev->display, 0, dev->digits));
    ESP_LOGI(__FUNCTION__, "Display updated");
    xSemaphoreGive(dev->mutex);
    return ESP_OK;
}

void minuteman_init(minuteman_t* dev, max7219_t* display){
    dev->display = display;
    sprintf(dev->digits, "00000000");
    dev->mutex = xSemaphoreCreateMutex();
}

static void render_handler(void* arg){
    ESP_LOGI(__FUNCTION__, "Render task started");
    while(1){
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
        ESP_LOGI(__FUNCTION__, "Render task awoken");
        ESP_ERROR_CHECK(render_display(&device));
    }

}

void encoder_handler(void *arg)
{
    rotary_encoder_event_t e;
    int32_t val = 0;

    while (1)
    {
        xQueueReceive(event_queue, &e, portMAX_DELAY);

        switch (e.type)
        {
            case RE_ET_BTN_PRESSED:
                break;
            case RE_ET_BTN_RELEASED:
                break;
            case RE_ET_BTN_CLICKED:
                break;
            case RE_ET_BTN_LONG_PRESSED:
                break;
            case RE_ET_CHANGED:
                val += e.diff;
                if(xSemaphoreTake(device.mutex, 0) == pdTRUE){
                    ESP_LOGI(__FUNCTION__, "Mutex aquired");
                    sprintf(device.digits, "%08d", val);
                    xSemaphoreGive(device.mutex);
                }
                render_display(&device);
                xTaskNotifyGive(&render_task);
                break;
            default:
                break;
        }
    }
}

void app_main(void)
{
    ESP_ERROR_CHECK(nvs_flash_init()); 
    display_init(&display);
    encoder_init(&encoder);
    minuteman_init(&device, &display);
    xTaskCreate(&render_handler, "render task", configMINIMAL_STACK_SIZE * 8, NULL, 5, &render_task);
    xTaskCreate(&encoder_handler, "encoder task", configMINIMAL_STACK_SIZE * 8, NULL, 5, NULL);
    vTaskDelay(pdMS_TO_TICKS(30000));
    for(;;);
}
```

So, nothing coredumped, no stack corruption errors. We get the following logs:

```
I (3791) encoder: Added rotary encoder 0, A: 34, B: 35, BTN: 32
I (3801) render_handler: Render task started
I (12391) encoder_handler: Mutex aquired
```

Which means that at the very least, the mutex code works as intended, but something funky is happening 
with `xTaskNotifyGive`, `ulTaskNotifyTake` or both. I'm going to try to chronologically set out
my thought process:

1. Maybe the tasks weren't started in the correct order and are deadlocked?
   Nope, changing the order doesn't work.

2. Perhaps the tasks don't live long enough because they are not static? Which would be weird as
   we didn't core dumped. Also no.

3. Maybe something to do with the scheduler? In some examples the scheduler seems to be explicitly
   started, so maybe that is it. Ok, something changed, now we are getting actually core dumped. Is this
   a good thing? Maybe the error is somewhere else? I check the earlier startup logs:

```
I (481) cpu_start: Starting scheduler on PRO CPU.
```

I search for this string on the esp-idf source tree and sure enough, 
[the scheduler is being started](https://github.com/espressif/esp-idf/blob/release/v4.2/components/esp32/cpu_start.c#L500),
so I don't think I have to call it twice.

4. Also, the call  to `vTaskDelay`  I copy pasted from an example doesn't make a lot of sense to me, so I'm removing it.

5. Maybe both tasks need to be created on the same core to enable inter-task communication? I change `xTaskCreate` into 
   `xTaskCreatePinnedToCore`. No avail.

6. Maybe the task communication functions I used weren't the appropiate ones?
   Many applications within the esp-idf use [xTaskNotifyWait()](https://www.freertos.org/xTaskNotifyWait.html) and 
   [ulTaskNotifyTake()](https://www.freertos.org/ulTaskNotifyTake.html). No dice.

7. Check if the flag `configUSE_TASK_NOTIFICATIONS` is set. The same for `configTASK_NOTIFICATION_ARRAY_ENTRIES`.

8. Finally, after taking a long walk and staring blankly into the screen for a while, I realized my mistake:

```c
// My declaration
static TaskHandle_t render_task = NULL;

// My function call
xTaskNotifyGive(&render_task);

// Which in turn expands into 
xTaskNotify((&render_task), 0, eIncrement);

// The function signature of xTaskNotify:
BaseType_t xTaskNotify(TaskHandle_t xTaskToNotify, uint32_t ulValue, eNotifyAction eAction);
```

¿Do you see it yet?

Let's go deeper:
```c
// What does TaskHandle_t expand to?
typedef void * TaskHandle_t;
```

So, in short: I was passing `void**` into a function that expects `void*`.

```c
// Now everything is working dandy.
xTaskNotifyGive(render_task);
```


```
I (1689862) render_handler: Render task awoken
I (1689862) render_display: Display updated

```

And that was it.

Don't you _just_ love uncaught typing errors?
