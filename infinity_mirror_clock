#include <EEPROM.h>
#include <FastLED.h>
#include <Encoder.h>
#include <DS3231.h>

//define number of LED and pin
#define NUM_LEDS 59
#define DATA_PIN 10

//define eeprom addresses
#define IS_UPLOADED 0
#define LED_BRIGHTNESS 1
#define HOUR_HSV_H 2
#define HOUR_HSV_S 3
#define MIN_HSV_H 4
#define MIN_HSV_S 5
#define SEC_HSV_H 6
#define SEC_HSV_S 7
#define BACKGROUND_HSV_H 8
#define BACKGROUND_HSV_S 9
#define RAINBOW_EFFECT 10

//define menu variables
#define ENCODER_1_PIN 2
#define ENCODER_2_PIN 3
#define ENCODER_BTN_PIN 4
#define MENU_VALUES_BUFFER_COUNT 2
#define MENUSTATES 10
#define MENU_SELECT_BLINK_RATE 500

#define DEBOUNCE_DELAY 10
#define MENU_TIMEOUT_DELAY 10000
#define SETTING_CHANGE_DELAY 200

#define SETTING_STEP 5

//display variables
#define MIN_BRIGHTNESS 50
#define DISPLAY_UPDATE_DELAY 30
#define NIGHT_BEGIN 21
#define NIGHT_END 8
#define EFFECT_UPDATE_DELAY 50
#define EFFECT_LAST_DELAY 60000

enum Direction : byte {
  NONE, LEFT, RIGHT
};


enum MenuState : byte {
  DISPLAY_NONE, SET_BRIGHTNESS, SET_HOUR, SET_MIN, SET_SEC, SET_BG_COLOR, SET_HOUR_COLOR, SET_MIN_COLOR, SET_SEC_COLOR, SET_RAINBOW
};

Direction encoderDirection = Direction::NONE;

unsigned long encoderBtnLastPress;
unsigned long lastMenuSettingChange;
unsigned long lastMenuSelectBlink;
bool menuSelectBlink;

bool encoderBtnState;
bool encoderBtnClicked;
MenuState menuState = MenuState::DISPLAY_NONE;

byte menuValuesBuffer[MENU_VALUES_BUFFER_COUNT] = {0, 0}; //used to temporary store data in menus to save on from writing to EEPROM

Encoder encoder(ENCODER_1_PIN, ENCODER_2_PIN);

// create the led object array
CRGB leds[NUM_LEDS];

// Init the DS3231 using the hardware interface
DS3231 rtc(SDA, SCL);

//display variables
unsigned long lastDisplayUpdate;
unsigned long lastDisplayEffect;
unsigned long leftEffectUpdate;
bool isDisplayingEffects;
byte effectStartIndex = 0;

void setup() {
  // init the LED object
  FastLED.addLeds<WS2812, DATA_PIN>(leds, NUM_LEDS);

  // put your setup code here, to run once:
  //Serial.begin(9600);
  pinMode(ENCODER_BTN_PIN, INPUT_PULLUP);

  rtc.begin();

  if (EEPROM.read(IS_UPLOADED) != 1) {
    EEPROM.write(IS_UPLOADED, 1);
    EEPROM.write(LED_BRIGHTNESS, 255);
    EEPROM.write(HOUR_HSV_H, 245);
    EEPROM.write(HOUR_HSV_S, 255);
    EEPROM.write(MIN_HSV_H, 230);
    EEPROM.write(MIN_HSV_S, 255);
    EEPROM.write(SEC_HSV_H, 150);
    EEPROM.write(SEC_HSV_S, 255);
    EEPROM.write(BACKGROUND_HSV_H, 50);
    EEPROM.write(BACKGROUND_HSV_S, 120);
    EEPROM.write(RAINBOW_EFFECT, 1);

  }
}

void loop() {
  long milliseconds = millis();
  Time time = rtc.getTime();

  handleInput(milliseconds);

  handleMenu(time, milliseconds);
  if (!isDisplayingEffects && time.min == 0 && time.sec == 0 && EEPROM.read(RAINBOW_EFFECT)) {
    //display rainbow effect every hour
    isDisplayingEffects = true;
    lastDisplayEffect = milliseconds;
  }

  if (isDisplayingEffects && menuState == MenuState::DISPLAY_NONE) {
    if ((milliseconds - lastDisplayEffect) > EFFECT_LAST_DELAY) {
      isDisplayingEffects = false;
    }
  } 

  if ((milliseconds - lastDisplayUpdate) > DISPLAY_UPDATE_DELAY) {

    displayClock(time, milliseconds);
    
    lastDisplayUpdate = milliseconds;
  }
}

