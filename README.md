# Sistema controlador para puente levadizo

## Memoria descriptiva

El sistema de control del puente levadizo simula el funcionamiento de un puente que puede elevarse y bajarse para permitir el paso de embarcaciones y vehículos. El sistema opera a través de una máquina de estados que gestiona las transiciones entre cuatro estados principales: Espera, Elevando, Elevado y Bajando.

### Estado Inicial: Espera
En este estado, el sistema está en reposo, a la espera de una señal para iniciar el movimiento de apertura del puente. El puente se encuentra en posición horizontal permitiendo el paso de vehículos. Tras la señal de subir, el sistema activará los mecanismos para elevar el puente (pasando al estado "Elevando"). Las barreras de seguridad permanecen en posición alta para garantizar la seguridad del tráfico vehicular.

### Elevando
Al recibir la señal de apertura, el sistema baja las barreras de seguridad para detener el tráfico vehicular (led rojo) y comienza a elevar el puente. Este estado continúa hasta que el puente alcanza la altura máxima establecida. Durante todo el proceso, las barreras permanecen en posición baja, evitando el paso de vehículos. Una vez que se alcanza la altura máxima, el sistema queda en espera de una nueva señal, generalmente para iniciar el descenso.

### Elevado
En el estado "Elevado", el puente ha alcanzado su posición máxima, permitiendo el paso seguro de embarcaciones por debajo. Durante este estado, el sistema permanece en espera de una señal de cierre para iniciar el descenso. Las barreras de seguridad para vehículos permanecen bajas, evitando el tráfico sobre el puente hasta que el proceso de bajada haya concluido. Este estado asegura que el puente se mantenga elevado de manera estable y sin movimiento hasta que se indique el siguiente cambio.

### Bajando
Tras recibir la señal de cierre, el sistema activa los mecanismos para bajar el puente de manera controlada. Las barreras de seguridad permanecen bajas durante todo este proceso, evitando el paso de vehículos hasta que el puente haya descendido completamente. El estado "Bajando" se mantiene hasta que el puente alcanza la posición horizontal. Al finalizar, las barreras de seguridad se levantan (led verde), permitiendo nuevamente el paso de vehículos, y el sistema regresa al estado de "Espera".


## Maquina de estados

A continuacion, se presenta un modelo de maquina de estados para el proyecto.

![](./maquina.png)


## Presentacion en Proteus

Se utiliza para el proyecto un microcontrolador ATMega 128, un puente H L298 para controlar el motor, dos leds (rojo y verde) para simular la barrera, un motor DC de 12v, y un potenciometro para leer la altura del puente levadizo.

![](./proteus.png)

## Firmware

### mylib.h

```c
#ifndef MYLIB_H
#define MYLIB_H
#include <avr/io.h>
#include "avr_api/avr_api.h"

#define ALTURA_MAX 127
#define ALTURA_MIN 0

//definiciones para los pulsadores de subir y bajar
#define SWITCH_PORT avr_GPIO_A
#define SWITCH_DOWN_PIN avr_GPIO_PIN_1
#define SWITCH_UP_PIN avr_GPIO_PIN_0
#define SWITCH_UP avr_GPIOA_IN_0
#define SWITCH_DOWN avr_GPIOA_IN_1

// Definiciones de puertos de salida
#define OUT_PORT avr_GPIO_D
#define LED_ROJO_PIN avr_GPIO_PIN_0
#define LED_VERDE_PIN avr_GPIO_PIN_1
#define ENA_PIN avr_GPIO_PIN_4
#define IN1_PIN avr_GPIO_PIN_3
#define IN2_PIN avr_GPIO_PIN_4
#define LED_ROJO avr_GPIOD_OUT_0
#define LED_VERDE avr_GPIOD_OUT_1
#define IN1 avr_GPIOD_OUT_2
#define IN2 avr_GPIOD_OUT_3
#define ENA avr_GPIOD_OUT_4

typedef enum {
    espera = 0,
    elevando = 1,
    elevado = 2,
    bajando = 3
} estados_t;

//prototipos estados
estados_t *(f_espera_puente)(void);
estados_t *(f_elevando_puente)(void);
estados_t *(f_elevado_puente)(void);
estados_t *(f_bajando_puente)(void);

//prototipos funciones
void init_puente(void);
void subir_barrera(void);
void bajar_barrera(void);
void activar_motor_subir(void);
unsigned int leer_sensor(void);
void activar_motor_bajar(void);
void apagar_motor(void);
void systick_led(void);

#endif
```
### estados.c

