# Seek Quence - Hardware


## Write up
In this challenge a pdf of a schematic was given, this schematic included a few components:
- AND gates
- OR gates
- Inverter gates (NOT)
- D-Flip flops
- 8 bit shift register
- 8 bit comparator
- Clock


After analyzing the schematic it could be seen that in the comparators some bytes are pulled high while some pulled low:
<img src=https://collab.kauotic.se/pad/uploads/118bf8e2-a706-4471-bf0a-805f9d6644f5.png width="200px"/>
<img src=https://collab.kauotic.se/pad/uploads/5602ad19-c1c9-4fa1-b6af-31dfcfcd369a.png width="200px"/>

This corresponds to the two bytes:
1. $00000101$
2. $00010101$

It can then be seen that these two are inverted at the output and connected to a OR gate.
At the output of this OR gate there is a label which says that to open the lock the OR gate should be off.
To accomplish this both the comparators must be inputed with the same bytes as the ones above.

The last bit of the right most shift register is also connected to the input of the second one, which basically extends the register to 16 bits so the bytes we want is actually $0001010100000101$ (connecting the two register into one string of bits).

To be able to send a $1$ to the shift register this AND gate shown in this image must be outputting $1$:
<img src=https://collab.kauotic.se/pad/uploads/908760bd-16fc-447d-b810-18313f959769.png width="600px"/>

To accomplish this, both the D-flip flops must output $1$ and the input needs to be $0$. To find out which sequence would accomplish this the first part of the schematic was simulated in Logisim-evolution:
<img src=https://collab.kauotic.se/pad/uploads/357a9acf-f71a-409c-b909-b4a6e09e53f3.png width="600px"/>

The sequence was found to activate the AND gate at the end was found to be $1010$ counting one bit per cycle as the D-flipflops activated on a rising-edge.

To then finally try to put it all together a python simulation of the program was made to be able to guarantee that the sequence is correct the code can be seen at the bottom.

When simulating it was found that adding a $10$ to the sequence would make the first shift register output $00000101$ which also makes it possible to add it again to create the sequence needed for the second shift register (which is $00010101$), now some padding is needed to move the bits over. A way was found to do this aswell as a way to also put the correct sequence in the first shift register at the same time as the second one.

The python simulator gave this output after fiddeling with the input a bit:
<img src=https://collab.kauotic.se/pad/uploads/79690e11-2810-4a1f-bd68-6197ae6f7f30.png width="100%"/>

The image contains the states of the output of all the gates, which at the last one the one called "o2" is $0$, which in this case is the one which opens the lock.

The input which caused this was <span style="color:red">$10101010$</span><span style="color:green">$1110$</span><span style="color:blue">$1010$</span>. Which can be broken down as:
1. $10101010$ putting $00010101$ in the first shift register
2. $1110$ adds the necessary padding.
3. $1010$ adds $00000101$ in the first shift register and causes the correct amount of shifting to move the earlier bits into the right spot in the second shift register which now has the output $00010101$

This makes the outputs of both comparators $0$ which causes the OR gate at the end to be $0$, causing the lock to unlock.

This sequence was then sent in and was confirmed to be the correct one.

**Flag taken**

---

## Code

```python
class Inverter:
    def __init__(self):
        self.output = 1
    def update(self,input):
        self.output = 1 if input == 0 else 0

class And:
    def __init__(self):
        self.output = 0
    def update(self,input):
        self.output = 1 if input[0] and input[1] else 0

class Or:
    def __init__(self):
        self.output = 0
    def update(self,input):
        self.output = 1 if input[0] or input[1] else 0

class DFlipFlop:
    def __init__(self):
        self.output = 0
        self.last_clock = 0
    def update(self,input):
        if self.last_clock == 0 and input[1] == 1:
            self.output = input[0]
        self.last_clock = input[1]
        
class ShiftRegister8Bit:
    def __init__(self):
        self.output = [0]*8
        self.last_clock = 0
    def update(self,input):
        if self.last_clock == 0 and input[1] == 1:
            self.output = [input[0]] + self.output[:-1]
        self.last_clock = input[1]

class Comparator8Bit:
    def __init__(self):
        self.output = 0
    def update(self,input):
        self.output = 0 if input[0] == input[1] else 1
              
def test(inputs):
    i1 = Inverter()
    i2 = Inverter()
    i3 = Inverter()
    a1 = And()
    a2 = And()
    a3 = And()
    a4 = And()
    a5 = And()
    o1 = Or()
    o2 = Or()
    d1 = DFlipFlop()
    d2 = DFlipFlop()
    s1 = ShiftRegister8Bit()
    s2 = ShiftRegister8Bit()
    c1 = Comparator8Bit()
    c2 = Comparator8Bit()
    clk = 0
    def update(input,clk):
        s2.update([s1.output[7],clk])
        s1.update([a5.output,clk])
        c2.update([[1,0,1,0,1,0,0,0],s2.output])
        c1.update([[1,0,1,0,0,0,0,0],s1.output])
        o2.update([c1.output,c2.output])
        
        i1.update(input)
        a1.update([i1.output,d2.output])
        a3.update([d1.output,i2.output])
        a2.update([input,a3.output])
        i2.update(d2.output)
        o1.update([a1.output,a2.output])
        d1.update([o1.output,clk])
        d2.update([input,clk])
        #Re run the first part of the circuit
        a1.update([i1.output,d2.output])
        a3.update([d1.output,i2.output])
        a2.update([input,a3.output])
        o1.update([a1.output,a2.output])
        
        #Update the rest of the circuit
        i3.update(input)
        a4.update([d1.output,d2.output])
        a5.update([i3.output,a4.output])
    states = []
    
    for input in inputs:
        #Run cycle
        update(input,clk)
        clk = 1 if clk == 0 else 0
        update(input,clk)
        state = {"input":input,"i1":i1.output,"a1":a1.output,"a2":a2.output,"a3":a3.output,"i2":i2.output,"o1":o1.output,"d1":d1.output,"d2":d2.output,"i3":i3.output,"a4":a4.output,"a5":a5.output,"s1":s1.output,"s2":s2.output,"c1":c1.output,"c2":c2.output,"o2":o2.output}
        clk = 1 if clk == 0 else 0
        states.append(state)
        
    return states

input_bits = [1,0,1,0,1,0,1,0,1,1,1,0,1,0,1,0]
res = test(input_bits)

for state in res:
    print(state)

print("".join([str(x["input"]) for x in res]))
```
