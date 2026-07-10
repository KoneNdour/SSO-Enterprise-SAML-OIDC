\# Configuration MFA TOTP — Keycloak



\## Realm : sso



\## Politique de mots de passe

\- Minimum Length : 12

\- Uppercase Characters : 1

\- Digits : 1

\- Special Characters : 1

\- Not Username : activé

\- Password Blacklist : activé

\- Expire Password : 90 jours



\## Brute Force Protection

\- Mode : Lockout temporarily

\- Max Login Failures : 5

\- Wait Increment : 15s

\- Max Wait : 900s

\- Failure Reset Time : 900s



\## MFA TOTP

\- OTP Type : TOTP (Time-based)

\- Algorithm : SHA1

\- Digits : 6

\- Period : 30 secondes

\- Flow : browser-mfa (lié au realm)

\- Applications supportées : Google Authenticator, FreeOTP

