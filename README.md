# home-assistant-myki
A simple package for integrating MyKi Gsm/GSP tracking watch in home-assistant

## Overview
I am using a [rest](https://www.home-assistant.io/integrations/rest/) integration to fetch the device location data from the [MyKi Web App](https://my.myki.watch/?lang=EN), and update my child's custom [device_tracker](https://www.home-assistant.io/integrations/device_tracker/) which is defined in `known_devices.yaml`.

First, we define a custom `device_tracker` in `/config/known_devices.yaml` 
```
# whichever device_id you want to define, I used myki_watch_1 here
myki_watch_1:
  # device friendly name
  name: MyKi Watch
  # made up, but unique mac address
  mac: FF:FF:FF:FF:FF:F1
  # default device picture
  picture: https://www.home-assistant.io/images/favicon-192x192.png
  # indicate that the device position will change
  track: true
# ... repeat for any other devices
```

Then, we create a myki package, where we bundle all the config related to this MyKi integration.
Contents of `/config/packages/myki.yaml` 
```
# I use rest integration to fetch the data from 
rest: 
  - authentication: basic
    username: ""
    password: ""
    # I have set it to update every 60s
    scan_interval: 60
    headers:
      Accept: application/json,text/plain,*/*
      Content-Type: application/json
      User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:106.0) Gecko/20100101 Firefox/106.0
      # The Authorization header copied from the MyKi web app and stored in secrets.yaml
      Authorization: !secret my_myki_token
    resource: https://my.myki.watch/api/watch/app-data
    params:
      # the MyKi Watch deviceid from the Web App (does not really need to be in secrets.yaml)
      id: !secret my_myki_device
    sensor:
      - name: "MyKi Watch 1"
        value_template: "OK"
        json_attributes_path: "$.data.current"
        json_attributes:
          - position
          - battery
          - steps
  # ... Copy and paste for other MyKi Watches

# Next, we update the device_tracker GPS position using the device_tracker.see service, 
# every time the command_line sensor is updated...
automation:
  - id: update_myki_watch_1_position 
    alias: Update MyKi position for watch 1
    description: ''
    trigger:
    - platform: state
      entity_id: sensor.myki_watch_1
    condition: []
    action:
    - service: device_tracker.see
      data:
        source_type: gps
        dev_id: myki_watch_1
        host_name: myki_watch_1
        gps:
        - '{{ state_attr(''sensor.myki_watch_1'', ''position'').latitude }}'
        - '{{ state_attr(''sensor.myki_watch_1'', ''position'').longitude }}'
        gps_accuracy: '{{ state_attr(''sensor.myki_watch_1'', ''position'').accuracy }}'
        battery: '{{ state_attr(''sensor.myki_watch_1'', ''battery'') | round(0) }}'
    mode: single
```
Make sure you enable packages in `configuration.yaml`:
```
homeassistant:
  packages: !include_dir_named packages/
```

Next, you go to your `Settings > People` and add your child and use `myki_watch_1` as a tracking device.

That's it. 
Cheers
