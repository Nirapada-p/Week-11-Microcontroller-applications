## คำถามสำหรับการทดลองที่

```
1. เมื่อหมุน Potentiometer ไปทางซ้ายสุด ค่า ADC ที่ได้คือเท่าไร?
        ADC 4095 แรงดัน 3.14 v. 100%
    2. เมื่อหมุน Potentiometer ไปทางขวาสุด ค่า ADC ที่ได้คือเท่าไร?
        ADC 0 แรงดัน 0.14 v. 0%
    3. หากต้องการให้ค่า ADC อยู่ประมาณ 2048 ต้องหมุน Potentiometer ไปที่ตำแหน่งใด?
       ตำแหน่งตรงกลางของ Potentiometer
    4. ความผิดพลาดของการวัด (Error) มีสาเหตุมาจากอะไรบ้าง?
       ข้อจำกัดของ ADC เอง (quantization, linearity, Vref ไม่ตรง) และ สัญญาณรบกวน รวมถึงการต่อวงจร ที่ทำให้แรงดันจริงไม่ถึงค่าที่กำหนดหรือค่าที่คาดการณ์
```
  ### การบันทึกผล
    "สร้างตารางบันทึกผลการทดลอง:
    
 ตำแหน่ง Potentiometer | ค่า ADC | แรงดัน (V) | เปอร์เซ็นต์ (%) |
|---|---:|---:|---:|
| ซ้ายสุด | 0 | 0.14 v | 0 % |  
| 1/4 | 1600 | 1.45 v | 31.1 % |  
| 1/2 (กลาง) | 2146 | 1.91 v  | 52.8 % |  
| 3/4 | 3010 | 2.58 v | 73.5 % |  
| ขวาสุด| 4095 | 3.14 v | 100 % |

## การทดลองที่ 2 - การอ่านค่าเซนเซอร์แสง (LDR)

```
หลักการทำงาน
LDR เป็นตัวต้านทานที่เปลี่ยนค่าตามแสง
เมื่อแสงมาก ความต้านทาน LDR ลดลง → แรงดันที่ ADC อ่านได้เพิ่มขึ้น
เมื่อแสงน้อย ความต้านทาน LDR เพิ่มขึ้น → แรงดันที่ ADC อ่านได้ลดลง
```
## โจทย์ท้าทาย ออกแบบระบบควบคุมแสง LED โดยใช้ค่าจาก LDR

```
#include <stdio.h>
#include <stdlib.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/adc.h"
#include "esp_adc_cal.h"
#include "driver/ledc.h"
#include "esp_log.h"

// ------------------ Pin Config ------------------
#define PIN_LDR       ADC1_CHANNEL_7   // LDR ต่อที่ GPIO35
#define PIN_LED       GPIO_NUM_18      // LED ต่อที่ GPIO18

// ------------------ ADC Config ------------------
#define DEF_VREF      1100
#define SAMPLE_COUNT  32               // ลดจำนวน sample
#define ADC_WIDTH_CFG ADC_WIDTH_BIT_12
#define ADC_ATTEN_CFG ADC_ATTEN_DB_11

// ------------------ PWM Config ------------------
#define PWM_TIMER     LEDC_TIMER_0
#define PWM_MODE      LEDC_HIGH_SPEED_MODE
#define PWM_CHANNEL   LEDC_CHANNEL_0
#define PWM_RES       LEDC_TIMER_10_BIT
#define PWM_FREQ      4000             // ปรับความถี่เป็น 4kHz

static esp_adc_cal_characteristics_t *adc_cali;
static const char *TAG = "LDR_PWM";

// ฟังก์ชันเริ่มต้น ADC
static void adc_setup(void) {
    adc1_config_width(ADC_WIDTH_CFG);
    adc1_config_channel_atten(PIN_LDR, ADC_ATTEN_CFG);

    adc_cali = calloc(1, sizeof(esp_adc_cal_characteristics_t));
    esp_adc_cal_characterize(
        ADC_UNIT_1, ADC_ATTEN_CFG, ADC_WIDTH_CFG,
        DEF_VREF, adc_cali
    );
}

// ฟังก์ชันเริ่มต้น PWM
static void pwm_setup(void) {
    ledc_timer_config_t timer_cfg = {
        .speed_mode       = PWM_MODE,
        .duty_resolution  = PWM_RES,
        .timer_num        = PWM_TIMER,
        .freq_hz          = PWM_FREQ,
        .clk_cfg          = LEDC_AUTO_CLK
    };
    ledc_timer_config(&timer_cfg);

    ledc_channel_config_t ch_cfg = {
        .gpio_num   = PIN_LED,
        .speed_mode = PWM_MODE,
        .channel    = PWM_CHANNEL,
        .timer_sel  = PWM_TIMER,
        .duty       = 0,
        .hpoint     = 0
    };
    ledc_channel_config(&ch_cfg);
}

// อ่านค่า LDR แล้ว return ค่าเฉลี่ย
static uint32_t read_ldr(void) {
    uint32_t val = 0;
    for (int i = 0; i < SAMPLE_COUNT; i++) {
        val += adc1_get_raw(PIN_LDR);
    }
    return val / SAMPLE_COUNT;
}

void app_main(void) {
    adc_setup();
    pwm_setup();

    ESP_LOGI(TAG, "เริ่มควบคุมความสว่าง LED ตามแสง");

    while (1) {
        uint32_t adc_val = read_ldr();
        float percent = (adc_val / 4095.0f) * 100.0f;

        // แปลง ADC -> duty (0 - 1023)
        int duty_val = (int)((adc_val / 4095.0f) * ((1 << PWM_RES) - 1));

        ledc_set_duty(PWM_MODE, PWM_CHANNEL, duty_val);
        ledc_update_duty(PWM_MODE, PWM_CHANNEL);

        ESP_LOGI(TAG, "ADC=%d  Light=%.1f%%  Duty=%d", adc_val, percent, duty_val);

        vTaskDelay(pdMS_TO_TICKS(700)); // หน่วงเวลาไม่ให้ถี่เกิน
    }
}
```
## โจทย์ท้าทาย สร้างระบบเตือนเมื่อค่าเซนเซอร์เกินขีดจำกัด

