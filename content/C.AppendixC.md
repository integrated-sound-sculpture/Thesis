(source_code_ESP)=
# Source code of microcontroller
This appendix shows the source code of the microcontroller.

The code is shown as three different parts:
-**main file:** The main puts everything together and gets executed first by the ESP32
-**input control** The input control regulates all the user control inputs. These are: Amplitude, waveform selection, visualization selection, calibration and auto-tune selection. 
-**pulse counter** The pulse counter module measures the input frequency from the oscillator. It will also convert this input frequency to a linear output frequency in the audible range. 



## Main file


```{code} cpp
:label: ESP-main
:caption: Implementation of main file of the microcontroller

#include <Arduino.h>
#include <cmath>
#include <pulse_cnt.h>
#include <input_control.h>
#include <iostream>
#include <chrono>

// UART2 pin definitions for communication with microprocessor
#define RXD2 16
#define TXD2 17

// Calibration constants
double A;
double B;

// Moving average filter variables
float samples[100];   // Buffer storing last 100 frequency measurements
int indx = 0;         // Current index in circular buffer
float sum = 0;        // Running sum of all samples

// Measurement variables
int in_freq;          // Raw input frequency from sensor
float freq;           // Output audio frequency
float d;              // Calculated hand distance
int amplitude;        // Volume/amplitude value from potentiometer

// Auto-tune selection flag
bool auto_tune;

void setup()
{
    // Initialize serial monitor
    Serial.begin(115200);

    // Initialize UART2 communication
    Serial2.begin(115200, SERIAL_8N1, RXD2, TXD2);

    // Initialize frequency measurement hardware
    init_frequencyMeter();

    // Initialize buttons, switches and potentiometers
    init_input_control();

    Serial.println("\nPress the button to input the frequency at 5 cm\n");

    // Number of calibration points
    int n = 4;

    // Known distances used for calibration (meters)
    float d[] = {0.05, 0.10, 0.20, 0.30};

    // Array to store measured frequencies
    float f[n];

    // Measure frequencies at the known distances
    freq_calibration2(f, n);

    //float f[] = {520000,560000,600000,610000};


    // Fit the frequency-distance model:
    fitModel(d, f, n, &A, &B);

    Serial.println("\nThe curve has been fitted A and B are:");
    Serial.println(A, 6);
    Serial.println(B, 6);

    Serial.println("\nSetup done");
    Serial.println("STARTING WITH MEASUREMENTS");

    delay(1500);
}

void loop()
{
    // Measure frequency from sensor
    in_freq = freq_measurement();
    Serial.print(in_freq);

    //--------------------------------------------------
    // Moving average filter (100 samples)
    //--------------------------------------------------

    // Remove oldest sample from running sum
    sum -= samples[indx];

    // Store new sample
    samples[indx] = in_freq;

    // Move to next position in circular buffer
    indx = (indx + 1) % 100;

    // Add new sample to running sum
    sum += in_freq;

    // Compute filtered frequency
    in_freq = sum / 100;

    Serial.print("               ");
    Serial.print(in_freq);

    //--------------------------------------------------
    // Convert frequency to hand distance
    //--------------------------------------------------

    d = convert_hand_lin(A, B, in_freq);

    //--------------------------------------------------
    // Convert distance to audible frequency
    //--------------------------------------------------
    /*
     * Parameters:
     * 4      = octave range
     * 130.8  = lowest note frequency (C3)
     * d      = measured hand distance
     * 30     = maximum distance (cm)
     * 5      = minimum distance (cm)
     */
    freq = convert_freq_log(4, 130.8, d, 0.30, 0.05);

    //--------------------------------------------------
    // Read volume control potentiometer
    //--------------------------------------------------
    amplitude = pot_meter();

    //--------------------------------------------------
    // Auto-tune option
    //--------------------------------------------------
    auto_tune = autotune_select();

    if (auto_tune)
    {
        // Snap frequency to nearest musical note
        freq = autotune(freq);
    }

    //--------------------------------------------------
    // Read waveform selection
    //--------------------------------------------------
    /*
     * Possible values:
     * 0 = Sine
     * 1 = Square
     * 2 = Sawtooth (depending on implementation)
     */
    int waveform = waveform_select();

    //--------------------------------------------------
    // Read visualization mode
    //--------------------------------------------------
    int visualization = visualization_select();

    //--------------------------------------------------
    // Create UART packet
    //--------------------------------------------------
    /*
     * Format:
     * frequency,amplitude,waveform
     *
     * Example:
     * 440.00,128,0
     */
    String UART_data =
        String(freq) + "," +
        String(amplitude) + "," +
        String(waveform) + "\n";

    // Send packet to connected device
    Serial2.println(UART_data);

    Serial.print("                ");
    Serial.println(UART_data);
    delay(2);

}


```





