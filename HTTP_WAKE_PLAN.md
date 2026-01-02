# HTTP Wake Implementation Plan: Android

## Overview

Add HTTP-based wake functionality as an alternative to standard Wake-on-LAN (WOL) in moonlight-android. This mirrors the feature already implemented in moonlight-qt.

## Files to Modify

| File | Changes |
|------|---------|
| `app/src/main/java/com/limelight/nvstream/http/ComputerDetails.java` | Add `wakeMethod` enum and `httpWakeUrl` field |
| `app/src/main/java/com/limelight/computers/ComputerDatabaseManager.java` | Add columns for wake config, update serialization |
| `app/src/main/java/com/limelight/nvstream/wol/WakeOnLanSender.java` | Add HTTP wake method alongside WOL |
| `app/src/main/java/com/limelight/computers/ComputerManagerService.java` | Add `updateWakeConfig()` method to sync in-memory state |
| `app/src/main/java/com/limelight/PcView.java` | Add "Configure Wake" context menu item and dialog |
| `app/src/main/res/values/strings.xml` | Add new string resources |

## Implementation Steps

### Step 1: Data Model Changes (ComputerDetails.java)

Add wake configuration fields:

```java
// Wake method enum
public enum WakeMethod {
    WOL,    // Standard Wake-on-LAN (default)
    HTTP    // HTTP GET request to configured URL
}

// New fields (add to class)
public WakeMethod wakeMethod = WakeMethod.WOL;
public String httpWakeUrl;
```

### Step 2: Database Schema Update (ComputerDatabaseManager.java)

**Add new columns:**
- `WakeMethod` (INTEGER) - 0=WOL, 1=HTTP
- `HttpWakeUrl` (TEXT) - URL for HTTP wake

**Migration strategy:**
- Use ALTER TABLE to add columns (with try-catch for existing columns)
- Default existing entries to WOL method

**Update methods:**
- `getComputerFromCursor()` - read new fields
- `updateComputer()` - write new fields

### Step 3: HTTP Wake Implementation (WakeOnLanSender.java)

**Add URL validation method:**
```java
public static boolean isValidWakeUrl(String url) {
    if (url == null || url.isEmpty()) {
        return false;
    }
    try {
        URL parsedUrl = new URL(url);
        String scheme = parsedUrl.getProtocol();
        String host = parsedUrl.getHost();
        return ("http".equalsIgnoreCase(scheme) || "https".equalsIgnoreCase(scheme))
                && host != null && !host.isEmpty();
    } catch (MalformedURLException e) {
        return false;
    }
}
```

**Add HTTP wake method:**
- Use shared static OkHttpClient instance for efficiency
- Validate URL scheme (http/https only) and host
- Throw IOException on failure (for proper error handling)
- Redact query parameters and credentials in logs

**Add routing method:**
```java
public static void sendWakePacket(ComputerDetails computer) throws IOException {
    if (computer.wakeMethod == ComputerDetails.WakeMethod.HTTP) {
        sendHttpWake(computer);
    } else {
        sendWolPacket(computer);
    }
}
```

### Step 4: Service Update (ComputerManagerService.java)

Add `updateWakeConfig()` method to binder to sync wake configuration changes to the in-memory computer details (with proper synchronization on `tuple.networkLock`).

### Step 5: UI - Context Menu (PcView.java)

**Add new menu constant and menu item in `onCreateContextMenu()`**

**Handle in `onContextItemSelected()`:**
```java
case CONFIGURE_WAKE_ID:
    showConfigureWakeDialog(computer.details);
    return true;
```

### Step 6: UI - Configuration Dialog (PcView.java)

- Programmatic AlertDialog with RadioGroup (WOL/HTTP) and EditText for URL
- Real-time URL validation using `WakeOnLanSender.isValidWakeUrl()`
- Show error message when URL is invalid
- Prevent save when HTTP is selected but URL is invalid
- Update both database and in-memory state via `managerBinder.updateWakeConfig()`

### Step 7: String Resources (strings.xml)

```xml
<string name="pcview_menu_configure_wake">Configure Wake</string>
<string name="pcview_configure_wake_title">Configure Wake: %1$s</string>
<string name="wake_method_wol">Standard Wake-on-LAN (magic packet)</string>
<string name="wake_method_http">HTTP Wake (for VPN/Tailscale)</string>
<string name="http_wake_url_label">HTTP Wake URL:</string>
<string name="http_wake_url_hint">https://example.com/wake</string>
<string name="http_wake_url_invalid">Invalid URL. Please enter a valid HTTP or HTTPS URL.</string>
<string name="http_wake_timeout_info">A simple HTTP GET request with a 10-second timeout will be sent to this URL.</string>
<string name="http_wake_waking_msg">It may take a few seconds for your PC to wake up.</string>
<string name="http_wake_fail">Failed to send HTTP wake request</string>
<string name="pcview_wake_config_saved">Wake configuration saved</string>
```

## Testing Strategy

1. **Backward compatibility**: Existing hosts should default to WOL with no UI changes
2. **Serialization**: Verify settings persist across app restarts
3. **URL validation**: Test invalid URLs are rejected (empty, non-http schemes, missing host)
4. **HTTP errors**: Test timeout, connection refused, 404, 500 responses (check logs)
5. **Local testing**: Use a simple HTTP server:
   ```bash
   python3 -m http.server 8080
   # Configure URL: http://localhost:8080/wake
   ```

## Notes

- Uses a 10-second timeout for HTTP requests
- URL query parameters and credentials are redacted in logs
- The wake method is per-host, allowing different hosts to use different methods
- HTTP wake uses a simple GET request with no authentication (relies on network-level security like Tailscale)
- URL validation ensures only http/https schemes with valid hosts are accepted
