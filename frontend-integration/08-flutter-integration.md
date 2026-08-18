# Flutter Integration Guide

Complete implementation notes for a Flutter app connected to a Total.js API Routing backend.

Reference stack: **Flutter + Dart + http + flutter_secure_storage + Provider or Riverpod**.

This guide mirrors the mobile integration patterns used successfully in React Native. Replace schema names, hostnames, and domain APIs with the ones from your project, but keep the same client architecture.

---

## Setup

Add the core packages:

```yaml
dependencies:
  flutter:
    sdk: flutter
  http: ^1.2.2
  flutter_secure_storage: ^9.2.2
  provider: ^6.1.2
  web_socket_channel: ^3.0.1
```

```bash
flutter pub get
```

For `flutter_secure_storage`, set Android `minSdkVersion` to at least `18` in `android/app/build.gradle`. iOS needs no extra setup for the default keychain behavior.

Recommended structure:

```text
lib/
  api/
    app_config.dart       # --dart-define config and URL building
    token_storage.dart    # secure token persistence
    schema_helpers.dart   # query builder and anonymous schema allowlist
    api_client.dart       # low-level HTTP, auth headers, errors
    api_request.dart      # apiRequest(), schema queries, normalization
    upload.dart           # multipart upload helper
  services/
    auth_service.dart
    posts_service.dart
    account_service.dart
  providers/
    auth_provider.dart
    app_state.dart
  screens/
  main.dart
```

---

## Environment Config

Flutter exposes build-time config through `--dart-define`. Do not bundle private secrets. Upload tokens are acceptable only when deliberately scoped to a file service.

| Define | Purpose |
|--------|---------|
| `APP_ENV` | `dev` or `production`; selects suffixed values when used. |
| `API_BASE_URL` | Default API host. |
| `API_BASE_URL_DEV` | Dev API host override. |
| `API_BASE_URL_PRODUCTION` | Production API host override. |
| `API_PATH` | Usually `/` for actions with `route: 'API /'` or `/api/` for conventional deployments. |
| `UPLOAD_URL` | File service upload base URL. |
| `UPLOAD_TOKEN` | Optional scoped upload token. |
| `UPLOAD_AUTH_HEADER` | Optional header name for the upload token, for example `Authorization`. |
| `ALLOW_LOCALHOST_NATIVE` | Set to `true` only when the runtime can really reach loopback. |

Native Android/iOS apps usually cannot reach the development machine through `localhost` or `127.0.0.1`. Use a LAN/proxy URL unless loopback is explicitly allowed.

```dart
class AppConfig {
  static const appEnv = String.fromEnvironment('APP_ENV', defaultValue: 'dev');
  static const apiPath = String.fromEnvironment('API_PATH', defaultValue: '/');
  static const allowLocalhostNative =
      bool.fromEnvironment('ALLOW_LOCALHOST_NATIVE', defaultValue: false);

  static String get apiBaseUrl {
    const base = String.fromEnvironment('API_BASE_URL');
    const dev = String.fromEnvironment('API_BASE_URL_DEV');
    const prod = String.fromEnvironment('API_BASE_URL_PRODUCTION');
    final selected = appEnv == 'production' && prod.isNotEmpty ? prod : dev;
    return selected.isNotEmpty ? selected : base;
  }

  static bool hasLoopbackHost(String url) {
    return RegExp(r'^https?://(localhost|127\.0\.0\.1|\[::1\])(?::\d+)?(?:/|$)',
            caseSensitive: false)
        .hasMatch(url.trim());
  }

  static Uri buildApiUri() {
    if (!allowLocalhostNative && hasLoopbackHost(apiBaseUrl)) {
      throw StateError('Use a LAN/proxy API URL for native Flutter runtimes.');
    }

    final base = apiBaseUrl.trim().replaceAll(RegExp(r'/+$'), '');
    final path = apiPath.trim();
    final normalizedPath = path.isEmpty || path == '/'
        ? '/'
        : '/${path.replaceAll(RegExp(r'^/+|/+$'), '')}/';
    return Uri.parse('$base$normalizedPath');
  }
}
```

