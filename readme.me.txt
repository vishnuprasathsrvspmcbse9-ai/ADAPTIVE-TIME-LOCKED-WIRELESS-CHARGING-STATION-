#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Servo.h>
#include <Keypad.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);
Servo doorServo;

// ---------------- KEYPAD ----------------

const byte ROWS = 4;
const byte COLS = 4;

char keys[ROWS][COLS] = {
  {'1', '2', '3', 'A'},
  {'4', '5', '6', 'B'},
  {'7', '8', '9', 'C'},
  {'*', '0', '#', 'D'}
};

byte rowPins[ROWS] = {2, 3, 4, 5};
byte colPins[COLS] = {6, 7, 8, 9};

Keypad keypad = Keypad(
  makeKeymap(keys), rowPins, colPins, ROWS, COLS
);

// ---------------- COMPONENT PINS ----------------

const byte SERVO_PIN = 10;
const byte RELAY_PIN = 11;
const byte LED_PIN = 12;

// SERVO ANGLES
const int DOOR_CLOSED = 0;
const int DOOR_OPEN = 90;

// ---------------- VARIABLES ----------------

String savedPassword = "";

unsigned long selectedSeconds = 0;
unsigned long remainingSeconds = 0;

bool emergencyExit = false;

// ---------------- KEYPAD INPUT ----------------

char waitForKey() {
  char key = NO_KEY;

  while (key == NO_KEY) {
    key = keypad.getKey();
  }

  return key;
}

// ---------------- PASSWORD ----------------

String getFourDigitPassword() {
  String pass = "";

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("ENTER PASSWORD");
  lcd.setCursor(0, 1);

  while (pass.length() < 4) {
    char key = waitForKey();

    if (key >= '0' && key <= '9') {
      pass += key;
      lcd.print('*');
    }
  }

  return pass;
}

// ---------------- DOOR ----------------

void openDoor() {
  doorServo.write(DOOR_OPEN);  // Open to 90 degrees
  delay(700);
}

void closeDoor() {
  doorServo.write(DOOR_CLOSED); // Close to 0 degrees
  delay(700);
}

// ---------------- RELAY ----------------

void relayON() {
  digitalWrite(RELAY_PIN, HIGH);
  digitalWrite(LED_PIN, HIGH);
}

void relayOFF() {
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(LED_PIN, LOW);
}

// ---------------- START SCREEN ----------------

void showStartScreen() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("CHARGING");
  lcd.setCursor(0, 1);
  lcd.print("STATION");

  delay(3000);

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("PRESS A TO START");

  while (waitForKey() != 'A') {
    // Wait for A
  }
}

// ---------------- CREATE PASSWORD ----------------

void createPassword() {
  savedPassword = getFourDigitPassword();

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("PASSWORD SAVED");

  delay(1000);

  openDoor();
}

// ---------------- ENTER CHARGING TIME ----------------

void enterChargingTime() {

  while (true) {

    String input = "";

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("ENTER TIME 1-999");
    lcd.setCursor(0, 1);
    lcd.print("D TO CONTINUE");

    // STEP 1: Enter the number first
    while (true) {

      char key = waitForKey();

      if (key >= '0' && key <= '9') {

        if (input.length() < 3) {
          input += key;
        }

        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("TIME: ");
        lcd.print(input);

        lcd.setCursor(0, 1);
        lcd.print("D TO CONTINUE");
      }

      // Press D to confirm the entered number
      else if (key == 'D') {

        long value = input.toInt();

        if (value < 1 || value > 999) {

          lcd.clear();
          lcd.setCursor(0, 0);
          lcd.print("INVALID TIME");
          lcd.setCursor(0, 1);
          lcd.print("ENTER 1-999");

          delay(1200);
          break;
        }

        // STEP 2: Choose minutes or hours
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("TIME: ");
        lcd.print(value);

        lcd.setCursor(0, 1);
        lcd.print("*=MIN #=HOUR");

        while (true) {

          char unitKey = waitForKey();

          if (unitKey == '*') {
            selectedSeconds = (unsigned long)value * 60UL;
            break;
          }

          else if (unitKey == '#') {
            selectedSeconds = (unsigned long)value * 3600UL;
            break;
          }
        }

        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("TIME SAVED");

        delay(1000);
        return;
      }
    }
  }
}