void handleInput(unsigned long milliseconds) {
  long encoderPosition = encoder.read();
  
  if (encoderPosition > 0) {
    encoderDirection = Direction::RIGHT;
  } else if (encoderPosition < 0) {
    encoderDirection = Direction::LEFT;
  } else {
    encoderDirection = Direction::NONE;
  }

  encoder.write(0);
}

void handleMenu(Time time, unsigned long milliseconds) {
  bool reading = digitalRead(ENCODER_BTN_PIN);

  if (reading != encoderBtnState) {
    encoderBtnLastPress = milliseconds;
  } else if ((milliseconds - encoderBtnLastPress) > DEBOUNCE_DELAY) {
    if (!encoderBtnClicked && reading == LOW) {
      setEEPROMMenuState(menuState);

      menuState = nextMenuState();

      switch (menuState) {
        case MenuState::SET_BRIGHTNESS:
          menuValuesBuffer[0] = EEPROM.read(LED_BRIGHTNESS);
          if (menuValuesBuffer[0] < MIN_BRIGHTNESS) {
            menuValuesBuffer[0] = MIN_BRIGHTNESS;
          }
          break;
        case MenuState::SET_HOUR_COLOR:
          menuValuesBuffer[0] = EEPROM.read(HOUR_HSV_H);
          menuValuesBuffer[1] = EEPROM.read(HOUR_HSV_S);
          break;
        case MenuState::SET_MIN_COLOR:
          menuValuesBuffer[0] = EEPROM.read(MIN_HSV_H);
          menuValuesBuffer[1] = EEPROM.read(MIN_HSV_S);
          break;
        case MenuState::SET_SEC_COLOR:
          menuValuesBuffer[0] = EEPROM.read(SEC_HSV_H);
          menuValuesBuffer[1] = EEPROM.read(SEC_HSV_S);
          break;
        case MenuState::SET_BG_COLOR:
          menuValuesBuffer[0] = EEPROM.read(BACKGROUND_HSV_H);
          menuValuesBuffer[1] = EEPROM.read(BACKGROUND_HSV_S);
          break;
        case MenuState::SET_RAINBOW:
          menuValuesBuffer[0] = EEPROM.read(RAINBOW_EFFECT);
          break;
        default:
          menuValuesBuffer[0] = 0;
          menuValuesBuffer[1] = 0;
          break;
      }

      encoderBtnClicked = true;
    } else if (reading == HIGH) {
      encoderBtnClicked = false;
    }
  }
  encoderBtnState = reading;

  unsigned long settingChangeMilli = (milliseconds - lastMenuSettingChange);
  if (menuState != MenuState::DISPLAY_NONE && settingChangeMilli > SETTING_CHANGE_DELAY) {
    //menu timeout occured, reset to normal operation
    if (settingChangeMilli > MENU_TIMEOUT_DELAY) {
      menuState = MenuState::DISPLAY_NONE;
    }

    switch (menuState) {
      case MenuState::SET_BRIGHTNESS:
        if (handleSetBrightness()) {
          lastMenuSettingChange = milliseconds;
        }
        break;
      case MenuState::SET_HOUR:
        if (handleSetHour(time)) {
          lastMenuSettingChange = milliseconds;
        }
        break;
      case MenuState::SET_MIN:
        if (handleSetMin(time)) {
          lastMenuSettingChange = milliseconds;
        }
        break;
      case MenuState::SET_SEC:
        if (handleSetSec(time)) {
          lastMenuSettingChange = milliseconds;
        }
        break;
      case MenuState::SET_HOUR_COLOR:
      case MenuState::SET_MIN_COLOR:
      case MenuState::SET_SEC_COLOR:
      case MenuState::SET_BG_COLOR:
        if (handleSetColor()) {
          lastMenuSettingChange = milliseconds;
        }
        break;
      case MenuState::SET_RAINBOW:
        if (handleSetRainbow()) {
          lastMenuSettingChange = milliseconds;
        }
        break;
    }

    encoderDirection = Direction::NONE;
  }

}

