# The Unofficial CHiP Bluetooth Low Energy Protocol
## Table of Contents
[Exciting CHiP Project Links](#exciting-chip-project-links)<br>
[Discovering CHiP Robots](#discovering-chip-robots)<br>
[CHiP's BLE Services & Characteristics](#chips-ble-services--characteristics)<br>
[Sending CHiP Requests](#sending-chip-requests)<br>
[Getting Responses from CHiP](#getting-responses-from-chip)<br>
[CHiP Command Reference](#chip-command-reference)<br>


## Exciting CHiP Project Links
[Official WowWee CHiP iOS SDK](https://github.com/WowWeeLabs/CHIP-iOS-SDK#wowwee-chip-ios-sdk)<br>
[Official WowWee CHiP Android SDK](https://github.com/WowWeeLabs/CHIP-Android-SDK#wowwee-chip-android-sdk)<br>

Have you built your own exciting and cool CHiP based project utilizing this documentation? If so, add a link to it here and send us a pull request so that others can check it out!

[CHiP C API for macOS](https://github.com/adamgreen/CHiP-Capi#readme) by [@adamgreen](https://www.github.com/adamgreen)<br>


## Discovering CHiP Robots
When WowWee toys like the MiP and CHiP advertise their BLE availability, they include two unique services in their advertising data:
```objective-c
// These are the services listed by CHiP in it's broadcast message.
// They aren't the ones we will actually use after connecting to the device though.
#define CHIP_BROADCAST_SERVICE1 "fff0"
#define CHIP_BROADCAST_SERVICE2 "ffb0"
```

The first step in finding connectable CHiP robots is to perform a BLE scan for advertising packets containing these 2 services. For example, the following code can be used in iOS and macOS to initiate the scan for CHiP robots in the area:
```objective-c
[manager scanForPeripheralsWithServices:[NSArray arrayWithObjects:[CBUUID UUIDWithString:@CHIP_BROADCAST_SERVICE1], 
                                                                  [CBUUID UUIDWithString:@CHIP_BROADCAST_SERVICE2], 
                                                                  nil] options:nil];
```

Once BLE devices with these 2 services are discovered, we need to take a peak at the manufacturer specific data in the advertisement packets to see if the first two bytes match the expected data for a CHiP robot {0x00 0x1D}. The following code is an example of performing this check on iOS or macOS:
```objective-c
// CHiP devices will have the following values in the first 2 bytes of their Manufacturer Data.
#define CHIP_MANUFACTURER_DATA_TYPE "\x00\x1d"

// Invoked when the central discovers CHiP robots while scanning.
- (void) centralManager:(CBCentralManager *)central didDiscoverPeripheral:(CBPeripheral *)aPeripheral advertisementData:(NSDictionary *)advertisementData RSSI:(NSNumber *)RSSI
{
    // Check the manufacturing data to make sure that the first two bytes are 0x00 0x1D to indicate that it is a CHiP device.
    NSData* manufacturerDataObject = [advertisementData objectForKey:CBAdvertisementDataManufacturerDataKey];
    uint8_t manufacturerData[2];
    [manufacturerDataObject getBytes:manufacturerData length:sizeof(manufacturerData)];
    if (0 == memcmp(manufacturerData, CHIP_MANUFACTURER_DATA_TYPE, sizeof(manufacturerData)))
    {
        // This is a CHiP robot!
        // If this is the CHiP robot with which we want to attach, then initiate the BLE connection.
        [manager connectPeripheral:peripheral options:nil];
    }
}
```


## CHiP's BLE Services & Characteristics
Once we have found the CHiP robot, we can connect to it and find the two services and characteristics that we will actually use to send requests and receive responses:
* CHIP_RECEIVE_DATA_SERVICE: 0xffe0
  * CHIP_RECEIVE_DATA_NOTIFY_CHARACTERISTIC: 0xffe4
* CHIP_SEND_DATA_SERVICE: 0xffe5
  * CHIP_SEND_DATA_WRITE_CHARACTERISTIC: 0xffe9

**Note:** The services used for actually communicating with the CHiP are different than the 2 services in the advertisement packet.

You should use your platform's BLE API to register for notifications whenever the CHIP_RECEIVE_DATA_NOTIFY_CHARACTERISTIC (0xffe4) changes. This is how the CHiP sends responses back to your application. You can send requests to the CHiP by writing to the CHIP_SEND_DATA_WRITE_CHARACTERISTIC (0xffe9).

The following code snippet shows an example of finding the required services/characteristics on iOS or macOS:
```objective-c
// These are the services used to send/receive data with the CHiP.
#define CHIP_RECEIVE_DATA_SERVICE    "ffe0"
#define CHIP_SEND_DATA_SERVICE       "ffe5"

// Characteristic of CHIP_RECEIVE_DATA_SERVICE which receives data from CHiP.
// The controller can register for notifications on this characteristic.
#define CHIP_RECEIVE_DATA_NOTIFY_CHARACTERISTIC "ffe4"
// Characteristic of CHIP_SEND_DATA_SERVICE to which data is sent to CHiP.
#define CHIP_SEND_DATA_WRITE_CHARACTERISTIC     "ffe9"

// Invoked whenever a connection is succesfully created with a CHiP robot.
// Start discovering available BLE services on the robot.
- (void) centralManager:(CBCentralManager *)central didConnectPeripheral:(CBPeripheral *)aPeripheral
{
    [aPeripheral setDelegate:self];
    [aPeripheral discoverServices:[NSArray arrayWithObjects:[CBUUID UUIDWithString:@CHIP_RECEIVE_DATA_SERVICE],
                                                            [CBUUID UUIDWithString:@CHIP_SEND_DATA_SERVICE], nil]];
}

// Invoked upon completion of a -[discoverServices:] request.
// Discover available characteristics on interested services.
- (void) peripheral:(CBPeripheral *)aPeripheral didDiscoverServices:(NSError *)error
{
    for (CBService *aService in aPeripheral.services)
    {
        /* CHiP specific services */
        if ([aService.UUID isEqual:[CBUUID UUIDWithString:@CHIP_RECEIVE_DATA_SERVICE]] ||
            [aService.UUID isEqual:[CBUUID UUIDWithString:@CHIP_SEND_DATA_SERVICE]])
        {
            [aPeripheral discoverCharacteristics:nil forService:aService];
        }
    }
}

// Invoked upon completion of a -[discoverCharacteristics:forService:] request.
// Perform appropriate operations on interested characteristics.
- (void) peripheral:(CBPeripheral *)aPeripheral didDiscoverCharacteristicsForService:(CBService *)service error:(NSError *)error
{
    /* CHiP Receive Data Service. */
    if ([service.UUID isEqual:[CBUUID UUIDWithString:@CHIP_RECEIVE_DATA_SERVICE]])
    {
        for (CBCharacteristic *aChar in service.characteristics)
        {
            /* Set notification on received data. */
            if ([aChar.UUID isEqual:[CBUUID UUIDWithString:@CHIP_RECEIVE_DATA_NOTIFY_CHARACTERISTIC]])
            {
                [peripheral setNotifyValue:YES forCharacteristic:aChar];
            }
        }
    }

    /* CHiP Send Data Service. */
    if ([service.UUID isEqual:[CBUUID UUIDWithString:@CHIP_SEND_DATA_SERVICE]])
    {
        for (CBCharacteristic *aChar in service.characteristics)
        {
            /* Remember Send Data Characteristic pointer. */
            if ([aChar.UUID isEqual:[CBUUID UUIDWithString:@CHIP_SEND_DATA_WRITE_CHARACTERISTIC]])
            {
                sendDataWriteCharacteristic = aChar;
            }
        }
    }
}
```


## Sending CHiP Requests
Sending requests to the CHiP robot is as easy as writing the request bytes documented in the [CHiP Command Reference](#chip-command-reference) section below to the CHIP_SEND_DATA_WRITE_CHARACTERISTIC that we discovered earlier. For example (again on iOS or macOS):
```objective-c
    // Send request to CHiP robot via Core Bluetooth.
    [peripheral writeValue:cmdData forCharacteristic:sendDataWriteCharacteristic type:CBCharacteristicWriteWithoutResponse];
```


## Getting Responses from CHiP
If you have registered for notifications on CHIP_RECEIVE_DATA_NOTIFY_CHARACTERISTIC changes then you will know whenever the CHiP has a new response for you to read and handle. 

If you are familiar with the [MiP BLE Protocol](https://github.com/WowWeeLabs/MiP-BLE-Protocol) then you know it has the additional complexity of needing to parse the hexadecimal text received to convert it to binary. The CHiP robot sends its responses back in binary so this parsing isn't required. 

Yet another iOS/macOS example:
```objective-c
// Invoked upon completion of a -[readValueForCharacteristic:] request or on the reception of a notification/indication.
- (void) peripheral:(CBPeripheral *)aPeripheral didUpdateValueForCharacteristic:(CBCharacteristic *)characteristic error:(NSError *)err
{
    if (err)
        NSLog(@"Read encountered error (%@)", err);

    // Response from CHiP command has been received.
    if ([characteristic.UUID isEqual:[CBUUID UUIDWithString:@CHIP_RECEIVE_DATA_NOTIFY_CHARACTERISTIC]])
    {
        const uint8_t* pResponseBytes = characteristic.value.bytes;
        NSUInteger responseLength = [characteristic.value length];
        
        // Handle the CHiP response.
}
```


## CHiP Command Reference
Requests are sent to the CHiP by writing a buffer of atleast one byte to the CHIP_SEND_DATA_WRITE_CHARACTERISTIC. The first element in this buffer will always contain the **Command Byte** from the table below. Some commands will require additional bytes to be placed in this outgoing buffer. The **Request Byte Count** column of the table below shows whether a particular command only requires the **Command Byte** to be sent or if it also requires additional bytes be sent as well. The specific notes for each command will document what is to be placed in these additional bytes when needed. 

The **Response Byte Count** column of the table indicates how many bytes of data the CHiP will respond with on the CHIP_RECEIVE_DATA_NOTIFY_CHARACTERISTIC when this particular command has been sent.  **N/A** indicates that this command expects no response data back from CHiP. When a response is expected, the first byte of the response will contain the **Command Byte** of the request to which it is the response. The **Response Byte Count** column will indicate how many other bytes are expected in the response and the notes for the particular command will indicate how each additional byte should be interpreted.

| Command Name                              | Command Byte | Request Byte Count | Response Byte Count
|-------------------------------------------|--------------|--------------------|--------------------
| [Drive](#drive)                           | 0x78         | 1 + 3              | N/A
| [Action](#action)                         | 0x07         | 1 + 1              | N/A
| [GetSpeed](#getspeed)                     | 0x46         | 1                  | 1 + 1
| [SetSpeed](#setspeed)                     | 0x45         | 1 + 1              | N/A
| [GetEyeBrightness](#geteyebrightness)     | 0x49         | 1                  | 1 + 1
| [SetEyeBrightness](#seteyebrightness)     | 0x48         | 1 + 1              | N/A
| [PlaySound](#playsound)                   | 0x06         | 1 + 2              | N/A
| [GetVolume](#getvolume)                   | 0x16         | 1                  | 1 + 1
| [SetVolume](#setvolume)                   | 0x18         | 1 + 1              | N/A
| [GetBatteryLevel](#getbatterylevel)       | 0x1C         | 1                  | 1 + 3
| [GetCurrentDateTime](#getcurrentdatetime) | 0x3A         | 1                  | 1 + 8
| [SetCurrentDateTime](#setcurrentdatetime) | 0x43         | 1 + 8              | N/A
| [GetAlarmDateTime](#getalarmdatetime)     | 0x4A         | 1                  | 1 + 6
| [SetAlarmDateTime](#setalarmdatetime)     | 0x44         | 1 + 6              | N/A
| [Version](#version)                       | 0x14         | 1                  | 1 + 10
| [Sleep](#sleep)                           | 0xFA         | 3                  | N/A


---
### Drive
Commands the CHiP to drive in a particular direction. The cool thing about the CHiP is that it is capable of omnidirectional motion where it can travel forward and reverse, strafe left and right, and spin in place. It can even be doing all 3 at once and this command allows you to control such motion. Let's give names to these 3 degrees of motion: forwardReverse, leftRight, and spin. The desired amount of motion in each of these directions can range between -32 and 32.

| Motion         | Value | Description |
|----------------|-------|-------------|
| forwardReverse | -32   | Maximum reverse velocity |
| <br>           | 0     | No forward/reverse motion |
| <br>           | 32    | Maximum forward velocity |
| leftRight      | -32   | Maximum strafe to the left |
| <br>           | 0     | No strafing motion |
| <br>           | 32    | Maximum strafe to the right |
| spin           | -32   | Maximum spin to the left |
| <br>           | 0     | No spin motion |
| <br>           | 32    | Maximum spin to the right |

Before these motion rates varying from -32 to 32 can be sent to the CHiP, they need to be converted into the format expected by the CHiP. I could try to describe the math needed here in words but I think some actual C code will convey the requirements better:
```c
    if (forwardReverse < 0)
        forwardReverse = 0x20 + (-forwardReverse);
    else if (forwardReverse > 0)
        forwardReverse = 0x00 + forwardReverse;

    if (spin < 0)
        spin = 0x60 + (-spin);
    else if (spin > 0)
        spin = 0x40 + spin;

    if (leftRight < 0)
        leftRight = 0xA0 + (-leftRight);
    else if (leftRight > 0)
        leftRight = 0x80 + leftRight;
```

#### Request Packet
| Byte 0 | Byte 1         | Byte 2 | Byte 3    |
|--------|----------------|--------|-----------|
| 0x78   | forwardReverse | spin   | leftRight |


---
### Action
Commands the CHiP to start performing one of its built-in actions such as sit, lie down, etc.

The supported actions include:

| Name                                           | Value |
|------------------------------------------------|-------|
| CHIP_ACTION_RESET                              | 0x01  |
| CHIP_ACTION_SIT                                | 0x02  |
| CHIP_ACTION_LIE_DOWN                           | 0x03  |
| CHIP_ACTION_ALL_IDLE_MODE                      | 0x04  |
| CHIP_ACTION_DANCE                              | 0x05  |
| CHIP_ACTION_VR_TRAINING1                       | 0x06  |
| CHIP_ACTION_VR_TRAINING2                       | 0x07  |
| CHIP_ACTION_RESET2                             | 0x08  |
| CHIP_ACTION_JUMP                               | 0x09  |
| CHIP_ACTION_YOGA                               | 0x0a  |
| CHIP_ACTION_WATCH_COME                         | 0x0b  |
| CHIP_ACTION_WATCH_FOLLOW                       | 0x0c  |
| CHIP_ACTION_WATCH_FETCH                        | 0x0d  |
| CHIP_ACTION_BALL_TRACKING                      | 0x0e  |
| CHIP_ACTION_BALL_SOCCER                        | 0x0f  |
| CHIP_ACTION_BASE                               | 0x10  |
| CHIP_ACTION_DANCE_BASE                         | 0x11  |
| CHIP_ACTION_STOP_OR_STAND_FROM_BASE            | 0x12  |
| CHIP_ACTION_GUARD_MODE                         | 0x13  |
| CHIP_ACTION_FREE_ROAM                          | 0x14  |
| CHIP_ACTION_FACE_DOWN_FOR_CONTROLLING_CHIPPIES | 0x15  |

#### Request Packet
| Byte 0 | Byte 1 |
|--------|--------|
| 0x07   | action |

---
### GetSpeed
Requests the current speed setting of the CHiP. It can be changed with the [SetSpeed](#setspeed) command.

The two supported speed settings for CHiP are:

| Name             | Value | Description |
|------------------|-------|-------------|
| CHIP_SPEED_ADULT | 0x00  | Full speed option |
| CHIP_SPEED_KID   | 0x01  | Reduced speed option, more suitable for young children |

#### Request Packet
| Byte 0 |
|--------|
| 0x46   |

#### Response Packet
| Byte 0 | Byte 1 |
|--------|--------|
| 0x46   | speed  |


---
### SetSpeed
Sets the current speed setting of the CHiP. The current setting can be retrieved by issuing the [GetSpeed](#getspeed) command.

The two supported speed settings for CHiP are:

| Name             | Value | Description |
|------------------|-------|-------------|
| CHIP_SPEED_ADULT | 0x00  | Full speed option |
| CHIP_SPEED_KID   | 0x01  | Reduced speed option, more suitable for young children |

#### Request Packet
| Byte 0 | Byte 1 |
|--------|--------|
| 0x45   | speed  |


---
### GetEyeBrightness
Requests the current setting for the brightness of CHiP's eye LEDs. The eye brightness can be changed by sending the [SetEyeBrightness](#seteyebrightness) command.

The brightness values can vary from 0x00 to 0xFF with the following values having special meanings:

| Brightness Value | Meaning |
|------------------|-------------|
| 0x00             | Default brightness - What CHiP would use if you hadn't changed it |
| 0x01             | Minimum brightness |
| 0xFF             | Maximum brightness |

#### Request Packet
| Byte 0 |
|--------|
| 0x49   |

#### Response Packet
| Byte 0 | Byte 1      |
|--------|-------------|
| 0x49   | brightness  |


---
### SetEyeBrightness
Sets the brightness of the CHiP's eye LEDs. The current setting can be retrieved by issuing the [GetEyeBrightness](#geteyebrightness) command.

The brightness values can vary from 0x00 to 0xFF with the following values having special meanings:

| Brightness Value | Meaning |
|------------------|-------------|
| 0x00             | Default brightness - What CHiP would use if you hadn't changed it |
| 0x01             | Minimum brightness |
| 0xFF             | Maximum brightness |

#### Request Packet
| Byte 0 | Byte 1      |
|--------|-------------|
| 0x48   | brightness  |


---
### PlaySound
Command the CHiP to start playing one of its built-in sounds.

The song to play can be one of the following:

| Sound Name | Value |
|------------|-------|
| CHIP_SOUND_BARK_X1_ANGRY_A34 | 1 |
| CHIP_SOUND_BARK_X1_CURIOUS_PLAYFUL_HAPPY_A34 | 2 |
| CHIP_SOUND_BARK_X1_NEUTRAL_A34 | 3 |
| CHIP_SOUND_BARK_X1_SCARED_A34 | 4 |
| CHIP_SOUND_BARK_X2_ANGRY_A34 | 5 |
| CHIP_SOUND_BARK_X2_CURIOUS_PLAYFUL_HAPPY_A34 | 6 |
| CHIP_SOUND_BARK_X2_NEUTRAL_A34 | 7 |
| CHIP_SOUND_BARK_X2_SCARED_A34 | 8 |
| CHIP_SOUND_CRY_A34 | 9 |
| CHIP_SOUND_GROWL_A_A34 | 10 |
| CHIP_SOUND_GROWL_B_A34 | 11 |
| CHIP_SOUND_GROWL_C_A34 | 12 |
| CHIP_SOUND_HUH_LONG_A34 | 13 |
| CHIP_SOUND_HUH_SHORT_A34 | 14 |
| CHIP_SOUND_LICK_1_A34 | 15 |
| CHIP_SOUND_LICK_2_A34 | 16 |
| CHIP_SOUND_PANT_FAST_A34 | 17 |
| CHIP_SOUND_PANT_MEDIUM_A34 | 18 |
| CHIP_SOUND_PANT_SLOW_A34 | 19 |
| CHIP_SOUND_SNIFF_1_A34 | 20 |
| CHIP_SOUND_SNIFF_2_A34 | 21 |
| CHIP_SOUND_YAWN_A_A34 | 22 |
| CHIP_SOUND_YAWN_B_A34 | 23 |
| CHIP_SOUND_ONE_A34 | 24 |
| CHIP_SOUND_TWO_A34 | 25 |
| CHIP_SOUND_THREE_A34 | 26 |
| CHIP_SOUND_FOUR_A34 | 27 |
| CHIP_SOUND_FIVE_A34 | 28 |
| CHIP_SOUND_SIX_A34 | 29 |
| CHIP_SOUND_SEVEN_A34 | 30 |
| CHIP_SOUND_EIGHT_A34 | 31 |
| CHIP_SOUND_NIGHT_A34 | 32 |
| CHIP_SOUND_TEN_A34 | 33 |
| CHIP_SOUND_ZERO_A34 | 34 |
| CHIP_SOUND_CHIP_DOG_COUGH_2_A34 | 35 |
| CHIP_SOUND_CHIP_DOG_CRY_2_A34 | 36 |
| CHIP_SOUND_CHIP_DOG_CRY_3_A34 | 37 |
| CHIP_SOUND_CHIP_DOG_CRY_4_A34 | 38 |
| CHIP_SOUND_CHIP_DOG_CRY_5_A34 | 39 |
| CHIP_SOUND_CHIP_DOG_EMO_CURIOUS_1_A34 | 40 |
| CHIP_SOUND_CHIP_DOG_EMO_CURIOUS_2_A34 | 41 |
| CHIP_SOUND_CHIP_DOG_EMO_CURIOUS_3_A34 | 42 |
| CHIP_SOUND_CHIP_DOG_EMO_EXCITED_1_A34 | 43 |
| CHIP_SOUND_CHIP_DOG_EMO_EXCITED_2_A34 | 44 |
| CHIP_SOUND_CHIP_DOG_EMO_EXCITED_3_A34 | 45 |
| CHIP_SOUND_CHIP_DOG_EMO_LAZY_1_A34 | 46 |
| CHIP_SOUND_CHIP_DOG_EMO_LAZY_2_A34 | 47 |
| CHIP_SOUND_CHIP_DOG_EMO_LAZY_3_A34 | 48 |
| CHIP_SOUND_CHIP_DOG_EMO_RESPONSE_1_A34 | 49 |
| CHIP_SOUND_CHIP_DOG_EMO_RESPONSE_2_A34 | 50 |
| CHIP_SOUND_CHIP_DOG_EMO_RESPONSE_3_A34 | 51 |
| CHIP_SOUND_CHIP_DOG_EMO_SCARED_YIP_2_A34 | 52 |
| CHIP_SOUND_CHIP_DOG_EMO_SCARED_YIP_3_A34 | 53 |
| CHIP_SOUND_CHIP_DOG_FART_1_A34 | 54 |
| CHIP_SOUND_CHIP_DOG_FART_2_A34 | 55 |
| CHIP_SOUND_CHIP_DOG_FART_3_A34 | 56 |
| CHIP_SOUND_CHIP_DOG_GROWL_1_A34 | 57 |
| CHIP_SOUND_CHIP_DOG_GROWL_2_A34 | 58 |
| CHIP_SOUND_CHIP_DOG_GROWL_4_A34 | 59 |
| CHIP_SOUND_CHIP_DOG_GROWL_5_A34 | 60 |
| CHIP_SOUND_CHIP_DOG_HICCUP_1_A34 | 61 |
| CHIP_SOUND_CHIP_DOG_HICCUP_2_A34 | 62 |
| CHIP_SOUND_CHIP_DOG_HOWL_1_A34 | 63 |
| CHIP_SOUND_CHIP_DOG_HOWL_2_A34 | 64 |
| CHIP_SOUND_CHIP_DOG_HOWL_3_A34 | 65 |
| CHIP_SOUND_CHIP_DOG_HOWL_4_A34 | 66 |
| CHIP_SOUND_CHIP_DOG_HOWL_5_A34 | 67 |
| CHIP_SOUND_CHIP_DOG_LICK_2_A34 | 68 |
| CHIP_SOUND_CHIP_DOG_LICK_3_A34 | 69 |
| CHIP_SOUND_CHIP_DOG_LOWBATTERY_1_A34 | 70 |
| CHIP_SOUND_CHIP_DOG_LOWBATTERY_2_A34 | 71 |
| CHIP_SOUND_CHIP_DOG_MUFFLE_1_A34 | 72 |
| CHIP_SOUND_CHIP_DOG_MUFFLE_2_A34 | 73 |
| CHIP_SOUND_CHIP_DOG_MUFFLE_3_A34 | 74 |
| CHIP_SOUND_CHIP_DOG_PANT_1_A34 | 75 |
| CHIP_SOUND_CHIP_DOG_PANT_2_A34 | 76 |
| CHIP_SOUND_CHIP_DOG_PANT_3_A34 | 77 |
| CHIP_SOUND_CHIP_DOG_PANT_4_A34 | 78 |
| CHIP_SOUND_CHIP_DOG_PANT_5_L_A34 | 79 |
| CHIP_SOUND_CHIP_DOG_SMOOCH_1_A34 | 80 |
| CHIP_SOUND_CHIP_DOG_SMOOCH_2_A34 | 81 |
| CHIP_SOUND_CHIP_DOG_SMOOCH_3_A34 | 82 |
| CHIP_SOUND_CHIP_DOG_SNEEZE_1_A34 | 83 |
| CHIP_SOUND_CHIP_DOG_SNEEZE_2_A34 | 84 |
| CHIP_SOUND_CHIP_DOG_SNEEZE_3_A34 | 85 |
| CHIP_SOUND_CHIP_DOG_SNIFF_1_A34 | 86 |
| CHIP_SOUND_CHIP_DOG_SNIFF_2_A34 | 87 |
| CHIP_SOUND_CHIP_DOG_SNIFF_4_A34 | 88 |
| CHIP_SOUND_CHIP_DOG_SNORE_1_A34 | 89 |
| CHIP_SOUND_CHIP_DOG_SNORE_2_A34 | 90 |
| CHIP_SOUND_CHIP_DOG_SPECIAL_1_A34 | 91 |
| CHIP_SOUND_CHIP_SING_DO1_SHORT_A34 | 92 |
| CHIP_SOUND_CHIP_SING_DO2_SHORT_A34 | 93 |
| CHIP_SOUND_CHIP_SING_FA_SHORT_A34 | 94 |
| CHIP_SOUND_CHIP_SING_LA_SHORT_A34 | 95 |
| CHIP_SOUND_CHIP_SING_MI_SHORT_A34 | 96 |
| CHIP_SOUND_CHIP_SING_RE_SHORT_A34 | 97 |
| CHIP_SOUND_CHIP_SING_SO_SHORT_A34 | 98 |
| CHIP_SOUND_CHIP_SING_TI_SHORT_A34 | 99 |
| CHIP_SOUND_CHIP_DOG_BARK_3_A34 | 100 |
| CHIP_SOUND_CHIP_DOG_BARK_4_A34 | 101 |
| CHIP_SOUND_CHIP_DOG_BARK_5_A34 | 102 |
| CHIP_SOUND_CHIP_DOG_BARK_MULTI_3_A34 | 103 |
| CHIP_SOUND_CHIP_DOG_BARK_MULTI_4_A34 | 104 |
| CHIP_SOUND_CHIP_DOG_BARK_MULTI_5_A34 | 105 |
| CHIP_SOUND_CHIP_DOG_BURP_1_A34 | 106 |
| CHIP_SOUND_CHIP_DOG_BURP_2_A34 | 107 |
| CHIP_SOUND_CHIP_DOG_COUGH_1_A34 | 108 |
| CHIP_SOUND_CHIO_DOG_EMO_RESPONSE_3A | 109 |
| CHIP_SOUND_CHIP_DEMO_MUSIC_2 | 110 |
| CHIP_SOUND_CHIP_DEMO_MUSIC_3 | 111 |
| CHIP_SOUND_CHIP_DOG_BARK_1 | 112 |
| CHIP_SOUND_CHIP_DOG_BARK_2 | 113 |
| CHIP_SOUND_CHIP_DOG_BARK_MULTI_1 | 114 |
| CHIP_SOUND_CHIP_DOG_BARK_MULTI_2 | 115 |
| CHIP_SOUND_CHIP_DOG_CRY_1 | 116 |
| CHIP_SOUND_CHIP_DOG_EMO_CURIOUS_2A | 117 |
| CHIP_SOUND_CHIP_DOG_EMO_EXCITED_3A | 118 |
| CHIP_SOUND_CHIP_DOG_EMO_LAZY_1A | 119 |
| CHIP_SOUND_CHIP_DOG_EMO_LAZY_2A | 120 |
| CHIP_SOUND_CHIP_DOG_EMO_LAZY_3A | 121 |
| CHIP_SOUND_CHIP_DOG_GROWL_3 | 122 |
| CHIP_SOUND_CHIP_DOG_HOWL_1A | 123 |
| CHIP_SOUND_CHIP_DOG_HOWL_3A | 124 |
| CHIP_SOUND_CHIP_DOG_HOWL_4A | 125 |
| CHIP_SOUND_CHIP_DOG_HOWL_5A | 126 |
| CHIP_SOUND_CHIP_DOG_LICK_1 |127 |
| CHIP_SOUND_CHIP_DOG_LOWBATTERY_1A | 128 |
| CHIP_SOUND_CHIP_DOG_LOWBATTERY_2A | 129 |
| CHIP_SOUND_CHIP_DOG_MUFFLE_1A | 130 |
| CHIP_SOUND_CHIP_DOG_SMOOCH_3A | 131 |
| CHIP_SOUND_CHIP_DOG_SNEEZE_1A | 132 |
| CHIP_SOUND_CHIP_DOG_SNIFF_3 | 133 |
| CHIP_SOUND_CHIP_DOG_SNIFF_4A | 134 |
| CHIP_SOUND_CHIP_MUSIC_DEMO_1 | 135 |
| CHIP_SOUND_CHIO_DOG_EMO_RESPONSE_1A | 136 |
| CHIP_SOUND_CHIO_DOG_EMO_RESPONSE_2A | 137 |
| CHIP_SOUND_SHORT_MUTE_FOR_STOP | 138 |

Specifing CHIP_SOUND_SHORT_MUTE_FOR_STOP as the sound to play will cause the currently playing sound to be immediately stopped.

#### Request Packet
| Byte 0 | Byte 1         | Byte 2 |
|--------|----------------|--------|
| 0x06   | sound          | 0x00   |


---
### GetVolume
Requests the current volume level setting of the CHiP. The volume can be changed by sending the [SetVolume](#setvolume) command.

The volume setting can vary from 1 to 11.

| Volume Setting | Meaning        |
|----------------|----------------|
| 1              | Mute           |
| 11             | Maximum volume |

#### Request Packet
| Byte 0 |
|--------|
| 0x16   |

#### Response Packet
| Byte 0 | Byte 1  |
|--------|---------|
| 0x16   | volume  |


---
### SetVolume
Sets the current volume level of the CHiP. The current setting can be retrieved by issuing the [GetVolume](#getvolume) command.

The volume setting can vary from 1 to 11.

| Volume Setting | Meaning        |
|----------------|----------------|
| 1              | Mute           |
| 11             | Maximum volume |

#### Request Packet
| Byte 0 | Byte 1  |
|--------|---------|
| 0x18   | volume  |


---
### GetBatteryLevel
Requests the current battery level and charging state from the CHiP.

The CHiP will respond with 3 properties: batteryLevel, chargingStatus, and chargerType.

#### batteryLevel
The batteryLevel returned from the CHiP can be converted via some math into a floating point value that varies from 0.0f (fully discharged) to 1.0f (fully charged). The following C code snippet shows this math:
```c
float floatBatteryLevel = (float)(batteryLevel - 0x7D) / 34.0f
```

#### chargingStatus
The chargingStatus returned from the CHiP can have one of the following 3 values:

| Value                                  | Description |
|----------------------------------------|-------------|
| CHIP_CHARGING_STATUS_NOT_CHARGING      | CHiP isn't currently being charged |
| CHIP_CHARGING_STATUS_CHARGING          | CHiP is currently being charged. See chargerType property to know which charger is being used |
| CHIP_CHARGING_STATUS_CHARGING_FINISHED | CHiP is still on charger but is now fully charged |

#### chargerType
The chargerType returned from the CHiP can have one of the following 2 values:

| Value                                  | Description                                        |
|----------------------------------------|----------------------------------------------------|
| CHIP_CHARGER_TYPE_DC                   | CHiP is being charged from DC barrel jack in chest |
| CHIP_CHARGER_TYPE_BASE                 | CHiP is being charged in base station (SmartBed)   |

#### Request Packet
| Byte 0 |
|--------|
| 0x1C   |

#### Response Packet
| Byte 0 | Byte 1         | Byte 2      | Byte 3       |
|--------|----------------|-------------|--------------|
| 0x1C   | chargingStatus | chargerType | batteryLevel |


---
### GetCurrentDateTime
Requests the current date and time from the CHiP. It can be set by sending the [SetCurrentDateTime](#setcurrentdatetime) command. The current date and time is only maintained while the CHiP is powered on. Powering off the CHiP will cause the date and time to be reset.

The response packet contains the following date/time fields:

| Field     | Description |
|-----------|-------------|
| year      | The current year |
| month     | The current month (1-January to 12-December) |
| day       | The current day of month (1 to 31) |
| hour      | The current hour (0 - 23) |
| minute    | The current minute (0 - 59) |
| second    | The current second (0 - 59) |
| dayOfWeek | The current day of week (0-Sunday to 6-Saturday) |

The year is 16-bit and stored in two bytes of the response packet (big endian ordering).
```c
    year = ((uint16_t)response[1] << 8) | (uint16_t)response[2];
```

#### Request Packet
| Byte 0 |
|--------|
| 0x3A   |

#### Response Packet
| Byte 0 | Byte 1          | Byte 2          | Byte 3 | Byte 4 | Byte 5 | Byte 6 | Byte 7 | Byte 8    |
|--------|-----------------|-----------------|--------|--------|--------|--------|--------|-----------|
| 0x3A   | year (High Byte)| year (Low Byte) | month  | day    | hour   | minute | second | dayOfWeek |


---
### SetCurrentDateTime
Sets the current date and time on the CHiP. The current date and time of the CHiP can be requested by sending the [GetCurrentDateTime](#getcurrentdatetime) command. The current date and time is only maintained while the CHiP is powered on. Powering off the CHiP will cause the date and time to be reset.

The request packet contains the following date/time fields:

| Field     | Description |
|-----------|-------------|
| year      | The current year |
| month     | The current month (1-January to 12-December) |
| day       | The current day of month (1 to 31) |
| hour      | The current hour (0 - 23) |
| minute    | The current minute (0 - 59) |
| second    | The current second (0 - 59) |
| dayOfWeek | The current day of week (0-Sunday to 6-Saturday) |

The year is 16-bit and stored in two bytes of the request packet (big endian ordering).
```c
    request[1] = (year >> 8) & 0xFF;
    request[2] = year & 0xFF;
```

#### Request Packet
| Byte 0 | Byte 1          | Byte 2          | Byte 3 | Byte 4 | Byte 5 | Byte 6 | Byte 7 | Byte 8    |
|--------|-----------------|-----------------|--------|--------|--------|--------|--------|-----------|
| 0x43   | year (High Byte)| year (Low Byte) | month  | day    | hour   | minute | second | dayOfWeek |


---
### GetAlarmDateTime
Requests the current alarm setting from the CHiP. When the alarm date/time arrives, the MiP will start barking and not stop until the user pets its head. The alarm date/time can be set by sending the [SetAlarmDateTime](#setalarmdatetime) command.

The response packet contains the following date/time fields:

| Field     | Description |
|-----------|-------------|
| year      | The current year |
| month     | The current month (1-January to 12-December) |
| day       | The current day of month (1 to 31) |
| hour      | The current hour (0 - 23) |
| minute    | The current minute (0 - 59) |

If all of the date/time fields in the response are 0 then there is currently no active alarm set in the CHiP.

The year is 16-bit and stored in two bytes of the response packet (big endian ordering).
```c
    year = ((uint16_t)response[1] << 8) | (uint16_t)response[2];
```

#### Request Packet
| Byte 0 |
|--------|
| 0x4A   |

#### Response Packet
| Byte 0 | Byte 1          | Byte 2          | Byte 3 | Byte 4 | Byte 5 | Byte 6 |
|--------|-----------------|-----------------|--------|--------|--------|--------|
| 0x4A   | year (High Byte)| year (Low Byte) | month  | day    | hour   | minute |


---
### SetAlarmDateTime
Sets the current alarm setting from the CHiP. When the alarm date/time arrives, the MiP will start barking and not stop until the user pets its head. The current alarm setting can be requested by sending the [GetAlarmDateTime](#getalarmdatetime) command.

The request packet contains the following date/time fields:

| Field     | Description |
|-----------|-------------|
| year      | The current year |
| month     | The current month (1-January to 12-December) |
| day       | The current day of month (1 to 31) |
| hour      | The current hour (0 - 23) |
| minute    | The current minute (0 - 59) |

The alarm can be cleared by setting all of the date/time fields in the request to 0.

The year is 16-bit and stored in two bytes of the request packet (big endian ordering).
```c
    request[1] = (year >> 8) & 0xFF;
    request[2] = year & 0xFF;
```

#### Request Packet
| Byte 0 | Byte 1          | Byte 2          | Byte 3 | Byte 4 | Byte 5 | Byte 6 |
|--------|-----------------|-----------------|--------|--------|--------|--------|
| 0x44   | year (High Byte)| year (Low Byte) | month  | day    | hour   | minute |


---
### Version
Requests the version information for CHiP's various hardware/firmware components.

The CHiP version response contains the following fields:

| Field Name |
|------------|
| bodyHardware |
| headHardware |
| mechanic |
| bleSpiFlash |
| nuvotonSpiFlash |
| bleBootloader |
| bleApromFirmware |
| nuvotonBootloaderFirmware |
| nuvotonApromFirmware |
| nuvoton |

#### Request Packet
| Byte 0 |
|--------|
| 0x14   |

#### Response Packet
| Byte 0 | Byte 1       | Byte 2       | Byte 3   | Byte 4      | Byte 5          | Byte 6        | Byte 7           | Byte 8                    | Byte 9               | Byte 10 |
|--------|--------------|--------------|----------|-------------|-----------------|---------------|------------------|---------------------------|----------------------|---------|
| 0x14   | bodyHardware | headHardware | mechanic | bleSpiFlash | nuvotonSpiFlash | bleBootloader | bleApromFirmware | nuvotonBootloaderFirmware | nuvotonApromFirmware | nuvoton |


---
### Sleep
Requests that the CHiP switch into sleep mode.

#### Request Packet
| Byte 0 | Byte 1 | Byte 2 |
|--------|--------|--------|
| 0xFA   | 0x12   | 0x34   |