Example:

```bash
flutter run \
  --dart-define=API_BASE_URL_DEV=http://192.168.1.20:8000 \
  --dart-define=API_PATH=/
```

---

## Token Storage

Store the session token in secure storage only. Non-secret preferences can live in shared preferences, Provider state, Riverpod state, or a local database.

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class TokenStorage {
  static const _storage = FlutterSecureStorage(
    aOptions: AndroidOptions(encryptedSharedPreferences: true),
  );
  static const _key = 'totaljs_session_token';

  static Future<String?> get() => _storage.read(key: _key);
  static Future<void> set(String token) => _storage.write(key: _key, value: token);
  static Future<void> clear() => _storage.delete(key: _key);
}
```

---

## API Client

The client has one public function:

```dart
Future<T> apiRequest<T>(
  String schema, {
  Map<String, dynamic>? data,
  Map<String, dynamic>? query,
  String method = 'POST',
})
```

Default to `POST /` or `POST /api/` with `{ schema, data }`. Some projects also expose `GET /?schema=...`; support it only as a convenience, not as the primary integration contract.

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;
import 'app_config.dart';
import 'schema_helpers.dart';
import 'token_storage.dart';

class ApiException implements Exception {
  final String message;
  final int? statusCode;
  const ApiException(this.message, {this.statusCode});
  @override
  String toString() => message;
}

class UnauthorizedException extends ApiException {
  const UnauthorizedException() : super('Session expired. Please log in again.', statusCode: 401);
}

class ApiClient {
  static final http.Client _http = http.Client();

  static Future<dynamic> request(
    String schema, {
    Map<String, dynamic>? data,
    String method = 'POST',
  }) async {
    final headers = <String, String>{'Content-Type': 'application/json'};
    final baseSchema = getBaseSchema(schema);
    final token = await TokenStorage.get();

    if (token != null && !isAnonymousApiSchema(baseSchema)) {
      headers['x-token'] = token;
      headers['token'] = token;
      headers['Authorization'] = 'Bearer $token';
    }

    final apiUri = AppConfig.buildApiUri();
    final response = method == 'GET'
        ? await _http.get(apiUri.replace(queryParameters: {'schema': schema}), headers: headers)
        : await _http.post(
            apiUri,
            headers: headers,
            body: jsonEncode({'schema': schema, if (data != null) 'data': data}),
          );

    if (response.statusCode == 401 && !isAnonymousApiSchema(baseSchema)) {
      await TokenStorage.clear();
      throw const UnauthorizedException();
    }

    if (response.statusCode >= 500) {
      throw ApiException('Server error (${response.statusCode})',
          statusCode: response.statusCode);
    }

    if (response.body.trim().isEmpty) return null;
    return jsonDecode(response.body);
  }
}
```

`x-token` is the portable Total.js baseline. Sending `token` and `Authorization: Bearer` as compatibility headers helps when a project has multiple middlewares or a separate file service.

---

## Schema Queries And Anonymous Schemas

Place schema helpers in `lib/api/schema_helpers.dart` so both `api_client.dart` and `api_request.dart` can use them without circular imports.

Build schema query strings programmatically and omit empty values:

```dart
String buildSchemaWithQuery(String schema, [Map<String, dynamic>? query]) {
  if (query == null || query.isEmpty) return schema;

  final parts = <String>[];
  for (final entry in query.entries) {
    final raw = entry.value;
    if (raw == null || raw == '') continue;
    final values = raw is Iterable ? raw : [raw];
    for (final value in values) {
      if (value == null || value == '') continue;
      parts.add('${Uri.encodeQueryComponent(entry.key)}='
          '${Uri.encodeQueryComponent(value.toString())}');
    }
  }

  return parts.isEmpty ? schema : '$schema?${parts.join('&')}';
}

String getBaseSchema(String schema) => schema.split('?').first;
```