```
#include <stdio.h>
#include <stdlib.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/adc.h"
#include "esp_adc_cal.h"
#include "driver/ledc.h"
#include "esp_log.h"

// --- LDR Config ---
#define LDR_CH         ADC1_CHANNEL_7   // GPIO35
#define ADC_WIDTH_CFG  ADC_WIDTH_BIT_12
#define ADC_ATTEN_CFG  ADC_ATTEN_DB_11
#define DEF_VREF_MV    1100
#define SAMPLE_NUM     32               // ลด sample

// --- Passive Buzzer Config ---
#define BUZZER_PIN     GPIO_NUM_18
#define BUZZER_MODE    LEDC_HIGH_SPEED_MODE
#define BUZZER_TIMER   LEDC_TIMER_0
#define BUZZER_CH      LEDC_CHANNEL_0
#define BUZZER_RES     LEDC_TIMER_10_BIT
#define BUZZER_FREQ    1500             // ปรับเป็น 1.5 kHz
#define BUZZER_DUTY    512

// --- Threshold (raw) ---
#define LDR_DARK_RAW   900
#define LDR_BRIGHT_RAW 1150

static esp_adc_cal_characteristics_t *adc_cali;
static const char *TAG = "LDR_BUZZ";

// ---------- Setup ----------
static void ldr_adc_setup(void) {
    adc1_config_width(ADC_WIDTH_CFG);
    adc1_config_channel_atten(LDR_CH, ADC_ATTEN_CFG);
    adc_cali = calloc(1, sizeof(esp_adc_cal_characteristics_t));
    esp_adc_cal_characterize(ADC_UNIT_1, ADC_ATTEN_CFG, ADC_WIDTH_CFG,
                             DEF_VREF_MV, adc_cali);
}

static uint32_t ldr_read_avg(void) {
    uint32_t total = 0;
    for (int i = 0; i < SAMPLE_NUM; i++) {
        total += adc1_get_raw((adc1_channel_t)LDR_CH);
    }
    return total / SAMPLE_NUM;
}

static void buzzer_setup(void) {
    ledc_timer_config_t timer_cfg = {
        .speed_mode      = BUZZER_MODE,
        .duty_resolution = BUZZER_RES,
        .timer_num       = BUZZER_TIMER,
        .freq_hz         = BUZZER_FREQ,
        .clk_cfg         = LEDC_AUTO_CLK
    };
    ledc_timer_config(&timer_cfg);

    ledc_channel_config_t ch_cfg = {
        .gpio_num   = BUZZER_PIN,
        .speed_mode = BUZZER_MODE,
        .channel    = BUZZER_CH,
        .timer_sel  = BUZZER_TIMER,
        .duty       = 0,
        .hpoint     = 0
    };
    ledc_channel_config(&ch_cfg);
}

static inline void buzzer_play(int on) {
    ledc_set_duty(BUZZER_MODE, BUZZER_CH, on ? BUZZER_DUTY : 0);
    ledc_update_duty(BUZZER_MODE, BUZZER_CH);
}

// ---------- Main App ----------
void app_main(void) {
    ldr_adc_setup();
    buzzer_setup();

    bool is_buzzer_active = false;

    while (1) {
        uint32_t raw = ldr_read_avg();
        float percent = (raw / 4095.0f) * 100.0f;

        if (!is_buzzer_active && raw < LDR_DARK_RAW) {
            is_buzzer_active = true;
        } else if (is_buzzer_active && raw > LDR_BRIGHT_RAW) {
            is_buzzer_active = false;
        }

        buzzer_play(is_buzzer_active);

        ESP_LOGW(TAG, "LDR Raw=%u (%.1f%% light) | Buzzer=%s",
                 raw, percent, is_buzzer_active ? "ON" : "OFF");

        vTaskDelay(pdMS_TO_TICKS(400)); // เวลาหน่วงต่างจากเพื่อน
    }
}
```

 # คำถามท้าทาย การคำนวณความละเอียด:

## ADC 12-bit มีความละเอียดเท่าไร? (ในหน่วย mV เมื่อใช้ช่วง 0-3.3V)
<img width="580" height="122" alt="image" src="https://github.com/user-attachments/assets/2b45d9f4-63d5-4f80-811b-e854abf13891" />

```
ประมาณ 0.805 mV ต่อ 1 LSB
```


## หากต้องการความละเอียด 1mV ต้องใช้ ADC กี่บิต?

```
ประมาณ 12-bit
```


## การวิเคราะห์ error: Quantization error สูงสุดของ 12-bit ADC คือเท่าไร?
```
±0.805 mV
```

## เหตุใดการใช้ oversampling จึงช่วยลด noise ได้?
```
Oversampling คือการอ่าน ADC หลายครั้ง แล้วหาค่าเฉลี่ย
Noise ที่เกิดขึ้นมักสุ่ม (random)  เฉลี่ยหลายครั้ง ค่าสุ่มจะถูกลดลงตามสถิติ
```

