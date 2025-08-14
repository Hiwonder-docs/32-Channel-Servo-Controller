# 2. LSC Series Controller Communication Protocol 

Serial communication, baud rate: 9600

| Frame Header | Data Length | Command | Parameters  |
|--------------|-------------|---------|-------------|
| 0x55 0x55    | Length      | Cmd     | Prm 1…Prm N |

Frame Header: Two consecutive bytes 0x55 0x55 indicate the start of a valid data packet.

Data Length: Equals the number of parameters N plus one byte for the command and one byte for the data length field itself. Formula: Length = N + 2

Command (Cmd): The control command code.

Parameters (Prm): Additional control data required for the command.

## 2.1 Sending Data from the User to the Controller

The TX pin of the user's device should be connected to the controller's RX pin. The controller and the user's system must share a common ground (GND). When the controller receives correct data, the **blue LED2** will blink once, indicating successful reception. If incorrect data is sent, **LED2** will remain steadily lit. The buzzer will emit two quick "**beep-beep**" sounds to indicate a data error.

(1) Command Name: CMD\_ SERVO_MOVE, Command Value: 3, Data Length (Length):

**Description:** Controls the movement of any number of servos. Length = (Number of Servos × 3) + 5

Prm1: Number of servos to control

Prm2: Time (low byte)

Prm3: Time (high byte)

Prm4: Servo ID

Prm5: Angle position (low byte)

Prm6: Angle position (high byte)

Parameters ...: Additional servos follow the same pattern as Prm4–Prm6 for different servo IDs.

Examples:

① Control servo ID 1 to move to position 2000 within 1000 ms:

| Frame Header | Data Length | Command | Parameters                    |
|--------------|-------------|---------|-------------------------------|
| 0x55 0x55    | 0x08        | 0x03    | 0x01 0xE8 0x03 0x01 0xD0 0x07 |

② Control servo 2 to move to position 1200 within 800 ms, and servo 9 to move to position 2300 within 800 ms:

| Frame Header | Data Length | Command | Parameters |
|----|----|----|----|
| 0x55 0x55 | 0x0B | 0x03 | 0x02 0x20 0x03 0x02 0xB0 0x04 0x09 0xFC 0x08 |

(2) Command Name: CMD \_ACTION_GROUP_RUN, Command Value: 6, Data Length (Length): 5

**Description:** Controls the execution of an action group. The action group must already be downloaded to the controller. You can specify the number of times the action group runs. If you want the action group to run indefinitely, set the repeat count parameter to 0.

Prm1: Action group ID to run

Prm2: Number of times to run (low byte)

Prm3: Number of times to run (high byte)

Examples:

① Control Action Group ID 8 to run once:

| Frame Header | Data Length | Command | Parameters     |
|--------------|-------------|---------|----------------|
| 0x55 0x55    | 0x05        | 0x06    | 0x08 0x01 0x00 |

② Control Action Group ID 2 to run indefinitely:

| Frame Header | Data Length | Command | Parameters     |
|--------------|-------------|---------|----------------|
| 0x55 0x55    | 0x05        | 0x06    | 0x02 0x00 0x00 |

(3) Command Name: CMD \_ACTION_GROUP_STOP, Command Value: 7, Data Length (Length): 2

**Description:** Stops the currently running action group. If no action group is running, sending this command will have no effect.

Parameters: None

Examples:

① Stop the currently running action group:

| Frame Header | Data Length | Command | Parameters |
|--------------|-------------|---------|------------|
| 0x55 0x55    | 0x02        | 0x07    | None       |

(4) Command Name: CMD_ACTION_GROUP \_SPEED, Command Value: 11, Data Length (Length): 5

**Description:** Adjusts the speed of an action group in percentage format. For example: setting speed to 200 means 200% of the original speed (twice as fast). If the Action Group ID is set to 0xFF, all downloaded action groups will have their speeds adjusted.

:::{Note}

