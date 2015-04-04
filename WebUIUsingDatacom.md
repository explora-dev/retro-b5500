# WebUI Using Datacom #



Data communications interfaces for computers were at a primitive state when the B5000 and B5500 were designed, and it shows in the equipment and software available with the B5500. Datacom was an I/O device. It multiplexed the traffic for multiple terminals onto one data path to and from the I/O Control Units.

The B5500 initially supported teletypes and other low-speed devices, but eventually supported the Burroughs line of poll-select video terminals, the TC500 (a programmable keyboard/printer device), and RJE through satellite B300 or IBM 1050 systems.



# Background #

There was an early "inquiry" system based on the B5480 Data Communications Control Unit (DCCU). This was somewhat limited, as it could not initiate messages to a remote terminal -- it could only send responses to messages from a terminal.

The second iteration was a "data transmission" system, which consisted of the B249 Data Transmission Control Unit (DTCU) plus up to 15 B487 Data Transmission Terminal Units (DTTU). The DTCU connected to the B5500 I/O Control Unit. It provided a multiplexed interface for the B487s, along with translation circuitry to convert ASCII and Baudot character sets to the BCL encoding used with the B5500.

Each B487 supported up to 15 communications circuits. Each circuit was terminated in an adapter card that plugged into the B487. There were different adapter cards for different types of circuits (e.g., teletype current-loop, Bell 103A modem, TWIX, etc.). The B487 buffered characters from each circuit to assemble messages, sending blocks of message data instead of individual characters through the B249 to the I/O Control Unit.

The B487 was limited, however, by a very small buffer memory -- 420 characters total for all circuits. 28 characters of this was allocated to each adapter slot, but multiple 28-character chunks could be assigned to an adapter by leaving the appropriate number of adapter slots following that adapter unoccupied. Thus, on one B487 you could configure lots of adapters with small buffers, or fewer adapters with larger buffers.

One problem with implementing a datacom interface in the web-based emulator is that web browsers act strictly as clients in a network connection, and what the emulator needs to do is act as a server. As a result, the web-based emulator cannot support something obvious, such as the Telnet protocol -- there just is not any way to get a browser to accept those connections as a server.

Therefore, we have decided to punt and implement a datacom interface for a _single_ terminal within the browser environment itself. While internally this terminal implements much of the mechanism of the B487 and B249, on the surface it works somewhat like the SPO device. It has a terminal-like window that emulates a teletype device. The device has been assigned four buffer segments, or 112 characters of buffer memory. Ping-pong buffering is not currently supported.

The window for the terminal interface we have developed for the web-based emulator looks like this:

