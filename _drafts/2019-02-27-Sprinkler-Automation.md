---
layout: post
title: Sprinkler Automation
tags: [Hardware, Raspberry Pi, Arduino, Web Development]
---

# Open Source Control

I set out to create an interface with the home sprinkler system in my rental house. Looking to add some features to the sprinkler that are not present in the existing system like web control, rain sensing, alternative schedules that work with the watering restrictions enforced by my city. Being that this is a rental I need to accomplish this without completely butchering the existing system, costing me my deposit. I need to leave the system the way I found it when I move out. This also presented as a great way to build something with all the various electronic parts that I have acquired over the years. So I dug into the system and learned all I could about sprinkler controls and what voltages I would be dealing with.

I have some past experience with arduino platform, though nothing to this level. I quickly found that the 24-28vDC that I was dealing with would not work directly with the microprocessor or the GPIO out of the Pi. Time for a quick lesson in electrical engineering and transient loads. The Raspberry and Arduino work great with low voltage loads, not so much with the voltage that the solenoids required out in my yard.

Before anyone says that I can accomplish this with just a Raspberry Pi or an arduino all on their own, I know. I didn't have an arduino Ethernet on hand to setup a web server for the control, and I didn't want to buy anything for this build. "But why not use the RaspberryPI GPIO to switch the relays?" I have other plans for those pins in the grand scheme of things and figured it was best to let the micro do what it does best. I may add more to this build as it will control my whole house audio system and monitor and control an equipment rack that lives in my garage.

## Project Constraints

I wanted to build this out of parts that I have in my garage already, and surprisingly I had almost everything on hand to assemble this thing on day one. Past this there were a few other requirements that drove the design in the direction I ended up in.

**Requirements -**

