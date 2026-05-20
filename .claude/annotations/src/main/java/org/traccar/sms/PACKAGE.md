# `sms/` — SMS sending

Three files for outbound SMS delivery used by `NotificatorSms` and for text-channel device commands.

## File index

| File | One-liner |
|---|---|
| `SmsManager.java` | Interface: `sendMessage(String phone, String message, boolean command)` |
| `HttpSmsClient.java` | HTTP POST to a generic SMS gateway API (configurable URL + auth headers) |
| `SnsSmsClient.java` | AWS SNS SMS via the AWS SDK v2 |

## Dependency graph

```
NotificatorSms (notificators/) → SmsManager
CommandsManager (database/) → SmsManager (text-channel commands)
StatisticsManager.registerSms() called by each implementation
```

## Configuration

```xml
<!-- Generic HTTP gateway (MSG91, Twilio, Exotel, etc.) -->
<entry key="sms.http.url">https://api.msg91.com/api/sendhttp.php?authkey=KEY&amp;mobiles={phone}&amp;message={message}&amp;route=4&amp;sender=TRACCAR</entry>
<entry key="sms.http.authorization">Bearer TOKEN</entry>

<!-- AWS SNS -->
<entry key="sms.aws.region">ap-south-1</entry>
<entry key="sms.aws.access">ACCESS_KEY</entry>
<entry key="sms.aws.secret">SECRET_KEY</entry>
```

URL template tokens in `HttpSmsClient`: `{phone}` and `{message}` (URL-encoded).

## Transport OS / AIS-140 note

AIS-140 certification requires SMS delivery for SOS panic alerts to 5 pre-configured numbers. Configure `HttpSmsClient` with an Indian SMS gateway:
- **MSG91** (India): `https://api.msg91.com/api/sendhttp.php?...`
- **Exotel** (India): `https://api.exotel.com/v1/Accounts/SID/Sms/send`
- **Twilio** (global): `https://api.twilio.com/2010-04-01/Accounts/SID/Messages.json`

The `command=true` parameter in `sendMessage` distinguishes command SMS from notification SMS — some gateways charge differently for transactional vs. promotional traffic.
