# 4. Do Not Implement Android Shortcuts

Date: 2026-08-02

## Status

Accepted

## Context

The app exposes a background service that performs a privileged action. The
service is declared `exported="true"` so that third-party automation apps
(e.g. Tasker) can start it. Access is gated by a **user-set access key**: the
key is configured by the user in the main activity, stored in the app's private
storage, and must be supplied as an intent extra for the service to act. The
user copies this key into whatever automation app they choose to authorize.

Adding support for the **Android shortcut system** was evaluated — specifically
a static (XML) shortcut backed by a dedicated shortcut activity. The shortcut
activity would read the access key from private storage and start/stop the
service on the user's behalf, giving the user a one-tap launcher entry point
without having to configure an automation app.

A fundamental security issue was identified:

1. **Exported Activity Requirement**: For the launcher to start the shortcut target activity, the
  activity must be declared as `android:exported="true"` in the manifest.

2. **Explicit Intent Access**: Any installed app can send an explicit intent to an exported activity
  using `ComponentName`, bypassing intent-filters. This is by Android design.

3. **Access Key Injection**: The shortcut handler must inject the user's access key
  (stored in SharedPreferences) into the Intent for MainService, as MainService requires
  `EXTRA_ACCESS_KEY` for both `ACTION_START` and `ACTION_STOP`.

4. **Security Gap**: This means any malicious app could send an explicit intent to the shortcut
  handling activity. The activity would then read the access key from preferences and forward it to
  MainService, effectively allowing the attacker to start or stop the VNC server on behalf of the
  user.

👉 **This essentially moves the "user interaction needed so we can inject access key on behalf of user"
point from within MainActivity to outside of it, where we cannot ensure there really _was_ user
interaction.**

### Alternatives

#### Alternative 1: getCallingPackage() Verification

Verify caller via `getCallingPackage()` against known launcher packages

**Rejected**: Caller verification is unreliable (`getCallingPackage()` returns null for `startActivity()`) and launcher packages vary by OEM. Blocklist maintenance was considered not sustainable.

#### Alternative 2: Use dynamic shortcuts with restricted visibility
Dynamic shortcuts published only to trusted callers could be used.

**Rejected**: Dynamic shortcuts were found to have the same exported activity requirement and do not solve the explicit intent problem.

## Decision

The Android shortcut system will **not** be adopted for service control. 
Users who want a trigger continue to use an automation app with their
access key, which is the existing, already-secured entry point.

## Consequences

- **Positive**: No security vulnerability introduced where malicious apps can control the VNC server
- **Negative**: Users do not gain the convenience of launcher shortcuts for start/stop actions
- **Workaround**: Users can still access start/stop through the app's main UI, widgets, or the
  existing Intent API with proper access key authentication