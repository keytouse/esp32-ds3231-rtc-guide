# DS3231 RTC Guide for ESP32 and ESP-IDF

## 1. Purpose

This guide explains how to integrate, initialize, validate, recover, and
maintain a DS3231 real-time clock (RTC) connected to an ESP32.

It combines general engineering guidance with a case study from an
ESP32-S3 project using ESP-IDF 6.2.

The essential rule is:

**Never treat an unverified RTC value as trusted current time.**

An RTC can return a plausible-looking date even when its timekeeping
history is uncertain. Applications that depend on time must check the
clock and handle failures explicitly.

## 2. What the DS3231 does

The DS3231 is a battery-backed real-time clock with calendar, timekeeping,
alarm, and temperature-measurement functions.

The ESP32 communicates with it over I2C. The RTC maintains its own calendar
and can continue timekeeping while the main system supply is unavailable,
provided its backup supply and oscillator remain functional.

The DS3231's Oscillator Stop Flag (OSF) is bit 7 of status register 0x0F.

- OSF = 1: the oscillator-stop condition has occurred since the flag was
  last cleared. Do not assume the calendar is trustworthy.
- OSF = 0: the flag is clear, but this alone does not prove that the date
  was initialized correctly or that the clock agrees with a trusted source.

OSF should be cleared only as part of a deliberate initialization or
recovery procedure, after the time has been set and verified.

Official reference:
https://www.analog.com/media/en/technical-documentation/data-sheets/ds3231.pdf

## 3. Hardware and wiring

### Typical connections

| DS3231 module | ESP32 |
| --- | --- |
| VCC | Supply compatible with the specific module |
| GND | Common ground |
| SDA | Configured ESP32 I2C SDA pin |
| SCL | Configured ESP32 I2C SCL pin |
| Backup battery | Correct battery type and polarity for the module |

The DS3231's usual 7-bit I2C address is 0x68.

Confirm the module's supply-voltage requirements, logic levels, pull-up
resistors, and battery circuitry before connecting it. Breakout boards
differ. Some contain charging circuitry intended for rechargeable cells;
do not install a non-rechargeable coin cell in a circuit that charges it.

The ESP32 GPIO assignments are board-specific. Do not copy pins from
another project without checking its schematic and other GPIO uses.

### This project's wiring

Verified source configuration:

- Board: ESP32-S3
- SDA: GPIO 1
- SCL: GPIO 2
- I2C controller: I2C_NUM_0
- RTC: DS3231
- I2C address: 0x68

These settings describe this project, not a universal ESP32 wiring scheme.

## 4. ESP-IDF integration

This project uses the esp-idf-lib components:

- esp-idf-lib__i2cdev
- esp-idf-lib__ds3231

The component dependencies are declared in main/CMakeLists.txt.

The project's public RTC interface is in main/rtc.h and its implementation
is in main/rtc.c. Its functions include:

- kt_rtc_init(sda_gpio, scl_gpio)
- kt_rtc_read(&epoch)
- kt_rtc_read_with_status(&epoch, &status)
- kt_rtc_write(epoch)
- kt_rtc_recover(trusted_utc)
- kt_rtc_clear_osf()
- kt_rtc_is_ok()

These are project-specific wrapper functions, not universal ESP-IDF APIs.
Review their implementation and the installed driver version before
reusing them in another project.

### Initialization and reading

A typical application should:

1. Initialize the I2C driver and RTC device descriptor.
2. Read the RTC status register.
3. Check OSF and read the calendar.
4. Validate the calendar fields and application-specific date range.
5. Mark the RTC usable only after the required checks succeed.
6. Keep time-dependent functions disabled if validation fails.

An I2C read succeeding is not sufficient proof that the time is correct.

## 5. UTC, Unix timestamps, and local time

Use a consistent time convention throughout the application.

This project interprets the DS3231 calendar as UTC and converts between
calendar fields and Unix timestamps using timegm() and gmtime_r().

A Unix timestamp represents seconds from 1970-01-01 00:00:00 UTC.

Recommended practice:

- Obtain initial time from a trusted source.
- Use UTC internally for storage, communication, and comparisons.
- Convert to local time only for display or explicitly local schedules.
- Define the intended timezone for each schedule.
- Account for daylight-saving rules if the deployment location uses them.

Do not use the ESP32's local timezone setting to reinterpret RTC calendar
fields that the firmware has already defined as UTC.

## 6. First-time setup

When a new RTC is installed, do not assume that its existing calendar is
correct.

1. Obtain a trusted current UTC timestamp, for example from a trusted
   network time source or a correctly set administrator device.
2. Validate the timestamp and the allowed date range.
3. Write the timestamp to the RTC.
4. Read the clock back and compare it with the trusted timestamp, allowing
   for the expected elapsed time and communication latency.
