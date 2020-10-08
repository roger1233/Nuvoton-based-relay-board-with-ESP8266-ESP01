# Nuvoton-based-relay-board-with-ESP8266-ESP01
Control relays on a nuvoton-based ESP-8266 ESP-01

Got two boards with 2 relays from Aliexpress (this one: https://www.aliexpress.com/item/32950213696.html ), before these I got others with only on relay, but based on a STC on-board serial controler.

The STC kind works well, you need to talk to it through the serial interface (TX pin) like so:

```
Serial.write(0xa0); // relay command
Serial.write(0x01); // first relay
Serial.write(0x00); // shut it off (0)
Serial.write(0xa1); // checksum
```

But, unfortunately, for my Nuvoton-based board, it did not work, my commands were ignored.

Then, I stepped on this article: https://tasmota.github.io/docs/devices/LC-Technology-WiFi-Relay/

In this one, they have this setup:
```
on System#Boot do Backlog Baudrate 115200
on SerialReceived#Data=41542B5253540D0A do SerialSend5 5749464920434f4e4e45435445440a5749464920474f542049500a41542b4349504d55583d310a41542b4349505345525645523d312c383038300a41542b43495053544f3d333630 endon
on Power1#State=1 do SerialSend5 A00101A2 endon
```
So, first thing, this one seems to listen at 115200 baud, where my other STC ones were 9600 baud. That's easy enough.

Translating hex to ASCII, we get (\n are newline character 0x0a):
    WIFI CONNECTED\nWIFI GOT IP\nAT+CIPMUX=1\nAT+CIPSERVER=1,8080\nAT+CIPSTO=360\n

And, they seem to wait for 41542B5253540D0A (AT+RST\c\n) before sending that string.

In my normal setups, I like to keep the RX pin for other uses since it's status does not influence boot (unlike GPIO 0 and 2). So, if I open the serial port like that:

    Serial.begin(115200,SERIAL_8N1,SERIAL_TX_ONLY); // This frees up GPIO3 (RX)
This will use only TX, but I won't be able to listen to RX and wait for AT+RST before sending my string.

So I decided to add a delay to let the nuvoton board initialize, and then send "blindly" the string, and it works!

So, here is the final code:

```
char nuvotonInit[] = "WIFI CONNECTED\nWIFI GOT IP\nAT+CIPMUX=1\nAT+CIPSERVER=1,8080\nAT+CIPSTO=360\n";

void setup()
{
  
  Serial.begin(115200,SERIAL_8N1,SERIAL_TX_ONLY); // This frees up GPIO3 (RX)

  
  // Do my normal init sequence, wifimanager, etc
  WiFiManager wifiManager;
  //wifiManager.resetSettings();
  wifiManager.setConfigPortalTimeout(300);
  wifiManager.autoConnect("nuvoton",""); // AP name in case of no config
  
  client.setServer("server.local",1883);
  client.setCallback(callback);

  // Initialize the nuvoton chip by waiting 5 seconds then blindingly sending it's init string
  delay(5000);
  Serial.write(nuvotonInit,sizeof(nuvotonInit));
  
}
```
This will properly initialize the board, BUT, just after that, if you send a relay ON or OFF string, it will work, but with a 10 second (approx) delay the first time. Then, it will work correctly all the time. Led D6 on the board needs to start blinking before the commands are working.

This means that after boot, you have a 5 second wait time before sending the string, and then a approx 10 seconds where even if you send commands, they seem to be cached and executed after 10 seconds. After that initial 10 second wait, all commands are executed immediately.

Lastly, here is my little function to send the relay commands instead of sending the hex code, I calculate it like so:

```
void relay(byte relay, byte state)
{
  byte msg = 0xa0;
  Serial.write(msg);
  Serial.write(relay);
  Serial.write(state);
  Serial.write(msg+relay+state);
}
```

So to set ON the second relay, I'd call: relay(2,1); and it would send the proper serial command to do so.

I searched the web a lot to fix this one, so I posted it all here in the hopes it will save somebody else the research. Tell me if it helped you!
