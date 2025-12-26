# HTTP Wake Implementation Plan: Android

## Overview

Add HTTP-based wake functionality as an alternative to standard Wake-on-LAN (WOL) in moonlight-android. This mirrors the feature already implemented in moonlight-qt.

## Files to Modify

| File | Changes |
|------|---------|
| `app/src/main/java/com/limelight/nvstream/http/ComputerDetails.java` | Add `wakeMethod` enum and `httpWakeUrl` field |
| `app/src/main/java/com/limelight/computers/ComputerDatabaseManager.java` | Add columns for wake config, update serialization |
| `app/src/main/java/com/limelight/nvstream/wol/WakeOnLanSender.java` | Add HTTP wake method alongside WOL |
| `app/src/main/java/com/limelight/PcView.java` | Add "Configure Wake" context menu item and dialog |
| `app/src/main/res/values/strings.xml` | Add new string resources |
| `app/src/main/res/layout/` | Add dialog layout for wake configuration (optional - can use programmatic AlertDialog) |

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
- `HttpWakeUrl` (TEXT) - URL template

**Migration strategy:**
- Create new database version (computers5.db or add to existing migration)
- Default existing entries to WOL method

**Update methods:**
- `getComputerByUUID()` - read new fields
- `updateComputer()` - write new fields

### Step 3: HTTP Wake Implementation (WakeOnLanSender.java)

Add new method:

```java
public static void sendHttpWake(ComputerDetails computer) {
    if (computer.httpWakeUrl == null || computer.httpWakeUrl.isEmpty()) {
        return;
    }

    // Expand URL template placeholders
    String url = computer.httpWakeUrl
        .replace("{mac}", computer.macAddress != null ? computer.macAddress : "")
        .replace("{localaddress}", computer.localAddress != null ? computer.localAddress.address : "")
        .replace("{remoteaddress}", computer.remoteAddress != null ? computer.remoteAddress.address : "");

    OkHttpClient client = new OkHttpClient.Builder()
        .connectTimeout(10, TimeUnit.SECONDS)
        .readTimeout(10, TimeUnit.SECONDS)
        .build();

    Request request = new Request.Builder()
        .url(url)
        .get()
        .build();

    try {
        Response response = client.newCall(request).execute();
        // Log result but don't throw - best effort
        LimeLog.info("HTTP wake response: " + response.code());
        response.close();
    } catch (IOException e) {
        LimeLog.warning("HTTP wake failed: " + e.getMessage());
    }
}
```

**Modify `sendWolPacket()`** to check wake method:

```java
public static void sendWolPacket(ComputerDetails computer) {
    if (computer.wakeMethod == ComputerDetails.WakeMethod.HTTP) {
        sendHttpWake(computer);
        return;
    }
    // ... existing WOL code
}
```

### Step 4: UI - Context Menu (PcView.java)

**Add new menu constant:**
```java
private static final int CONFIGURE_WAKE_ID = 11;  // After VIEW_DETAILS_ID = 8
```

**Add menu item in `onCreateContextMenu()`:**
```java
menu.add(Menu.NONE, CONFIGURE_WAKE_ID, 6, getResources().getString(R.string.pcview_menu_configure_wake));
```

**Handle in `onContextItemSelected()`:**
```java
case CONFIGURE_WAKE_ID:
    showConfigureWakeDialog(computer);
    return true;
```

### Step 5: UI - Configuration Dialog (PcView.java)

```java
private void showConfigureWakeDialog(final ComputerDetails computer) {
    AlertDialog.Builder builder = new AlertDialog.Builder(this);
    builder.setTitle(R.string.pcview_configure_wake_title);

    View view = getLayoutInflater().inflate(R.layout.dialog_configure_wake, null);
    RadioGroup wakeMethodGroup = view.findViewById(R.id.wake_method_group);
    RadioButton wolRadio = view.findViewById(R.id.wake_method_wol);
    RadioButton httpRadio = view.findViewById(R.id.wake_method_http);
    EditText httpUrlEdit = view.findViewById(R.id.http_wake_url);
    View httpSection = view.findViewById(R.id.http_wake_section);

    // Initialize state
    wolRadio.setChecked(computer.wakeMethod == ComputerDetails.WakeMethod.WOL);
    httpRadio.setChecked(computer.wakeMethod == ComputerDetails.WakeMethod.HTTP);
    httpUrlEdit.setText(computer.httpWakeUrl);
    httpSection.setVisibility(httpRadio.isChecked() ? View.VISIBLE : View.GONE);

    wakeMethodGroup.setOnCheckedChangeListener((group, checkedId) -> {
        httpSection.setVisibility(checkedId == R.id.wake_method_http ? View.VISIBLE : View.GONE);
    });

    builder.setView(view);
    builder.setPositiveButton(R.string.ok, (dialog, which) -> {
        computer.wakeMethod = httpRadio.isChecked() ?
            ComputerDetails.WakeMethod.HTTP : ComputerDetails.WakeMethod.WOL;
        computer.httpWakeUrl = httpUrlEdit.getText().toString().trim();

        // Save to database
        ComputerDatabaseManager dbManager = new ComputerDatabaseManager(this);
        dbManager.updateComputer(computer);
    });
    builder.setNegativeButton(R.string.cancel, null);
    builder.show();
}
```

### Step 6: String Resources (strings.xml)

```xml
<string name="pcview_menu_configure_wake">Configure Wake</string>
<string name="pcview_configure_wake_title">Configure Wake: %1$s</string>
<string name="wake_method_wol">Standard Wake-on-LAN (magic packet)</string>
<string name="wake_method_http">HTTP Wake (for VPN/Tailscale)</string>
<string name="http_wake_url_label">HTTP Wake URL:</string>
<string name="http_wake_url_hint">https://wakeonlan.your-tailnet.ts.net/wake?mac=aa:bb:cc:dd:ee:ff</string>
<string name="http_wake_timeout_info">A simple HTTP GET request with a 10-second timeout will be sent to this URL.</string>
```

## Testing Strategy

1. **Backward compatibility**: Existing hosts should default to WOL with no UI changes
2. **Serialization**: Verify settings persist across app restarts
3. **Placeholder expansion**: Test {mac}, {localaddress}, {remoteaddress} substitution
4. **HTTP errors**: Test timeout, connection refused, 404, 500 responses (check logs)
5. **Local testing**: Use a simple HTTP server:
   ```bash
   python3 -m http.server 8080
   # Configure URL: http://localhost:8080/wake?mac={mac}
   ```

## Notes

- Uses a 10-second timeout for HTTP requests
- URL query parameters and credentials should be redacted in logs
- The wake method is per-host, allowing different hosts to use different methods
- HTTP wake uses a simple GET request with no authentication (relies on network-level security like Tailscale)