Total.js action declarations are useful references, but do not treat route metadata as a portable mobile auth contract:

```javascript
NEWACTION('Posts|read', {
  input: '*id',
  route: '+API /',
  action: function($, model) {
    // model.id contains the validated record ID
  }
});
```

Confirm public/protected behavior from backend middleware and real responses, then mirror public schemas in the Flutter client:

```dart
const anonymousApiSchemas = <String>{
  'Account|create',
  'Account|login',
  'Account|login_google',
  'Account|login_github',
  'Account|oauth',
  'Account|reset',
  'Account|password_reset',
  'Account|verify',
  'Posts|list',
  'Posts|read',
  'Categories|list',
};

bool isAnonymousApiSchema(String schema) => anonymousApiSchemas.contains(getBaseSchema(schema));
```

This prevents stale tokens from being attached to login, registration, public discovery, and OTP calls. It also prevents public `401` responses from logging users out.

---

## Response Normalization

Total.js responses may be plain payloads, `{ success, value, error }` envelopes, or one-item array envelopes. Normalize once in the API layer so screens never parse transport shapes.

Rules:

- If the response is `[ { success/value/error/token } ]`, collapse it to the first item.
- If `success == false` or `error` exists, throw a normalized `ApiException`.
- If root `token` exists and `value` is an object, merge the token into the returned object.
- If `value` exists, return `value`.
- If `success == true` with no `value`, return `null`.
- For list endpoints, accept both arrays and `{ items: [] }`.

```dart
dynamic normalizeApiResponse(dynamic payload) {
  var data = payload;
  if (data is List && data.length == 1 && data.first is Map) {
    data = data.first;
  }

  if (data is Map) {
    final error = data['error'] ?? data['message'];
    if (data['success'] == false || error != null) {
      throw ApiException(error?.toString() ?? 'Request failed');
    }

    if (data.containsKey('token') && data['value'] is Map) {
      return {...Map<String, dynamic>.from(data['value']), 'token': data['token']};
    }

    if (data.containsKey('value')) return data['value'];
    if (data['success'] == true) return null;
  }

  return data;
}

Future<T> apiRequest<T>(
  String schema, {
  Map<String, dynamic>? data,
  Map<String, dynamic>? query,
  String method = 'POST',
}) async {
  final schemaWithQuery = buildSchemaWithQuery(schema, query);
  final raw = await ApiClient.request(schemaWithQuery, data: data, method: method);
  return normalizeApiResponse(raw) as T;
}

List<T> extractItems<T>(dynamic payload) {
  if (payload is List) return payload.cast<T>();
  if (payload is Map && payload['items'] is List) {
    return (payload['items'] as List).cast<T>();
  }
  return <T>[];
}
```

Keep one error normalizer for HTTP errors, Total.js array errors, envelope errors, and plain strings. Map network and timeout errors to UI-friendly messages before they reach widgets.

---

## Auth And Session Hydration

Restore the token from `flutter_secure_storage` during app bootstrap, then hydrate account data. Provider, Riverpod, Bloc, or another state manager can own the in-memory user snapshot and language.

Startup flow:

```text
App starts
  -> restore persisted non-secret preferences
  -> load token from secure storage
  -> if no token: clear auth state and show the public shell
  -> if token: set token in memory and call Account|read
  -> hydrate user and any cheap bootstrap counts
  -> show the authenticated shell
```

Login/register flow:

```text
Account|login
  -> normalize { token, user? } or a plain token string
  -> save token to secure storage
  -> set token/user in app state
  -> call Account|read in the background
```

Logout flow:

```text
Account|logout best-effort
  -> delete secure storage token
  -> clear auth state
  -> preserve non-sensitive preferences such as language
```

Example service:

