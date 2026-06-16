# Source Code of the Microprocessor

## Theory

### UART

```{code}cpp
:label: code-UART
:caption: Implementation of the termios library to access the serail port of the RPi.

int uart0_filestream = -1;

struct termios options;

void set_interface_attribs(void)
{
    //Open uart
    uart0_filestream = open("/dev/ttyS0", O_RDWR | O_NOCTTY);		//Open in non blocking read/write mode
    if (uart0_filestream == -1) {
        //ERROR - CAN'T OPEN SERIAL PORT
        printf("Error: Unable to open UART\n");
    }

    //Configure uart
    
    tcgetattr(uart0_filestream, &options);

    cfsetispeed(&options, B115200);
    cfsetospeed(&options, B115200);

    options.c_cflag |= (CLOCAL | CREAD);
    options.c_cflag &= ~PARENB;
    options.c_cflag &= ~CSTOPB;
    options.c_cflag &= ~CSIZE;
    options.c_cflag |= CS8;

    options.c_lflag = 0;
    options.c_oflag = 0;
    options.c_iflag = IGNPAR;

    options.c_cc[VMIN]  = 1;
    options.c_cc[VTIME] = 1;

    tcsetattr(uart0_filestream, TCSANOW, &options);
}

bool read_line(int fd, char* buffer, size_t maxlen)
{
    size_t idx = 0;

    while (idx < maxlen - 1)
    {
        char c;

        // read serial character
        int n = read(fd, &c, 1);

        if (n > 0)
        {
            // newline reached
            if (c == '\n')
            {
                break;
            }

            // ignore carriage return
            if (c != '\r')
            {
                buffer[idx++] = c;
            }
        }
    }

    buffer[idx] = '\0';

    return idx > 0;
}

void parameter_aquisition(void)
{
    while(true){
        //----- CHECK FOR ANY RX BYTES -----
        if (uart0_filestream != -1)
        {
            char line[128];

            if(read_line(uart0_filestream, line, sizeof(line)))
            {
                printf("RX = [%s]\n", line);
            }
        }
    }
}

int main(void)
{
    set_interface_attribs();
    parameter_aquisition();
}
```

### Audio output

```{code} cpp
:label: code-alsa-init
:caption: ALSA API initialisation
// Audio
int err;
snd_pcm_t *handle;
// Open and configure the playback device
err = snd_pcm_open(&handle, "hw:CARD=Headphones", SND_PCM_STREAM_PLAYBACK, 0);
if (err < 0) {
    fprintf(stderr, "Playback open error: %s\n", snd_strerror(err));
    return EXIT_FAILURE;
}

err = snd_pcm_set_params(handle,
                            SND_PCM_FORMAT_S16_LE,
                            SND_PCM_ACCESS_RW_INTERLEAVED,
                            1, SAMPLE_RATE, 1, 40000);
if (err < 0) {
    fprintf(stderr, "Hardware configuration error: %s\n", snd_strerror(err));
    snd_pcm_close(handle);
    return EXIT_FAILURE;
}
```

```{code} cpp
:label: code-alsa-frames
:caption: ALSA send frames.
// Send frames to ALSA pipeline
snd_pcm_sframes_t frames = snd_pcm_writei(handle, buffer, BUFFER_SIZE);
if (frames < 0) {
    frames = snd_pcm_recover(handle, frames, 0);
}
if (frames < 0) {
    fprintf(stderr, "snd_pcm_writei failed: %s\n", snd_strerror(frames));
    break;
}
```

