The **NeoSWSerial** class is intended as an more-efficient drop-in replacement for the Arduino built-in class `SoftwareSerial`.  If you could use `Serial`, `Serial1`, `Serial2` or `Serial3`, you should use [NeoHWSerial](https://github.com/SlashDevin/NeoHWSerial) instead.  If you could use an Input Capture pin (ICP1, pins 8 & 9 on an UNO), you should consider  [NeoICSerial](https://github.com/SlashDevin/NeoICSerial) instead.

**NeoSWSerial** is limited to four baud rates: 9600 (default), 19200, 31250 (MIDI) and 38400.

The class methods are nearly identical to the built-in `SoftwareSerial`, except for two new methods, `attachInterrupt` and `detachInterrupt`:

```
    typedef void (* isr_t)( uint8_t );
    void attachInterrupt( isr_t fn );
    void detachInterrupt() { attachInterrupt( (isr_t) NULL ); };

  private:
    isr_t  _isr;
```

There are five, nay, **six** advantages over `SoftwareSerial`:

**1)** It uses *much* less CPU time.

**2)** Simultaneous transmit and receive is fully supported.

**3)** Interrupts are not disabled for the entire RX character time.  (They are disabled for most of each TX character time.)

**4)** It is much more reliable (far fewer receive data errors).

**5)** Characters can be handled with a user-defined procedure at interrupt time.  This should prevent most input buffer overflow problems.  Simply register your procedure with the 'NeoSWSerial' instance:

```
    #include <NeoSWSerial.h>
    NeoSWSerial ss( 4, 3 );

    volatile uint32_t newlines = 0UL;

    static void handleRxChar( uint8_t c )
    {
      if (c == '\n')
        newlines++;
    }

    void setup()
    {
      ss.attachInterrupt( handleRxChar );
      ss.begin( 9600 );
    }
```

Remember that the registered procedure is called from an interrupt context, and it should return as quickly as possible.  Taking too much time in the procedure will cause many unpredictable behaviors, including loss of received data.  See the similar warnings for the built-in [`attachInterrupt`](https://www.arduino.cc/en/Reference/AttachInterrupt) for digital pins.

The registered procedure will be called from the ISR whenever a character is received.  The received character **will not** be stored in the `rx_buffer`, and it **will not** be returned from `read()`.  Any characters that were received and buffered before `attachInterrupt` was called remain in `rx_buffer`, and could be retrieved by calling `read()`.

If `attachInterrupt` is never called, or it is passed a `NULL` procedure, the normal buffering occurs, and all received characters must be obtained by calling `read()`.

**6)** The NeoSWSerial ISRs can be disabled.  This can help you avoid linking conflicts with other PinChangeInterrupt libraries, like EnableInterrupt:

```
void myDeviceISR()
{
  NeoSWSerial::rxISR( *portInputRegister( digitalPinToPort( RX_PIN ) ) );
  // if you know the exact PIN register, you could do this:
  //    NeoSWSerial::rxISR( PIND );
}

void setup()
{
  myDevice.begin( 9600 );
  enableInterrupt( RX_PIN, myDeviceISR, CHANGE );
  enableInterrupt( OTHER_PIN, otherISR, RISING );
}
```

This class supports the following MCUs at 16MHz: ATmega328P (Pro, UNO, Nano), ATmega2560 (Mega), ATmega256RFR2 (Xplained Pro, Altair, Pinoccio), ATmega1284P (MightyCore), ATmega32U4 (Micro, Leonardo), ATtinyx61, ATtinyx4, ATtinyx5 (Trinket), and AT90USB1286 (Teensy++)

These MCUs are also supported at 8MHz: ATmega328P (Pro, Fio, Feather 328), ATmega2560 (Mega), ATmega256RFR2, ATmega1284P (Mayfly, Mbili), ATmega32U4 (Feather 32U4, Flora), and ATtinyx5 (Trinket, Gemma).  To run on 8MHz boards, NeoSWSerial must set another timer prescaler - timer4 on the 32U4, timer1 on the AtTiny, and timer2 on the others.  For the vast majority of cases, this will not be a problem.  But, if not used carefully, this will cause the tone() function (and possibly others) to behave strangely.  It could also cause the [EnviroDIY SDI-12](https://github.com/EnviroDIY/Arduino-SDI-12) library (which was partly modeled on NeoSWSerial) to malfunction.  To avoid these problems make sure that you ignore() or end() all instances of NeoSWSerial before using the other functions/libraries.  You must then begin() or listen() again to restart NeoSWSerial.
