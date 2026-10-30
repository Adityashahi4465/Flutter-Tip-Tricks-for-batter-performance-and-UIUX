# Reusable Flutter auth form and keyboard UX

Use this guide when building login, registration, password reset, OTP, or profile setup screens in another Flutter application. It describes the pattern used in KavyaKunj and provides components that can be copied without its theme or Firebase dependencies.

## 1. Expected behaviour

| User action or screen state | Expected behaviour |
| --- | --- |
| Keyboard opens | The visible area resizes; fields and actions remain reachable by scrolling. |
| Content fits | The layout fills the available height without unnecessary blank space. |
| Content exceeds the available height | The screen scrolls vertically without overflow. |
| User drags the form | The keyboard closes. |
| User taps outside a field | The keyboard closes. |
| User presses Next | Focus moves to the next input field. |
| User presses Done/check on the final field | The keyboard closes; the form is not submitted automatically. |
| User presses the submit button | Validate, show errors, then submit valid input. |
| Validation adds error text | The layout grows and remains scrollable. |

Use explicit next-field focus targets when controls such as password visibility buttons might otherwise enter the focus order. Moving to the next field does not inherently mean the current value is valid; validation remains a separate responsibility.

Dismiss-on-drag is a reasonable default for short auth forms. For long editing forms, consider retaining the keyboard during scrolling so users can inspect nearby content without interrupting typing. Do not dismiss focus merely because the app automatically scrolls a field into view.

## 2. Reusable scroll layout

Create `lib/core/common/keyboard_scroll_view.dart`:

```dart
import 'package:flutter/material.dart';

/// Use inside a bounded Scaffold body, after SafeArea.
class KeyboardScrollView extends StatelessWidget {
  final Widget child;
  final EdgeInsetsGeometry padding;

  const KeyboardScrollView({
    super.key,
    required this.child,
    this.padding = const EdgeInsets.symmetric(vertical: 16),
  });

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) => GestureDetector(
        behavior: HitTestBehavior.translucent,
        onTap: () => FocusScope.of(context).unfocus(),
        child: SingleChildScrollView(
          keyboardDismissBehavior: ScrollViewKeyboardDismissBehavior.onDrag,
          physics: const AlwaysScrollableScrollPhysics(),
          child: ConstrainedBox(
            constraints: BoxConstraints(minHeight: constraints.maxHeight),
            child: IntrinsicHeight(
              child: Padding(padding: padding, child: child),
            ),
          ),
        ),
      ),
    );
  }
}
```

The `Scaffold` resizes for the keyboard. `LayoutBuilder` measures that remaining space. The minimum height preserves the full-screen form layout, while the scroll view lets taller content extend beyond it. `IntrinsicHeight` allows short form columns containing `Spacer` to work within this arrangement. `AlwaysScrollableScrollPhysics` allows a dismissing drag even when the content fits.

This layout is intended for short forms. Intrinsic measurement has a layout cost; use a `ListView` or slivers for long or dynamically generated forms. Do not place a lazy list or another vertical scroll view inside this wrapper. Its parent must provide a finite maximum height; if it is inside another `Column`, give it bounded space with `Expanded`.

## 3. Reusable keyboard actions for fields

If your app already has a shared text field, add the following options and behaviour there. Otherwise, create `lib/core/common/keyboard_form_field.dart`:

```dart
import 'package:flutter/material.dart';

class KeyboardFormField extends StatelessWidget {
  final TextEditingController controller;
  final String label;
  final FocusNode? focusNode;
  final FocusNode? nextFocusNode;
  final TextInputAction textInputAction;
  final TextInputType? keyboardType;
  final bool obscureText;
  final String? Function(String?)? validator;
  final Iterable<String>? autofillHints;

  const KeyboardFormField({
    super.key,
    required this.controller,
    required this.label,
    this.focusNode,
    this.nextFocusNode,
    this.textInputAction = TextInputAction.done,
    this.keyboardType,
    this.obscureText = false,
    this.validator,
    this.autofillHints,
  });

  @override
  Widget build(BuildContext context) {
    return TextFormField(
      controller: controller,
      focusNode: focusNode,
      keyboardType: keyboardType,
      textInputAction: textInputAction,
      obscureText: obscureText,
      autocorrect: false,
      enableSuggestions: !obscureText,
      autofillHints: autofillHints,
      scrollPadding: const EdgeInsets.all(24),
      decoration: InputDecoration(labelText: label),
      validator: validator,
      onEditingComplete: () {
        final scope = FocusScope.of(context);
        if (textInputAction == TextInputAction.next) {
          if (nextFocusNode != null) {
            scope.requestFocus(nextFocusNode);
          } else {
            scope.nextFocus();
          }
        } else {
          scope.unfocus();
        }
      },
    );
  }
}
```