### GPIO Control
```{code} cpp
:label: code-gpiod 
:caption: GPIO library implementation.

#include <gpiod.h>
#include <unistd.h>
#include <stdlib.h>

#define S0 17
#define S1 27
#define S2 22
#define S3 23
#define E0 25
#define E1 24

// GPIO chip and request for libgpiod v2.x
struct gpiod_chip *gpiod_chip_handle = NULL;
struct gpiod_line_request *gpio_request = NULL;
unsigned int gpio_offsets[6] = {S0, S1, S2, S3, E0, E1};

void init_gpio(void)
{
    // Open the GPIO chip (typically /dev/gpiochip0 on Raspberry Pi)
    gpiod_chip_handle = gpiod_chip_open("/dev/gpiochip0");
    if (!gpiod_chip_handle) {
        perror("gpiod_chip_open");
        exit(1);
    }

    // Create line configuration for all pins as outputs
    struct gpiod_line_config *line_cfg = gpiod_line_config_new();
    if (!line_cfg) {
        perror("gpiod_line_config_new");
        gpiod_chip_close(gpiod_chip_handle);
        exit(1);
    }

    struct gpiod_line_settings *settings = gpiod_line_settings_new();
    if (!settings) {
        perror("gpiod_line_settings_new");
        gpiod_line_config_free(line_cfg);
        gpiod_chip_close(gpiod_chip_handle);
        exit(1);
    }

    // Configure settings as output
    gpiod_line_settings_set_direction(settings, GPIOD_LINE_DIRECTION_OUTPUT);
    // gpiod_line_settings_set_output_value(settings, GPIOD_LINE_VALUE_ACTIVE);

    // Add all pins with these settings
    if (gpiod_line_config_add_line_settings(line_cfg, gpio_offsets, 6, settings) < 0) {
        perror("gpiod_line_config_add_line_settings");
        gpiod_line_settings_free(settings);
        gpiod_line_config_free(line_cfg);
        gpiod_chip_close(gpiod_chip_handle);
        exit(1);
    }

    // Create request config
    struct gpiod_request_config *req_cfg = gpiod_request_config_new();
    if (!req_cfg) {
        perror("gpiod_request_config_new");
        gpiod_line_settings_free(settings);
        gpiod_line_config_free(line_cfg);
        gpiod_chip_close(gpiod_chip_handle);
        exit(1);
    }

    gpiod_request_config_set_consumer(req_cfg, "wave_generation");

    // Request the lines
    gpio_request = gpiod_chip_request_lines(gpiod_chip_handle, req_cfg, line_cfg);
    if (!gpio_request) {
        perror("gpiod_chip_request_lines");
        gpiod_request_config_free(req_cfg);
        gpiod_line_settings_free(settings);
        gpiod_line_config_free(line_cfg);
        gpiod_chip_close(gpiod_chip_handle);
        exit(1);
    }

    // Clean up temporary configs
    gpiod_request_config_free(req_cfg);
    gpiod_line_settings_free(settings);
    gpiod_line_config_free(line_cfg);
}

void cleanup_gpio(void)
{
    // Release the line request
    if (gpio_request) {
        gpiod_line_request_release(gpio_request);
    }

    // Close the chip
    if (gpiod_chip_handle) {
        gpiod_chip_close(gpiod_chip_handle);
    }
}

void drive_leds(void)
{
    uint8_t value = 0;
    int i;
    float f;
    enum gpiod_line_value array[5];

    while(true) {
        value = 10; //obtain the value from some other variable
        if (value < 0) {
            value = 0;
        } else if (value > 32) {
            value = 32;
        }

        for (i = 0; i < 5; i++) {
            array[i] = static_cast<enum gpiod_line_value>((value >> i) & 1);
        }
        
        // Write values using libgpiod v2.x API
        gpiod_line_request_set_value(gpio_request, gpio_offsets[0], array[0]);  // S0
        gpiod_line_request_set_value(gpio_request, gpio_offsets[1], array[1]);  // S1
        gpiod_line_request_set_value(gpio_request, gpio_offsets[2], array[2]);  // S2
        gpiod_line_request_set_value(gpio_request, gpio_offsets[3], array[3]);  // S3
        gpiod_line_request_set_value(gpio_request, gpio_offsets[4], array[4]);  // E0
        gpiod_line_request_set_value(gpio_request, gpio_offsets[5], static_cast<enum gpiod_line_value>(~array[4])); // E1 (inverted)

        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
}

int main(void)
{
    init_gpio();
    drive_leds();
}
```

### Communication Visualisation
```{code} cpp
:label: code-socket
:caption: A connection via an inet socket.

#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <netinet/in.h>

int sock = socket(AF_INET, SOCK_DGRAM, 0);

int main(void)
{
    // Communication to Python over static internal network
    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    addr.sin_addr.s_addr = inet_addr("127.0.0.1");

    int buffer[100];

    for(int i=0; i < 100; i++)
    {
        buffer[i] = i;
    }

    sendto( sock,
            buffer,
            sizeof(buffer),
            0,
            (sockaddr*)&addr,
            sizeof(addr));
}


```

### Multithreading

```{code} cpp
:label: code-multithread
:caption: Implementation of multithreading in C++.

#include <thread>

void myfunction(void)
{   
    for(int i=0; i < 10; i++)
    {
        printf("i: %d\n", i);
    }
}

int main(void)
{
    std::thread t0(my_function);

    t0.detach();

    // wait 1 second
    std::this_thread::sleep_for(std::chrono::milliseconds(1000));
}
```

## Method

```{include} figures/11.SP_RPI/final_code_rpi.cpp
:lang: cpp
:label: code-rpi-full
:caption: The full integration of all required features of the RPi.

```