```python
from machine import Pin, unique_id
from umqtt.robust import MQTTClient
import asyncio
from lib.dfplayer import DFPlayer
from myservo import myServo
from binascii import hexlify
import random
import json

df = DFPlayer(1) 
df.init(tx=4, rx=5)

mp3_er_ad_spila = Pin(21, Pin.IN, Pin.PULL_UP) # 0 mp3 er að spila, 1 mp3 er ekki að spila
hljod_stada_1 = Pin(7, Pin.IN, Pin.PULL_UP)
hljod_stada_2 = Pin(8, Pin.IN, Pin.PULL_UP)
hljod_stada_3 = Pin(9, Pin.IN, Pin.PULL_UP)

servo = myServo(10)
servo.myServoWriteDuty(128)
servo.myServoWriteAngle(180)

mp3_a_ad_spila = False
mp3_a_ad_stoppa = False
mp3_spila_hljod_numer = 1
sena_i_pasu = True

WIFI_SSID = 'TskoliVESM'
WIFI_LYKILORD = 'Fallegurhestur'

def do_connect():
    import network
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    if not wlan.isconnected():
        print('connecting to network...')
        wlan.connect(WIFI_SSID, WIFI_LYKILORD)
        while not wlan.isconnected():
            pass
    print('network config:', wlan.ifconfig())
    
do_connect()

async def senda_mqtt_skilabod(mqtt_client_inn, topic, skilabod):
    mqtt_client_inn.publish(topic, skilabod)
    
def fekk_skilabod(topic, skilabod):
    global mp3_a_ad_spila, mp3_a_ad_stoppa, mp3_spila_hljod_numer, sena_i_pasu
    print(f"TOPIC: {topic.decode()}, skilaboð: {skilabod}")
    if skilabod == b"sena1" or skilabod == b"polo":
        if mp3_a_ad_stoppa == False or skilabod == b"sena1":
            mp3_a_ad_spila = True
            if skilabod != b"polo":
                mp3_a_ad_stoppa = False
            mp3_spila_hljod_numer = 1
            sena_i_pasu = False
        
    elif skilabod == b"stoppa":
        mp3_a_ad_stoppa = True
    elif skilabod.decode().isdigit():
        servo.myServoWriteAngle(int(skilabod))
        mqtt_client.publish(b"LidA1_nodered",json.dumps({"gradur":int(skilabod)}))

print("Initializing MQTT")
mqtt_client = MQTTClient(hexlify(unique_id()), "test.mosquitto.org", keepalive=60)
mqtt_client.set_callback(fekk_skilabod)
mqtt_client.connect()
mqtt_client.subscribe(b"LidA1")
print("done!")

async def spila():
    global mp3_a_ad_spila, mp3_a_ad_stoppa, mp3_spila_hljod_numer, sena_i_pasu
    await df.wait_available()
    while True:
        if mp3_er_ad_spila.value() == 1 and not sena_i_pasu:
            if mp3_a_ad_spila == True:
                print("spila senu")
                await df.play(1, mp3_spila_hljod_numer)
                mp3_a_ad_spila = False
            else:
                await df.stop()
                print("sena búin")
                sena_i_pasu = True
                
                if mp3_spila_hljod_numer == 1:
                    random_ms = random.randint(1500, 2500)
                    await asyncio.sleep_ms(random_ms)
                    mqtt_client.publish(b"LidA2", "marco")
                    print("marco")
                    await asyncio.sleep_ms(500)
                    
        elif mp3_er_ad_spila.value() == 0 and mp3_a_ad_stoppa == True:
            await df.stop()
            print("sena stoppuð handvirkt")
                
        hljodstyrkur = 30
        await df.volume(hljodstyrkur)
        await asyncio.sleep_ms(0)
        
async def tala():
    while True:
        #Lesa stöðu á ljósaperunum á hljóðskynjaranum og hreyfa kjaftinn í takt við hávaða
        gradur = 180 - 25 * 3
        if hljod_stada_3.value() == 0:
            #print("staða 3 - fullopinn kjaftur")
            gradur = 180
            servo.myServoWriteAngle(gradur)
        elif hljod_stada_2.value() == 0:
            #print("staða 2 - 66% opinn kjaftur")
            gradur = 180 - 25
            servo.myServoWriteAngle(gradur)
        elif hljod_stada_1.value() == 0:
            #print("staða 1 - 33% opinn kjaftur")
            gradur = 180 - 25 * 2
            servo.myServoWriteAngle(gradur)
        else:
            #print("staða 0 - lokaður kjaftur")
            gradur = 180 - 25 * 3
            servo.myServoWriteAngle(gradur)
            
        mqtt_client.publish(b"LidA1_nodered",json.dumps({"gradur":gradur}))

        await asyncio.sleep_ms(175)
        
async def main():
    asyncio.create_task(spila())
    asyncio.create_task(tala())

    while True:
        mqtt_client.check_msg()
        await asyncio.sleep_ms(100)

asyncio.run(main())
```