`scrollPadding` gives Flutter room around the focused field when revealing it in a scrollable. It does not guarantee that every validation message will automatically become visible. After validation, focus the first invalid field and, if necessary, reveal its container with `Scrollable.ensureVisible` after the updated layout has rendered.

Controllers and externally supplied focus nodes belong to the screen state. Create them once, outside `build`, and dispose them when the screen closes. Keep password text unchanged when submitting; do not trim passwords.

For multiline inputs, preserve newline behaviour rather than applying this auth field wrapper unchanged. Search or keyboard-submit actions should also be handled explicitly if your app needs them.

## 4. Screen integration example

The following example assumes both shared components are imported. Add these members to a `State` class:

```dart
final formKey = GlobalKey<FormState>();
final emailController = TextEditingController();
final passwordController = TextEditingController();
final passwordFocusNode = FocusNode();

@override
void dispose() {
  emailController.dispose();
  passwordController.dispose();
  passwordFocusNode.dispose();
  super.dispose();
}
```

Use this structure in `build`:

```dart
return Scaffold(
  resizeToAvoidBottomInset: true,
  appBar: AppBar(title: const Text('Log in')),
  body: SafeArea(
    child: KeyboardScrollView(
      padding: const EdgeInsets.all(16),
      child: AutofillGroup(
        child: Form(
          key: formKey,
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.stretch,
            children: [
              const Text('Welcome back'),
              const SizedBox(height: 24),
              KeyboardFormField(
                controller: emailController,
                label: 'Email',
                keyboardType: TextInputType.emailAddress,
                textInputAction: TextInputAction.next,
                nextFocusNode: passwordFocusNode,
                autofillHints: const [AutofillHints.username],
                validator: (value) => (value?.trim().isEmpty ?? true)
                    ? 'Enter your email'
                    : null,
              ),
              const SizedBox(height: 16),
              KeyboardFormField(
                controller: passwordController,
                label: 'Password',
                focusNode: passwordFocusNode,
                obscureText: true,
                autofillHints: const [AutofillHints.password],
                validator: (value) => (value?.isEmpty ?? true)
                    ? 'Enter your password'
                    : null,
              ),
              const SizedBox(height: 24),
              FilledButton(
                onPressed: () {
                  FocusScope.of(context).unfocus();
                  if (!formKey.currentState!.validate()) return;
                  // Call your auth controller here.
                  // Email may be trimmed; password must stay unchanged.
                },
                child: const Text('Log in'),
              ),
            ],
          ),
        ),
      ),
    ),
  ),
);
```

These sample validators only check that values are present. Add your application's email validation, server error handling, loading state, duplicate-submit prevention, and invalid-field focus behaviour.

## 5. Apply the pattern to each auth screen

| Screen | Field order and keyboard actions |
| --- | --- |
| Login | Email: Next → Password: Done. |
| Registration | Full name: Next → Email: Next → Password: Next → Confirm password: Done. |
| Password reset | Email: Done; reset button submits. |
| Profile setup | Username: Done; Finish button submits. |
| OTP | Numeric input; Done dismisses; a completed code may trigger verification once. |

For registration, create focus nodes for email, password, and confirmation. Connect each preceding field's `nextFocusNode` to the next field's `focusNode`. Use a name keyboard for full name, an email keyboard for email, and obscured input for both password fields.

For OTP, use `AutofillHints.oneTimeCode` where the field or OTP package supports it. Keep a visible Verify action, guard against simultaneous verification requests, and verify the code on the server. Some iOS numeric keyboard layouts do not provide a Done key; provide a visible dismiss control or keyboard accessory instead of relying on the keyboard action alone.

Keep decorative backgrounds outside the scrollable and form content inside it. For example, an auth background can wrap `SafeArea → KeyboardScrollView → Form`. Avoid moving background artwork with every form scroll unless that is an intentional design choice.

## 6. Optional refinements