```dart
class AuthService {
  Future<Map<String, dynamic>> login(String email, String password) async {
    final res = await apiRequest<dynamic>('Account|login', data: {
      'email': email,
      'password': password,
    });

    final token = res is String ? res : (res as Map)['token']?.toString();
    if (token == null || token.isEmpty) throw const ApiException('Missing session token');

    await TokenStorage.set(token);
    return res is Map ? Map<String, dynamic>.from(res) : {'token': token};
  }

  Future<Map<String, dynamic>> account() async {
    return apiRequest<Map<String, dynamic>>('Account|read');
  }

  Future<void> logout() async {
    try {
      await apiRequest<dynamic>('Account|logout');
    } catch (_) {
      // Logout should still clear local state when the backend is unreachable.
    }
    await TokenStorage.clear();
  }
}
```

The mobile app can be guest-first: public screens load without auth, auth opens as a modal or route, and protected navigation mounts only when authenticated.

---

## Auth Gate

Use an auth gate for public screens that expose protected actions.

```dart
typedef PendingAuthAction = Future<void> Function();

class AuthGateController {
  PendingAuthAction? _pending;

  void requireAuth({
    required bool isAuthenticated,
    required PendingAuthAction action,
    required VoidCallback openAuth,
  }) {
    if (isAuthenticated) {
      action();
      return;
    }

    _pending = action;
    openAuth();
  }

  Future<void> consumePendingAuthAction() async {
    final action = _pending;
    _pending = null;
    if (action != null) await action();
  }
}
```

Call `consumePendingAuthAction()` after successful login or registration.

---

## Domain API Layer

Do not call schema strings directly from widgets. Keep typed domain APIs thin and centralized.

```dart
class PostsApi {
  Future<List<Map<String, dynamic>>> list({
    int? page,
    String? search,
  }) async {
    final payload = await apiRequest<dynamic>(
      'Posts|list',
      query: {'page': page, 'search': search},
    );
    return extractItems<Map<String, dynamic>>(payload);
  }

  Future<Map<String, dynamic>> read(String id) {
    return apiRequest<Map<String, dynamic>>(
      'Posts|read',
      data: {'id': id.trim()},
    );
  }

  Future<Map<String, dynamic>> create(Map<String, dynamic> data) {
    return apiRequest<Map<String, dynamic>>('Posts|create', data: data);
  }
}
```

Useful mobile domains:

- `AuthApi`: login, register, profile, password reset, OAuth.
- `PostsApi`: public lists and authenticated writes.
- `AccountApi`: current user and settings.

---

## Uploads

File uploads do not use the Total.js API envelope. Upload multipart form data to the file service, then store returned URL/metadata through a normal schema when the domain requires it.

Build the upload path from the current user id, or `anonymous`:

```dart
class UploadConfig {
  static const uploadUrl = String.fromEnvironment('UPLOAD_URL');
  static const uploadToken = String.fromEnvironment('UPLOAD_TOKEN');
  static const uploadAuthHeader = String.fromEnvironment('UPLOAD_AUTH_HEADER');
}

Uri buildUploadUri(String userId) {
  final id = Uri.encodeComponent(userId.isEmpty ? 'anonymous' : userId);
  final hasPlaceholder =
      UploadConfig.uploadUrl.contains('{id}') || UploadConfig.uploadUrl.contains('{0}');

  var raw = UploadConfig.uploadUrl.replaceAll('{id}', id).replaceAll('{0}', id);
  var uri = Uri.parse(raw);

  if (!hasPlaceholder) {
    uri = uri.replace(path: '${uri.path.replaceAll(RegExp(r'/+$'), '')}/$id/');
  } else {
    uri = uri.replace(path: '${uri.path.replaceAll(RegExp(r'/+$'), '')}/');
  }

  if (UploadConfig.uploadToken.isNotEmpty &&
      UploadConfig.uploadAuthHeader.isEmpty &&
      !uri.queryParameters.containsKey('token')) {
    uri = uri.replace(queryParameters: {
      ...uri.queryParameters,
      'token': UploadConfig.uploadToken,
    });
  }

  return uri;
}
```

