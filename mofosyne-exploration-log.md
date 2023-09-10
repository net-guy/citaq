# Mofosyne Exploration Log

---

# CITAQ H10-3 SDK

## Where is the print logic in POSFactory App?

```
~/git/Citaq-H10-3/CitaqSDK/src/com/citaq/citaqfactory/PrintActivity.java
```

### How does the print demo text work? (`PrintActivity.java`)

This button to print demo text will call either `Command.getPrintDemoZH()` and `Command.getPrintDemo()` from `./CitaqSDK/src/com/citaq/util/Command.java` which would output a byte array of ESC/POS formatted text.

This is sent to `printerWrite()` in `PrintActivity.java` which calls `mSendThread.addData()` from `public class SendThread extends Thread` which would call either `serialWrite()` or `usbWrite()` to send the ESC/POS data to the internal printer.

**Serial**

```java
public class PrintActivity extends SerialPortActivity{
	private void initSerial(){
			mSerialPort = mApplication.getPrintSerialPort();
			mOutputStream = mSerialPort.getOutputStream();
    ... } ... } ...
```

Where `getPrintSerialPort()` from `./CitaqSDK/src/com/citaq/citaqfactory/CitaqApplication.java` uses `SerialPort()` from `./CitaqSDK/src/android_serialport_api/SerialPort.java`

```java
	public SerialPort getPrintSerialPort() throws SecurityException, IOException, InvalidParameterException {
		if (mSerialPort == null) {
			/* Open the serial port */
			mSerialPort = new SerialPort(new File("/dev/ttyS1"), 115200, 0, true);
		}
		return mSerialPort;
	}
```

Where

```java
	public SerialPort(File device, int baudrate, int flags, boolean flowCon)
```

But the main point to learn from the above function is that the internal serial port we should be using to talk to the internal printer is:
 - CTE-RP80 Internal Printer Serial Port
    - path: `/dev/ttyS1`
    - baud: 115200
    - flag: none
    - flow control: enabled


## How does led bar works

The test page for this led bar main business logic page is at `./CitaqSDK/src/com/citaq/citaqfactory/LedActivity.java`

Which essentally calls functions like `trunOnRedRight()` which is from

```
./CitaqSDK/src/com/citaq/util/LEDControl.java
```

The virtual gpio file that this function writes is dependent on cpu type (different board revision?)

This is determined via `getCpuHardware()` which calls a linux command `cat /proc/cpuinfo` that dumps the board hardware info and scans for lines starting with `Hardware:` and grab the hardware ID after it.

| getCpuHardware() Recognised Board Type | hardware ID string (/proc/cpuinfo) |
|----------------------------------------|------------------------------------|
| SMDKV210                               | `SMDKV210`                         |
| RK3188                                 | `RK30BOARD`                        |
| RK30BOARD                              | `SUN50IW1P1`                       |
| MSM8625Q                               | `QRD MSM8625Q SKUD`                |
| RK3368                                 | `RK3368`                           |

Fortunately as shown in trunOnRedRight()... in both CPU/board models, you just need to write `1` to turn led on or `0` to turn led off as shown in `trunOnRedRight()`. As for what files should be written to based on board type, there is some clues based on what `trunOnRedRight()` and `trunOnBlueRight()`

 - MainBoardUtil.RK3188 or MainBoardUtil.RK30BOARD
    - RED: `/sys/class/gpio/gpio190/value`
    - BLUE: `/sys/class/gpio/gpio172/value`
 - MainBoardUtil.RK3368
    - RED: `/sys/class/gpio/gpio124/value`
    - BLUE: `/sys/class/gpio/gpio106/value`

There is also `MainBoardUtil.MSM8625Q` referenced in `trunOnRedRight()` to control the led bar, but instead of control via linux gpio virual files interface... it is via calling `isRedlightOn()` function contained within a JNI (Java Native Interface) libarary from `/CitaqSDK/libs/x86/libposctrl_jni.so`. This fits my experience when working with Qualcomm SoC in that they don't typically give you datasheets for direct access to the gpio memory map as those tend to be behind NDAs for some reason...

As for other boards/cpu mentioned above, there is no led reference to it... so its likely an older model that may not have an led bar.

But anyway for the context of this repository this does not matter. We got what we needed for system speccing now.


## Whats with the other serial virtual file ports???

In `./CitaqSDK/src/com/citaq/citaqfactory/CitaqApplication.java` we see

| serial port getter name   | Serial Port Devices |   Baud | Flow Control |
|---------------------------|---------------------|--------|--------------|
| getPrintSerialPort()      |        '/dev/ttyS1` | 115200 |         true |
| getPrintSerialPortMT()    |       '/dev/ttyMT0` | 115200 |         true |
| getMSRSerialPort()        |        '/dev/ttyS2` |  19200 |        false |
| getCtmDisplaySerialPort() |        '/dev/ttyS3` |   9600 |        false |
| getMSRSerialPort_S4()     |        '/dev/ttyS4` |  19200 |        false |

So let's at least annotate this for documentation

(Note: My best guess is that MSR means Magnetic Stripe Reader in the context of Point Of Sales equiment)

| CitaqApplication.java func() | Serial Port Devices |   Baud | Flow Control | Note On Usage            |
|------------------------------|---------------------|--------|--------------|--------------------------|
| getPrintSerialPort()         |        '/dev/ttyS1` | 115200 |         true | Internal Thermal Printer |
| getPrintSerialPortMT()       |       '/dev/ttyMT0` | 115200 |         true | Unknown Usage |
| getMSRSerialPort()           |        '/dev/ttyS2` |  19200 |        false | Magnetic Stripe Reader via serial port |
| getCtmDisplaySerialPort()    |        '/dev/ttyS3` |   9600 |        false | Customer Display facing serial port. Used by FSK Caller ID and PD (ESC/POS Printer Device) example in POSFactory app |
| getMSRSerialPort_S4()        |        '/dev/ttyS4` |  19200 |        false | Magnetic Stripe Reader via serial port |

I'm not sure which one would control the serial port that is externally accessible from the CITAQ H10-3