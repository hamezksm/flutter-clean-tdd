# UI: Apple-style design in Flutter

If the `apple-design` skill (emilkowalski) is installed, read it for the principles - it is the design authority. But its code is for the web (CSS, Pointer Events, Motion). Never port CSS or JS literally; use the Flutter equivalents below. Skip its "Initial Response" instruction when it is read from within this skill. If it isn't installed, this file stands on its own.

Ask once per project: **Cupertino everywhere, or platform-adaptive** (Cupertino on Apple platforms, Material 3 on Android)? Default to Cupertino everywhere if the user just wants "Apple design".

## Principle → Flutter mapping

| Principle | Flutter |
|---|---|
| Respond on press, not release | `CupertinoButton` (built-in pressed opacity), or `GestureDetector(onTapDown/onTapUp/onTapCancel)` driving a `useState` pressed flag (visual-only, allowed) |
| 1:1 drag tracking, respect grab offset | `GestureDetector` / `onVerticalDragUpdate` adding `details.delta` to the current value - never jump to the finger position |
| Interruptible, start from current value | `AnimationController.stop()` on drag start and continue from `controller.value` (the live on-screen value) |
| Springs, not fixed durations | `SpringSimulation` + `controller.animateWith(...)` |
| Velocity handoff | Pass `details.velocity.pixelsPerSecond` (converted to controller units) as the simulation's velocity |
| Momentum projection | `project()` below, then snap to nearest point to the projected end |
| Rubber-banding | `BouncingScrollPhysics` for scrollables (keep it - never force Clamping on iOS); `rubberBand()` below for custom drags |
| Same path in and out | `CupertinoPageRoute`, `showCupertinoModalPopup` / Cupertino sheet routes; custom transitions use the same curve reversed |
| Translucent materials | `CupertinoNavigationBar`/`CupertinoTabBar` with translucent `backgroundColor` and content scrolling under; custom: `ClipRect` + `BackdropFilter(filter: ImageFilter.blur(...))` |
| Multimodal feedback | `HapticFeedback.selectionClick()` / `lightImpact()` fired in the same callback as the visual state change, only for meaningful moments |
| Reduced motion | `MediaQuery.disableAnimationsOf(context)` → cross-fade or no animation, no bounce |
| Higher contrast | `MediaQuery.highContrastOf(context)` → solid surfaces, stronger borders. Flutter has no reduce-transparency flag; treat high contrast as the trigger to drop blur |
| Dynamic Type | Never clamp `MediaQuery.textScalerOf(context)`; layouts must reflow at 2x text. Test it. |
| System font, size-specific tracking | Keep `CupertinoTheme`'s system font; define a type scale in `core/ui/theme/` with tighter `letterSpacing` and `height` for large sizes, near 0 tracking for body |
| Light/dark | `CupertinoDynamicColor` / `CupertinoColors.systemBackground` etc., resolved via context. Never hardcode colors in widgets |

## Motion math lives in `core/ui/motion/` (pure Dart, TDD)
These are pure functions - write their unit tests first.

```dart
import 'dart:math' as math;
import 'package:flutter/physics.dart';

/// Apple-style spring from designer params: response (s) and damping ratio.
SpringDescription appleSpring({double response = 0.4, double dampingRatio = 1.0}) {
  const mass = 1.0;
  final stiffness = math.pow(2 * math.pi / response, 2) * mass;
  final damping = 4 * math.pi * dampingRatio * mass / response;
  return SpringDescription(mass: mass, stiffness: stiffness.toDouble(), damping: damping);
}

abstract final class Springs {
  static final standard = appleSpring(response: 0.4, dampingRatio: 1.0); // default, no overshoot
  static final sheet = appleSpring(response: 0.3, dampingRatio: 0.8);    // only after a flick
  static final rotation = appleSpring(response: 0.4, dampingRatio: 0.8);
}

/// Distance a flick would travel with exponential deceleration.
double project(double velocityPxPerSec, {double decelerationRate = 0.998}) =>
    (velocityPxPerSec / 1000) * decelerationRate / (1 - decelerationRate);

/// Progressive resistance past a boundary.
double rubberBand(double overshoot, double dimension, {double c = 0.55}) =>
    (overshoot * dimension * c) / (dimension + c * overshoot.abs());

double nearestSnap(double value, List<double> snaps) =>
    snaps.reduce((a, b) => (a - value).abs() <= (b - value).abs() ? a : b);
```

Default to `Springs.standard` (no overshoot). Use bounce only when the user's gesture carried momentum.

## Draggable sheet pattern (hooks, no StatefulWidget)
```dart
class SnapSheet extends HookWidget {
  const SnapSheet({super.key, required this.child, required this.height});
  final Widget child;
  final double height;

  @override
  Widget build(BuildContext context) {
    final offset = useAnimationController(
      lowerBound: double.negativeInfinity, upperBound: double.infinity,
    ); // value = px offset from open position
    useAnimation(offset);
    final reduceMotion = MediaQuery.disableAnimationsOf(context);
    final snaps = [0.0, height];

    return GestureDetector(
      onVerticalDragStart: (_) => offset.stop(),               // interrupt from live value
      onVerticalDragUpdate: (d) {
        final next = offset.value + d.delta.dy;
        offset.value = next < 0 ? -rubberBand(-next, height) : next;
      },
      onVerticalDragEnd: (d) {
        final v = d.velocity.pixelsPerSecond.dy;
        final target = nearestSnap(offset.value + project(v), snaps);
        if (reduceMotion) { offset.value = target; return; }
        offset.animateWith(SpringSimulation(
          v.abs() > 300 ? Springs.sheet : Springs.standard, offset.value, target, v,
        ));
        HapticFeedback.selectionClick();
      },
      child: Transform.translate(offset: Offset(0, offset.value), child: child),
    );
  }
}
```
Whether the sheet is "open" as app state (e.g. it drives navigation or data) belongs in a notifier; the pixel offset is visual and stays in the hook.

## Performance
- `BackdropFilter` is expensive: one per surface, wrapped in `ClipRect`, never inside list items.
- Animate transforms and opacity; wrap animating subtrees in `RepaintBoundary` when profiling shows repaint cost.
- Use `const` widgets so animations rebuild only what moves.

## Widget tests for UI
Every screen gets tests at default settings, with `disableAnimations: true`, with `highContrast: true`, and at `TextScaler.linear(2)` to prove nothing overflows.