MenuState nextMenuState() {
  switch(menuState) {
    case MenuState::DISPLAY_NONE:
      menuState = MenuState::SET_BRIGHTNESS;
      break;
    case MenuState::SET_BRIGHTNESS:
      menuState = MenuState::SET_HOUR;
      break;
    case MenuState::SET_HOUR:
      menuState = MenuState::SET_MIN;
      break;
    case MenuState::SET_MIN:
      menuState = MenuState::SET_SEC;
      break;
    case MenuState::SET_SEC:
      menuState = MenuState::SET_BG_COLOR;
      break;
    case MenuState::SET_BG_COLOR:
      menuState = MenuState::SET_HOUR_COLOR;
      break;
    case MenuState::SET_HOUR_COLOR:
      menuState = MenuState::SET_MIN_COLOR;
      break;
    case MenuState::SET_MIN_COLOR:
      menuState = MenuState::SET_SEC_COLOR;
      break;
    case MenuState::SET_SEC_COLOR:
      menuState = MenuState::SET_RAINBOW;
      break;
    
    default:
      isDisplayingEffects = false;
      menuState = MenuState::DISPLAY_NONE;
      break;
  }

  return menuState;
}

bool handleSetHour(Time t) {
  byte hour = t.hour;
  switch (encoderDirection) {
    case Direction::LEFT:
      setHour(t, --hour);
      return true;
    case Direction::RIGHT:
      setHour(t, ++hour);
      return  true;
  }
  return false;
}

bool handleSetMin(Time t) {
  byte min = t.min;
  switch (encoderDirection) {
    case Direction::LEFT:
      setMin(t, --min);
      return true;
    case Direction::RIGHT:
      setMin(t, ++min);
      return  true;
  }
  return false;
}

bool handleSetSec(Time t) {
  byte sec = t.sec;
  switch (encoderDirection) {
    case Direction::LEFT:
      setSec(t, --sec);
      return true;
    case Direction::RIGHT:
      setSec(t, ++sec);
      return  true;
  }
  return false;
}

bool handleSetBrightness() {
  switch (encoderDirection) {
    case Direction::LEFT:
      if (menuValuesBuffer[0] > MIN_BRIGHTNESS) {
        menuValuesBuffer[0] -= SETTING_STEP;
      }
      return true;
    case Direction::RIGHT:
      if (menuValuesBuffer[0] < 255) {
        menuValuesBuffer[0] += SETTING_STEP;
      }
      return true;
  }
  return false;
}

bool handleSetColor() {
  switch (encoderDirection) {
    case Direction::LEFT:
      menuValuesBuffer[0] += SETTING_STEP * 2;
      return true;
    case Direction::RIGHT:
      menuValuesBuffer[1] += SETTING_STEP * 2;
      return true;
  }
  return false;
}

bool handleSetRainbow() {
  switch (encoderDirection) {
    case Direction::LEFT:
      menuValuesBuffer[0] = 0;
      return true;
    case Direction::RIGHT:
      menuValuesBuffer[0] = 1;
      return  true;
  }
  return false;
}

void setEEPROMMenuState(MenuState menuState) {
  switch (menuState) {
    case MenuState::SET_BRIGHTNESS:
      EEPROM.update(LED_BRIGHTNESS, menuValuesBuffer[0]);
      break;
    case MenuState::SET_HOUR_COLOR:
      EEPROM.update(HOUR_HSV_H, menuValuesBuffer[0]);
      EEPROM.update(HOUR_HSV_S, menuValuesBuffer[1]);
      break;
    case MenuState::SET_MIN_COLOR:
      EEPROM.update(MIN_HSV_H, menuValuesBuffer[0]);
      EEPROM.update(MIN_HSV_S, menuValuesBuffer[1]);
      break;
    case MenuState::SET_SEC_COLOR:
      EEPROM.update(SEC_HSV_H, menuValuesBuffer[0]);
      EEPROM.update(SEC_HSV_S, menuValuesBuffer[1]);
      break;
    case MenuState::SET_BG_COLOR:
      EEPROM.update(BACKGROUND_HSV_H, menuValuesBuffer[0]);
      EEPROM.update(BACKGROUND_HSV_S, menuValuesBuffer[1]);
      break;
    case MenuState::SET_RAINBOW:
      EEPROM.update(RAINBOW_EFFECT, (menuValuesBuffer[0] == 1));
      break;
  }
}

