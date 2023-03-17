# ESP_Wifi_Multisensor
Early Wiring and Software Development for Wireless ECG

Hello and welcome to another episode of Karl fiddles around!

Today we’ll be working on combining all the sensors of the basic baby boot to see if there will be any conflicts between the sensors. I’m starting with heart rate (PPG), respiration rate (RR), electrocardiogram (ECG), blood oxygen (sPO2) and temperature (T). For those I have a Protocentral MAX86150 for the ECG and PPG, and a DS10B20 for T. RR is a bonus signal that can be determined from ECG and PPG, provided the signal is clean enough. With that, the goal is to get as full of a signal as possible from these two sensors plotting in the Arduino Serial Plotter, so that we can next get that signal moving into RStudio for live analysis and then eventually add other sensors to the pack, including CO2, glucose and pH from custom builds. 

Because the base program only takes 21% of program space and 7% of dynamic memory on the ESP32, I’m comfortable skipping the step of using the Arduino for base testing, as adding the wifi and websocket layers won’t add enough overhead to overwhelm the ESP. 

Fortunately, I already have a program available from MIT that can perform Respiration Rate calculations. I’ll be starting with that code, and adding in a DS10B20 temperature probe. 

So, here’s a description of the wiring. 


I plugged in the ESP32 to upload and run the unedited code to see what I get. 
Initially I had to recheck the wiring, but got it to run, and the serial looked pretty clean. 

After that, I started importing the code for the digital temperature sensor, and though it ran, just a straight copy paste slowed down the ecg readings considerably, which means I’ll need to go back and see how to run the temperature sensor without blocking. 

For that, I found this github example for running a nonblocking DS18b20. Trying to compile once, it discovered a missing library, which I went to install and found another library, “nonblockingdallas” which was for the same sensor, which had an example code that I’ll use instead. 

Once merged, I found out there’s some conflict between the libraries, because although the ecg readings can stream again, none of the intermittent readings (HR, RR, or T) display anything but 0. I wasn’t sure about the HR and RR by themselves because there was documentation on the sensor saying that without the electrodes there would be artifacts preventing calculation of HR and RR, but the T is definitely not 0C in the house, so I’ll need to go through and debug the conflict that’s preventing proper reading of the temperature sensor. This is a fork in the road, because the easier route would be to switch to a thermistor, but I would prefer to get digital probe readings, so that norming to standard measurements is already hardwired into the system for measuring T, and I won’t have to norm it again later. However, using a thermistor is definitely the easier option because analog readings stream nonblocking automatically. 

So, after some research, it looks like there’s not a simple way to resolve wire.h and onewire.h, so I’m switching to the thermistor. And, with a quick hunt, there is a calibrated version of the thermistor code here, which means I don’t need to fuss with that later either. 

The thermistor works, which means the very next step is to merge this with the ECG code. 

And, it works! The temperature is reading reasonable values for temperature, and that means that the calculations for RR and HR just couldn’t separate out the signal from the artifacts, which means it’s time to use one of the disposable sticker leads on me to see if we can get real HR and RR readings from this same code. 

Wiring for the Thermistor is in the image shown on the references page

Wiring for the MAX86150 follows the QWIIC standard wiring. 

After checking with the electrodes, I found that there’s significant signal noise for the ECG, which is well documented in the online community. The ESP has a phantom ac voltage coming back through the ground because of the other digital sensors running, which means the majority of the signal coming through the ECG is a record of the power draw at each sampling interval for the other chip. I’ll need to rewire the ground somehow to clean up the reference voltage for the ECG, though I’m not sure how. Some online references mention a capacitor to smooth out the signal. However, the same online resources mention that the wifi on the ESP32 also causes flux in the ground, so I probably should wait to solve that problem until it’s all put together, so I know the full scope of the noise issue for the full package. 

Which means…I’m done for this part! I got this signal set: (See reference page)



And I can tell there’s a spO2 signal (PPG) and a ECG signal in the noise. The temperature also reads, though respiration and heart rate can’t read, which is obviously because of the high noise content in the ECG it would try to calculate from…plus possibly other bugs, but since that’s not a sensor issue, and the code was unchanged for that part, I’ll assume the source code for that was robust, and move on to the next stage; setting this up for wifi!