## Input control


```{code} cpp
:label: ESP-controlh
:caption: Implementation of the header file of input control on the microcontroller

#ifndef INPUT_CONTROL_H
#define INPUT_CONTROL_H

const int calibration_button = 5;
const int ampl_slider_Pin = 4;
const int wave_select_pin = 2;
const int autotune_switch = 12;
const int waveform_pin1 = 22;
const int waveform_pin2 = 23;
const int vis_select1 = 18;
const int vis_select2 = 19;

void init_input_control();
void freq_calibration(int* freq_high, int* freq_low);
int pot_meter();
bool autotune_select();
float autotune(float freq);
int waveform_select();
int visualization_select();

void freq_calibration2(float freq[], int n);


#endif

```



```{code} cpp
:label: ESP-control
:caption: Implementation of input control on the microcontroller

#include <Arduino.h>
#include "stdio.h"  
#include "input_control.h"
#include "pulse_cnt.h"


int potValue = 0;
bool lastState = LOW;

void init_input_control(){    //init calibration button and autotune_switch
    pinMode(calibration_button, INPUT_PULLUP);
    pinMode(autotune_switch, INPUT_PULLUP);

}


void freq_calibration(int* freq_high, int* freq_low){  //calibrates the low and high frequencies
  bool calibrate_high_freq = false;
  bool calibrate_low_freq = false;
  bool currentState;

  while(calibrate_low_freq == false){                         // loop until button has been pressed twice
    currentState = digitalRead(calibration_button);         // read the button stat 
    int x = freq_measurement();                             // measure frequency so it is already enabled (is not necessary)

    if (lastState == HIGH && currentState == LOW) {         // detect press (pull down button) only detect when state changes
        if(calibrate_high_freq == false){                     // when high freq has not yet been calibrated 
            *freq_high = freq_measurement();                // measure high freq
            calibrate_high_freq = true;                     // set flag high 
            Serial.println("High frequency calibrated:");       
            Serial.println(*freq_high);
            delay(500);
            Serial.println("\nPress the button to input the lowest frequency\n");

        }
        else{                                               // if high frequencie has already been calibrated
            calibrate_low_freq = true;                      // set flag high, so the while stops
            *freq_low = freq_measurement();                 // measure the low freq
            Serial.println("Low Frequency calibrated");      
            Serial.println(*freq_low);
        }
    }

    lastState = currentState;                              

    delay(100);         //delay to avoid jittering inputs from button

  }
}



void freq_calibration2(float freq[], int n){
    bool currentState;
    bool lastState = LOW;

    int frequency;

    float samples[50] = {0};   // Buffer storing last 100 frequency measurements
    int indx = 0;         // Current index in circular buffer
    float sum = 0;        // Running sum of all samples
    for (int i = 0; i<n; i++){

        while(true){ 

            frequency = freq_measurement();      
            
                // Remove oldest sample from running sum
            sum -= samples[indx];

            // Store new sample
            samples[indx] = frequency;

            // Move to next position in circular buffer
            indx = (indx + 1) % 50;

            //Serial.println(indx);
            // Add new sample to running sum
            sum += frequency;

            //Serial.println(sum);

            // Compute filtered frequency
            frequency = sum / 50;

            //Serial2.println(frequency);
            
            // measure frequency so it is already enabled (is not necessary)
            currentState = digitalRead(calibration_button);   
            
            if (lastState == HIGH && currentState == LOW) {
                freq[i] = frequency;

                Serial.print("frequency calibrated: ");       
                Serial.println(freq[i]);

                Serial2.println("Calibrate 1");
                Serial2.println(freq[i]);

                delay(300);
                Serial.println("\nPress the button to input the next frequency\n");
                lastState = currentState;
                break;
            }

            delay(20);
            lastState = currentState;

        }
    }
}



bool autotune_select(){                                 // autotune selection returns bool when auto tune has been selected
    bool autotune = digitalRead(autotune_switch);
    return autotune;
}

float autotune(float freq){                                //this function changes the a frequency to a perfect note
    float note = 12 * std::log2((float) freq / 65.406);     // 1) change frequency to note this will be a decimal.
    int auto_note = std::round(note);                       // 2) round the note to nearest integer
    float auto_freq = 65.406 * pow(2,(float)auto_note/12);  // 3) convert note to frequency
    return auto_freq;
}

int pot_meter(){                                        // potmeter is used to change amplitude of the sound
    int potValue = analogRead(ampl_slider_Pin);         // read the slider value (0-4096)
    if(potValue < 350){
        potValue = 0;
    }
    return potValue;
}
 

int waveform_select(){
    bool waveform_select1 = digitalRead(waveform_pin1);
    bool waveform_select2 = digitalRead(waveform_pin2);
    int waveform = waveform_select1 + 2 * waveform_select2;
    return waveform;
}


int visualization_select(){
    bool vis_select1 = digitalRead(vis_select1);
    bool vis_select2 = digitalRead(vis_select2);
    int visualization= vis_select1 + 2 * vis_select2;
    return visualization;
}

```