These are enhancements to consider; they are not all implemented in KavyaKunj:

- Reduce the illustration height while the keyboard is visible. Detect visibility with `MediaQuery.viewInsetsOf(context).bottom > 0`; use a small `AnimatedSize` or `AnimatedContainer` transition. Keep enough space for the focused field and avoid large layout jumps.
- Add autofill hints to shared fields: username/password for login, email/newPassword for registration, and oneTimeCode for OTP. The reusable sample above exposes hints; KavyaKunj's current `CustomTextField` does not yet expose them.
- After failed validation, reveal and focus the first invalid field. Preserve entered values and use specific inline messages.
- Disable duplicate submissions during requests and give a clear loading state. If a loading overlay sits outside the `Scaffold`, give its text a `Material` ancestor so it inherits a normal text style.
- Check TalkBack, VoiceOver, hardware Tab navigation, large text, and password-manager behaviour.

## 7. Avoid these layout mistakes

- A form in a non-scrollable `Column` can overflow when the keyboard reduces the viewport.
- `resizeToAvoidBottomInset: false` can leave fields or actions behind the keyboard unless you deliberately handle the insets yourself.
- A container with the full screen's fixed height ignores the smaller keyboard-visible area. Let content determine its height instead.
- A `Spacer` or `Expanded` directly inside an unconstrained scroll view can produce unbounded-height errors. Use the bounded/minimum-height pattern above, or remove flex children.
- Adding full keyboard-height bottom padding when the `Scaffold` already resizes can double-count the inset and create a large gap.
- Nested vertical scroll views can make dismissal and field visibility unpredictable.
- Globally dismissing focus on every scroll notification can close the keyboard during automatic field scrolling. Prefer `keyboardDismissBehavior: onDrag`.

## 8. Verification checklist

Run the following on every auth screen:

- Open the keyboard on a small phone and focus each field. Confirm there are no overflow warnings.
- Reach the final field and submit button without closing the keyboard manually.
- Press Next through the form. Confirm focus skips decorative controls and password visibility buttons.
- Press Done/check on the final field. Confirm the keyboard closes without submitting.
- Drag the form and tap empty space. Confirm both dismiss the keyboard.
- Move directly from one input to another. Confirm the new field receives focus normally.
- Trigger empty/invalid values so validation text increases the form height. Confirm errors and actions remain reachable.
- Repeat with large text and landscape orientation where the app supports it.
- Test Android and iOS separately, including an iOS numeric keypad.
- Verify autofill and OTP behaviour on devices; simulated keyboard actions do not prove that native autofill works.

Automated widget tests should cover Next/Done focus transitions, layout with simulated keyboard insets, scrolling to the bottom of a tall form, drag dismissal, and short layouts containing `Spacer`. KavyaKunj has three focused tests in `test/auth_keyboard_test.dart`; they cover these basic layout and focus behaviours, not every native keyboard or autofill scenario.

## 9. Existing KavyaKunj implementation

- Shared scroll layout: [`keyboard_scroll_view.dart`](../lib/core/common/keyboard_scroll_view.dart)
- Shared field keyboard options: [`custom_text_field.dart`](../lib/core/common/custom_text_field.dart)
- Login example: [`login_screen.dart`](../lib/features/auth/views/login_screen.dart)
- Registration focus chain: [`registration_screen.dart`](../lib/features/auth/views/registration_screen.dart)
- OTP example: [`verify_email_screen.dart`](../lib/features/auth/views/verify_email_screen.dart)
- Regression tests: [`auth_keyboard_test.dart`](../test/auth_keyboard_test.dart)

The `KeyboardFormField` above is a portable example for other apps. KavyaKunj uses its existing themed `CustomTextField` instead.

## References

- [Flutter: dismissing the keyboard on drag](https://api.flutter.dev/flutter/widgets/ScrollViewKeyboardDismissBehavior.html)
- [Flutter: SingleChildScrollView and viewport constraints](https://api.flutter.dev/flutter/widgets/SingleChildScrollView-class.html)
- [Flutter: TextInputAction](https://api.flutter.dev/flutter/services/TextInputAction.html)
- [Flutter: IntrinsicHeight and its layout cost](https://api.flutter.dev/flutter/widgets/IntrinsicHeight-class.html)
- [Apple: Return key types](https://developer.apple.com/documentation/UIKit/UIReturnKeyType)