// ---------------- CLOSE DOOR ----------------

void waitForClose() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("PRESS C TO");
  lcd.setCursor(0, 1);
  lcd.print("CLOSE DOOR");

  while (waitForKey() != 'C') {
    // Wait for C
  }

  closeDoor();
}

// ---------------- COUNTDOWN DISPLAY ----------------

void displayCountdown() {
  unsigned long hours = remainingSeconds / 3600UL;
  unsigned long minutes = (remainingSeconds % 3600UL) / 60UL;
  unsigned long seconds = remainingSeconds % 60UL;

  lcd.setCursor(0, 0);
  lcd.print("CHARGING ON     ");

  lcd.setCursor(0, 1);

  if (hours < 10) lcd.print('0');
  lcd.print(hours);
  lcd.print(':');

  if (minutes < 10) lcd.print('0');
  lcd.print(minutes);
  lcd.print(':');

  if (seconds < 10) lcd.print('0');
  lcd.print(seconds);

  lcd.print("   ");
}

// ---------------- EMERGENCY ACCESS ----------------

bool emergencyAccess() {
  relayOFF();

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("TIMER PAUSED");

  String pass = getFourDigitPassword();

  if (pass == savedPassword) {

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("PASSWORD OK");

    delay(1000);

    openDoor();
    return true;
  }

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("WRONG PASSWORD");
  lcd.setCursor(0, 1);
  lcd.print("RESUMING");

  delay(1000);

  relayON();
  return false;
}

// ---------------- CHARGING PROCESS ----------------

void chargingProcess() {
  emergencyExit = false;
  remainingSeconds = selectedSeconds;

  unsigned long lastTick = millis();

  relayON();

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("CHARGING ON");

  delay(500);

  while (remainingSeconds > 0) {

    displayCountdown();

    char key = keypad.getKey();

    // Emergency access during charging
    if (key == 'A') {

      if (emergencyAccess()) {

        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("REMOVE PHONE");
        lcd.setCursor(0, 1);
        lcd.print("PRESS C TO CLOSE");

        while (waitForKey() != 'C') {
          // Wait for C
        }

        closeDoor();
        relayOFF();

        emergencyExit = true;
        remainingSeconds = 0;
        selectedSeconds = 0;
        savedPassword = "";

        return;
      }

      lastTick = millis();
    }

    unsigned long now = millis();

    if (now - lastTick >= 1000UL) {

      unsigned long elapsed = (now - lastTick) / 1000UL;

      if (elapsed >= remainingSeconds) {
        remainingSeconds = 0;
      } else {
        remainingSeconds -= elapsed;
      }

      lastTick += elapsed * 1000UL;
    }
  }

  // Charging completed
  relayOFF();

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("CHARGING DONE");
  lcd.setCursor(0, 1);
  lcd.print("PRESS A");

  while (waitForKey() != 'A') {
    // Wait for A
  }
}

// ---------------- AFTER CHARGING ----------------

void afterCharge() {

  while (true) {

    String pass = getFourDigitPassword();

    if (pass == savedPassword) {

      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("PASSWORD OK");

      delay(700);

      openDoor();

      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("REMOVE PHONE");
      lcd.setCursor(0, 1);
      lcd.print("PRESS C TO CLOSE");

      while (waitForKey() != 'C') {
        // Wait for C
      }

      closeDoor();
      return;
    }

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("WRONG PASSWORD");
    lcd.setCursor(0, 1);
    lcd.print("TRY AGAIN");

    delay(1200);
  }
}

// ---------------- SETUP ----------------

void setup() {
  Wire.begin();

  lcd.init();
  lcd.backlight();

  doorServo.attach(SERVO_PIN);
  doorServo.write(DOOR_CLOSED);

  pinMode(RELAY_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);

  relayOFF();
}

// ---------------- MAIN LOOP ----------------

void loop() {

  showStartScreen();

  createPassword();

  enterChargingTime();

  waitForClose();

  chargingProcess();

  if (!emergencyExit) {
    afterCharge();
  }

  relayOFF();
  doorServo.write(DOOR_CLOSED);

  savedPassword = "";
  selectedSeconds = 0;
  remainingSeconds = 0;
  emergencyExit = false;

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("SYSTEM RESET");

  delay(1500);
}