## Pulse counter

```{code} cpp
:label: ESP-counterh
:caption: Implementation of the header file of the pulse counter on the microcontroller

#ifndef PULSE_CNT_H
#define PULSE_CNT_H



void init_frequencyMeter();
int freq_measurement();

float convert_freq_log(int num_octaves, float min_freq, float freq, float in_max, float in_min);
float convert_freq_lin(int in_min, int in_max, int out_min,int out_max, int freq);
float convert_hand_lin(float A, float B, int freq);
void  fitModel(float d[], float f[], int n, double* A, double* B);

#endif 

```

```{code} cpp
:label: ESP-counter
:caption: Implementation of the pulse counter on the microcontroller

#include <Arduino.h>
// BLOG Eletrogate
// ESP32 Frequency Meter
// ESP32 DevKit 38 pins + LCD
// https://blog.eletrogate.com/esp32-frequencimetro-de-precisao
// Rui Viana and Gustavo Murta august/2020

#include "stdio.h"                                                        // Library STDIO
#include "driver/pcnt.h"                                                  // Library ESP32 PCNT
#include "soc/pcnt_struct.h"
#include "pulse_cnt.h"
#include <cmath>




#define PCNT_COUNT_UNIT       PCNT_UNIT_0                                 // Set Pulse Counter Unit - 0 
#define PCNT_COUNT_CHANNEL    PCNT_CHANNEL_0                              // Set Pulse Counter channel - 0 

#define PCNT_INPUT_SIG_IO     GPIO_NUM_34                                 // Set Pulse Counter input - Freq Meter Input GPIO 34
#define PCNT_INPUT_CTRL_IO    GPIO_NUM_35                                 // Set Pulse Counter Control GPIO pin - HIGH = count up, LOW = count down  
#define OUTPUT_CONTROL_GPIO   GPIO_NUM_32                                 // Timer output control port - GPIO_32
#define PCNT_H_LIM_VAL        overflow                                    // Overflow of Pulse Counter 

#define IN_BOARD_LED          GPIO_NUM_2                                  // ESP32 native LED - GPIO 2

bool            flag          = true;                                     // Flag to enable print frequency reading
uint32_t        overflow      = 25000;                                    // Max Pulse Counter value
int16_t         pulses        = 0;                                        // Pulse Counter value
uint32_t        multPulses    = 0;                                        // Quantidade de overflows do contador PCNT
uint32_t        sample_time   = 20000;                                    // sample time of 1 second to count pulses
float           frequency     = 0;                                        // frequency value
char            buf[32];                                                  // Buffer

esp_timer_create_args_t create_args;                                      // Create an esp_timer instance
esp_timer_handle_t timer_handle;                                          // Create an single timer

portMUX_TYPE timerMux = portMUX_INITIALIZER_UNLOCKED;                     // portMUX_TYPE to do synchronism

//----------------------------------------------------------------------------------
static void IRAM_ATTR pcnt_intr_handler(void *arg)                        // Counting overflow pulses
{
  portENTER_CRITICAL_ISR(&timerMux);                                      // disabling the interrupts
  multPulses++;                                                           // increment Overflow counter
  PCNT.int_clr.val = BIT(PCNT_COUNT_UNIT);                                // Clear Pulse Counter interrupt bit
  portEXIT_CRITICAL_ISR(&timerMux);                                       // enabling the interrupts
}

//----------------------------------------------------------------------------------
void init_PCNT(void)                                                      // Initialize and run PCNT unit
{
  pcnt_config_t pcnt_config = { };                                        // PCNT unit instance

  pcnt_config.pulse_gpio_num = PCNT_INPUT_SIG_IO;                         // Pulse input GPIO 34 - Freq Meter Input
  pcnt_config.ctrl_gpio_num = PCNT_INPUT_CTRL_IO;                         // Control signal input GPIO 35
  pcnt_config.unit = PCNT_COUNT_UNIT;                                     // Unidade de contagem PCNT - 0
  pcnt_config.channel = PCNT_COUNT_CHANNEL;                               // PCNT unit number - 0
  pcnt_config.counter_h_lim = PCNT_H_LIM_VAL;                             // Maximum counter value - 20000
  pcnt_config.pos_mode = PCNT_COUNT_INC;                                  // PCNT positive edge count mode - inc
  pcnt_config.neg_mode = PCNT_COUNT_INC;                                  // PCNT negative edge count mode - inc
  pcnt_config.lctrl_mode = PCNT_MODE_DISABLE;                             // PCNT low control mode - disable
  pcnt_config.hctrl_mode = PCNT_MODE_KEEP;                                // PCNT high control mode - won't change counter mode
  pcnt_unit_config(&pcnt_config);                                         // Initialize PCNT unit

  pcnt_counter_pause(PCNT_COUNT_UNIT);                                    // Pause PCNT unit
  pcnt_counter_clear(PCNT_COUNT_UNIT);                                    // Clear PCNT unit

  pcnt_event_enable(PCNT_COUNT_UNIT, PCNT_EVT_H_LIM);                     // Enable event to watch - max count
  pcnt_isr_register(pcnt_intr_handler, NULL, 0, NULL);                    // Setup Register ISR handler
  pcnt_intr_enable(PCNT_COUNT_UNIT);                                      // Enable interrupts for PCNT unit

  pcnt_counter_resume(PCNT_COUNT_UNIT);                                   // Resume PCNT unit - starts count
}

//----------------------------------------------------------------------------------
void read_PCNT(void *p)                                                   // Read Pulse Counter
{
  gpio_set_level(OUTPUT_CONTROL_GPIO, 0);                                 // Stop counter - output control LOW
  pcnt_get_counter_value(PCNT_COUNT_UNIT, &pulses);                       // Read Pulse Counter value
  flag = true;                                                            // Change flag to enable print
}

//---------------------------------------------------------------------------------
void init_frequencyMeter()
{
  init_PCNT();                                                            // Initialize and run PCNT unit

  gpio_pad_select_gpio(OUTPUT_CONTROL_GPIO);                              // Set GPIO pad
  gpio_set_direction(OUTPUT_CONTROL_GPIO, GPIO_MODE_OUTPUT);              // Set GPIO 32 as output

  create_args.callback = read_PCNT;                                       // Set esp-timer argument
  esp_timer_create(&create_args, &timer_handle);                          // Create esp-timer instance

  gpio_set_direction(IN_BOARD_LED, GPIO_MODE_OUTPUT);                     // Set LED inboard as output

  gpio_matrix_in(PCNT_INPUT_SIG_IO, SIG_IN_FUNC226_IDX, false);           // Set GPIO matrin IN - Freq Meter input
  gpio_matrix_out(IN_BOARD_LED, SIG_IN_FUNC226_IDX, false, false);        // Set GPIO matrix OUT - to inboard LED
}

//----------------------------------------------------------------------------------------

int freq_measurement()
{
  if (flag == true)                                                     // If count has ended
  {
    flag = false;                                                       // change flag to disable print
    frequency = (pulses + (multPulses * overflow)) / 2.0;               // Calculation of frequency
    multPulses = 0;                                                     // Clear overflow counter

    pcnt_counter_clear(PCNT_COUNT_UNIT);                                // Clear Pulse Counter
    esp_timer_start_once(timer_handle, sample_time);                    // Initialize High resolution timer (1 sec)
    gpio_set_level(OUTPUT_CONTROL_GPIO, 1);                             // Set enable PCNT count
  }
  return frequency * (1/0.02);
}


//---------------------------------------------------------------------------------

float convert_freq_lin(int in_min, int in_max, int out_min, int out_max, int freq){        //converting the frequency using a linear scale
  float a = (float)(in_max-in_min)/(float)(out_max-out_min);
  float b = in_min - a*out_min;
  float y = a*freq +b;
  return y;
}


//---------------------------------------------------------------------------------

float convert_freq_log(int num_octaves, float min_freq, float freq, float in_max, float in_min){    //converting teh frequency using a logaritmic scale

  float x = (float)(freq-in_min)/(float)(in_max-in_min); //convert input freq to range between 0 and 1
  
  //Serial.println(x);
  float y = min_freq * pow(2,x*num_octaves);  //convert to logaritmic range 
  return y;
}


//---------------------------------------------------------------------------------

// Converts a measured frequency into a hand distance.
//
// The formula is derived from the fitted sensor model:
//
//      f = A / sqrt(1 + B/d)
//
// Rearranging for distance d gives:
//
//      d = B / (A²/f² - 1)
//
// Parameters:
//   A, B  -> calibration constants obtained from fitModel()
//   freq  -> measured sensor frequency
//
// Returns:
//   Estimated hand distance
//

float convert_hand_lin(float A, float B, int freq)
{
    float d = B / (std::pow(A, 2) / std::pow(freq, 2) - 1);
    return d;
}

//---------------------------------------------------------------------------------

// Fits the sensor model:
//
//      1/f² = (1/A²) + (B/A²)(1/d)
//
// This transforms the nonlinear relationship into a linear form:
//
//      Y = c + mX
//
// where:
//
//      X = 1/d
//      Y = 1/f²
//      c = 1/A²
//      m = B/A²
//
// Linear regression is then used to determine m and c,
// from which A and B are recovered.
//

void fitModel(float d[], float f[], int n, double* A, double* B)
{
    // Variables used to compute least-squares regression
    double sumX  = 0.0f;   // ΣX
    double sumY  = 0.0f;   // ΣY
    double sumXX = 0.0f;   // ΣX²
    double sumXY = 0.0f;   // ΣXY

    // Build regression sums from calibration points
    for (int i = 0; i < n; i++)
    {
        // Ignore invalid calibration points
        if (d[i] <= 0 || f[i] <= 0)
            continue;

        // Linearized variables
        float X = 1.0f / d[i];
        float Y = 1.0f / (f[i] * f[i]);

        // Accumulate regression sums
        sumX  += X;
        sumY  += Y;
        sumXX += X * X;
        sumXY += X * Y;
    }

    // Denominator used in least-squares formulas
    float denom = n * sumXX - sumX * sumX;

    // Calculate slope (m) of best-fit line
    float m = (n * sumXY - sumX * sumY) / denom;

    // Calculate intercept (c) of best-fit line
    float c = (sumY - m * sumX) / n;

    // Recover model parameter A
    *A = 1.0f / sqrtf(c);

    // Recover model parameter B
    *B = m / c;
}

```