* Allow an override to the whole system for those days it rains
* Independent control of each sprinkler zone
* graphical feedback on which zone was on
* The ability to return this to the original state when we move out (It's a rental after all)
* Custom schedules for watering
* Additional zones not available on the basic sprinkler controller (Drip Zones for the Veggie garden)

**Wish list -**

* Moisture sensing automatic watering
* Automatic rain cutoff (If it has rained don't waste the water)
* Free dinner


## Parts

As I said most of the parts I had on hand already which greatly reduced the initial cost for this. 

Here's the list of parts I used to set this al up.

* Arduino UNO
* Raspberry Pi b+
* SPST relay 5v trigger (AMAZON)
* NPN transistor 2N222
* 1k resistors
* 10k resistors
* 4148 diode
* Blue LED
* Red LED
* Green LED
* 75HC595 Shift Regulator
* Perf Board
* Jumper Wire
* Headers
* 12 port terminal strip
* spade lugs
* 14ga wire
* Scrap Plywood
* rain sensor (I had to purchase this Rainbird unit)



## Build 

I have connected the relays NC pin to the sprinkler system to allow normal operation most of the time, and during arduino “outages”. The common pin on the relay connected to the solenoids in the yard. I connected 24v directly from the power supply to the NO pin of the relay to open the solenoids manually when wanted. To accomplish a complete system override I pulled the COM leg from the sprinkler controller through a relay as well. Connected the same as each zone except the NO pin is not connected. This allows me to override when its rained, or is going to rain.

I used the plywood as a backboard to mount all the necessary goodies for the project. The existing sprinkler controller got remounted to the plywood as well. 

Heres the code for the arduino;


```c++
//Arduino wired up to 2 74hc595 shift regulatiors connected to transistors, leds and relays to
//control higher voltage than arduino/74hc595 can
//------------------------------------------------------------------------
// constants won't change. Used here to 
// set pin numbers:
int SER_Pin = 11;   //pin 14 on the 75HC595
int RCLK_Pin = 8;  //pin 12 on the 75HC595
int SRCLK_Pin = 12; //pin 11 on the 75HC595
int MAN_SW = 2; //switch connected to pin 2
int RAIN_SENS = A0; //Rain sensor on roof
int TEMP_SENS = A1; //Temp sensor wired to pin A1
int SOIL_SENS1 = A2; //Soin Sensor in the ground of veggie garden bare nails
int SOIL_SENS2 = A3; //Soin Sensor in the ground of veggie garden plaster mold
//int SOIL_SENS3 = A4; //Soin Sensor in the ground of the ??? FUTURE 
//------------------------------------------------------------------------
// Variables will change:
int rainVal = 0; // variable to store the value coming from the sensor
int tempVal = 0; // variable to store the value coming from the sensor
int soilVal1 = 0; //Variable to store the value coming from the sensor
int soilVal2 = 0; //Variable to store the value coming from the sensor
//int soilVal3 = 0; //Variable to store the value coming from the sensor
//------------------------------------------------------------------------
// State machine for sensor checking. Will use the time set in interval to wait. 
long sensorMillis = 0;        // will store last time SENSORS were updated
// the follow variables is a long because the time, measured in miliseconds,
// will quickly become a bigger number than can be stored in an int.
long sensorInterval = 30000;           // interval at which to check in seconds)
//------------------------------------------------------------------------ 
//State machine for the longest allowed run time of a zone. Used as a fail safe if
//you leave a zone on. set a variable to the current time when zone on, then 
unsigned long runtimeMillis = 0;        // will store last time the Zone was turned on
long RunTimeOn = 1;     // milliseconds of on-time
long RunTimeOFF = 300;    // milliseconds of on-time
//------------------------------------------------------------------------ 
//How many of the shift registers - change this
#define number_of_74hc595s 2 
//------------------------------------------------------------------------
//do not touch
#define numOfRegisterPins number_of_74hc595s * 8
boolean registers[numOfRegisterPins];
//------------------------------------------------------------------------
boolean hasRained = false;
boolean veggieDry1 = false;
boolean veggieDry2 = false;
//boolean GardenDry = false;
boolean isHot = false;
//------------------------------------------------------------------------
void setup(){
  Serial.begin(9600);
  pinMode(SER_Pin, OUTPUT);
  pinMode(RCLK_Pin, OUTPUT);
  pinMode(SRCLK_Pin, OUTPUT);

  //reset all register pins
  clearRegisters();
  writeRegisters();
}               

//set all register pins to LOW
void clearRegisters(){
  for(int i = numOfRegisterPins - 1; i >=  0; i--){
     registers[i] = LOW;
  }
} 

//Set and display registers
//Only call AFTER all values are set how you would like (slow otherwise)
void writeRegisters(){
  digitalWrite(RCLK_Pin, LOW);
  for(int i = numOfRegisterPins - 1; i >=  0; i--){
    digitalWrite(SRCLK_Pin, LOW);
    int val = registers[i];
    digitalWrite(SER_Pin, val);
    digitalWrite(SRCLK_Pin, HIGH);
  }
  digitalWrite(RCLK_Pin, HIGH);
}

//set an individual pin HIGH or LOW
void  RelayZone(int index, int value){
  registers[index] = value;
}

//check sensor values and set status accordingly/
void checkVal (){
rainVal = analogRead(RAIN_SENS);
Serial.print("Rain Sensor,   ");
Serial.print(rainVal);
Serial.print("\t");
tempVal = analogRead(TEMP_SENS);
Serial.print("Temp Sensor,   ");
Serial.print(tempVal);
Serial.print("\t");
soilVal1 = analogRead(SOIL_SENS1);
Serial.print("Soil 1 Sensor,   ");
Serial.print(soilVal1);
Serial.print("\t");
soilVal2 = analogRead(SOIL_SENS2);
Serial.print("Soil 2 Sensor,   ");
Serial.print(soilVal2);
Serial.print("\t");
//soilVal3 = analogRead(SOIL_SENS3);
//Serial.print("Soil 3 Sensor,   ");
//Serial.print(soilVal3);
//Serial.print("\t");

Serial.print("\n");

}

//Set status of sensor

// use to set indivigual zones high.
  //RelayZone(1, HIGH);  //Zone 1
    //RelayZone(2, HIGH);  //Zone 2
    //RelayZone(3, HIGH);  //Zone 3
    //RelayZone(4, HIGH);  //Zone 4
    //RelayZone(5, HIGH);  //Zone 5
    //RelayZone(6, HIGH);  //Zone 6
    //RelayZone(7, HIGH);  //Zone 7
    //RelayZone(8, HIGH);  //Zone 8
    //RelayZone(9, HIGH);  //Zone 9
    //RelayZone(10, HIGH);  //Future zone
    //RelayZone(11, HIGH);  //Future zone
    //RelayZone(12, HIGH); //Connected to the COMM leg for RAIN cutoff
    //RelayZone(13, HIGH);  //Not Connected
    //RelayZone(14, HIGH);  //Not Connected
    //RelayZone(15, HIGH);  //Not Connected
    //RelayZone(16, HIGH);  //Not Connected

void loop(){
  
  // check to see if it's time to update the LED; that is, if the 
  // difference between the current time and last time you Checked 
  // the Sensors is bigger than the interval at which you want to 
  // check the Sensors.
  
  unsigned long currentMillis = millis(); //used as the refference for the time. Checks millis for time and sets currentMillis

 //Serial.print("\n");

  if(currentMillis - sensorMillis > sensorInterval) {
    Serial.print(currentMillis);  //print currentMills
    Serial.print("\n");  //newline
    // save the last time you checked the sensors 
    sensorMillis = currentMillis;   
    checkVal();// Call to check the Sensors 
   
   //Set the rain sensors values to trigger if value is greater than XXX
      if (rainVal <= 100){
        hasRained = true;
       RelayZone(12, HIGH); //Connected to the COMM leg for RAIN cutoff
  }
      else {
        hasRained = false; 
       RelayZone(12, LOW); //Connected to the COMM leg for RAIN cutoff
  }
  }
  
  

  writeRegisters();  //MUST BE CALLED TO DISPLAY CHANGES
  //Only call once after the values are set how you need.
}
```


I used some perf-board to setup the relay circuit. It was what I had on hand that was large enough to hold all those relays. This turned out to be a pain to solder up, but I managed. This allowed me enough room to mount the arduino directly to it and left room for a rain sensor and moisture sensors to come later. Once I got the polarity correct on the LEDs and tested the board with a basic sketch I mounted it to the plywood. This turned out better than I though it would. I sanded the LEDs to diffuse the light some. They were way to bright without.
I mounted some male/female headers to the perf board to interface the Relays and LED's to the arduino.  

We soldered up a board to handle the connection of the 75HC595 shift regulator, and transistors with all required resistors. This shield allows us to break out the various pins from the arduino we have freed up by using the shift regulators. We now route 3 digital pins to the zone control functions, and through code we can control a ton of sprinkler zones, 16 as configured but could be expanded easily. 

I will have an actual shield printed up after successful testing of the circuit and will incorporate relays, LEDs and power for both raspi, arduino and the sprinkler timer / zones.


I am using terminal strips to connect the sprinkler solenoids to the arduino, as well as the sprinkler controller through the relays. These are mounted to the plywood in various places. Whatever made wiring easy.

I wired the solenoids to the bottom left terminal strip, made jumpers to run to the arduino terminal strip and routed along the bottom.
Then I made jumpers from the sprinkler controller to the arduino control board.
I soldered  leads to the relays on the perf board and conneted them to the terminal strips. One lead from Normally Closed (NC), Common ( C), and Normally Open (NO) 
Having multiple terminal strips allows me to patch together the correct wiring for the system. I was not sure what the solenoids needed for me to be able to override the system when I started and I wanted the most flexibility.








The Code - 

We have various parts of the code here and I will break it out in parts to understand whats going on.

Shift Register control - 
 


I'm using the Ethernet library to serve up a web page with some buttons on it to turn on or turn off the zones. This proves useful for a test sketch to allow independent control of the relays. 
It turns out that the Ethernet library consumes a ton of memory. This is the first sketch I have had fill an arduino. I managed to get a page written that will show the status of the digital pins connected to zone relays. I had initially intended on letting the arduino serve the entire page for control and monitoring, but both will not fit. Global variables use 1,798 bytes and this is the minimal amount of sketch that I know how to write. Im sure it can be trimed down a bit more, or re-written completely to work. But as they say “that wil do pig, That will do.

To control the arduino you send a get or post command from a form, or link a button to the address with the command prepended to the end of the ip address. So for instance, if I wanted to trigger relay 1 on my system I would say http://192.168.10.15/?relay1on since that is the command the I programmed the arduino to listen for. This can be changed to anything that would work for the project to make it clear what is going on. When you send the get request you are redirected to the arduino served page, which shows the status of the relays curently.

NEED TO ADD A RAIN SENSOR and program it.


Notes -
During the install I learned a few things and I would like to make my pitfals your success.

* This circuit will only turn on 4 relays simultaneously
        ◦ 5 will overdrive the arduino with out an external power supply
* Perf board is not the best for a perminate install. I will look at making a printed circiut board for this in the future to make more stable
        ◦  If I did this again I would use a blank circuit bord with solder pads on each hole
* the led anode needs to go to 5v not ground. I had them wired incorrectly and had all lights with low status
