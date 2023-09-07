---
title: 'Multithreading with Finder Opta and Finder 7M'
description: "Learn how to run multiple threads on the Finder Opta to simultaneously
              read from a Finder 7M and send the measure to a remote HTTP server."
author: 'Fabrizio Trovato'
libraries:
  - name: 'ArduinoHttpClient'
    url: https://www.arduino.cc/reference/en/libraries/arduinohttpclient/
  - name: 'Finder 7M for Finder Opta library'
    url: https://github.com/dndg/Finder7M
  - name: 'ArduinoRS485'
    url: https://www.arduino.cc/reference/en/libraries/arduinors485
  - name: 'ArduinoModbus'
    url: https://www.arduino.cc/reference/en/libraries/arduinomodbus
difficulty: intermediate
tags:
  - Multithreading
  - ModbusRTU
  - Finder 7M
software:
  - ide-v1
  - ide-v2
  - arduino-cli
  - web-editor
hardware:
  - hardware/07.opta/opta-family/opta
---

## Overview

In previous examples we discussed [how to use the Finder Opta to read from a
Finder 7M energy
meter](https://github.com/dndg/FinderOpta7MTutorial/blob/main/content.md) using
the Modbus protocol. In this tutorial, we are going to learn how to implement
multithreading on the Finder Opta to simultaneously read the Finder 7M and send
the obtained values to a remote HTTP server via Ethernet. In particular, one
thread will continuously read a MID certified counter from the Finder 7M, while
the `loop()` will transmit the read value every five seconds.

## Goals

* Learn how to implement multithreading on the Finder Opta.
* Learn how to simultaneously read from a Finder 7M energy meter and
  periodically send the measure to a remote HTTP server within the same sketch.

## Required Hardware and Software

### Hardware Requirements

* Finder Opta PLC with RS-485 support (x1).
* Finder 7M energy meter (x1).
* 12VDC/1A DIN rail power supply (x1).
* USB-C® cable (x1).
* ETH RJ45 cable (x1).
* Wire with either specification for RS-485 connection (x3):
  * STP/UTP 24-18AWG (Unterminated) 100-130Ω rated.
  * STP/UTP 22-16AWG (Terminated) 100-130Ω rated.

### Software Requirements

* [Arduino IDE 1.8.10+](https://www.arduino.cc/en/software), [Arduino IDE
2.0+](https://www.arduino.cc/en/software) or [Arduino Web
Editor](https://create.arduino.cc/editor).
* If you choose an offline Arduino IDE, you must install the
`ArduinoHttpClient`, `ArduinoRS485` and `ArduinoModbus` libraries. You can
install them using the Library Manager of the Arduino IDE. Additionally, you
will need to install the `Finder 7M for Finder Opta library`: you can clone it
[from GitHub](https://github.com/dndg/Finder7M) directly into the directory
containing all the rest of your libraries.
* [Example code](assets/OptaTasksExample.zip).
* An HTTP server to print the data received from the Finder Opta: for the sake
  of simplicity we recommend something like
  [http-echo-server](https://github.com/watson/http-echo-server) which can be
  easily setup on a computer with a couple commands.

## Finder Opta and Multithreading

Being an Mbed OS based board, the Finder Opta can leverage the [`Thread`
class](https://os.mbed.com/docs/mbed-os/v6.16/apis/thread.html) to create and
control parallel tasks. This allows us to write additional loops that can run
concurrently next to the `loop()` function, which is instead the main thread.

Additionally, the Finder Opta can leverage the `millis()` function to access
its stopwatch, meaning the number of milliseconds that passed since the Arduino
board began running the current sketch: by means of this stopwatch we can
orchestrate the execution of the different threads in our program.

## Instructions

### Setting Up the Arduino IDE

This tutorial will need [the latest version of the Arduino
IDE](https://www.arduino.cc/en/software). If it is your first time setting up
the Finder Opta, check out the [getting started
tutorial](/tutorials/opta/getting-started).

Make sure you install the latest version of the
[ArduinoHttpClient](https://www.arduino.cc/reference/en/libraries/arduinohttpclient/),
[ArduinoModbus](https://www.arduino.cc/reference/en/libraries/arduinomodbus/)
and the
[ArduinoRS485](https://www.arduino.cc/reference/en/libraries/arduinors485/)
_libraries_, as they will be used to implement the Modbus RTU communication
protocol.

Finally, to install the `Finder 7M for Finder Opta library` you can clone it
[from GitHub](https://github.com/dndg/Finder7M) and then move it to the
libraries folder inside your sketchbook. For further details on how to manually
install libraries refer to [this
article](https://support.arduino.cc/hc/en-us/articles/5145457742236-Add-libraries-to-Arduino-IDE).

### Connecting the Finder Opta and Finder 7M

Just like [in the previous
tutorial](https://github.com/dndg/FinderOpta7MTutorial/blob/main/content.md#connecting-the-finder-opta-and-finder-7m)
we will need to power the Finder Opta with the 12VDC/1A supply and connect it
to the 7M via RS-485 serial connection. Additionally, for this tutorial we will
connect the Finder Opta to the computer running the HTTP server using the
Ethernet RJ45 cable. To complete these steps refer to the following diagram:

![Connecting Opta and Finder 7M](assets/connection.svg)

For the example code to work, you will need to set the communication parameters
for the 7M to:

* Modbus address `2`.
* Baudrate `38400`.
* Serial configuration `8-N-1`.

This can easily be done with your smartphone using [the Finder Toolbox
application](https://www.findernet.com/en/worldwide/support/software-and-apps/)
via NFC.

### Code Overview

The goal of the following example is to implement multithreading on the Finder
Opta to simultaneously read the Finder 7M and send the obtained values to a
remote HTTP server via Ethernet.

The full code of the example is available
[here](assets/OptaMultithreadingExample.zip): after extracting the files the
sketch can be compiled and uploaded to the Finder Opta.

#### Setting up the program

In the `setup()` we are going to:

* Initialize the Modbus connection towards the Finder 7M energy meter using the
  built-in function from the `Finder7M` library.
* Assign a static IP address to the Finder Opta so that it belongs to the same
  IP network as your HTTP server (eg. it is on the same LAN as your computer
  which hosts the `http-echo-server`). It might be necessary to change the code
  to set a different IP address, depending on your network configuration.
* Configure the HTTP server parameters, most notably IP address and port. It
  might be necessary to edit both values, depending on your setup.
* Start a separate thread which will run alongside the main `loop()`, executing
  a function in loop.

```cpp
#include <Finder7M.h>
#include <ArduinoRS485.h>
#include <ArduinoModbus.h>

#include "mbed.h"

#include "opta_info.h"
#include <SPI.h>
#include <Ethernet.h>
#include <ArduinoHttpClient.h>

const uint8_t modbusAddress = 2; // Modbus address of the Finder 7M device.
Finder7M f7m;
volatile float importActiveEnergy;

long int chrono;
const int readDelay = 300;  // Read measures every 0.3s.
const int sendDelay = 5000; // Send measures every 5s.

static rtos::Thread MIDsThread;

OptaBoardInfo *info;
OptaBoardInfo *boardInfo();

IPAddress ip(192, 168, 10, 25); // Static IP address assigned to the Opta.
EthernetClient ethernetClient;
HttpClient httpClient = HttpClient(ethernetClient, "192.168.10.1", 64738); // IP address and port of the netcat server.

void setup()
{
    Serial.begin(38400);

    delay(2000);

    if (!f7m.init())
    {
        Serial.println("Error! Could not init Finder 7M connection via Modbus.");
        while (1)
        {
        }
    }

    if (!setIPAddress())
    {
        Serial.println("Error! Could not assign static IP address to the Opta.");
        while (1)
        {
        }
    }

    // Start a separate thread to send the measures from MID registers
    // via HTTP while the main loop reads from the Finder 7M.
    MIDsThread.start(readMIDs);

    chrono = millis();
}
```

Note how this code declares a volatile variable that the second thread will
update on behalf of the main `loop()`, which will instead transmit its value as
we will see later. Finally, at the end of the `setup()` we also read the
stopwatch using the `millis()` function and we store its current value in a
variable.

#### Reading from the Finder 7M in a thread

Below we find the code of the function that will be executed by the second
thread, that we created during the `setup()`:

```cpp
void readMIDs()
{
    long int t;

    while (1)
    {
        t = millis();

        Measure inActive = f7m.getMIDInActiveEnergy(modbusAddress);
        importActiveEnergy = inActive.toFloat();
        Serial.println("Mantissa: " + String(inActive.mantissa()));
        Serial.println("Exponent: " + String(inActive.exponent()));

        t = millis() - t;

        if (t < readDelay)
        {
            rtos::ThisThread::sleep_for(readDelay - t);
        }
    }
}
```

This function reads the value of the import active energy MID certified counter
using the `Finder7M` library, it updates the volatile variable and it prints to
the serial monitor the mantissa and exponent of the measure. The thread then
either repeats the same operation, or if the time passed since the previous
read is lower than 0.3s, it sleeps: notice how this implementation relies on
the stopwatch to control the execution flow of the code, without relying on
delays.

#### Sending measures to the HTTP server

The `loop()` function contains the following code:

```cpp
void loop()
{
    if (millis() - chrono > sendDelay)
    {
        sendMIDs();
        chrono = millis();
    }
    delay(500); // Check every 0.5s if we need to send.
}
```

This loop is executed every 0.5s and it uses the stopwatch to check if 5s
passed since the last update to the `chrono` variable, which is initialized at
the end of the `setup()` function and then updated whenever a measure is sent
to the HTTP server. Notice how again to control the flow of the execution we
rely on the stopwatch, as the delay is only introduced for convenience to avoid
performing the time check over and over for 5s.

The code that sends data to the HTTP server is the following:

```cpp
String contentType = "application/x-www-form-urlencoded";
String postData = "importActiveEnergy=" + String(importActiveEnergy) +
                    "+time=" + String(millis());

httpClient.post("/", contentType, postData);

String response = httpClient.responseBody();

Serial.println(response);
```

The HTTP server will receive a POST request of type `x-www-form-urlencoded`
that contains the float value of the import active energy and the value of the
stopwatch when the request was sent.

## Conclusion

This tutorial illustrates how to implement two threads on the Finder Opta to
have them perform separate tasks using the stopwatch of the board.
Additionally, the tutorial shows how it is possible to use the `Finder7M`
library to easily read MID certified counters from a Finder 7M, and send the
measure to an HTTP server via Ethernet.