* The speed adjustment will not be saved after power-off. It must be set again after reboot.
* Servos have a maximum physical rotation speed. Setting the speed to a multiple higher than their limit will not increase the actual movement speed.

:::

Parameter 1: Action Group ID

Parameter 2: Speed percentage (Low byte)

Parameter 3: Speed percentage (High byte)

**Examples:**

① Control Action Group ID 8 to run at 50% speed:

| Frame Header | Data Length | Command | Parameters     |
|--------------|-------------|---------|----------------|
| 0x55 0x55    | 0x05        | 0x0B    | 0x08 0x32 0x00 |

② Adjust all downloaded action groups (0xFF) to 300% speed:

| Frame Header | Data Length | Command | Parameters     |
|--------------|-------------|---------|----------------|
| 0x55 0x55    | 0x05        | 0x0B    | 0xFF 0x2C 0x01 |

(5) Command Name: CMD_GET_BATTERY_VOLTAGE, Command Value: 15, Data Length (Length): 2

**Description:** Requests the battery voltage of the controller, in millivolts (mV). After this command is sent, the controller immediately returns a data packet containing two parameter bytes.

Parameters: None

Send Format:

| Frame Header | Data Length | Command | Parameters |
|--------------|-------------|---------|------------|
| 0x55 0x55    | 0x02        | 0x0F    | None       |

Return: The controller's response contains: Parameter 1 – Low 8 bits of the voltage value, Parameter 2 – High 8 bits of the voltage value

| Frame Header | Data Length | Command | Parameters |
|--------------|-------------|---------|------------|
| 0x55 0x55    | 0x04        | 0x0F    | 0x4C 0x1D  |

## 2.2 Data Sent by the Controller to User

During operation, when the controller's status changes, it will actively send data to the user via the serial port. For example, when an action group finishes running. There are multiple ways to operate the controller, such as using a PS2 controller, connecting via a Bluetooth module, or using the user's custom serial interface. Hence, it is necessary for different control methods to be aware of the current status of the controller to facilitate proper management and operation. The following are the instructions that the controller returns to the user.

(1) Command Name: CMD \_ACTION_GROUP_RUN, Command Value: 6, Data Length (Length): 5

**Description:** When the controller starts running an action group, it immediately sends data with parameter information. The return data format is the same as the format used by the user to control the action group run command.

Prm1: Action group ID to run

Prm2: Number of times to run (low byte)

Prm3: Number of times to run (high byte)

Examples:

When action group 8 is running and is set to run once, the controller returns the following data packet to the user:

| Frame Header | Data Length | Command | Parameters     |
|--------------|-------------|---------|----------------|
| 0x55 0x55    | 0x05        | 0x06    | 0x08 0x01 0x00 |

:::{Note}

Even if the user initiates the action group run via the serial port, once the controller starts executing the command, it will immediately send back this status message via the serial port.

:::

(2) Command Name: CMD \_ACTION_GROUP_STOP, Command Value: 7, Data Length (Length): 2

**Description:** When a running action group is forcibly stopped by another method, for example, via the wireless controller or by the user sending a stop command, this command is returned.

The return data format is the same as the format used by the user to stop the action group running command.

Parameters: None Examples:

① When an action group that is currently running is forcibly stopped, the following data is returned:

| Frame Header | Data Length | Command | Parameters |
|--------------|-------------|---------|------------|
| 0x55 0x55    | 0x02        | 0x07    | None       |

(3) Command Name: CMD \_ACTION_GROUP_COMPLETE, Command Value: 8, Data Length (Length): 5

**Description:** This command is returned when an action group finishes running naturally, which was not forcefully stopped, but ended because its programmed duration was reached.

Prm1: Action group ID to run

Prm2: Number of times to run (low byte)

Prm3: Number of times to run (high byte)

Examples:

When action group No. 8 is set to run once and finishes naturally, the controller returns the following data to the user:

| Frame Header | Data Length | Command | Parameters     |
|--------------|-------------|---------|----------------|
| 0x55 0x55    | 0x05        | 0x08    | 0x08 0x01 0x00 |
