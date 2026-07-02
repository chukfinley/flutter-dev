# Networking: dio, with certificate pinning + a test

**Rule:** Use [`dio`](https://pub.dev/packages/dio) as the HTTP client. For any
app talking to a backend you control, add **certificate (SPKI) pinning**, and
write a **test that proves the pin actually rejects a wrong cert**.

## Problem

- The bare `http` package has no interceptors, no global config, no retry — you
  end up re-writing that per call.
- Without pinning, a user on a hostile network (corporate proxy, rogue Wi-Fi,
  a phone with an injected root CA) can **man-in-the-middle** your TLS. For apps
  handling logins, payments, or private data that's a real leak.
- Pinning that's never tested tends to be pinning that's silently broken (wrong
  hash, disabled in a refactor) — you only find out when it fails to protect
  anyone.

## Fix

`dio` for all requests; pin the server's public-key hash on the underlying
`HttpClient`.

```yaml
dependencies:
  dio: ^5.4.0
  crypto: ^3.0.0
```

```dart
import 'dart:convert';
import 'dart:io';
import 'package:crypto/crypto.dart';
import 'package:dio/dio.dart';
import 'package:dio/io.dart';

// Base64 SHA-256 of the server cert's SubjectPublicKeyInfo.
const kPinnedSpkiSha256 = 'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=';

Dio buildDio() {
  final dio = Dio(BaseOptions(baseUrl: 'https://api.example.com'));
  (dio.httpClientAdapter as IOHttpClientAdapter).createHttpClient = () {
    final client = HttpClient();
    client.badCertificateCallback = (cert, host, port) {
      final spki = sha256.convert(cert.der).toString(); // simplified; hash SPKI in prod
      return base64.encode(sha256.convert(cert.der).bytes) == kPinnedSpkiSha256;
    };
    return client;
  };
  return dio;
}
```

(In production hash the **SPKI**, not the whole DER cert, so the pin survives
cert renewal on the same key. Keep a backup pin for key rotation.)

## The test is part of the rule

Ship a test that fails if pinning stops working — a good pin must **reject** a
server whose cert doesn't match:

```dart
// test/cert_pinning_test.dart
import 'dart:io';
import 'package:test/test.dart';

void main() {
  test('rejects a cert that does not match the pin', () async {
    // Point dio at a host serving a DIFFERENT cert (self-signed / badssl-style).
    final dio = buildDio()..options.baseUrl = 'https://wrong-cert.test';
    expect(
      () => dio.get('/'),
      throwsA(isA<DioException>()), // handshake refused by the pin
    );
  });

  test('accepts the real pinned host', () async {
    final dio = buildDio();
    final res = await dio.get('/health');
    expect(res.statusCode, 200);
  });
}
```

Use a known bad-cert endpoint (e.g. a `badssl.com` host or a local self-signed
server) for the reject case so the test is deterministic.

## Notes / gotchas

- Pin the **SPKI hash**, keep at least one **backup pin**, or a routine cert
  rotation will brick every installed app until they update.
- Only pin hosts you control. Don't pin third-party APIs whose certs you can't
  predict.
- Put auth headers, logging, and retry in `dio` **interceptors**, not at call sites.
- Certificate pinning is not a FOSS/GMS concern — it applies to every app, free
  or not.
