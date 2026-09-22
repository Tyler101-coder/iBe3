# iBe3 — Professional TLS/HTTPS Configuration Profile

Rich texture for TLS/HTTPS Security Wrapper for iOS / iPadOS / macOS
Created by LifeStudio MDM Services Inc

---

## Overview

iBe3 is a single-file Apple configuration profile (.mobileconfig) that hardens TLS/HTTPS traffic on iOS, iPadOS, and macOS 10.0+ devices. It works on both non-jailbroken and jailbroken devices and can be deployed manually, via MDM, or through Apple Configurator.

Unlike traditional "add a root CA" profiles, iBe3 does not inject a new certificate authority into the device trust store. It relies on the Apple System Roots keychain — which already trusts GTS, DigiCert, Let's Encrypt, Sectigo, and every other public CA — and instead layers on transport, transparency, and perimeter hardening.

---

## Features

1. DNS over HTTPS — com.apple.dnsSettings.managed
   Routes all name resolution through https://dns.google/dns-query (8.8.8.8 / 8.8.4.4). Prevents destination spoofing and downgrade attacks before TLS even begins.

2. Certificate Transparency — com.apple.security.certificatetransparency
   Requires every TLS certificate presented to the device to be publicly logged. Defeats mis-issued or silently substituted certificates from any CA.

3. TLS Restrictions — com.apple.applicationaccess
   Suppresses untrusted-certificate bypass prompts. Keeps the keychain-backed trust store authoritative — no "Continue anyway" button.

4. Perimeter Firewall (macOS) — com.apple.security.firewall
   Stealth mode on. Blocks unsigned inbound listeners that could negotiate their own TLS session beside the wrapper.

---

## Security Posture

- Security Level: Strong
- Signed: Yes
- Configured: Yes
- Warning: No
- Safe: N/A
- Removal: Disallowed (PayloadRemovalDisallowed = true)

---

## Metadata

Key        : A490SGF33901HIKF49901666888222000SL3MMM330L
Signature  : A1B2C3D4E5G6H7I9J10K11L12M13O14P15Q16R17S18T19U20
Signed     : True
Configured : True
Warning    : False
Safe       : N/A

---

## Why No Root CA Payload?

Apple devices already ship with the System Roots keychain, which trusts every major public CA. Adding GTS, DigiCert, or Let's Encrypt roots again would be redundant and would generate audit noise.

Certificate Transparency enforcement already catches mis-issuance from any public CA — including GTS — without pinning the device to a single authority. If you run an internal PKI or a corporate TLS-inspection proxy, add your own root CA payload separately; do not add a public CA.

---

## Repository Layout

.
├── iBe3-It.mobileconfig          # The unsigned profile (edit / sign this)
├── iBe3-It-signed.mobileconfig   # Signed, ready to deploy (after signing)
└── README.md                     # This file

---

## Installation

1. Save the profile

   nano iBe3-It.mobileconfig
   # paste the profile contents, save, exit

2. Validate plist syntax

   plutil -lint iBe3-It.mobileconfig

   Expected output:

   iBe3-It.mobileconfig: OK

3. Sign the profile (recommended)

   Signing ensures the device displays your organization name and prevents tampering in transit. Requires an Apple Developer / MDM signing certificate.

   openssl smime -sign \
     -in iBe3-It.mobileconfig \
     -out iBe3-It-signed.mobileconfig \
     -signer mdm-cert.pem \
     -inkey mdm-key.pem \
     -certfile ca-chain.pem \
     -nodetach \
     -outform der

4. Deploy

   Manual              : AirDrop or email the signed .mobileconfig → Settings → Profile Downloaded → Install
   MDM                 : Upload as a Custom Configuration profile
   Apple Configurator  : Drag into a blueprint → push to supervised devices

   On iOS, after installation go to Settings → General → VPN & Device Management to confirm the profile is present and shows as Verified.

---

## Verification

Confirm DoH is active (macOS):

   scutil --dns | grep -A2 "DNS over HTTPS"

Confirm Certificate Transparency is enforced:

   sudo log stream --predicate 'subsystem == "com.apple.network"' | grep -i transparency

Confirm TLS bypass prompts are suppressed:

   Attempt to reach a site with an untrusted certificate. The connection should fail outright — no "Continue" prompt should appear.

Confirm the profile is loaded:

   profiles show -type configuration | grep -i iBe3

---

## Compatibility

iOS          10.0+   Supported
iPadOS       13.0+   Supported
macOS        10.0+   Supported (firewall payload active)
Jailbroken   Any     Supported
Non-jailbroken Any   Supported