5. Wait briefly and read it again.
6. Confirm that the clock advances at the expected rate.
7. Clear OSF only after the time has been deliberately set and verified.
8. Read the status and calendar again.
9. Confirm that OSF remains clear and that the clock continues advancing.
10. Only then declare the RTC healthy for time-dependent application logic.

If any step fails, report the failure and leave time-dependent operations
disabled. Do not clear OSF merely to make a warning disappear.

## 7. Recovery after an invalid clock

Possible symptoms include:

- Dates near 2000 or 2001.
- OSF set.
- Time not advancing.
- I2C communication errors.
- Time disagreeing with a trusted source.

A safe recovery sequence is:

1. Diagnose the hardware and power supplies.
2. Obtain a trusted UTC timestamp.
3. Write that time explicitly.
4. Verify the time against the supplied timestamp.
5. Verify that the clock advances.
6. Clear OSF only after these checks pass.
7. Read the status and calendar again.
8. Confirm OSF remains clear and the clock still advances.
9. Enable time-dependent operations only after all required checks pass.

Do not use the firmware compilation date as a substitute for current time.
A build timestamp may be old and does not prove that the RTC is healthy.

If recovery fails, preserve the error logs and investigate the RTC, I2C
connections, power supply, backup battery, and driver behavior.

## 8. Scheduler and application safety

Schedules, timed switching, expiry checks, and other time-dependent actions
must have an explicit policy for invalid time.

Recommended policy:

- If RTC initialization fails, mark time as untrusted.
- If OSF is set, treat time as untrusted until recovery succeeds.
- If a read fails or the calendar is implausible, do not execute schedules.
- Do not silently replace invalid time with a build timestamp.
- Log why scheduling is disabled.
- Revalidate time when the RTC or its power state may have changed.

A plausible date is not a substitute for a trusted time source and validation.

## 9. Project case study: ESP32-S3 and DS3231

### Verified configuration

The following details were confirmed from the project source:

- ESP32-S3 target and ESP-IDF 6.2 environment.
- SDA GPIO 1 and SCL GPIO 2.
- I2C_NUM_0 and DS3231 address 0x68.
- UTC interpretation using timegm() and gmtime_r().
- Calendar year validation from 2024 through 2100.
- Read-only diagnostic configuration:
  - KT_RTC_FORCE_SET = 0
  - KT_RTC_TEMP_SET_TEST = 0
  - KT_RTC_DIAGNOSTIC_READ_ONLY = 1
  - KT_RTC_PERIODIC_MONITOR = 0

With this configuration, normal initialization does not automatically set
the clock or clear OSF. An invalid calendar or set OSF causes initialization
to reject the time.

### Recovery implementation

The project's kt_rtc_recover() routine:

1. Requires an initialized DS3231 device.
2. Rejects trusted timestamps outside 2024–2100.
3. Marks the RTC unhealthy while recovery is in progress.
4. Writes the supplied UTC timestamp.
5. Reads back and checks the difference from the supplied timestamp.
6. Waits and verifies that the RTC advances.
7. Clears OSF only after those checks pass.
8. Reads the calendar and status again.
9. Verifies OSF remains clear and that the clock continues advancing.
10. Marks the RTC healthy only after the final checks succeed.

In the inspected implementation, the first read is accepted when it is
between 1 second behind and 20 seconds ahead of the supplied timestamp.
Each progression check accepts an advance of 1–4 seconds over an
approximately 1.2-second wait. These are implementation-specific
tolerances, not universal DS3231 specifications, and should be reviewed
if the recovery code or timing behavior changes.

The project also takes care when clearing OSF so that writing the status
register does not inadvertently clear alarm flags A1F and A2F.

### Recovery endpoint security

Administrative time-recovery interfaces should require authentication,
authorization, strict input validation, and careful error handling.
Avoid placing credentials in URLs because URLs may be recorded in logs,
browser history, and other systems.

Plain HTTP does not encrypt credentials or request data. Restrict access
to trusted networks and use an appropriately secured transport when
requests cross networks that are not trusted.

Before deploying any recovery interface, review its access controls,
transport security, validation, logging, and operational safeguards.

### Evidence and test limits

The project's boot log on 2026-10-24 showed:

- RTC calendar read succeeded.
- OSF was clear.
- The reported UTC clock was 2026-10-24 03:50:37.
- The ESP32 application subsequently reported local display time
  2026-10-24 08:50:37 in Pakistan time.

The recovery endpoint previously returned:
{"ok":true,"rtc":"verified"}

A normal ESP32 reboot was observed to preserve the valid RTC time while
the OSF flag remained clear.

A controlled removal and restoration of the board's main power has not yet
been documented as a completed test. Backup-battery endurance and behavior
with a depleted battery also remain unverified.

These observations apply to the tested board and firmware. They are not a
guarantee for every DS3231 module or ESP32 design.

## Code examples

