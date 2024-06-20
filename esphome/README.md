# ESPHome Config

The config is used to automate multiple not so important tasks.  
As hardware, we are using Shelly devices.

Parts of the config are based on the config from [jcallaghan](https://github.com/jcallaghan/esphome-config/). Thank you for the beautiful config.

---

| Directory | Use case                                                              |
|-----------|-----------------------------------------------------------------------|
| /         | One config for each device. Device-specific settings.                 |
| /devices  | Config file for each device model (Shelly 1PM, Shelly 2PM, and so on) |
| /boards   | Config file for each Board/CPU used                                   |
| /common   | Common config parts which are used on all devices                     |