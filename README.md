# PID Project
## Table of Contents
* [Planning](#Planning)
* [Code](#Code)
* [CAD](#CAD)
* [Assembly](#Assembly)

## Planning
### Project Description and Requirements
😃 Design, Build, and Program a device that uses PID feedback control. 😃 

Requirements:
Use only an Arduino and other standard components in the Sigma Lab
Use 4 or 6 AA batteries and a battery pack for power
Include a power switch and an LED power indicator
Ensure everything is securely mounted (the Arduino can use standoffs or rubber feet)

We created a ping pong ball floater using PID limits. The motor is housed in a box with a tube extending out of the top and is controlled by a rotary controller and LCD. 

### Bill of Materials
* Metro and prototyping shield
* DC motor
* Distance sensor
* LCD screen
* 9V battery pack
* Fan blades (3D printed)
* Rotary encoder
* Switch
* Ping pong ball
* Plastic tube

### Planning Document

Link to the [planning document](https://docs.google.com/document/d/1F2JCuq7YCHW3Lv81fLc3tMGUtD7HIeB3eArvS5VSfUk/edit)

### Sketches
<img src="https://github.com/honklin/PID-Project/assets/112962114/38efeac4-c5ee-45e3-8089-61fcac12e671" width="250" height="350">



<img src="https://github.com/honklin/PID-Project/assets/112962114/1dfa361e-4c4d-43a3-a721-a11e3b7278d5" width="250" height="350">

## Code

### Description
The code for this project uses PID to control the speed of a motor based on the input of a distance sensor to keep a ping pong ball floating at a certain height.

### Wiring Diagram

![PID Wiring](https://github.com/honklin/PID-Project/assets/121810694/be6a0463-50cd-4c57-927b-61d566eccf6d)
We made our wiring diagram neat and organized because otherwise the wiring is very hard to understand

### Code
```python
# Lucia and Hazel
# PID Project
# This code will use PID to increase or decrease the speed of a fan
# depending on the height of the ball in order to keep the ball at the correct height
import board
from lcd.lcd import LCD # lcd libraries
from lcd.i2c_pcf8574_interface import I2CPCF8574Interface
import adafruit_hcsr04 # distance sensor library
import rotaryio # rotary encoder library
import digitalio
import simpleio # map function library
from PID import PID # PID library
import pwmio   
import time 

pid = PID(100,800,1000) # pid object with pid variables of 100, 800, and 1000
pid.setpoint = 35 # height of ball
pid.output_limits = (24000,34000) # keeps motor from making the ball move too fast

motor = pwmio.PWMOut(board.D8,duty_cycle = 65535,frequency=5000) # motor
motor.duty_cycle = 0

i2c = board.I2C() # lcd object
lcd = LCD(I2CPCF8574Interface(i2c, 0x27), num_rows=2, num_cols=16)

encoder = rotaryio.IncrementalEncoder(board.D3,board.D4) # rotary encoder knob object

button = digitalio.DigitalInOut(board.D2) # rotary encoder button object       
button.pull = digitalio.Pull.UP
button.direction = digitalio.Direction.INPUT

dist = adafruit_hcsr04.HCSR04(trigger_pin = board.D5, echo_pin = board.D6) # distance sensor object

option = -1 # variable that lets user switch functions

def setpoint(): # changes the height the ball is set to
    newHeight = pid.setpoint
    lcd.set_cursor_pos(0,0) # print new position
    lcd.print("Press to set")
    lcd.set_cursor_pos(1,7)
    lcd.print(str(newHeight))
    oldPos = encoder.position
    while (True): # loops until user exits
        oldHeight = newHeight
        pos = encoder.position
        add = pos - oldPos
        if (add > 0 and newHeight < 57): # keeps user from exceeding bounds
            newHeight = newHeight + add
        elif (add < 0 and newHeight > 0):
            newHeight = newHeight + add
        if (newHeight > 57):
            newHeight = 57
        elif (newHeight < 0):
            newHeight = 0
        if (newHeight != oldHeight): # reprints height if knob is rotated
            lcd.clear()
            lcd.set_cursor_pos(0,0)
            lcd.print("Press to set")
            lcd.set_cursor_pos(1,7)
            lcd.print(str(newHeight))
        if (button.value == False): # exits once user presses and releases rotary encoder
            while (button.value == False):
                pid.setpoint = newHeight
            break
        oldPos = pos
    lcd.clear()

def manual(): # allows the user to change the motor speed by twisting the rotary encoder
    motor.duty_cycle = 0
    newSpeed = motor.duty_cycle
    lcd.set_cursor_pos(0,0) # prints new speed
    lcd.print("Motor speed:")
    lcd.set_cursor_pos(1,6)
    lcd.print(str(newSpeed))
    newPos = encoder.position
    oldPosition = encoder.position
    while True: # runs until user exits back to PID control
        oldSpeed = newSpeed
        position = encoder.position
        add = position - oldPosition
        if (add > 0 and newPos < 100): # keeps rotary encoder within limits of 0-100
            newPos = newPos + add
        elif (add < 0 and newPos > 0):
            newPos = newPos + add
        if (newPos > 100):
            newPos = 100
        elif (newPos < 0):
            newPos = 0
        oldPosition = position
        newSpeed = int(simpleio.map_range(newPos,0,100,0,65535)) # converts rotary encoder value to motor speed
        if (oldSpeed != newSpeed): # prints new speed if knob is rotated
            motor.duty_cycle = newSpeed
            lcd.set_cursor_pos(0,0)
            lcd.print("Motor speed:")
            lcd.set_cursor_pos(1,6)
            lcd.print(str(newSpeed))
        if (button.value == False): # exits once user presses and releases rotary encoder
            while (button.value == False):
                lcd.clear()
                motor.duty_cycle = 0
            break

while True: # runs PID control
    try: # checks if distance sensor will throw an error
        height = 57 - dist.distance # counts height from bottom of tube
        speed = int(pid(height)) # PID library outputs speed from height input
        motor.duty_cycle = speed
    except RuntimeError: # retries if distance sensor throws and error
        time.sleep(.001)
    time.sleep(.1)
    if (encoder.position % 10 < 5 and option != 0): # if encoder is rotated to select changing the height
        option = 0 # selects changing setpoint option
        lcd.clear()
        lcd.set_cursor_pos(0,0)
        lcd.print("Press to change")
        lcd.set_cursor_pos(1,0)
        lcd.print("height")
    elif (encoder.position % 10 >= 5 and option != 1): # if encoder is rotated to select manual motor control
        option = 1 # selects controlling the motor manually
        lcd.clear()
        lcd.set_cursor_pos(0,0)
        lcd.print("Press to change")
        lcd.set_cursor_pos(1,0)
        lcd.print("motor speed")
    if (button.value == False): # once user presses and releases rotary encoder
        while (button.value == False):
            motor.duty_cycle = 0
        lcd.clear()
        if (option == 0): # calls change setpoint function
            setpoint()
        if (option == 1): # calls manual motor control function
            manual()
        time.sleep(1)
```
Code for PID Project

## Code Reflection
One of the most difficult parts of this code was controling the user input. The rotary encoder is helpful in that it lets the user select between several different options, but because the knob will count up indefinitely, it makes it difficult to set limits for the output. If I just used encoder.position, the values would keep counting even once they got past the limits, so I would have to twist the knob until it got back within range, but it was difficult to know which was was the right way to twist the knob. Because of this, I had to use several if statements to make sure that the code doesn't overshoot the limits. I also had to use a while loop inside an if statement to make sure that the user had let go of the button before proceeding in the code because otherwise the code would just run straight past and select other options that the user didn't mean to select.

PID is a complicated code concept that essentially adjusts an output based on an input information to keep the input at the right value. In our code, I used PID so that the height of the ball input to the PID function output the correct motor speed to keep the ball at the correct height. It has three variables: Proportional, Integral and Derivative. The Proportional variable controls how fast the motor runs to reach the setpoint. The higher this variable is, the faster the ball will go up, but it will very likely overshoot and need a lot of time to adjust. If this variable is slower, the motor speed will increase slowly and may take a while to reach the correct height. The Integral variable controls how much the motor speed changes as it tries to adjust to get the ball to the right level. If this variable is high, the motor will change values by large amounts and will likely overshoot and undershoot while trying to adjust. If this variable is low, the motor will change very slowly and will take a long time to adjust to the right speed. The Derivative variable shrinks the adjustments as the height reaches the setpoint so that the motor is less likely to overshoot the setpoint. I used output limits on my PID because I measured the speeds for the motor where the ball would lift and fall very slowly so that the ball did not oscillate a lot. Otherwise, the ball would shoot up and down and the PID wouldn't be able to correct the overadjustments fast enough. However, I noticed that I had to adjust the output limits when we changed batteries because the motor would need a faster or slower speed to be able to lift the ball than it had before because it was being supplied with higher or lower voltages.

One factor that I had to keep in mind when using PID to control the motor speed is that it tends to overshoot because the motor will not automatically adjust to the speed it is set to. It takes a few moments for the motor to slowly speed up or slow down to get to the desired speed. This made PID tuning hard because the PID would make the motor increase and decrease by large amounts because the changes didn't kick in immediately.

Another problem that we ran into with the code was that we needed the project to be completely assembled before the code could ever be run or tested. I was able to write a basic outline of the code while Lucia was designing and assembling the box and tube, but I couldn't test my code and my PID especially until it was assembled, which didn't happen until May.
This gave me a limited amount of time to work on the code and tune the PID before the due date.

## CAD

### Description
Link to the [Onshape Document](https://cvilleschools.onshape.com/documents/cae23d9f292dc30b34fb14cb/w/1f6cba44752eedd2b7e14d8a/e/930f21f8e03fe640d7c35464)

### Evidence

<img src="https://github.com/honklin/PID-Project/blob/main/images/Assembly%201%20(2).png" width="400" height="500">

Front view of PID box


Back view of PID box

<img src="https://github.com/honklin/PID-Project/blob/main/images/Assembly%201%20(1).png" width="400" height="500">


Back right view of PID box

<img src="https://github.com/honklin/PID-Project/blob/main/images/Assembly%201.png" width="400" height="500">


Side angle of fan 
<img src="https://github.com/honklin/PID-Project/assets/112962114/ebe8a898-2f9b-4f17-b540-33bbbbd98565" width="400" height="500">

## CAD Reflection
In the begining of the project, CAD design was started before we fully understood what method was going to be used to propell the ball intothe air. I started designing a funnel for a NMB fan. Mr Deirlof kindly desgined a squirrel fan for all of the people with this project idea. After we adapted the fan to our model, we were able to start designing the box around the fan and all. The overall design for creating this in onshape was fairly straightforward and easy. Probably the most important aspect of the design process in onshape would be effectivly using the edit in context tool and basing all the finished project off of the existing parts we had. 

There was quite a bit of T-slot neglegence. I definetley realized what not do when using T-Slots and Laser Joints. The T-Slots/laser joints arent actually hindering any part of the design, they are just really asymetrical and ugly. The only sight problem this created was the asymitericalness caused the top of the box to only fit one way. We reaized this after it was assembled and had to do a bit of reassembly so that al the parts would fit.

Along the way of designing the CAD, I created a seperate context and started using that one to use all of the update-in-contexts. I am not entirey sure of what the effect of doing this was, but it created some confusing senarios I had to navigate my way around. In the future my goal is to attempt to use the original context for as much of the design before I add another context. 
One thing that I learned was the usefulness of using limits on planar mates. For different parts that need to be touching a wall, but also don't need an incredibly presise location, it is very useful to drag the part to where you want it and then use the limits to confine that piece to a rough location. 
## Assembly

### Description
Our assembly consisted of a laser cut box that housed the fan LCD screen, rotary encoder, switch, arduino and battery pack. The tube we used was actually an old face shield that was used during covid. There is a space cut in the back of the box that allows airflow to the fan 

### Video
![gogogo](https://github.com/honklin/PID-Project/assets/112962114/365a28d6-a7f3-48dd-b3e0-f406ac16cda5)


## Assembly Reflection
One thing about the finished project that could be better would be the support for the tube. The tube itself is very tall and with height, it brings a few structural issues. None of these issues are hugely detrimental to the overall soundness of the build, but it does create a little wobble when moving the box and adjusting the various parts of the box.

Another thing that was a bit of an oversight was the wire collection on the side of the tube. Due to the fact that we are this late in the game, the only reasonable solutionto this issue would be using rubberbands or another form of security of wires. Again, this by itself does not show much of an issue, but it reduces proffesionality. 

One win that came of this project was the fact that we only needed to cut one round of acrylic. This was due to the fact that I had made sure there were no visible errors in the CAD before printing anything. In past projects I had not done this to the extent that I did in this project. Even though the engineering teachers stress this fact a lot, this was the first project I had worked one where the effect of this was extremely visable in our progress. During all stages of this project, Hazel and I were consistently ahead of schedule. 

The only thing that put a small slow in the process of fabrication was when the top of the box had to br removes due to the error that came of the CAD flaw. The T-Slots created a situation in which the top sheet was not symmetrical. This created a situation where we had to completely remove the top of the box and take off the LCD, Rotary Encoder and switch which cost us a class period of figuring out the right orientation of the box sides.