> ![https://googledrive.com/host/0BxqKm7v4xBswRjNYQnpqM0ItbkU/B5500-Datacom-Terminal.png](https://googledrive.com/host/0BxqKm7v4xBswRjNYQnpqM0ItbkU/B5500-Datacom-Terminal.png)

You type messages into this window, and the window displays the system's responses -- all at ten characters per second -- just like any good 1960s terminal would do.


# Terminal Control Panel #

The B249 DTCU and B487 DTTU do not have a user interface. The terminal itself is all that appears, and roughly models a Teletype Model 33 KSR unit. Across the top of its window are a **Connect** button and a series of red indicators reflecting the status of the terminal's buffer in the DTTU:

  * **NR** -- Not Ready -- the buffer and/or DTCU/DTTU is not ready.
  * **IDLE** -- the buffer is in an idle state, ready to receive input from the user or output from the system.
  * **RR** -- Read Ready -- The user has entered a message into the buffer that is ready to be sent to the system. A buffer becomes Read Ready when the user enters an end-of-message character (see below) or the buffer becomes completely full. The buffer signals the system that it has data to send. The state reverts to Idle once the system reads the buffer.
  * **WR** -- Write Ready -- this state is normally set after the system sends data to the buffer without an end-of-message character. Once the buffer sends the all of data to the terminal, the buffer enters the Write Ready state and signals the system that it is ready for more output.
  * **IBZ** -- Input Busy -- the buffer enters this state when the user types the first character of a message. It stays in this state until either the user enters an end-of-message character or the buffer becomes full, at which point it enters the Read Ready state.
  * **OBZ** -- Output Busy -- the buffer enters this state after receiving an output message from the system. It stays in this state while it is transmitting the data to the terminal. Once the buffer is empty, it enters either the Idle or Write Ready state, depending on whether it has seen an end-of-message character from the system or the buffer has become full.
  * **AB** -- Abnormal -- this is a flag that has different meaning depending on the state of the buffer. For example, Input Busy Abnormal indicates that the user has entered a "?" in the message (used to indicate a control message to the MCP). Write Ready Abnormal indicates that either the terminal has just connected to the DTTU or the user has entered the WRU (who-are-you) character. It is generally reset the next time the buffer changes state.
  * **INT** -- Interrupt Requested -- this is another flag that can be set for several states when the buffer requires attention from the system. It indicates than an Inquiry Request interrupt has been sent to the B5500. It will be reset once the B5500 performs an Interrogate operation to determine the status of the highest-priority buffer that initiated the interrupt.
  * **FB** -- Full Buffer -- this flag indicates that the buffer has been filled on either input or output without sensing an end-of-character message. It will be reset once the buffer is emptied.


# Using the Datacom Terminal #

The **Connect** button is used to initiate a connection between the terminal and the DTTU, similar to dialing a telephone number and establishing a modem connection. Simply click the button to connect. It will light. Click again to disconnect. After connecting, a system running the Datacom MCP should respond with an identification message within a few seconds:

```
    B-5500 01/00
```

The numbers following "`B-5500`" are the DTTU number and buffer number within that DTTU. Since the emulator currently supports only one terminal buffer, these numbers are constant.

Because the B5500 had a character set consisting of only 64 printable characters, it had no native provision for the types of control characters (carriage-return, backspace, etc.) we are now accustomed to using with ASCII. Therefore, certain of the Model 33 printable characters were reserved for this purpose. On input, the following characters have special meaning:

  * `<` -- backspace.
  * (left-arrow) -- end of message. The Model 33 Teletype had this character, but on modern keyboards this is represented by the underscore character (`_`). For the web-based emulator, we use both underscore and the tilde (`~`) to represent the left arrow. The ASCII DC1 character (also known as X-on or control-Q) will also end an input message.
  * (control-B) -- also known as ASCII STX, this is the equivalent of sending a Break signal from the terminal.
  * (control-E) -- also known as ASCII ENQ, this is the WRU (who-are-you) character.
  * (control-L) -- also known as ASCII FF, this character would clear the input buffer after the user started entering a message, but leave the buffer in an Input Busy state.
  * `!` -- disconnect -- entered on input, this character will disconnect the terminal from the DTTU.

The emulated terminal supports two user-interface features that the DTTU Teletype adapter did not:

  1. You can press the **Enter** key on your keyboard instead of the "`~`" to end a message. The terminal will print a "`~`".
  1. You can press the Backspace key to correct errors. The terminal will print a "`<`".

See pages 3-15 and 3-16 in the [B5500 Handbook](http://bitsavers.org/pdf/burroughs/1031986_B5500_Handbook_Aug70.pdf) for information on the character substitutions used by the B249/B487.

You may move and resize the terminal window, and minimize it, but _do not close the window_. The system will warn you if you attempt to do this. Closing the window will render the terminal inoperable until the emulator is reinitialized with the **POWER ON** button.

When you resize the terminal window, the "paper" area will change size accordingly. If the window becomes too narrow for the current output, the output lines will be clipped on the right, although the output itself is unaffected and will reappear if the window is made larger. Below a certain minimum (and essentially usable) window size, the window contents will no longer resize and will be clipped.

The area representing the "paper" for the terminal has a 5000-line scrollback. You can copy portions of this text using  ordinary click-and-drag functions of your pointing device. If you double-click anywhere in the text, however, the emulator will open a temporary window and copy the entire contents of the "paper" into it. From there, you can save the text in a file or copy/paste it into another application. When you have finished with the temporary window, simply close it.