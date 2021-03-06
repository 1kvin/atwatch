# sput-ino

Sput is an unit testing framework for C/C++ that focuses on simplicity of use and maximum portability.  
https://www.use-strict.de/sput-unit-testing/

This project is Sput unit testing framework port to Arduino.

- Run unit tests on any Arduino device
- Run same unit tests on desktop
- Very simple (single header file with macros)

detailed manual (in Russian): https://habr.com/post/419445/  
good post on unit testing on Arduino: https://stackoverflow.com/questions/780819/how-can-i-unit-test-arduino-code#11437456

## Install
To install, just clone https://github.com/sadr0b0t/sput-ino git repo to $HOME/Arduino/libraries

~~~bash
cd ~/Arduino/libraries/
git clone https://github.com/sadr0b0t/sput-ino.git
~~~

and restart IDE.

Or on github page click Clone or download > Download ZIP, then install sput-ino-master.zip ZIP via Arduino library installation menu.

Basic example should appear in Arduino examples menu: File > Examples > sput-ino

## Self-contained Arduino sketch example with tests
_For extended project layout see https://github.com/sadr0b0t/sput-ino-demo_

https://github.com/sadr0b0t/sput-ino/blob/master/examples/sput-ino-monolith/sput-ino-monolith.ino

Write some code to test
~~~cpp
/**
 * @return a plus b
 */
int a_plus_b(int a, int b) {
    return a + b;
}

/**
 * @return a minus b
 */
int a_minus_b(int a, int b) {
    return a - b;
}

/** 
 * Switch led on if number is even
 * @param pin len pin id
 * @param num number
 * @param true, if num is even
 */
bool led_on_even(int pin, int num) {
    if(num % 2 == 0) {
        digitalWrite(pin, HIGH);
    } else {
        digitalWrite(pin, LOW);
    }
    return num % 2 == 0;
}
~~~

