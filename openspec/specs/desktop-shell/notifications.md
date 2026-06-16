# Notification and Sound System

> 系统通知和音效反馈，用于在达到使用率阈值时提醒用户。

## ADDED Requirements

### Requirement: Notification Types

`NotificationManager` in `rust/src/notifications.rs` manages system notifications.

#### Scenario: Notification type enum

- **WHEN** a notification is evaluated
- **THEN** the type is one of: `HighUsage` (≥75%), `CriticalUsage` (≥90%), `Exhausted` (100%), `StatusIssue` (provider error), `SessionDepleted` (session quota exhausted), `SessionRestored` (session quota recovered)

#### Scenario: Notification deduplication

- **WHEN** the same provider and notification type occurs multiple times
- **THEN** `sent_notifications: HashSet<(ProviderId, NotificationType)>` prevents duplicate notifications
- **AND** notifications are only sent once per state transition

### Requirement: Threshold Notification Logic

Notifications fire when usage crosses configured thresholds.

#### Scenario: High usage notification

- **WHEN** `check_and_notify(provider, used_percent, settings)` is called
- **AND** `used_percent >= settings.high_usage_threshold` (default 75%)
- **AND** this threshold was not already notified for this provider
- **THEN** a "High Usage" toast notification is shown
- **AND** the `(provider, HighUsage)` pair is added to `sent_notifications`

#### Scenario: Critical usage notification

- **WHEN** `used_percent >= settings.critical_usage_threshold` (default 90%)
- **THEN** a "Critical Usage" toast notification with higher priority is shown
- **AND** the `(provider, CriticalUsage)` pair is added to `sent_notifications`

#### Scenario: Threshold crossing

- **WHEN** usage drops below a previously notified threshold (e.g., after reset)
- **THEN** the `(provider, threshold_type)` is removed from `sent_notifications`
- **AND** the notification can fire again on next crossing

### Requirement: Session Quota Transition Notifications

Notifications for session-based quota state changes.

#### Scenario: Session depleted notification

- **WHEN** `check_session_transition(provider, current_percent, settings)` detects a transition from non-exhausted to exhausted
- **THEN** a `SessionDepleted` notification is shown

#### Scenario: Session restored notification

- **WHEN** a session resets and transitions from exhausted to non-exhausted
- **THEN** a `SessionRestored` notification is shown

### Requirement: Status Issue Notifications

Provider fetch errors can trigger status notifications.

#### Scenario: Status issue notification

- **WHEN** `notify_status_issue(provider, description, settings)` is called
- **THEN** a `StatusIssue` notification is shown with the error description
- **AND** only if `settings.show_notifications` is true

#### Scenario: Clear status issue

- **WHEN** `clear_status_issue(provider)` is called (on successful fetch)
- **THEN** the status issue notification is cleared
- **AND** the provider returns to normal notification state

### Requirement: Toast Notification Display

Windows toast notifications are shown via the OS notification system.

#### Scenario: Windows toast

- **WHEN** `show_toast(title, body)` is called on Windows
- **THEN** a Windows toast notification appears with the specified title and body text
- **AND** respects the system notification settings (Do Not Disturb, Focus Assist)

### Requirement: Sound Feedback

`rust/src/sound.rs` provides audio feedback for threshold events.

#### Scenario: Sound playback

- **WHEN** a threshold notification fires and `settings.sound_enabled` is true
- **THEN** a sound is played at `settings.sound_volume` level
- **AND** different sounds may be used for different notification types

#### Scenario: Sound disabled

- **WHEN** `settings.sound_enabled` is false
- **THEN** no sounds are played regardless of notification type

### Requirement: Tray Icon Rendering

`rust/src/tray/render.rs` renders pixel-level tray icons.

#### Scenario: Normal icon rendering

- **WHEN** `render_bar_icon_rgba(session_percent, weekly_percent, has_error)` is called
- **THEN** it produces a 32x32 RGBA image with usage bars
- **AND** colors follow the `UsageLevel::from_percent()` mapping: green (0-50%) → yellow (50-75%) → red (75-100%)
- **AND** the icon shows two bars when `weekly_percent` is `Some`: session (top, thicker) and weekly (bottom, thinner)
- **AND** the icon shows one centered bar when `weekly_percent` is `None`

#### Scenario: Error state rendering

- **WHEN** `has_error` is true
- **THEN** the icon background alpha is reduced (180 instead of 255)
- **AND** bar colors are desaturated to grayscale

#### Scenario: Render output format

- **WHEN** the render function completes
- **THEN** it returns `(Vec<u8>, u32, u32)` — raw RGBA bytes, width, height
- **AND** the width and height are both 32 (TRAY_ICON_SIZE)
