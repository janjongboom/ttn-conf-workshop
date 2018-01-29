# TTN Conference - Firmware updates workshop

## Prerequisites

We need to install a few pieces of software that we'll be using.

On your computer:

1. Install a recent version of [node.js](https://nodejs.org) (6.x or higher).
1. Install Python 2.7.
1. Download the software pack for this workshop from [here]() - and unpack in a convenient location.

**Windows**

If you are on Windows, also install:

1. [Mbed Windows serial driver](http://os.mbed.com/media/downloads/drivers/mbedWinSerial_16466.exe) - serial driver for the board.
1. [Tera term](https://osdn.net/projects/ttssh2/downloads/66361/teraterm-4.92.exe/) - to see debug messages from the board.
1. [OpenSSL](https://slproweb.com/download/Win32OpenSSL-1_0_2n.exe) - to sign firmware updates.

**Linux**

If you're on Linux, also install:

1. OpenSSL - e.g. via `sudo apt install openssl`

## Building the circuit

We're using the [L-TEK FF1705](https://os.mbed.com/platforms/L-TEK-FF1705/) development board, which contains the Multi-Tech xDot module. Let's connect some sensors and verify that the board works.

Grab:

* Temperature sensor.
* Moisture sensor.
* Six jumper wires.

The Grove sensors have four wires. Yellow = data line, white = another data line, red = power, black = ground. In this workshop we'll only use yellow, red and black. We'll use the jumper wires to connect the sensor to the board (because we don't have Grove base shields).

1. Take the jumper wires and connect them to the holes matching yellow, black and red.
1. Plug the jumper wires into the dev board as follows:
    * Temperature sensor yellow -> GPIO2
    * Temperature sensor red -> 3V3
    * Temperature sensor black -> GND
    * Moisture sensor yellow -> SPI_MOSI
    * Moisture sensor red -> 3V3
    * Moisture sensor black -> GND
1. Now connect the board to your computer using a micro-USB cable.

## 1. A simple application

Now let's build a simple application which reads the sensor data and prints it to the serial console.

1. Go to [https://os.mbed.com](https://os.mbed.com) and sign up (or sign in).
1. Go to the [L-TEK FF1705](https://os.mbed.com/platforms/L-TEK-FF1705/) platform page and click *Add to your Mbed compiler*.
1. Open the [Online Compiler](https://os.mbed.com/compiler/).
1. Click *Import > Click here to import from URL*.
1. Enter `https://github.com/armmbed/mbed-os-example-blinky` and click *Import*.
1. In the top right corner make sure you selected 'L-TEK FF1705'.
1. Now open `main.cpp`, and replace the content with:

    ```cpp
    #include "mbed.h"

    DigitalOut led1(LED1);

    // For an overview of all the pins, see https://os.mbed.com/platforms/L-TEK-FF1705/
    AnalogIn moisture(PB_15);
    AnalogIn thermistor(GPIO2);

    float read_temperature() {
        unsigned int a, beta = 3975;
        float temperature, resistance;

        a = thermistor.read_u16(); /* Read analog value */

        /* Calculate the resistance of the thermistor from analog votage read. */
        resistance= (float) 10000.0 * ((65536.0 / a) - 1.0);

        /* Convert the resistance to temperature using Steinhart's Hart equation */
        temperature=(1/((log(resistance/10000.0)/beta) + (1.0/298.15)))-273.15;

        return temperature;
    }

    int main() {
        while (true) {
            printf("Moisture is %.3f, temperature is %.2f\n", moisture.read(), read_temperature());

            // Over 5% moisture, turn LED on
            led1 = moisture.read() > 0.05f;

            wait(0.5);
        }
    }
    ```

1. Click *Compile*.
1. A file downloads, use drag-and-drop to drag the file to the DAPLINK device (like a USB mass storage device).
1. When flashing is complete, hit the **RESET** button on the board (next to USB).

Now when you touch the moisture sensor with your fingers, you should see the blue LED light up. Let's look at the logs now.

## 2. Showing logs

If all is well, you should see something similar to:

```
Moisture is 0.010, temperature is 24.90
Moisture is 0.011, temperature is 24.86
Moisture is 0.097, temperature is 24.86
Moisture is 0.010, temperature is 24.88
```

#### Windows

To see debug messages, install:

1. [Mbed Windows serial driver](http://os.mbed.com/media/downloads/drivers/mbedWinSerial_16466.exe) - serial driver for the board.
    * See above for more instructions.
1. [Tera term](https://osdn.net/projects/ttssh2/downloads/66361/teraterm-4.92.exe/) - to see debug messages from the board.

When you open Tera Term, select *Serial*, and then select the Mbed COM port.

#### OS/X

No need to install a driver. Open a terminal and run:

```
screen /dev/tty.usbm            # now press TAB to autocomplete and then ENTER
```

To exit, press: `CTRL+A` then `CTRL+\` then press `y`.

#### Linux

If it's not installed, install GNU screen (`sudo apt-get install screen`). Then open a terminal and find out the handler for your device:

```
$ ls /dev/ttyACM*
/dev/ttyACM0
```

Then connect to the board using screen:

```
sudo screen /dev/ttyACM0 9600                # might not need sudo if set up lsusb rules properly
```

To exit, press `CTRL+A` then type `:quit`.

## 3. Connecting to The Things Network

We have set up a special instance of TTN at https://console.fotademo.thethings.network/, so do not use the normal console.

You can log in with:

* Username: `arm-ttn-conference`
* Password: `omgfota1`

...

### Cloning the repository

We have set up a repository which contains all the bits needed for a firmware update. It's already configured with the right credentials to talk to the special instance of TTN. Let's build this application in the online compiler.

1. In the Online Compiler, click *Import > Click here to import from URL*.
1. Enter `https://github.com/armmbed/lorawan-fota-demo` and click *Import*.

Now we need to generate a certificate, that we can use to verify updates.

1. Open a terminal (or command prompt).
1. Navigate to the `package-signer` directory (in the software package for this workshop).
1. Then run:
    * Windows:

        ```
        $ SET PATH=%PATH%;C:\OpenSSL-Win32\bin
        $ node generate-keys.js organization.com your-device-model
        ```

    * Mac / Linux:

        ```
        $ node generate-keys.js organization.com your-device-model
        ```

1. This generates your public / private key (see certs/ directory), and the `UpdateCerts.h` file, which contains the public key and the UUIDs for your device class.
1. Copy the content of `UpdateCerts.h`.
1. Create a new file in the online compiler (named `UpdateCerts.h`) and add the content.

## 4. Testing without breaking TTN

Firmware updates have a very significant effect on the network, so we are not going to run 25 different updates. But we can simulate one.

1. In the Online Compiler, click *Import > Click here to import from URL*.
1. Enter `https://github.com/janjongboom/lorawan-fragmentation-in-flash` and click *Import*.
1. Go into the `src` folder and open `UpdateCerts.h`.
1. On your computer, locate `UpdateCerts.h` and paste the content into the online compiler.
1. Now we can sign a package... We have created a simple blinky application for you.
1. In the terminal, go to the package-signer folder and run:

    ```
    $ node create-packets-h.js xdot-l151cc-blinky_application.bin
    ```

    Note on Windows: run `SET PATH=%PATH%;C:\OpenSSL-Win32\bin` first.

1. This should output something like this:

    ```
    Signed signature is 3045022100bd5f0d2ddd0fe47e39d25cce73a185f84fb4b96ec47feaed2a
    e366a236edaa52022071c9091ee6762975415ae39672a7b53398b0fc4523743f2f092263d487ff6c
    c8
    CRC64 hash is 275eaae88c83a779
    Done, written to packets.h
    ```

1. Go back to the online compiler, go to the `src` folder and remove `packets.h`.
1. Now drag the `packets.h` file from your local computer (in the local `src` folder) and drag it into the online compiler.
1. Now hit *Compile* and flash the binary to your board.

On the serial port you'll see the fragmentation session, then CRC64 validation, and then flashing the new binary.

## 5. Building locally and binary diffs

...

## Install

* A recent version of Node.js
* OpenSSL on windows (e)
* OpenSSL on Linux (`sudo apt install openssl`)

## Create public/private key pair

* Windows:

    ```
    $ SET PATH=%PATH%;C:\OpenSSL-Win32\bin
    ```

    ```
    $ c:\Users\janjon01\Desktop\package-signer>SET PATH=%PATH%;z:\ttn-workshop-resources\jojodiff\win32
    ```