```c
#include "mylib.h"

estados_t *(f_espera_puente)(void) {
    if (SWITCH_UP==0) {
        return elevando;
    } else {
        return espera;
    }
}

estados_t *(f_elevando_puente)(void) {
       bajar_barrera();
       activar_motor_subir();
       apagar_motor();
    return elevado;
}

estados_t *(f_elevado_puente)(void) {

    if (SWITCH_DOWN==0) {
        return bajando;
    } else {
        return elevado;
    }
}

estados_t *(f_bajando_puente)(void) {
    activar_motor_bajar();
    subir_barrera();
    apagar_motor();
    
    return espera;
}

```

### funciones.c

```c
#include "mylib.h"

volatile unsigned int led_time=0;

void init_puente(void) {
    GpioInitStructure_AVR salida;
    GpioInitStructure_AVR entrada;
    //AdcInitStructure_AVR adc_config;
    SystickInitStructure_AVR systick_conf;

    //configuracion de pines de salida
    salida.port = OUT_PORT;
    salida.modo = avr_GPIO_mode_Output;
    salida.pines = LED_ROJO_PIN | LED_VERDE_PIN | ENA_PIN | IN1_PIN | IN2_PIN;
    init_gpio(salida);
    
    //configuracion de pines de entrada para los switch
    entrada.port = SWITCH_PORT;
    entrada.modo = avr_GPIO_mode_Input; 
    entrada.pines = SWITCH_DOWN_PIN | SWITCH_UP_PIN; //input SUBIR / BAJAR
    init_gpio(entrada);

    /*Configuracion del ADC
    adc_config.mode = avr_ADC_MODE_Single_Conversion; 
    adc_config.prescaler = avr_ADC_Prescaler_64;      // Divisor del reloj: 64
    adc_config.channel = avr_ADC_canal0;             // Canal ADC0 (PF0)
    adc_config.resolution = avr_ADC_RES_8Bit;       // mapeo de 8 bits
    adc_config.reference = avr_ADC_REF_AVcc;         // referencia AVcc
    init_adc(adc_config);
    /*/
    // Inicializar el ADC
    //if ( init_adc(*adc_config)!= avr_ADC_OK) {
     //   return -1;
    //}
    
    LED_VERDE = 1; 
    SWITCH_DOWN = 1; 
    SWITCH_UP = 1; 
    
    //configuracion del timer
    systick_conf.timernumber = avr_TIM0;
    systick_conf.time_ms = 1;
    systick_conf.avr_systick_handler = systick_led;
    init_Systick_timer(systick_conf);
    sei();
    return;
}

void bajar_barrera(void) {
    int ciclos=0; 
    while (ciclos < 4) {
        if (led_time > 500) {
            LED_ROJO ^= 1;
            led_time = 0;
            ciclos += 1;
        }
    }
    LED_VERDE=0; 
    return;
}

void activar_motor_subir(void) {
    ENA = 1;   //activar el motor ENA=1
    // Rotacion en direccion de elevacion
    IN1 = 1;   // IN1=1
    IN2 = 0; //IN2=0
    return;
}

unsigned int leer_sensor(void) {
      return leer_ADC(avr_ADC_canal0);
}

void activar_motor_bajar(void) {
    ENA = 1;    //activar el motor
    //configuraciÃ³n para rotacion en direccion de descenso
    IN1 = 0;  
    IN2 = 1;
    return;
}

void subir_barrera(void) {
    int ciclos=0;
    activar_motor_bajar();
    apagar_motor();
    while (ciclos < 4) {
        if (led_time > 500) {
            LED_ROJO ^= 1;
            led_time = 0;
            ciclos += 1;
        }
    }
    LED_VERDE=1; //apagar LED verde
    return;
}

void apagar_motor(void) {
    ENA = 0; 
    return;
}

void systick_led(void) {
    led_time++;
    return;
}
```
### main.c
```c
#include "mylib.h"

int main() {
    estados_t estado_actual = espera;
    estados_t (*vectorEstados[])(void) = {f_espera_puente, f_elevando_puente, f_elevado_puente, f_bajando_puente};

    init_puente();    

    while (1) {
        estado_actual = vectorEstados[estado_actual]();
    }
 return 0;
}
```