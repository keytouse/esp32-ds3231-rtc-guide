# DS3231 RTC Guide for ESP32 and ESP-IDF

Practical documentation for integrating and troubleshooting a DS3231
real-time clock (RTC) with ESP32 projects using ESP-IDF.

Topics include:

- Hardware connections and I2C integration
- UTC timestamps and local-time handling
- Invalid clock recovery and the oscillator-stop flag (OSF)
- Safe handling of time-dependent application actions
- Diagnostic examples and troubleshooting

## Documentation

See [DS3231-RTC-GUIDE.md](DS3231-RTC-GUIDE.md).

## Important

Examples must be adapted and tested for the target application.
A valid calendar reading alone does not guarantee clock accuracy.
Recovery requires a trusted current time source.

Review all files for secrets and project-specific information before
publishing them publicly.
