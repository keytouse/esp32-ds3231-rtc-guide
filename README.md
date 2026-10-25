# ESP32 DS3231 RTC Guide for ESP32 and ESP-IDF

**Status: Work in Progress — technical review and hardware validation ongoing.**

Practical documentation for integrating, diagnosing, validating, and
recovering a DS3231 real-time clock (RTC) in ESP32 projects using ESP-IDF.

## Topics

- Hardware connections and I2C integration
- UTC timestamps, Unix time, and local-time handling
- Calendar validation and the oscillator-stop flag (OSF)
- Safe RTC initialization and recovery from a trusted time source
- Fail-safe handling of schedules and other time-dependent actions
- Diagnostic examples and troubleshooting

## Documentation

Read the [DS3231 RTC Guide](DS3231-RTC-GUIDE.md).

## Current validation status

The guide includes observations from an ESP32-S3 project using ESP-IDF 6.2.
Those observations apply to the tested setup and are not guarantees for
other boards or DS3231 modules.

Further review and testing are ongoing. In particular, controlled
main-power-loss testing, backup-battery endurance, and behavior with a
depleted backup battery have not yet been documented as completed tests.

The code examples must be checked against the actual driver version and
target application before reuse. A successful I2C read or a plausible
calendar value does not, by itself, establish that the clock is trustworthy.

## Safety principle

**Never allow time-dependent actions to run on unverified time.**

If the RTC fails validation, keep dependent operations disabled until
time has been recovered from a trusted source and the required checks pass.

## References

- [Analog Devices DS3231 datasheet](https://www.analog.com/media/en/technical-documentation/data-sheets/ds3231.pdf)
- [ESP-IDF documentation](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/)
- [esp-idf-lib components](https://github.com/esp-idf-lib)

---

This guide is an evolving technical document. Findings and validation
status will be updated as testing progresses.
