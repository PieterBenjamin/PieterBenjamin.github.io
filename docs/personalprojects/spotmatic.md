---
title: Fixing an Old Film Camera
layout: post
has_children: false
parent: Personal Projects
back_to_top: true
---

I've been really enjoying film photography the last couple years. 

<img src="/assets/images/camera/sutro_tower_film.jpeg">

Recently I thrifted an original Pentax Spotmatic SP. It was just the body, and the film advance lever was stuck. So I took the bottom off, watched some quick YouTube videos, and poked at the gears until things started turning.

<img src="/assets/images/camera/workbench.JPEG">

Things looked promising so I shot a test roll. After getting it developed, I saw there were some issues with the shutter curtains. 

<img src="/assets/images/camera/Scan7.jpg">

I found [pentax-manuals.com](http://www.pentax-manuals.com/manuals/service/servicemanuals.htm) which had some useful content. But after looking more closely at the [spotmatic manual](http://www.pentax-manuals.com/manuals/service/spotmatic_sm.pdf) I realized the it was actually for a Spotmatic SP II, and I had the SP 1(?). Eventually I got lucky with a manual on [learncamerarepair.com](https://learncamerarepair.com/product.php?product=827&category=0&secondary=0).

Once I found where to tweak the tension of each shutter curtain, I realized I would need a new tool to time how long light was allowed to pass through the shutter curtains. I thought it would be fun to make this on my own, so I worked the [Arduino Student Kit](https://www.arduino.cc/education/student-kit/) and made a super simple circuit with the phototransistor that came in the kit.

```c
#define LIGHT_THRESHOLD 120 // picked arbirtarily while manually reading inputs
#define RECEIVER_PIN A0

void setup() {
  Serial.begin(9600);
}

void loop() {
  int light_val = analogRead(RECEIVER_PIN);     

  if (light_val > LIGHT_THRESHOLD) {
    long exposure_duration_millis = get_exposure_time_millis();
    Serial.print("exposure_time_millis:");
    Serial.println(exposure_duration_millis);
  }
}

int get_exposure_time_millis() {
  // https://www.arduino.cc/reference/en/language/functions/time/millis/
  // "This number will overflow (go back to zero), after approximately 70 minutes"
  long start_millis = millis();
  
  Serial.print("recording exposure time... ");
  while (true) { 
    if (analogRead(RECEIVER_PIN) <= LIGHT_THRESHOLD) {
      return millis() - start_millis;
    }
  }
}
```

<img src="/assets/images/camera/arduino.JPEG">

After maybe ~40 rotations to the "closing curtain take-up roller worm" I was able to measure the following exposure speeds

| 1/n second exposure time | expected exposure time (ms) |  measured exposure time (ms) |
|:-------------------------|:----------------------------|:-----------------------------|
| 1                        | 1000                        | 843                          | 
| 2                        | 500                         | 400                          | 
| 4                        | 250                         | 209                          | 
| 8                        | 125                         | 102                          | 
| 15                       | 67                          | 62                           | 
| 30                       | 33                          | 36                           | 
| 60                       | 16                          | 18                           | 
| 125                      | 8                           | 8                            | 
| 250                      | 4                           | 4                            | 
| 500                      | 2                           | 2                            | 
| 1000                     | 1                           | 1                            | 

Which is a terrific success! Things might be slightly undexposed at the slowest exposure settings, but that's something I can manually correct for using the in-body light meter! Currently I'm off shooting/developing a new roll to see if the fix worked.