Upload with auth priority:

1. If `UPLOAD_TOKEN` and `UPLOAD_AUTH_HEADER` are configured, send the upload token in that header.
2. If `UPLOAD_TOKEN` is configured without a header, add it as `?token=...`.
3. Otherwise send the user session token as `token`, `x-token`, and `Authorization`.

```dart
Future<Map<String, dynamic>> uploadFile(String filePath, String userId) async {
  final request = http.MultipartRequest('POST', buildUploadUri(userId));
  request.files.add(await http.MultipartFile.fromPath('file', filePath));

  if (UploadConfig.uploadToken.isNotEmpty && UploadConfig.uploadAuthHeader.isNotEmpty) {
    request.headers[UploadConfig.uploadAuthHeader] = UploadConfig.uploadToken;
  } else if (UploadConfig.uploadToken.isEmpty) {
    final token = await TokenStorage.get();
    if (token != null) {
      request.headers['x-token'] = token;
      request.headers['token'] = token;
      request.headers['Authorization'] = 'Bearer $token';
    }
  }

  final response = await request.send();
  final body = await response.stream.bytesToString();
  if (response.statusCode < 200 || response.statusCode >= 300) {
    throw ApiException('Upload failed (${response.statusCode})');
  }

  return jsonDecode(body) as Map<String, dynamic>;
}
```

Normalize relative upload responses into absolute URLs using the upload service origin.

---

## WebSocket

Use `web_socket_channel` when the backend exposes realtime routes. Pass the token as a query parameter only when the socket endpoint expects it.

```dart
import 'dart:async';
import 'dart:convert';
import 'package:web_socket_channel/web_socket_channel.dart';

class BackendSocket {
  BackendSocket({
    required this.uri,
    required this.onMessage,
    this.onDisconnected,
  });

  final Uri uri;
  final void Function(Map<String, dynamic>) onMessage;
  final void Function()? onDisconnected;

  WebSocketChannel? _channel;
  StreamSubscription? _sub;

  void connect() {
    _channel = WebSocketChannel.connect(uri);
    _sub = _channel!.stream.listen(
      (raw) => onMessage(jsonDecode(raw.toString()) as Map<String, dynamic>),
      onDone: onDisconnected,
      onError: (_) => onDisconnected?.call(),
      cancelOnError: true,
    );
  }

  Future<void> disconnect() async {
    await _sub?.cancel();
    await _channel?.sink.close(1000);
  }
}
```

---

## Production Checklist

- `apiRequest()` is the only low-level API entrypoint.
- API base URL and upload URL are driven by `--dart-define`.
- Native localhost is blocked or replaced unless explicitly allowed.
- Session token lives in `flutter_secure_storage`, not plain preferences.
- Anonymous schemas are explicitly allowlisted and do not receive stale tokens.
- `401` clears auth only for protected schemas.
- Response normalization handles envelopes, arrays, root tokens, and raw lists.
- Domain APIs own schema strings; widgets call typed functions/providers.
- Auth bootstrap completes before navigation decisions.
- Uploads use the file service and normalize returned URLs.
- Development logs mask token, password, secret, and authorization fields.

---

## Platform Differences

| Concern | React Native | Flutter |
|---------|--------------|---------|
| HTTP client | `axios` | `http` |
| Token storage | `expo-secure-store` | `flutter_secure_storage` |
| State management | Zustand | Provider, Riverpod, Bloc, or app state classes |
| Config | `EXPO_PUBLIC_*` env vars | `--dart-define` |
| 401 handling | navigation/auth events | app state update and root navigation rebuild |
| File upload | `fetch` + `FormData` | `http.MultipartRequest` |
| WebSocket | `WebSocket` API | `web_socket_channel` |

The API contract is identical across mobile stacks: schema strings, `{ schema, data }` request body, normalized `{ success, value, error }` responses, and the `x-token` auth header.