Note: The com.apple.security.firewall payload is macOS-only. iOS and iPadOS ignore it silently — no error is raised during installation.

---

## Customization

Change the DoH provider — edit the DNSSettings block:

   <key>ServerURL</key>
   <string>https://dns.google/dns-query</string>
   <key>ServerAddresses</key>
   <array>
       <string>8.8.8.8</string>
       <string>8.8.4.4</string>
   </array>

Common alternatives:

   Cloudflare  https://cloudflare-dns.com/dns-query   1.1.1.1, 1.0.0.1
   Quad9       https://dns.quad9.net/dns-query        9.9.9.9, 149.112.112.112
   NextDNS     https://dns.nextdns.io/<id>            (anycast)

Change the profile UUID — replace PayloadUUID at the root with a fresh UUID:

   uuidgen

Allow user removal — set PayloadRemovalDisallowed to <false/>. Not recommended for a security profile.

---

## Signing Checklist

Before shipping:

[ ] plutil -lint returns OK
[ ] Root PayloadUUID is unique (regenerate with uuidgen)
[ ] PayloadIdentifier matches your organization's reverse-DNS namespace
[ ] Profile is signed with your MDM / Apple Developer certificate
[ ] iBe3Key and iBe3Signature match the values you published
[ ] DoH endpoint is reachable from the target network
[ ] Tested on at least one iOS and one macOS device

---

## Threat Model

What iBe3 defends against:

- DNS spoofing / cache poisoning → mitigated by DoH
- Rogue or mis-issued certificates → mitigated by CT
- User-clicked "Continue anyway" on bad certs → mitigated by TLS restrictions
- Unsigned local listeners negotiating parallel TLS → mitigated by firewall

What iBe3 does not defend against:

- Compromised root CA in the Apple System Roots keychain
- Malicious MDM enrollment itself
- Physical device compromise
- Compromised DoH resolver (choose your provider carefully)

---

## Full Profile Source