Write tests with sput
(for more details see http://www.use-strict.de/sput-unit-testing/tutorial.html)

~~~cpp
#include "sput.h"

/** Test a_plus_b call */
void test_a_plus_b() {
    sput_fail_unless(a_plus_b(2, 2) == 4, "2 + 2 == 4");
    sput_fail_unless(a_plus_b(-2, 2) == 0, "-2 + 2 == 0");
    
    // this one would pass on 32-bit controllers and would fail on 16-bit Arduino
    sput_fail_unless(a_plus_b(34000, 34000) == 68000, "34000 + 34000 == 68000");
}

/** Test a_minus_b call */
void test_a_minus_b() {
    sput_fail_unless(a_minus_b(115, 6) == 109, "115 - 6 == 109");
    sput_fail_unless(a_minus_b(13, 17) == -4, "13 - 17 == -4");
}

/** Test test_led_on_even call */
bool test_led_on_even() {
  
    // 1) this test shows that we can also test code, that uses Arduino API
    // 2) please, consider this test also as kind of antipattern:
    // - we make here device-specific calls which may pass or fail
    //     on different devices
    // - what we should test here is NOT that digitalWrite did MAKE changes
    //     to GPIO register (so we check this with device-specific
    //     pinMode+digitalRead), but that we did CALL digitallWrite
    //     with specific params in tested code (this can hardly be
    //     checked when we run test on device, but can easily be checked
    //     when we run test on desktop with Arduino API mockups).
    //     See desktop+mockup version of this test:
    //       example-desktop/mylib-test-desktoponly.cpp
    // - when we run this specific test on desktop, digitalRead
    //     would be implemented as mockup anyway, so, probably, whatever
    
    pinMode(13, OUTPUT);
    
    sput_fail_unless(led_on_even(13, 2), "num=2 => led#13 on");
    // would pass on desktop, might fail or pass on difference devices
    // (e.g.: Arduino Due - fail, ChipKIT Uno32 - pass)
    sput_fail_unless(digitalRead(13) == HIGH, "num=2 => led#13 on");
    
    sput_fail_unless(!led_on_even(13, 5), "num=5 => led#13 off");
    sput_fail_unless(digitalRead(13) == LOW, "num=5 => led#13 off");
    
    sput_fail_unless(led_on_even(13, 18), "num=18 => led#13 on");
    sput_fail_unless(digitalRead(13) == HIGH, "num=18 => led#13 on");
}
~~~

Pack tests to test suites
~~~cpp
/** Test suite for a_plus_b call */
int mylib_test_suite_a_plus_b() {
    sput_start_testing();

    sput_enter_suite("a plus b");
    sput_run_test(test_a_plus_b);
    
    sput_finish_testing();
    return sput_get_return_value();
}

/** Test suite for a_minus_b call */
int mylib_test_suite_a_minus_b() {
    sput_start_testing();
    
    sput_enter_suite("a minus b");
    sput_run_test(test_a_minus_b);
    
    sput_finish_testing();
    return sput_get_return_value();
}

/** Test suite for led_on_even call */
int mylib_test_suite_led_on_even() {
    sput_start_testing();
    
    sput_enter_suite("led on even");
    sput_run_test(test_led_on_even);
    
    sput_finish_testing();
    return sput_get_return_value();
}

/** All tests in one bundle */
int mylib_test_suite() {
    sput_start_testing();

    sput_enter_suite("a plus b");
    sput_run_test(test_a_plus_b);
    
    sput_enter_suite("a minus b");
    sput_run_test(test_a_minus_b);
    
    sput_enter_suite("led on even");
    sput_run_test(test_led_on_even);
    
    sput_finish_testing();
    return sput_get_return_value();
}
~~~

Run tests on device
~~~cpp
void run_tests() {
    Serial.println("#################### Start testing...");

    // comment out specific test suites if firmware does not
    // fit to device memory
    
    // Test suite for a_plus_b call
    mylib_test_suite_a_plus_b();

    // Test suite for a_minus_b call
    mylib_test_suite_a_minus_b();
    
    // Test suite for led_on_even call
    mylib_test_suite_led_on_even();

    // All tests in one bundle
    //mylib_test_suite();
    
    Serial.println("#################### Finished testing");
}
~~~

Add setup/loop as usual and call test runner
~~~cpp
void setup() {
    Serial.begin(9600);
    while (!Serial);

    // run tests
    run_tests();

    // other code - kinda application business logic
    Serial.println("Just show that we call functions from tested lib, nothing useful here");
    
    pinMode(13, OUTPUT);
    
    Serial.print("14+23=");
    Serial.println(a_plus_b(14, 23));
    Serial.print("14-23=");
    Serial.println(a_minus_b(14, 23));
    Serial.print("34000+34000=");
    Serial.println(a_plus_b(34000, 34000));
}

void loop() {
    static int i = 0;
    led_on_even(13, i++);
    delay(2000);
}
~~~

Compile and upload sketch to device, watch results in serial monitor.

On ChipKIT Uno32, PIC32-based Arduino-based board:
~~~
#################### Start testing...

== Entering suite #1, "a plus b" ==

[1:1]  test_a_plus_b:#1  "2 + 2 == 4"  pass
[1:2]  test_a_plus_b:#2  "-2 + 2 == 0"  pass
[1:3]  test_a_plus_b:#3  "34000 + 34000 == 68000"  pass

--> 3 check(s), 3 ok, 0 failed (0.00%)

==> 3 check(s) in 1 suite(s) finished after 0.00 second(s),
    3 succeeded, 0 failed (0.00%)

[SUCCESS]

== Entering suite #1, "a minus b" ==

[1:1]  test_a_minus_b:#1  "115 - 6 == 109"  pass
[1:2]  test_a_minus_b:#2  "13 - 17 == -4"  pass

--> 2 check(s), 2 ok, 0 failed (0.00%)

==> 2 check(s) in 1 suite(s) finished after 0.00 second(s),
    2 succeeded, 0 failed (0.00%)

[SUCCESS]

== Entering suite #1, "led on even" ==

[1:1]  test_led_on_even:#1  "num=2 => led#13 on"  pass
[1:2]  test_led_on_even:#2  "num=2 => led#13 on"  pass
[1:3]  test_led_on_even:#3  "num=5 => led#13 off"  pass
[1:4]  test_led_on_even:#4  "num=5 => led#13 off"  pass
[1:5]  test_led_on_even:#5  "num=18 => led#13 on"  pass
[1:6]  test_led_on_even:#6  "num=18 => led#13 on"  pass

--> 6 check(s), 6 ok, 0 failed (0.00%)

==> 6 check(s) in 1 suite(s) finished after 0.00 second(s),
    6 succeeded, 0 failed (0.00%)

[SUCCESS]
#################### Finished testing
Just show that we call functions from tested lib, nothing useful here
14+23=37
14-23=-9
34000+34000=68000
~~~

on Arduino Uno (AVR 16 bit int):
~~~
#################### Start testing...

== Entering suite #1, "a#################### Start testing...

== Entering suite #1, "a plus b" ==

[1:1]  test_a_plus_b:#1  "2 + 2 == 4"  pass
[1:2]  test_a_plus_b:#2  "-2 + 2 == 0"  pass
[1:3]  test_a_plus_b:#3  "34000 + 34000 == 68000"  FAIL
!    Type:      fail-unless
!    Condition: a_plus_b(34000, 34000) == 68000
!    Line:      14

--> 3 check(s), 2 ok, 1 failed (?%)

==> 3 check(s) in 1 suite(s) finished after ? second(s),
    2 succeeded, 1 failed (?%)

[FAILURE]

== Entering suite #1, "a minus b" ==

[1:1]  test_a_minus_b:#1  "115 - 6 == 109"  pass
[1:2]  test_a_minus_b:#2  "13 - 17 == -4"  pass

--> 2 check(s), 2 ok, 0 failed (?%)

==> 2 check(s) in 1 suite(s) finished after ? second(s),
    2 succeeded, 0 failed (?%)

[SUCCESS]

== Entering suite #1, "led on even" ==

[1:1]  test_led_on_even:#1  "num=2 => led#13 on"  pass
[1:2]  test_led_on_even:#2  "num=2 => led#13 on"  pass
[1:3]  test_led_on_even:#3  "num=5 => led#13 off"  pass
[1:4]  test_led_on_even:#4  "num=5 => led#13 off"  pass
[1:5]  test_led_on_even:#5  "num=18 => led#13 on"  pass
[1:6]  test_led_on_even:#6  "num=18 => led#13 on"  pass

--> 6 check(s), 6 ok, 0 failed (?%)

==> 6 check(s) in 1 suite(s) finished after ? second(s),
    6 succeeded, 0 failed (?%)

[SUCCESS]
#################### Finished testing
Just show that we call functions from tested lib, nothing useful here
14+23=37
14-23=-9
34000+34000=2464
~~~

## Separate tested code and tests: split monolith sketch to modules

https://github.com/sadr0b0t/sput-ino/tree/master/examples/sput-ino-modules

## Run tests on desktop

Go to "sput-ino/example-desktop" dir
https://github.com/sadr0b0t/sput-ino/tree/master/example-desktop

~~~sh
cd $HOME/Arduino/libraries/sput-ino/example-desktop
~~~

build tests with gcc
~~~sh
#!/bin/sh
# simple build script, feel free to modify or convert it
# to your favourite build system config

#gcc -c c_file_stub.c
#g++ -std=c++11 -c cpp_file_stub.cpp

g++ -std=c++11 -c \
    -I. -I../examples/sput-ino-modules -I$HOME/Arduino/libraries/sput-ino/src \
    Arduino.cpp \
    ../examples/sput-ino-modules/mylib.cpp \
    ../examples/sput-ino-modules/mylib-test.cpp \
    mylib-test-desktoponly.cpp \
    mylib-test-main.cpp
g++ *.o -o test_mylib
~~~

run compiled tests:
~~~sh
./test_mylib
~~~

see result in console (identical to above).

