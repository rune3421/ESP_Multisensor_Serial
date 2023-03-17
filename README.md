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


Hello and welcome back to Karl Fiddles Around!
Last time we worked on setting up the full collection of actual sensors that would be used for the baby boot. We included an ECG, a PPG, and a Thermistor with code to read accurate temperature, heart rate, respiration rate, and electrical heart pulse. With those, we found a signal filtering problem, that would probably be aggravated by the wifi, so instead of solving the partial problem before hooking it up and coming up against it a second time, we decided to get the full picture by integrating the working code with the Websocket Server to Client to Serial code prepared earlier, to see what the complete sequence of circuits would do to the noise coming from the ground into the ECG. 

Here we go!

For this one, I have created both codes in house, so first step is to open both of them, see if they both still compile, and then start the merge!

The codes I’m merging will be:
BabyBoot_rev3.ino
ESP_to_ESP_Socket_Server
ESP_to_ESP_Socket_Client

Because Highcharts was conclusively too slow to handle a streaming websocket, I’ll first get rid of the website section of the server side, and merge the BabyBoot sensors with the server, set to print to serial just to check for any conflicts. 

Merging the Server and the BabyBoot multisensor worked, the signals still came through Serial from the server side. Now that the server is flashed, I can work on the Client side . 

And, with that, I’ve successfully merged the client side, and checked the signal. 
Although I’m still not getting respiration or heart rates, and the thermistor seems to have burned out (go figure), I have a clear heartbeat and ECG signal. I’m alive!

That said, there does seem to be some stability issues, where the stream lags about half the time, which doesn’t yield a consistent waveform the whole time. There’s also placement problems, where the oximeter will occasionally spike…and I just can’t tell if it’s reading because the size of the chart in the Arduino Serial Plotter doesn’t render the small variations well when the values between signals are extreme. 

So, despite not figuring out the electrode backfeed problem (which we can kick down the road to the actual boot, being that it’s a hardware problem and will be heavily dependent on the final placement and layers of each sensor), we have a working stream!

Now, just for kicks and grins, it would be good to get the thermistor to work, so I’ll check on the reason for its failure and see if switching out will work. 

Doubling back to the original thermistor code without the other peripherals, the thermistor is actually working, so it is probably also suffering from a backflush from the ground corrupting the comparison. This means I have two sources agreeing on the root problem; a diagnosis!

I need to see what is happening with the ground voltage, and see if I can find a way to ensure it’s at a steady 0V…or at least a steady V in general. I do have some microfarad capacitors that should smooth it out…let’s see if hooking those in a couple places in the setup will help smooth out the ground. 

Adding in a microfarad capacitor across the thermistor did nothing to fix the voltage drop error, so I tried checking if there was a variable conflict in the code. I rewrote “T” for temperature as “TC”, but that didn’t change the output either. It definitely is a voltage drop error, which means I’ll need either a digital thermometer or a stronger voltage regulator. 

And, after a bit more research, I’ve found out that the current, not the voltage, is the most important factor for a calibrated Thermistor, so seeing that I’ve been using a set of calculations for calibrated T that doesn’t take into account the new system with altered current values, I’m going to stick with printing the raw sensor values for now, and only calibrate after I’ve finished assembling the whole circuit. 

This also means that I’ll need to calibrate the thermistor for every revision of the circuit from here on out…except that that doesn’t fix the problem. Now that I’m printing the raw voltage reading, I can see that the actual value is 0, meaning there’s no response from the thermistor at all. So, last try before I switch to a digital thermometer again, I’m plugging the thermistor into a completely different power source. 

After that didn’t work, I tried checking whether the thermistor could be swapped for the KY-028 Thermometer module from Elegoo, and since it’s based on the same thermistor probe with the circuitry included, it’s got the same problem. Also, in the middle of checking these things, I discovered that the battery (9v) has already burned down far enough that the circuit couldn’t run anymore. I swapped the battery, and am going to start keeping track of burn time for each battery, and I’m considering adding in some power restriction benchmarks for the system. 

I do have the DS18B20 that was running a digital read error when merged with the other codes before, so I think it’s time to go back to that. The DS might be able to run on lower power, and give a consistent temperature without being influenced by electrical artifacts in the circuit. 

The DS does have a nonblocking option, so we’ll try to merge that with the server side and see what happens. The DS nonblocking works! Ok, now it’s time to try to add in the other two sensors. It wouldn’t be a multisensor suite without all the sensors, right?

So. here’s the full set I’m attempting:

PPG
ECG
Temperature
CO2
pH
Glucose (eventually)

I have standins for the CO2 and pH that I can use. The SGP30 air quality sensor includes a raw ambient CO2 reading. Though it’s not recording venous CO2, it’s close enough to work, and we could check whether ambient CO2 is sensitive enough to be usable. 

I also have an aqueous, analog pH sensor that I’d calibrated earlier, so it can be used again here. Aqueous means it needs a buffer, and venous pH is the most responsive, but we don’t want to stick a needle in the baby, so we’ll at least be able to see if surface pH can do the trick if the sensor is working. 

First up is the CO2 sensor. Since it also works on the QWIIC connect system, I was able to daisychain its wiring with the ECG/PPG sensor. Including the Wire.h could be a problem, but let’s see how it loads. 

And it works! Sorta. There’s definitely a growing lag issue, and the battery life is worrisome already. Looking at the compilation, the server is using 60% of program space, and the client is using 64% 

So, this concludes this session of Karl Fiddles Around, because after all, we’re looking at storing and post-processing the data, so as long as it comes in, it doesn't matter how clean because an ML system is reading it. So, we’re all done! Next up, adding an SD card storage system to the client side. See you there. 