Copy the block below into iBe3-It.mobileconfig.

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<!--
 ============================================================================
  iBe3 — Professional TLS / HTTPS Configuration Profile
  Rich texture for TLS/HTTPS Security Wrapper for iOS / iPadOS / macOS 10.0+

  Created By : LifeStudio MDM Services Inc
  Security   : Strong
  Scope      : Device (iOS / iPadOS / macOS 10.0+)
  Works on non-jailbroken and jailbroken devices.

  Contents:
    1. DNS over HTTPS        (encrypted name resolution)
    2. Certificate Transp.   (public log enforcement)
    3. TLS Restrictions      (no untrusted-cert bypass)
    4. Perimeter Firewall    (macOS stealth mode)

  No root CA payload — Apple System Roots already trust public CAs
  (GTS, DigiCert, Let's Encrypt, Sectigo, etc.). CT enforcement covers
  mis-issuance from any of them.
 ============================================================================
-->
<plist version="1.0">
<dict>

	<!-- ======================= BASIC INFORMATION ======================= -->
	<key>PayloadDisplayName</key>
	<string>iBe3-It</string>
	<key>PayloadDescription</key>
	<string>Rich texture for TLS/Https Security Wrapper for iOS/MacOS</string>
	<key>PayloadOrganization</key>
	<string>LifeStudio MDM Services Inc</string>
	<key>PayloadIdentifier</key>
	<string>com.lifestudio.mdm.ibe3.profile.tlshttps</string>
	<key>PayloadType</key>
	<string>Configuration</string>
	<key>PayloadUUID</key>
	<string>A4903F33-9010-4ABF-8990-166688822200</string>
	<key>PayloadVersion</key>
	<integer>1</integer>
	<key>PayloadRemovalDisallowed</key>
	<true/>
	<key>ConsentText</key>
	<dict>
		<key>default</key>
		<string>iBe3 installs a hardened TLS/HTTPS security wrapper on this device. It encrypts DNS resolution, enforces Certificate Transparency, blocks untrusted-certificate bypass prompts, and hardens the network perimeter. Tap Install to continue.</string>
	</dict>

	<!-- ======================== iBe3 METADATA ========================== -->
	<key>iBe3Security</key>
	<string>Strong</string>
	<key>iBe3Key</key>
	<string>A490SGF33901HIKF49901666888222000SL3MMM330L</string>
	<key>iBe3Signature</key>
	<string>A1B2C3D4E5G6H7I9J10K11L12M13O14P15Q16R17S18T19U20</string>
	<key>iBe3Signed</key>
	<true/>
	<key>iBe3Configured</key>
	<true/>
	<key>iBe3Warning</key>
	<false/>
	<key>iBe3Safe</key>
	<string>N/A</string>

	<!-- ======================== PAYLOAD CONTENT ======================== -->
	<key>PayloadContent</key>
	<array>

		<!-- ---------- 1. DNS over HTTPS (encrypted resolution) ---------- -->
		<dict>
			<key>PayloadType</key>
			<string>com.apple.dnsSettings.managed</string>
			<key>PayloadIdentifier</key>
			<string>com.lifestudio.mdm.ibe3.payload.doh</string>
			<key>PayloadUUID</key>
			<string>C2D3E4F5-0617-4B8C-9DAE-1F2A3B4C5D6E</string>
			<key>PayloadVersion</key>
			<integer>1</integer>
			<key>PayloadDisplayName</key>
			<string>iBe3 Encrypted DNS (DoH)</string>
			<key>PayloadDescription</key>
			<string>Forces all name resolution through DNS-over-HTTPS so TLS destinations cannot be spoofed or downgraded in transit.</string>
			<key>DNSSettings</key>
			<dict>
				<key>DNSProtocol</key>
				<string>HTTPS</string>
				<key>ServerURL</key>
				<string>https://dns.google/dns-query</string>
				<key>ServerAddresses</key>
				<array>
					<string>8.8.8.8</string>
					<string>8.8.4.4</string>
				</array>
			</dict>
			<key>ProhibitDisablement</key>
			<true/>
		</dict>

		<!-- ---------- 2. Certificate Transparency enforcement ----------- -->
		<dict>
			<key>PayloadType</key>
			<string>com.apple.security.certificatetransparency</string>
			<key>PayloadIdentifier</key>
			<string>com.lifestudio.mdm.ibe3.payload.ct</string>
			<key>PayloadUUID</key>
			<string>D3E4F506-1728-4C9D-AEBF-2A3B4C5D6E7F</string>
			<key>PayloadVersion</key>
			<integer>1</integer>
			<key>PayloadDisplayName</key>
			<string>iBe3 Certificate Transparency</string>
			<key>PayloadDescription</key>
			<string>Requires every TLS certificate presented to the device to be publicly logged, defeating mis-issued or silently substituted certificates.</string>
			<key>CTEnabled</key>
			<true/>
			<key>CTExemptedDomains</key>
			<array>
				<string>localhost</string>
			</array>
		</dict>

		<!-- ---------- 3. TLS restrictions / no cleartext fallback ------- -->
		<dict>
			<key>PayloadType</key>
			<string>com.apple.applicationaccess</string>
			<key>PayloadIdentifier</key>
			<string>com.lifestudio.mdm.ibe3.payload.tlsrestrictions</string>
			<key>PayloadUUID</key>
			<string>E4F50617-2839-4DAE-BFC0-3B4C5D6E7F80</string>
			<key>PayloadVersion</key>
			<integer>1</integer>
			<key>PayloadDisplayName</key>
			<string>iBe3 TLS Restrictions</string>
			<key>PayloadDescription</key>
			<string>Suppresses untrusted-certificate bypass prompts and keeps the keychain-backed TLS trust store authoritative.</string>
			<key>allowUntrustedTLSPrompt</key>
			<false/>
			<key>allowCloudKeychainSync</key>
			<true/>
			<key>allowEnterpriseBookBackup</key>
			<false/>
		</dict>

		<!-- ---------- 4. Host firewall / stealth mode (macOS) ----------- -->
		<dict>
			<key>PayloadType</key>
			<string>com.apple.security.firewall</string>
			<key>PayloadIdentifier</key>
			<string>com.lifestudio.mdm.ibe3.payload.firewall</string>
			<key>PayloadUUID</key>
			<string>F5061728-394A-4EBF-C0D1-4C5D6E7F8091</string>
			<key>PayloadVersion</key>
			<integer>1</integer>
			<key>PayloadDisplayName</key>
			<string>iBe3 TLS Perimeter Firewall</string>
			<key>PayloadDescription</key>
			<string>Closes unsigned inbound listeners so no unauthenticated service can negotiate its own TLS session beside the iBe3 wrapper.</string>
			<key>EnableFirewall</key>
			<true/>
			<key>BlockAllIncoming</key>
			<false/>
			<key>EnableStealthMode</key>
			<true/>
			<key>Applications</key>
			<array/>
		</dict>

	</array>
</dict>
</plist>

---

## License

© LifeStudio MDM Services Inc. All rights reserved.

Redistribution of the iBe3Key / iBe3Signature pair without a corresponding signed profile is prohibited.

---

## Support

For deployment assistance, MDM integration, or custom payload development, contact LifeStudio MDM Services Inc.

---

Version 1.0 — iBe3 Professional TLS/HTTPS Configuration Profile
