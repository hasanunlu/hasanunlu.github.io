---
layout: post
title:  "Speed of Light Measurement using TI TDC7201"
date:   2017-10-25 21:46:39 -0700
# categories: jekyll update
---
In this experiment, the time delay of light in a 15-meter fiber cable is measured using the TDC7201 Time-to-Digital Converter. Commonly available components were utilized for this setup. The fiber cables employed are inexpensive Toslink optical cables. The measured delay in the 15-meter cable is 99.2 ns. In a vacuum, the light would travel the same distance in 50 ns. Generally, the refractive index of fiber cables is 1.3, but this specific cable seem different. In a previous test using a 10-meter cable of the same brand, the measured delay was 65.8 ns, indicating correlation between the results from the same cable type. Their refractive indexes are same.

**Components list**
* Arduino Due or a similar microcontroller with SPI and 3.3V logic level
* TI TDC7201 evaluation board [datasheet](http://www.ti.com/lit/ds/snas686/snas686.pdf)
* Red light laser
* 2 photodiodes
* 7404 for basic laser diode driving (or any transistor-based driver would suffice)
* OLED or another display (optional), as output is also provided through the serial terminal
* 15-meter optical cable
* Optical splitter

![Experiment Setup](/assets/tdc7201_setup.png)

Arduino Due code

```c
//#define OLED_LCD

#ifdef OLED_LCD
#define OLED_RESET 4
#define NUMFLAKES 10
#define XPOS 0
#define YPOS 1
#define DELTAY 2
#define LOGO16_GLCD_HEIGHT 16 
#define LOGO16_GLCD_WIDTH  16 
Adafruit_SSD1306 display(OLED_RESET);
#endif

#define CS        10
#define EN        7
#define TRIGG     4
#define LASER     5
#define FINISH    8
#define FILT_SIZE 64
#define TDC_CLK   8 //8 MHZ

enum reg_list
{
  TDCx_CONFIG1,
  TDCx_CONFIG2,
  TDCx_INT_STATUS,
  TDCx_INT_MASK,
  TDCx_COARSE_CNTR_OVF_H,
  TDCx_COARSE_CNTR_OVF_L,
  TDCx_CLOCK_CNTR_OVF_H,
  TDCx_CLOCK_CNTR_OVF_L,
  TDCx_CLOCK_CNTR_STOP_MASK_H,
  TDCx_CLOCK_CNTR_STOP_MASK_L,
  TDCx_TIME1=0x10,
  TDCx_CLOCK_COUNT1,
  TDCx_TIME2,
  TDCx_CLOCK_COUNT2,
  TDCx_TIME3,
  TDCx_CLOCK_COUNT3,
  TDCx_TIME4,
  TDCx_CLOCK_COUNT4,
  TDCx_TIME5,
  TDCx_CLOCK_COUNT5,
  TDCx_TIME6,
  TDCx_CALIBRATION1,
  TDCx_CALIBRATION2
};
enum states
{
  START,
  MEASURE
};

volatile uint8_t trig_ready=0;
volatile uint8_t done=0;
uint8_t state=0;
double last[FILT_SIZE];
uint8_t i=0;
double mean;

uint8_t readRegister(uint8_t reg_add) 
{
  SPI.transfer(CS, reg_add, SPI_CONTINUE);
  return SPI.transfer(CS, 0x00, SPI_LAST);
}


void writeRegister(uint8_t reg_add, uint8_t data) 
{
  SPI.transfer(CS, 1 << 6 + reg_add, SPI_CONTINUE);
  SPI.transfer(CS, data, SPI_LAST);
}

uint32_t read3(uint8_t reg_add)
{
  SPI.transfer(CS, reg_add, SPI_CONTINUE);
  uint32_t data = SPI.transfer(CS, 0x00, SPI_CONTINUE) << 16;
  data += SPI.transfer(CS, 0x00, SPI_CONTINUE) << 8;
  data += SPI.transfer(CS, 0x00, SPI_LAST);
  return data;
}

void trigger()
{
  trig_ready = 1;
}

void finish()
{
  done = 1;
}

void setup() {

#ifdef OLED_LCD
  // initialize with the I2C addr 0x3C (for the 128x32)
  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);  
  display.clearDisplay();
  
  display.setTextSize(2);
  display.setTextColor(WHITE);
  display.setCursor(0,0);
  display.println("TDC"); 
  display.display();
#endif

  // initialize digital pin LED_BUILTIN as an output.
  SPI.begin(CS);
  SPI.beginTransaction(SPISettings(50000, MSBFIRST, SPI_MODE0));
  SPI.setClockDivider(CS, 84);

  pinMode(LED_BUILTIN, OUTPUT);
  pinMode(LASER, OUTPUT);
  pinMode(EN, OUTPUT);
  digitalWrite(LASER, HIGH); 

  digitalWrite(EN, LOW); 
  delay(100); 
  digitalWrite(EN, HIGH); 
  delay(100);  

#ifdef OLED_LCD
  display.clearDisplay();
  display.setCursor(0,0);
  display.print("0x"); 
  display.println(readRegister(1), HEX);
  display.print("0x"); 
  display.println(readRegister(3), HEX);
  display.display();
#endif

  Serial.begin(115200);

  pinMode(TRIGG, INPUT_PULLUP);
  attachInterrupt(TRIGG, trigger, RISING);
  
  pinMode(FINISH, INPUT_PULLUP);
  attachInterrupt(FINISH, finish, FALLING);
  interrupts();
}

double tof()
{
  double diff = (double)read3(TDCx_CALIBRATION2);
  diff -= (double)read3(TDCx_CALIBRATION1);
  diff /= (10.0-1.0);
  diff = (1000.0 / TDC_CLK) / diff;
  diff *= (double)read3(TDCx_TIME1);
  return diff;
}

void loop() {

  switch(state){
  case 0:
    writeRegister(TDCx_CONFIG1, 0x01);
    state = 1;
  break;
  case 1:
    if(trig_ready)
    {
      trig_ready = 0;
      digitalWrite(LASER, LOW); 
      state = 2;
    }
  break;  
  case 2:
    delay(20);
    if(done)
    {
      done = 0;
      digitalWrite(LASER, HIGH); 
      state = 3; 
    }
  break;  
  case 3:
    last[++i % FILT_SIZE] = tof();
    mean = 0;
    for(uint8_t j = 0; j < FILT_SIZE; j++)
      mean += last[j];
    
    Serial.println(mean / FILT_SIZE, 1);

#ifdef OLED_LCD
    noInterrupts();
    
    display.clearDisplay();
    display.setTextSize(2);
    display.setTextColor(WHITE);
    display.setCursor(0,0);
    display.print(mean / FILT_SIZE,1); 
    display.println(" ns");
    display.display();
    interrupts();
#endif
    state = 0;
  break;
  default:
  break;
  } 
}
```