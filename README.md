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

### Planning Document

Link to the [planning document](https://docs.google.com/document/d/1F2JCuq7YCHW3Lv81fLc3tMGUtD7HIeB3eArvS5VSfUk/edit)

### Sketches

## Code

### Description
The code for this project uses PID to control the speed of a motor based on the input of a distance sensor to keep a ping pong ball floating at a certain height.

### Code
```python
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
        if (button.value == False): # exits once user presses and releases button
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
        if (add > 0 and newPos < 100):
            newPos = newPos + add
        elif (add < 0 and newPos > 0):
            newPos = newPos + add
        if (newPos > 100):
            newPos = 100
        elif (newPos < 0):
            newPos = 0
        oldPosition = position
        newSpeed = int(simpleio.map_range(newPos,0,100,0,65535)) 
        if (oldSpeed != newSpeed):
            motor.duty_cycle = newSpeed
            lcd.set_cursor_pos(0,0)
            lcd.print("Motor speed:")
            lcd.set_cursor_pos(1,6)
            lcd.print(str(newSpeed))
        if (button.value == False):
            while (button.value == False):
                lcd.clear()
                motor.duty_cycle = 0
            break

while True:
    try:
        height = 57 - dist.distance
        speed = int(pid(height))
        motor.duty_cycle = speed
        print("speed")
        print(speed)
        print("height")
        print(height)
        print(" ")
    except RuntimeError:
        print("retry")
    time.sleep(.1)
    if (encoder.position % 10 < 5 and option != 0):
        option = 0
        lcd.clear()
        lcd.set_cursor_pos(0,0)
        lcd.print("Press to change")
        lcd.set_cursor_pos(1,0)
        lcd.print("height")
    elif (encoder.position % 10 >= 5 and option != 1):
        option = 1
        lcd.clear()
        lcd.set_cursor_pos(0,0)
        lcd.print("Press to change")
        lcd.set_cursor_pos(1,0)
        lcd.print("motor speed")
    if (button.value == False):
        while (button.value == False):
            motor.duty_cycle = 0
        lcd.clear()
        if (option == 0):
            setpoint()
        if (option == 1):
            manual()
        time.sleep(1)
```

### Wiring Diagram

![PID](https://github.com/honklin/PID-Project/assets/121810694/ebdb1ad4-41ef-4e67-868b-61fa972533c0)
Wiring diagram for PID project

### Reflection
One of the most difficult parts of this code was controling the user input

PID has three components, proportional, integral and derivative. The PID function is the brain of the operation. Kp serves as the variable that lets the arduino know how much power it needs to supply to the motor to reach the setpoint. After we have the original jump to the stepoint, Ki takes care of repeating errors and allows the fan to be more accurate and correct long term errors. Lastly the Kd monitors how fast the fan is engaging to reduce occilation. If the Kd is too low, the fan will occilate a lot. Another aspect of the PID challenge included adding limits to the speed in which how fast or slow the ball sunk. 

Another difficulty we faced in this project was only being able to tune the code once the project was fabricated. This reduced the time we had to make update to both the code and the design of the project. The motor also doesn't automatically start spinnig at the set duty cycle. It takes a little bit for it to set so it makes the PID hard becuase it overcorrects.

### Tuning

## CAD

### Description
Link to the [Onshape Document](https://cvilleschools.onshape.com/documents/cae23d9f292dc30b34fb14cb/w/1f6cba44752eedd2b7e14d8a/e/930f21f8e03fe640d7c35464)

### Evidence
![Assemby CAD Image 1](https://github.com/honklin/PID-Project/blob/main/images/Assembly%201%20(1).png)

![Assemby CAD front view](https://github.com/honklin/PID-Project/blob/main/images/Assembly%201%20(2).png)

![Assemby CAD angle view](https://github.com/honklin/PID-Project/blob/main/images/Assembly%201.png)

![part studio box](https://github.com/honklin/PID-Project/blob/main/images/Part%20Studio%201.png)

![Part Studio 1](https://github.com/honklin/PID-Project/assets/112962114/ebe8a898-2f9b-4f17-b540-33bbbbd98565)
This is the 

### Reflection
In the begining of the project, CAD design was started before we fully understood what method was going to be used to propell the ball intothe air. I started designing a funnel for a NMB fan. Mr Deirlof kindly desgined a squirrel fan for all of the people with this project idea. After we adapted the fan to our model, we were able to start designing the box around the fan and all. The overall design for creating this in onshape was fairly straightforward and easy. Probably the most important aspect of the design process in onshape would be effectivly using the edit in context tool and basing all the finished project off of the existing parts we had. 

There was quite a bit of T-slot neglegence. I definetley realized what not do when using T-Slots and Laser Joints. The T-Slots/laser joints arent actually hindering any part of the design, they are just really asymetrical and ugly. The only sight problem this created was the asymitericalness caused the top of the box to only fit one way. We reaized this after it was assembled and had to do a bit of reassembly so that al the parts would fit.

Along the way of designing the CAD, I created a seperate context and started using that one to use all of the update-in-contexts. I am not entirey sure of what the effect of doing this was, but it created some confusing senarios I had to navigate my way around. In the future my goal is to attempt to use the original context for as much of the design before I add another context. 
One thing that I learned was the usefulness of using limits on planar mates. For different parts that need to be touching a wall, but also don't need an incredibly presise location, it is very useful to drag the part to where you want it and then use the limits to confine that piece to a rough location. 
## Assembly

### Description
Our assembly consisted of a laser cut box that housed the fan LCD screen, rotary encoder, switch, arduino and battery pack. The tube we used was actually an old face shield that was used during covid. There is a space cut in the back of the box that allows airflow to the fan 
### Reflection
One thing about the finished project that could be better would be the support for the tube. The tube itself is very tall and with height, it brings a few structural issues. None of these issues are hugely detrimental to the overall soundness of the build, but it does create a little wobble when moving the box and adjusting the various parts of the box.

Another thing that was a bit of an oversight was the wire collection on the side of the tube. Due to the fact that we are this late in the game, the only reasonable solutionto this issue would be using rubberbands or another form of security of wires. Again, this by itself does not show much of an issue, but it reduces proffesionality. 

One win that came of this project was the fact that we only needed to cut one round of acrylic. This was due to the fact that I had made sure there were no visible errors in the CAD before printing anything. In past projects I had not done this to the extent that I did in this project. Even though the engineering teachers stress this fact a lot, this was the first project I had worked one where the effect of this was extremely visable in our progress. During all stages of this project, Hazel and I were consistently ahead of schedule. 

The only thing that put a small slow in the process of fabrication was when the top of the box had to br removes due to the error that came of the CAD flaw. The T-Slots created a situation in which the top sheet was not symmetrical. This created a situation where we had to completely remove the top of the box and take off the LCD, Rotary Encoder and switch which cost us a class period of figuring out the right orientation of the box sides.