void displayClock(Time t, unsigned long milliseconds) {
  FastLED.clear();

  CHSV hourHSV = CHSV(0, 0, 0),
       minHSV = CHSV(0, 0, 0),
       secHSV = CHSV(0, 0, 0),
       bgHSV = CHSV(0, 0, 0),
       menuSelectHSV = CHSV(100, 255, 255);

  bool brightnessReduce;
  byte brightness = EEPROM.read(LED_BRIGHTNESS);

  switch (menuState) {
    case MenuState::SET_BRIGHTNESS:
      brightness = menuValuesBuffer[0];
      bgHSV = CHSV(EEPROM.read(BACKGROUND_HSV_H), EEPROM.read(BACKGROUND_HSV_S), (brightness - 15));
      break;
    case MenuState::SET_HOUR_COLOR:
      hourHSV = CHSV(menuValuesBuffer[0], menuValuesBuffer[1], brightness);
      break;
    case MenuState::SET_MIN_COLOR:
      minHSV = CHSV(menuValuesBuffer[0], menuValuesBuffer[1], brightness);
      break;
    case MenuState::SET_SEC_COLOR:
      secHSV = CHSV(menuValuesBuffer[0], menuValuesBuffer[1], brightness);
      break;
    case MenuState::SET_BG_COLOR:
      bgHSV = CHSV(menuValuesBuffer[0], menuValuesBuffer[1], (brightness - 15));
      break;
    case MenuState::SET_RAINBOW:
      if (menuValuesBuffer[0] == 1) {
        isDisplayingEffects = true;
      } else {
        isDisplayingEffects = false;
      }
      break;

    case MenuState::DISPLAY_NONE:
    default:
      if (t.hour >= NIGHT_BEGIN || t.hour <= NIGHT_END) {
        brightness /= 2;
        //require minimium brightness
        if (brightness < MIN_BRIGHTNESS) {
          brightness = MIN_BRIGHTNESS;
        }
      }

      hourHSV = CHSV(EEPROM.read(HOUR_HSV_H), EEPROM.read(HOUR_HSV_S), brightness);
      minHSV = CHSV(EEPROM.read(MIN_HSV_H), EEPROM.read(MIN_HSV_S), brightness);
      secHSV = CHSV(EEPROM.read(SEC_HSV_H), EEPROM.read(SEC_HSV_S), brightness);
      bgHSV = CHSV(EEPROM.read(BACKGROUND_HSV_H), EEPROM.read(BACKGROUND_HSV_S), (brightness - 25));
      break;
  }

  if (isDisplayingEffects) {
    handleEffects(milliseconds, brightness);
  } else {
    //Finally, set the clock leds
    for (int i = 0; i < NUM_LEDS; i++) {
      leds[i] = bgHSV;
    }
 
    byte hour = (t.hour >= 12) ? t.hour - 12 : t.hour;
    hour = (hour * 5) - (t.min / 12);
    leds[hour] = hourHSV;
    leds[t.min] = minHSV;
    leds[t.sec] = secHSV;

    //display the menuSelect
    if (menuState != MenuState::DISPLAY_NONE) {
      if ((milliseconds - lastMenuSelectBlink) > MENU_SELECT_BLINK_RATE) {
        menuSelectBlink = !menuSelectBlink;
        lastMenuSelectBlink = milliseconds;
      }
      if (menuSelectBlink) {
        leds[menuState] = menuSelectHSV;
      }
    } else {
      menuSelectBlink = false;
    }
  }

  FastLED.show();
}

void handleEffects(unsigned long milliseconds, byte brightness) {
    if ((milliseconds - leftEffectUpdate) > EFFECT_UPDATE_DELAY) {
      effectStartIndex += 3;
      leftEffectUpdate = milliseconds;
    }

    byte index = effectStartIndex;
    for (int i = 0; i < NUM_LEDS; i++) {
        leds[i] = ColorFromPalette(RainbowColors_p, index, brightness, LINEARBLEND);
        index += 3;
    }
}

void setHour(Time t, byte hour) {
  if (hour > 23) {
    hour = 0;
  }
  rtc.setTime(hour, t.min, t.sec);
}

void setMin(Time t, byte minute) {
  if (minute > 59) {
    minute = 0;
  }
  rtc.setTime(t.hour, minute, t.sec);
}

void setSec(Time t, byte second) {
  if (second > 59) {
    second = 0;
  }
  rtc.setTime(t.hour, t.min, second);
}