The examples below use the project RTC API. The scheduling example is illustrative; it is not evidence that the current PWA implements a scheduler.

### Example 1: Read the RTC

```c
#include "rtc.h"
#include "esp_log.h"

static const char *TAG = "rtc_example";

esp_err_t read_rtc_time(time_t *out_utc)
{
    if (out_utc == NULL) {
        return ESP_ERR_INVALID_ARG;
    }

    if (!kt_rtc_is_ok()) {
        return ESP_ERR_INVALID_STATE;
    }

    esp_err_t err = kt_rtc_read(out_utc);
    if (err != ESP_OK) {
        ESP_LOGW(TAG, "RTC read failed: %s", esp_err_to_name(err));
    }
    return err;
}
```

`kt_rtc_is_ok()` is an internal health flag, not a continuous hardware check. A successful read alone does not prove that the clock is accurate or advancing.

### Example 2: Check the oscillator-stop flag

```c
time_t now_utc = 0;
uint8_t status = 0xFF;

esp_err_t err = kt_rtc_read_with_status(&now_utc, &status);
if (err != ESP_OK || status == 0xFF || (status & 0x80U) != 0) {
    /* Treat the time as untrusted; disable time-dependent actions. */
}
```

The DS3231 OSF (oscillator-stop flag) is bit 7 of status register `0x0F`. This API can return a successful calendar read even when reading the status register fails, so callers must explicitly reject the `0xFF` status sentinel.

### Example 3: Recover from a trusted UTC timestamp

```c
/* trusted_utc must come from a trusted, current time source. */
esp_err_t err = kt_rtc_recover(trusted_utc);
if (err != ESP_OK) {
    /* Keep time-dependent actions disabled and log the error. */
}
```

Do not substitute a firmware build timestamp or an invented fixed date for trusted current time. `kt_rtc_recover()` performs the project recovery checks; clearing OSF alone is not a substitute for recovery.

### Example 4: Gate a scheduled action (illustrative)

```c
time_t now_utc = 0;
uint8_t status = 0xFF;

esp_err_t err = kt_rtc_read_with_status(&now_utc, &status);
if (err == ESP_OK &&
    status != 0xFF &&
    (status & 0x80U) == 0 &&
    kt_rtc_is_ok()) {
    /* Evaluate a validated schedule using now_utc. */
    /* Execute only if due, with duplicate-execution protection. */
} else {
    /* Do not execute time-dependent actions. */
}
```

This is a reusable safety pattern, not a description of an existing PWA scheduler. Production code must also handle invalid schedules, read failures, clock progression, and recovery without accidentally repeating overdue actions.

---

## 10. Troubleshooting checklist

### I2C communication fails

- Verify power and common ground.
- Check SDA/SCL wiring and configured GPIO numbers.
- Confirm pull-ups and compatible logic levels.
- Check the I2C controller and device address.
- Review driver initialization and I2C error logs.

### Calendar is implausible

- Treat the time as untrusted.
- Read the raw calendar and status registers for diagnosis.
- Obtain a trusted timestamp.
- Use the explicit recovery procedure rather than a build timestamp.

### OSF is set

- Do not clear it just to suppress the warning.
- Establish the correct time.
- Verify time progression.
- Clear OSF only after the recovery checks pass.
- Verify the flag remains clear afterward.

### Time is lost after power removal

- Check the backup battery, polarity, condition, and module circuitry.
- Check whether the module's design supports the installed battery type.
- Repeat the recovery procedure with a trusted timestamp.
- Perform controlled power tests only when it is safe to do so.

### Clock appears to stop or jump

- Compare consecutive RTC readings against elapsed time.
- Check the main and backup supplies.
- Inspect OSF and communication errors.
- Do not enable scheduling until the cause is understood.

## 11. Maintenance and firmware updates

The DS3231 is a separate hardware device, so an ordinary ESP32 reboot or
application firmware update does not inherently erase its calendar.

However, firmware can change the RTC if it writes to the clock or its
registers. Review changes to initialization, recovery, I2C, and status
handling before deployment.

A replacement board with a different RTC needs its own initialization and
validation. A moved RTC may retain time if its backup supply and hardware
remain healthy.

For every release, preserve the distinction between:

- Reading and validating time.
- Setting time from a trusted source.
- Clearing OSF.
- Declaring the RTC healthy.
- Enabling time-dependent application features.

These operations should not be treated as interchangeable.

## 12. Further references

- Analog Devices DS3231 datasheet:
  https://www.analog.com/media/en/technical-documentation/data-sheets/ds3231.pdf
- ESP-IDF documentation:
  https://docs.espressif.com/projects/esp-idf/en/latest/esp32/
- esp-idf-lib components used by this project:
  https://github.com/esp-idf-lib

---

Document status: Initial guide based on inspected project source and
reported hardware tests. Review against the exact driver version and target
hardware before using the examples in another product.
