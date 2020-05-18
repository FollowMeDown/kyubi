<p align="center"><img width="250" src="https://user-images.githubusercontent.com/2095991/74606706-f9a64880-50d2-11ea-9808-3c44dc956612.png" alt="kyubi logo"></p>

# Kyubi

**Kyubi** is a HOTP validation server (HMAC-based One-Time Password) compatible with the YubiKey. It was created for an educational purpose, to learn more about how YubiKey OTP are working.

## A quick tour

A Yubico OTP is a 44-character, one use, secure, 128-bit encrypted Public ID and Password, nearly impossible to spoof. The OTP is comprised of two majorparts: the first 12 characters remain constant and represent the Public ID of the YubiKey device itself.  The remaining 32 characters make up a unique passcode for each OTP generated

The passcode is generated from a multitude of random sources, including counters for both YubiKey sessions and OTPs generated.  When a Yubico OTP is verified, the session and OTP counter values are compared to last values submitted. If the counters are less than the previously used values the OTP is rejected.  Copying an OTP will not allow another user to spoof a YubiKey. The counter value will allow the validation server to know which OTPs have already been used

Several other implementations are available. The official one is **yubikey-val** which stands for Yubikey Validation Server, written in PHP for use behind web servers such as Nginx. This server is supposed to talk to a KSM service for decrypting the OTPs, to avoid storing any AES keys on the validation server. Our project **Kyubi** handles both the validation and the KSM. Although functional, it does not manage all the security features of a traditional KSM, as this project is mainly a study project. 

<p align="center"><img src="https://user-images.githubusercontent.com/2095991/74607098-fbbdd680-50d5-11ea-8653-8be71547d684.png" alt="otp details"></p>

## Usage

1. Build the project with `go build`

2. Use the Yubikey Personalization Tool to generate credentials on an available slot and note the *Public Identity* key and the *Secret AES Key*. We'll need them at step 4.

3. Create a new keyring of YubiKeys with `./kyubi new -name mynewkeyring`. It will then output the API key generated for the keyring and the keyring ID. A keyring can handle as many YubiKey as you want.

4. Add you YubiKey to a keyring with `./kyubi add -id 1 -public vvcnducdrkbb -secret e091bd9af8ebe1d3d6628aee07cd68bf` 

5. Run the validation server with `./kyubi run`, which is then available on http://127.0.0.1:4242/wsapi/2.0/verify?otp=myYubiKeyOtp&id=1&nonce=randomStuff.

## API

Once the server is started, the API is made available to validate given OTP. All valid submitted OTP are consumed by the API and the YubiKey counter is incremented in the SQLite database, allowing us to protect against replay attacks. The Kyubi API is available under the route `/wsapi/2.0/verify` and take three parameters.

* `id` which is the keyring ID handling your yubikeys
* `otp` which is the OTP generated by the Yubikey
* `nonce` a random nonce

Each response has a timestamp and HMAC-SHA1 signed using the API key generated at the creation of the keyring. The API also returns a status message allowing us to know the result of the validation.

* `MISSING_PARAMETER`  The request lacks a parameter.
* `NO_SUCH_CLIENT` The request id does not exist.
* `BACKEND_ERROR` Unexpected error in our server.
* `REPLAYED_OTP` The OTP has already been seen by the service.
* `BAD_OTP` The OTP has an invalid format.
* `OK` The OTP is valid.

## Why ModHex ?

**ModHex** stands for MODified HEXadecimal. It is an encoding scheme used by YubiKeys to handle different keyboard layouts. Here is the answer from the hardware engineering team at Yubico.

*We've been asked quite a few times now regarding the modhex scheme used by the Yubikey. Why don't you use for example base64 instead as this would allow six bits to be sent for each keystroke instead of just four for the modhex scheme.*

*The answer is keyboard layouts. Most countries have different keyboard layouts and what most people don't think about it that the keyboard mapping is not known by the keyboard itself. The keyboard is luckily unaware of what text is printed on top of the keys. It simply just sends the key number or "scan code" and then the computer translates it depending on your keyboard settings.*

*After quite some investigation we found out that there are a few keys that are mapped to the same scan code for almost all national settings. There we have the modhex codes.*


## Useful resources

* https://tools.ietf.org/html/rfc4226
* https://developers.yubico.com/OTP/
