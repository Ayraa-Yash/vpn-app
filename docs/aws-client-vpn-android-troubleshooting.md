# AWS Client VPN on Android: "Peer certificate verification failure"

Summary of a troubleshooting session (2026-10-07) for connecting to our AWS Client VPN
endpoint (`cvpn-endpoint-0940242fcc0df9146`, us-east-1) from Android using a
certificate-based (`<cert>` / `<key>`) `.ovpn` profile.

## 1. Running AWS Client VPN `.ovpn` profiles

First check which authentication the profile uses:

| In the `.ovpn` | Auth type | Android? |
|---|---|---|
| `auth-federate` | SAML / SSO | ❌ Only the AWS VPN Client supports it (no Android app) |
| `<cert>` + `<key>` blocks | Mutual certificate auth | ✅ |
| `auth-user-pass` | Active Directory | ✅ |

- **Desktop (Windows / macOS / Ubuntu):** AWS VPN Client
  (https://aws.amazon.com/vpn/client-vpn-download/) → File → Manage Profiles → Add Profile.
- **Linux CLI:** `sudo openvpn --config file.ovpn` (cert / AD auth only).
- **Android:** AWS has no Android client. Use **OpenVPN Connect** or
  **OpenVPN for Android** (Arne Schwabe) → Import profile → File.

> The `.ovpn` contains a private key. Treat it like a password.

## 2. Symptom

OpenVPN Connect (Android, core 3.11.1) failed immediately after reaching the server:

```
Transport Error: ... error:0A000086:SSL routines::certificate verify failed
EVENT: CERT_VERIFY_FAIL
```

The phone reached the endpoint, so network and DNS were fine. The **client was rejecting the
server's certificate**. The same profile worked on the desktop AWS VPN Client.

## 3. Diagnosis

```bash
# Server certificate used by the endpoint
ARN=$(aws ec2 describe-client-vpn-endpoints \
  --client-vpn-endpoint-ids cvpn-endpoint-0940242fcc0df9146 --region us-east-1 \
  --query 'ClientVpnEndpoints[0].ServerCertificateArn' --output text)
aws acm get-certificate --certificate-arn "$ARN" --region us-east-1 \
  --query Certificate --output text > server.pem
openssl x509 -in server.pem -noout -issuer -dates
# issuer=CN = visheshsachdev.domain.tld, valid Dec 8 2025 to Dec 7 2028

# Compare every local profile's <ca> against the server cert
for f in ~/Downloads/*.ovpn; do
  echo "=== $f"
  sed -n '/<ca>/,/<\/ca>/p' "$f" | sed '1d;$d' > /tmp/ca.pem
  openssl crl2pkcs7 -nocrl -certfile /tmp/ca.pem | openssl pkcs7 -print_certs -noout | grep subject
  openssl verify -CAfile /tmp/ca.pem server.pem
done
```

Findings:

- `ayraaprod-yashpurohit.ovpn` has the wrong CA (`CN = Easy-RSA CA`). Don't use it as is.
- `yash-ayraa-vpn-2025.ovpn` and `yash_xdge_2025_vpn.ovpn` have the right CA
  (`CN = visheshsachdev.domain.tld`) and pass the basic `openssl verify`.

`yash-ayraa-vpn-2025.ovpn` (the profile used on the phone) still failed on Android, so we ran a
stricter check, like the one OpenVPN 3 does:

```bash
openssl verify -auth_level 2 -purpose sslserver -CAfile /tmp/ca.pem server.pem
# error 26 at 1 depth lookup: unsuitable certificate purpose

openssl x509 -in /tmp/ca.pem -noout -text | grep -A1 -E "Basic Constraints|Key Usage"
#   X509v3 Basic Constraints: critical  CA:TRUE
#   X509v3 Key Usage: critical          Digital Signature, Certificate Sign, CRL Sign
#   X509v3 Extended Key Usage:          TLS Web Client Authentication   <-- problem
```

## 4. Root cause

The CA certificate `CN = visheshsachdev.domain.tld` has an **Extended Key Usage of only
`TLS Web Client Authentication`**. That restricts the CA to vouching for *client* certificates.
OpenVPN 3 (OpenVPN Connect on Android and iOS) enforces this and rejects the server certificate
it signed. The desktop AWS VPN Client (OpenVPN 2 based) doesn't check this, which is why laptops
work. Changing the app's Security Level or re-importing the file does not fix it.

## 5. Fix

### Preferred: reissue only the CA certificate with the same key

Needs the existing CA private key (`ca.key`). No changes in AWS: the server certificate in ACM
stays as it is.

```bash
openssl x509 -in ca.crt -x509toreq -signkey ca.key -out ca.csr
printf "basicConstraints=critical,CA:TRUE\nkeyUsage=critical,digitalSignature,keyCertSign,cRLSign\nsubjectKeyIdentifier=hash\n" > ca-ext.cnf
openssl x509 -req -in ca.csr -signkey ca.key -days 3650 -sha256 -extfile ca-ext.cnf -out ca-new.crt

# Must print "server.pem: OK"
openssl verify -auth_level 2 -purpose sslserver -CAfile ca-new.crt server.pem
```

Then replace the `<ca>…</ca>` contents in the mobile `.ovpn` profiles with `ca-new.crt`. Existing
desktop profiles keep working. Leave the endpoint's client root CA setting as it is.

### Fallback: if the CA key is lost, build a new CA and server cert

```bash
./easyrsa init-pki
./easyrsa build-ca nopass
./easyrsa build-server-full server nopass

aws acm import-certificate --region us-east-1 \
  --certificate fileb://pki/issued/server.crt \
  --private-key fileb://pki/private/server.key \
  --certificate-chain fileb://pki/ca.crt

aws ec2 modify-client-vpn-endpoint --region us-east-1 \
  --client-vpn-endpoint-id cvpn-endpoint-0940242fcc0df9146 \
  --server-certificate-arn <NEW_ARN>
```

Every user's `.ovpn` `<ca>` must then be replaced with the new `pki/ca.crt`. Existing profiles,
including desktop ones, stop connecting until they are updated. Client certificates don't change
as long as the client root CA setting is left alone.

### Possible workaround meanwhile

Try **OpenVPN for Android** (Arne Schwabe) with the OpenVPN 2.x engine, which may not enforce
this check. Not guaranteed.

## 6. Other tips

- Run shell commands one at a time. A trailing `\` joins lines into one broken command.
- Set the phone's date and time to automatic. A wrong clock also causes certificate verify errors.
- `reneg-sec` / `resolv-retry` "unused option" notes in the Connect log are harmless.
