---
layout: post
title:  "Full Speed Bit-Bang I2C for RISC-V Architecture HiFive-1 uC Board"
date:   2018-8-27 21:46:39 -0700
# categories: jekyll update
---
The HiFive-1 represents the initial Risc-V board compatible with Arduino. Regrettably, it is only equipped with PWM, UART, and SPI hardware. During the past weekend, I developed a bit-bang I2C implementation that clocks up to 400KHz for my HiFive-1 board. The demonstration involves reading data from an MPU6050 sensor for all axes and displaying the Z-axis value in *g*.

![Reading MPU6050](/assets/i2c_prints.png){:style="display:block; margin-left:auto; margin-right:auto"}
<div align="center">
Reading MPU6050
</div>
\
\
![Experiment setup](/assets/hifive1setup.png)
<div align="center">
Experiment setup
</div>

Full source code is [here](https://github.com/hasanunlu/i2c_demo_for_HiFive1).