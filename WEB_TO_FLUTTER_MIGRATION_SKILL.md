---
name: web-to-flutter-native
description: >
  Complete end-to-end skill for migrating a web application (HTML/CSS/JS or React)
  to production-grade native Flutter code targeting Android and iOS. Covers design
  token translation, component mapping, visual inspection loop via MCP tools,
  automated screenshot-diff-fix cycles, and pixel-accurate design fidelity.
  Use this skill whenever migrating a web UI to Flutter, verifying Flutter UI
  against a web reference, or running an automated visual QA loop on a running
  Flutter emulator or simulator.
triggers:
  - "migrate web to flutter"
  - "convert web app to android"
  - "web to native"
  - "flutter migration"
  - "web design to flutter"
  - "visual inspection loop flutter"
  - "screenshot diff flutter"
---

# Web → Flutter Native Migration Skill

> **AGENT PRIME DIRECTIVE:** You are executing a full web-to-native migration.
> Your job is not just to translate code — it is to produce a Flutter app that a
> human, holding it next to the web version, cannot visually distinguish from the
> original. Every colour, every radius, every shadow, every animation, every
> interaction state must be reproduced in Flutter with the same fidelity.
> You will not stop until the visual loop passes. You will not guess at values —
> you will read them from the source. You will not skip a component — every visible
> element gets migrated.

---

## PART 0 — BEFORE YOU WRITE A SINGLE LINE

Read these files in order before doing anything else:

```
1. The web source (HTML/CSS/JS files or design system .md)
2. ORDR_BRAND_DESIGN_GUIDELINES_V1.md (if present — the design token truth)
3. This SKILL.md (you are reading it now)
```

Then answer these four questions internally before starting:

1. **What is the primary emotion this Flutter UI must produce?** (Must match web)
2. **What are the exact accent, floor, surface, and text hex values?**
3. **What are the three depth planes and how are they expressed in Flutter?**
4. **Which MCP tools are available?** (Run: `flutter --version` and check MCP config)

---

## PART 1 — ENVIRONMENT SETUP

### 1.1 — Verify MCP Tool Availability

Before starting migration, confirm which visual tools are available:

```bash
# Check Flutter installation
flutter --version
flutter doctor -v

# Check if Dart MCP server is available
dart mcp-server --version 2>/dev/null && echo "DART_MCP=available" || echo "DART_MCP=missing"

# Check if Marionette is in pubspec
grep -r "marionette" pubspec.yaml 2>/dev/null && echo "MARIONETTE=available" || echo "MARIONETTE=missing"

# List connected devices
flutter devices
```

Capture outputs. Store in `migration_state.json` under `environment`.

### 1.2 — MCP Tool Decision Tree

```
IF dart_mcp=available AND marionette=available:
  → USE FULL VISUAL LOOP (Part 5)
  → Agent can screenshot, inspect widget tree, tap, type, scroll

ELSE IF dart_mcp=available ONLY:
  → USE CODE-ONLY LOOP (Part 5B)
  → Agent can analyze errors, hot-reload, check widget tree at build time

ELSE IF marionette=available ONLY:
  → USE VISUAL-ONLY LOOP (Part 5C)
  → Agent can screenshot and interact but not analyze build errors

ELSE:
  → USE MANUAL LOOP (Part 5D)
  → Agent runs flutter build, reads logs, produces screenshots via ADB
```

### 1.3 — Project Bootstrap

If no Flutter project exists yet:

```bash
# Create project
flutter create --org com.ordr --template=app \
  --platforms=android,ios ordr_app

cd ordr_app

# Add required dependencies
flutter pub add \
  google_fonts \
  flutter_svg \
  go_router \
  riverpod \
  flutter_riverpod \
  cached_network_image \
  lottie \
  shimmer

# Add dev dependencies
flutter pub add --dev \
  flutter_test \
  integration_test \
  golden_toolkit

# If Marionette is available
flutter pub add marionette_flutter
flutter pub add --dev marionette_mcp
```

### 1.4 — Required pubspec.yaml Structure

```yaml
name: ordr_app
description: Court-ready case management — native

environment:
  sdk: ">=3.2.0 <4.0.0"
  flutter: ">=3.16.0"

dependencies:
  flutter:
    sdk: flutter
  google_fonts: ^6.1.0
  flutter_svg: ^2.0.9
  go_router: ^13.0.0
  flutter_riverpod: ^2.4.0
  shimmer: ^3.0.0
  lottie: ^3.0.0
  # Add marionette_flutter if using visual loop
  marionette_flutter: ^1.0.0   # REMOVE IN PRODUCTION BUILD

dev_dependencies:
  flutter_test:
    sdk: flutter
  golden_toolkit: ^0.15.0
  marionette_mcp: ^1.0.0       # REMOVE IN PRODUCTION BUILD

flutter:
  uses-material-design: true
  assets:
    - assets/icons/
    - assets/images/
    - assets/fonts/
```

---

## PART 2 — DESIGN TOKEN TRANSLATION

### 2.1 — CSS → Flutter Color Translation

The agent MUST translate every CSS token to a Flutter `Color` or `ColorScheme`.
Use this exact mapping table. Never hardcode hex values elsewhere in the codebase.

**Create `lib/core/theme/tokens.dart`:**

```dart
import 'package:flutter/material.dart';

/// ORDR Design System — SCALES v1
/// Single source of truth for all visual tokens.
/// Mirrors ORDR_BRAND_DESIGN_GUIDELINES_V1.md exactly.
/// DO NOT add magic numbers anywhere else in the codebase.
abstract class OrdrTokens {

  // ─── FLOOR ────────────────────────────────────────────────────────────
  static const Color floorBase    = Color(0xFF09090B);
  static const Color floorAmbient = Color(0xFF0F0F12);

  // ─── SURFACES ─────────────────────────────────────────────────────────
  static const Color surfaceCard     = Color(0x0AFFFFFF); // rgba(255,255,255,0.04)
  static const Color surfaceElevated = Color(0x12FFFFFF); // rgba(255,255,255,0.07)
  static const Color surfaceMuted    = Color(0x06FFFFFF); // rgba(255,255,255,0.025)
  static const Color surfaceFrost    = Color(0x14F8F8FC); // rgba(248,248,252,0.08)

  // ─── TEXT ─────────────────────────────────────────────────────────────
  static const Color textPrimary   = Color(0xFFF4F0E8);
  static const Color textSecondary = Color(0x9EF4F0E8); // 0.62 alpha
  static const Color textTertiary  = Color(0x5CF4F0E8); // 0.36 alpha

  // ─── ACCENT ───────────────────────────────────────────────────────────
  static const Color accent       = Color(0xFFC9A84C);
  static const Color accentStrong = Color(0xFFE8C86A);
  static const Color accentGhost  = Color(0x1FC9A84C); // 0.12 alpha
  static const Color accentBorder = Color(0x33C9A84C); // 0.20 alpha
  static const Color onAccent     = Color(0xFF1A1408);

  // ─── GRADIENTS ────────────────────────────────────────────────────────
  static const LinearGradient accentGradient = LinearGradient(
    begin: Alignment.topLeft,
    end: Alignment.bottomRight,
    stops: [0.0, 0.5, 1.0],
    colors: [Color(0xFFC9A84C), Color(0xFFE8C86A), Color(0xFFB8952F)],
  );

  static const LinearGradient accentGradientSimple = LinearGradient(
    begin: Alignment.topLeft,
    end: Alignment.bottomRight,
    colors: [Color(0xFFC9A84C), Color(0xFFE8C86A)],
  );

  static const RadialGradient atmosphereTop = RadialGradient(
    center: Alignment(-0.70, -0.80),
    radius: 1.2,
    colors: [Color(0x0FC9A84C), Colors.transparent],
  );

  static const RadialGradient atmosphereBottom = RadialGradient(
    center: Alignment(0.70, 0.80),
    radius: 1.1,
    colors: [Color(0x0AC9A84C), Colors.transparent],
  );

  // ─── SEMANTIC ─────────────────────────────────────────────────────────
  static const Color signalPositive       = Color(0xFF4CAF7D);
  static const Color signalPositiveBg     = Color(0x264CAF7D); // 0.15 alpha
  static const Color signalPositiveBorder = Color(0x404CAF7D); // 0.25 alpha
  static const Color signalDanger         = Color(0xFFE05252);
  static const Color signalDangerBg       = Color(0x1FE05252); // 0.12 alpha

  // ─── BORDERS ──────────────────────────────────────────────────────────
  static const Color borderSubtle = Color(0x12FFFFFF); // rgba(255,255,255,0.07)
  static const Color borderMedium = Color(0x1AFFFFFF); // rgba(255,255,255,0.10)
  static const Color borderStrong = Color(0x2EFFFFFF); // rgba(255,255,255,0.18)

  // ─── RADIUS ───────────────────────────────────────────────────────────
  static const double radiusCard  = 22.0;
  static const double radiusInput = 14.0;
  static const double radiusPill  = 999.0;
  static const double radiusIcon  = 12.0;
  static const double radiusBadge = 20.0;
  static const double radiusSmall = 10.0;

  // ─── SPACING ──────────────────────────────────────────────────────────
  static const double space1 = 4.0;
  static const double space2 = 8.0;
  static const double space3 = 16.0;
  static const double space4 = 24.0;
  static const double space5 = 40.0;
  static const double space6 = 64.0;
  static const double space7 = 96.0;
  static const double cardPaddingMin = 28.0; // NEVER go below this

  // ─── ELEVATION ────────────────────────────────────────────────────────
  // Flutter doesn't use CSS box-shadows directly.
  // Use these BoxShadow lists instead.
  static const List<BoxShadow> shadowSoft = [
    BoxShadow(color: Color(0x73000000), blurRadius: 24, offset: Offset(0, 2)),
  ];
  static const List<BoxShadow> shadowCard = [
    BoxShadow(color: Color(0x47000000), blurRadius: 32, offset: Offset(0, 8)),
    BoxShadow(color: Color(0x24000000), blurRadius: 8,  offset: Offset(0, 2)),
  ];
  static const List<BoxShadow> shadowFloat = [
    BoxShadow(color: Color(0xB3000000), blurRadius: 48, offset: Offset(0, 8)),
    BoxShadow(color: Color(0x47000000), blurRadius: 16, offset: Offset(0, 4)),
  ];
  static const List<BoxShadow> shadowAccentGlow = [
    BoxShadow(color: Color(0x4DC9A84C), blurRadius: 24, offset: Offset(0, 4)),
  ];
  static const List<BoxShadow> shadowAccentStrong = [
    BoxShadow(color: Color(0x73C9A84C), blurRadius: 32, offset: Offset(0, 8)),
  ];

  // ─── MOTION DURATIONS ────────────────────────────────────────────────
  // CSS spring curves → Flutter Curves
  static const Duration durationFast   = Duration(milliseconds: 150);
  static const Duration durationMedium = Duration(milliseconds: 300);
  static const Duration durationSlow   = Duration(milliseconds: 600);
  static const Duration durationEntrance = Duration(milliseconds: 700);

  // CSS --spring-light  → springLight
  static const Curve springLight  = Curves.elasticOut;    // overshoot, fast
  // CSS --spring-medium → springMedium
  static const Curve springMedium = Curves.easeOutCubic;  // smooth, gentle overshoot
  // CSS --spring-heavy  → springHeavy
  static const Curve springHeavy  = Curves.easeOutBack;   // dramatic spring
}
```

**AGENT RULE:** After creating this file, grep the entire codebase:
```bash
grep -rn "Color(0x\|Colors\." lib/ | grep -v "tokens.dart" | grep -v "theme.dart"
```
If any hardcoded color values appear outside `tokens.dart` or `theme.dart`, fix them before proceeding.

---

### 2.2 — Typography Translation

**Create `lib/core/theme/typography.dart`:**

```dart
import 'package:flutter/material.dart';
import 'package:google_fonts/google_fonts.dart';
import 'tokens.dart';

/// ORDR Typography System — mirrors web SCALES v1
/// Cormorant Garamond → display/editorial only
/// Instrument Sans → all UI chrome
abstract class OrdrTextStyles {

  // ─── DISPLAY / EDITORIAL (Cormorant Garamond) ─────────────────────────

  /// Hero page headline — maps to CSS clamp(52px, 8vw, 108px)
  /// Flutter equivalent: scaled via LayoutBuilder or MediaQuery
  static TextStyle heroXL(BuildContext ctx) {
    final size = MediaQuery.of(ctx).size.width;
    final fs = (size * 0.08).clamp(52.0, 108.0);
    return GoogleFonts.cormorantGaramond(
      fontSize: fs,
      fontWeight: FontWeight.w300,
      height: 0.95,         // line-height: 0.95
      letterSpacing: fs * -0.02,  // letter-spacing: -0.02em
      color: OrdrTokens.textPrimary,
    );
  }

  /// Section title — clamp(36px, 5vw, 60px)
  static TextStyle sectionTitle(BuildContext ctx) {
    final size = MediaQuery.of(ctx).size.width;
    final fs = (size * 0.05).clamp(36.0, 60.0);
    return GoogleFonts.cormorantGaramond(
      fontSize: fs,
      fontWeight: FontWeight.w300,
      height: 1.1,
      letterSpacing: fs * -0.01,
      color: OrdrTokens.textPrimary,
    );
  }

  /// Card heading — 24px
  static final TextStyle cardHeading = GoogleFonts.cormorantGaramond(
    fontSize: 24,
    fontWeight: FontWeight.w400,
    height: 1.2,
    letterSpacing: -0.24, // -0.01em
    color: OrdrTokens.textPrimary,
  );

  /// Step/feature title — 28px
  static final TextStyle stepTitle = GoogleFonts.cormorantGaramond(
    fontSize: 28,
    fontWeight: FontWeight.w400,
    letterSpacing: -0.28,
    color: OrdrTokens.textPrimary,
  );

  /// Pull quote / testimonial — 18px italic
  static final TextStyle pullQuote = GoogleFonts.cormorantGaramond(
    fontSize: 18,
    fontStyle: FontStyle.italic,
    fontWeight: FontWeight.w400,
    height: 1.5,
    color: OrdrTokens.textPrimary,
  );

  /// Hero stat number — 42px tabular
  static final TextStyle heroNumber = GoogleFonts.cormorantGaramond(
    fontSize: 42,
    fontWeight: FontWeight.w300,
    height: 1.0,
    letterSpacing: -0.84, // -0.02em
    fontFeatures: [FontFeature.tabularFigures()],
    color: OrdrTokens.textPrimary,
  );

  /// Timer display — 72px
  static final TextStyle timerDisplay = GoogleFonts.cormorantGaramond(
    fontSize: 72,
    fontWeight: FontWeight.w300,
    height: 1.0,
    letterSpacing: -2.88, // -0.04em
    fontFeatures: [FontFeature.tabularFigures()],
    color: OrdrTokens.accentStrong,
  );

  /// Plan price — 48px
  static final TextStyle planPrice = GoogleFonts.cormorantGaramond(
    fontSize: 48,
    fontWeight: FontWeight.w300,
    height: 1.0,
    letterSpacing: -1.44, // -0.03em
    fontFeatures: [FontFeature.tabularFigures()],
    color: OrdrTokens.textPrimary,
  );

  // ─── UI / FUNCTIONAL (Instrument Sans) ────────────────────────────────

  /// Page body copy — 15px
  static final TextStyle body = GoogleFonts.instrumentSans(
    fontSize: 15,
    fontWeight: FontWeight.w400,
    height: 1.7,
    letterSpacing: 0.15, // 0.01em
    color: OrdrTokens.textSecondary,
  );

  /// Card body — 13px
  static final TextStyle bodySm = GoogleFonts.instrumentSans(
    fontSize: 13,
    fontWeight: FontWeight.w400,
    height: 1.65,
    color: OrdrTokens.textSecondary,
  );

  /// Caption — 12px
  static final TextStyle caption = GoogleFonts.instrumentSans(
    fontSize: 12,
    fontWeight: FontWeight.w400,
    color: OrdrTokens.textTertiary,
  );

  /// Micro UI label — 10px UPPERCASE — ALWAYS UPPERCASE
  static final TextStyle label = GoogleFonts.instrumentSans(
    fontSize: 10,
    fontWeight: FontWeight.w600,
    letterSpacing: 1.8, // 0.18em
    color: OrdrTokens.textTertiary,
  );
  static String labelText(String s) => s.toUpperCase(); // ALWAYS call this

  /// Accent label (section eyebrow) — 11px UPPERCASE
  static final TextStyle accentLabel = GoogleFonts.instrumentSans(
    fontSize: 11,
    fontWeight: FontWeight.w600,
    letterSpacing: 1.76, // 0.16em
    color: OrdrTokens.accent,
  );

  /// Button — 13px
  static final TextStyle button = GoogleFonts.instrumentSans(
    fontSize: 13,
    fontWeight: FontWeight.w600,
    letterSpacing: 0.13,
    color: OrdrTokens.onAccent,
  );

  /// Nav link — 13px
  static final TextStyle navLink = GoogleFonts.instrumentSans(
    fontSize: 13,
    fontWeight: FontWeight.w500,
    color: OrdrTokens.textSecondary,
  );

  /// Badge text — 9px UPPERCASE
  static final TextStyle badge = GoogleFonts.instrumentSans(
    fontSize: 9,
    fontWeight: FontWeight.w700,
    letterSpacing: 0.9, // 0.10em
    color: OrdrTokens.accent,
  );
  static String badgeText(String s) => s.toUpperCase(); // ALWAYS call this
}
```

---

### 2.3 — ThemeData Construction

**Create `lib/core/theme/theme.dart`:**

```dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'tokens.dart';

ThemeData ordrTheme() {
  return ThemeData(
    brightness: Brightness.dark,
    scaffoldBackgroundColor: OrdrTokens.floorBase,
    colorScheme: const ColorScheme.dark(
      background:    OrdrTokens.floorBase,
      surface:       OrdrTokens.surfaceCard,
      primary:       OrdrTokens.accent,
      onPrimary:     OrdrTokens.onAccent,
      secondary:     OrdrTokens.accentStrong,
      onSecondary:   OrdrTokens.onAccent,
      error:         OrdrTokens.signalDanger,
      onError:       OrdrTokens.textPrimary,
      onBackground:  OrdrTokens.textPrimary,
      onSurface:     OrdrTokens.textPrimary,
    ),
    // No Material card defaults — use OrdrCard widget
    cardTheme: const CardTheme(
      color:       OrdrTokens.surfaceCard,
      elevation:   0,
      margin:      EdgeInsets.zero,
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.all(Radius.circular(OrdrTokens.radiusCard)),
        side: BorderSide(color: OrdrTokens.borderSubtle, width: 1),
      ),
    ),
    // Input fields
    inputDecorationTheme: InputDecorationTheme(
      filled: true,
      fillColor: OrdrTokens.surfaceElevated,
      border: OutlineInputBorder(
        borderRadius: BorderRadius.circular(OrdrTokens.radiusInput),
        borderSide: const BorderSide(color: OrdrTokens.borderMedium, width: 1),
      ),
      enabledBorder: OutlineInputBorder(
        borderRadius: BorderRadius.circular(OrdrTokens.radiusInput),
        borderSide: const BorderSide(color: OrdrTokens.borderMedium, width: 1),
      ),
      focusedBorder: OutlineInputBorder(
        borderRadius: BorderRadius.circular(OrdrTokens.radiusInput),
        borderSide: const BorderSide(color: OrdrTokens.accentBorder, width: 1.5),
      ),
      hintStyle: TextStyle(color: OrdrTokens.textTertiary),
    ),
    // Disable default Material splash (use custom)
    splashFactory: NoSplash.splashFactory,
    highlightColor: Colors.transparent,
    // Status bar style
    appBarTheme: const AppBarTheme(
      backgroundColor: Colors.transparent,
      elevation: 0,
      systemOverlayStyle: SystemUiOverlayStyle(
        statusBarBrightness: Brightness.dark,
        statusBarIconBrightness: Brightness.light,
      ),
    ),
    // Ensure fonts are available globally
    fontFamily: 'InstrumentSans', // fallback only — prefer GoogleFonts in styles
    useMaterial3: true,
  );
}
```

---

## PART 3 — COMPONENT TRANSLATION TABLE

### 3.1 — Web Element → Flutter Widget Mapping

| Web Element | CSS Class | Flutter Widget | File |
|---|---|---|---|
| `body` atmosphere | `body::before / ::after` | `OrdrFloorBackground` | `widgets/floor_background.dart` |
| Nav pill (top) | `.nav` | `OrdrTopNav` | `widgets/nav/top_nav.dart` |
| Nav pill (mobile bottom) | `.mobile-bottom-nav` | `OrdrBottomNav` | `widgets/nav/bottom_nav.dart` |
| Primary button | `.btn-primary` | `OrdrPrimaryButton` | `widgets/buttons/primary_button.dart` |
| Secondary button | `.btn-secondary` | `OrdrSecondaryButton` | `widgets/buttons/secondary_button.dart` |
| Ghost button | `.btn-ghost` | `OrdrGhostButton` | `widgets/buttons/ghost_button.dart` |
| Standard card | `.bento` | `OrdrCard` | `widgets/cards/card.dart` |
| Accent card | `.bento.feature-main` | `OrdrAccentCard` | `widgets/cards/accent_card.dart` |
| Hero badge | `.hero-badge` | `OrdrHeroBadge` | `widgets/badges/hero_badge.dart` |
| Status badge | `.badge` | `OrdrStatusBadge` | `widgets/badges/status_badge.dart` |
| AI chip | `.ai-chip` | `OrdrAiChip` | `widgets/badges/ai_chip.dart` |
| Section eyebrow | `.section-eyebrow` | `OrdrEyebrow` | `widgets/typography/eyebrow.dart` |
| Progress bar | `.progress-bar` | `OrdrProgressBar` | `widgets/indicators/progress_bar.dart` |
| Skeleton | `.skeleton` | `OrdrSkeleton` | `widgets/indicators/skeleton.dart` |
| List row | `.list-row` | `OrdrListRow` | `widgets/lists/list_row.dart` |
| Timer orb | `.timer-orb` | `OrdrTimerOrb` | `widgets/decorative/timer_orb.dart` |
| Ticker belt | `.ticker-wrap` | `OrdrTicker` | `widgets/decorative/ticker.dart` |
| Reveal animation | `.reveal` | `OrdrReveal` | `widgets/animation/reveal.dart` |
| Hero section | `.hero` | `LandingHeroSection` | `features/landing/hero_section.dart` |
| Stats bar | `.stats-bar` | `LandingStatsBar` | `features/landing/stats_bar.dart` |
| Bento grid | `.bento-grid` | `LandingBentoGrid` | `features/landing/bento_grid.dart` |
| How it works | `.how-it-works` | `LandingHowItWorks` | `features/landing/how_it_works.dart` |
| Testimonials | `.testimonials` | `LandingTestimonials` | `features/landing/testimonials.dart` |
| Pricing | `.pricing` | `LandingPricing` | `features/landing/pricing.dart` |
| CTA section | `.cta-section` | `LandingCta` | `features/landing/cta_section.dart` |
| Footer | `.footer` | `LandingFooter` | `features/landing/footer.dart` |

### 3.2 — Critical Component Recipes

#### OrdrFloorBackground
The three-layer atmospheric background. Must wrap every Scaffold.

```dart
// lib/widgets/floor_background.dart
import 'package:flutter/material.dart';
import '../core/theme/tokens.dart';

class OrdrFloorBackground extends StatelessWidget {
  final Widget child;
  const OrdrFloorBackground({super.key, required this.child});

  @override
  Widget build(BuildContext context) {
    return Container(
      decoration: const BoxDecoration(
        color: OrdrTokens.floorBase,
      ),
      child: Stack(
        children: [
          // Layer 1: Top-left gold radial
          Positioned.fill(
            child: DecoratedBox(
              decoration: BoxDecoration(
                gradient: RadialGradient(
                  center: const Alignment(-0.70, -0.80),
                  radius: 1.0,
                  colors: [
                    OrdrTokens.accent.withOpacity(0.06),
                    Colors.transparent,
                  ],
                ),
              ),
            ),
          ),
          // Layer 2: Bottom-right gold radial
          Positioned.fill(
            child: DecoratedBox(
              decoration: BoxDecoration(
                gradient: RadialGradient(
                  center: const Alignment(0.70, 0.80),
                  radius: 0.9,
                  colors: [
                    OrdrTokens.accent.withOpacity(0.04),
                    Colors.transparent,
                  ],
                ),
              ),
            ),
          ),
          // Layer 3: Content
          child,
        ],
      ),
    );
  }
}
```

**Scaffold pattern — use everywhere:**
```dart
Scaffold(
  backgroundColor: Colors.transparent,
  body: OrdrFloorBackground(
    child: /* your content */,
  ),
)
```

#### OrdrPrimaryButton
```dart
// lib/widgets/buttons/primary_button.dart
import 'package:flutter/material.dart';
import '../../core/theme/tokens.dart';
import '../../core/theme/typography.dart';

class OrdrPrimaryButton extends StatefulWidget {
  final String label;
  final VoidCallback? onPressed;
  final Widget? icon;
  final double? width; // null = intrinsic

  const OrdrPrimaryButton({
    super.key,
    required this.label,
    this.onPressed,
    this.icon,
    this.width,
  });

  @override
  State<OrdrPrimaryButton> createState() => _OrdrPrimaryButtonState();
}

class _OrdrPrimaryButtonState extends State<OrdrPrimaryButton>
    with SingleTickerProviderStateMixin {
  late final AnimationController _ctrl;
  late final Animation<double> _scale;
  late final Animation<List<BoxShadow>> _shadow;

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(
      vsync: this,
      duration: OrdrTokens.durationFast,
    );
    _scale = Tween<double>(begin: 1.0, end: 0.97).animate(
      CurvedAnimation(parent: _ctrl, curve: Curves.easeOut),
    );
  }

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTapDown: (_) => _ctrl.forward(),
      onTapUp: (_) {
        _ctrl.reverse();
        widget.onPressed?.call();
      },
      onTapCancel: () => _ctrl.reverse(),
      child: AnimatedBuilder(
        animation: _ctrl,
        builder: (_, child) => Transform.scale(
          scale: _scale.value,
          child: child,
        ),
        child: Container(
          width: widget.width,
          padding: const EdgeInsets.symmetric(horizontal: 22, vertical: 12),
          decoration: BoxDecoration(
            gradient: OrdrTokens.accentGradient,
            borderRadius: BorderRadius.circular(OrdrTokens.radiusPill),
            boxShadow: OrdrTokens.shadowAccentGlow,
          ),
          child: Row(
            mainAxisSize: widget.width != null ? MainAxisSize.max : MainAxisSize.min,
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              if (widget.icon != null) ...[
                widget.icon!,
                const SizedBox(width: 8),
              ],
              Text(widget.label, style: OrdrTextStyles.button),
            ],
          ),
        ),
      ),
    );
  }
}
```

#### OrdrCard (Standard Bento Card)
```dart
// lib/widgets/cards/card.dart
import 'package:flutter/material.dart';
import '../../core/theme/tokens.dart';

class OrdrCard extends StatefulWidget {
  final Widget child;
  final EdgeInsetsGeometry? padding;
  final bool hoverable;

  const OrdrCard({
    super.key,
    required this.child,
    this.padding,
    this.hoverable = true,
  });

  @override
  State<OrdrCard> createState() => _OrdrCardState();
}

class _OrdrCardState extends State<OrdrCard>
    with SingleTickerProviderStateMixin {
  late final AnimationController _ctrl;
  late final Animation<double> _elevation;

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(
      vsync: this,
      duration: OrdrTokens.durationMedium,
    );
    _elevation = Tween<double>(begin: 0, end: 1).animate(
      CurvedAnimation(parent: _ctrl, curve: OrdrTokens.springMedium),
    );
  }

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return MouseRegion(
      // hoverable on desktop
      onEnter: widget.hoverable ? (_) => _ctrl.forward() : null,
      onExit: widget.hoverable ? (_) => _ctrl.reverse() : null,
      child: AnimatedBuilder(
        animation: _elevation,
        builder: (_, child) {
          return Transform.translate(
            offset: Offset(0, -3 * _elevation.value), // translateY(-3px)
            child: Container(
              decoration: BoxDecoration(
                color: OrdrTokens.surfaceCard,
                borderRadius: BorderRadius.circular(OrdrTokens.radiusCard),
                border: Border.all(
                  color: Color.lerp(
                    OrdrTokens.borderSubtle,
                    OrdrTokens.accentBorder,
                    _elevation.value,
                  )!,
                  width: 1,
                ),
                boxShadow: [
                  ...OrdrTokens.shadowSoft.map((s) => BoxShadow(
                    color: s.color,
                    blurRadius: s.blurRadius + (20 * _elevation.value),
                    offset: s.offset,
                  )),
                ],
              ),
              child: ClipRRect(
                borderRadius: BorderRadius.circular(OrdrTokens.radiusCard),
                child: child,
              ),
            ),
          );
        },
        child: Padding(
          padding: widget.padding ?? const EdgeInsets.all(OrdrTokens.cardPaddingMin),
          child: widget.child,
        ),
      ),
    );
  }
}
```

#### OrdrReveal (Scroll-triggered entrance animation)
```dart
// lib/widgets/animation/reveal.dart
import 'package:flutter/material.dart';
import '../../core/theme/tokens.dart';

class OrdrReveal extends StatefulWidget {
  final Widget child;
  final Duration delay;

  const OrdrReveal({
    super.key,
    required this.child,
    this.delay = Duration.zero,
  });

  @override
  State<OrdrReveal> createState() => _OrdrRevealState();
}

class _OrdrRevealState extends State<OrdrReveal>
    with SingleTickerProviderStateMixin {
  late final AnimationController _ctrl;
  late final Animation<double> _opacity;
  late final Animation<Offset> _slide;

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(
      vsync: this,
      duration: OrdrTokens.durationEntrance,
    );
    _opacity = Tween<double>(begin: 0, end: 1).animate(
      CurvedAnimation(parent: _ctrl, curve: OrdrTokens.springMedium),
    );
    _slide = Tween<Offset>(
      begin: const Offset(0, 0.05), // 28px on ~560px screen ≈ 5%
      end: Offset.zero,
    ).animate(CurvedAnimation(parent: _ctrl, curve: OrdrTokens.springMedium));
  }

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  void _trigger() {
    if (!_ctrl.isCompleted) {
      Future.delayed(widget.delay, () {
        if (mounted) _ctrl.forward();
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    return VisibilityDetector(
      key: Key('reveal_${widget.hashCode}'),
      onVisibilityChanged: (info) {
        if (info.visibleFraction > 0.1) _trigger();
      },
      child: AnimatedBuilder(
        animation: _ctrl,
        builder: (_, child) => FadeTransition(
          opacity: _opacity,
          child: SlideTransition(position: _slide, child: child),
        ),
        child: widget.child,
      ),
    );
  }
}
// Note: Add visibility_detector package to pubspec.yaml
```

#### OrdrSkeleton
```dart
// lib/widgets/indicators/skeleton.dart
import 'package:flutter/material.dart';
import 'package:shimmer/shimmer.dart';
import '../../core/theme/tokens.dart';

class OrdrSkeleton extends StatelessWidget {
  final double? width;
  final double height;
  final double borderRadius;

  const OrdrSkeleton({
    super.key,
    this.width,
    this.height = 16,
    this.borderRadius = 8,
  });

  @override
  Widget build(BuildContext context) {
    // Use palette tones — NEVER grey
    return Shimmer.fromColors(
      baseColor: OrdrTokens.surfaceCard,
      highlightColor: OrdrTokens.surfaceElevated,
      child: Container(
        width: width,
        height: height,
        decoration: BoxDecoration(
          color: OrdrTokens.surfaceCard,
          borderRadius: BorderRadius.circular(borderRadius),
        ),
      ),
    );
  }
}

// Usage:
// OrdrSkeleton(width: 200, height: 20)           // text line
// OrdrSkeleton(height: 140, borderRadius: 22)    // card
```

---

## PART 4 — PROJECT FILE STRUCTURE

The agent MUST create this exact directory structure. No exceptions.

```
lib/
├── main.dart                          # App entry + Marionette init
├── app.dart                           # MaterialApp + router
├── core/
│   ├── theme/
│   │   ├── tokens.dart                # ALL design tokens
│   │   ├── typography.dart            # ALL text styles
│   │   └── theme.dart                 # ThemeData construction
│   ├── router/
│   │   └── router.dart                # GoRouter config
│   └── constants/
│       └── strings.dart               # ALL user-facing strings
├── features/
│   ├── landing/
│   │   ├── landing_screen.dart        # Screen scaffold + scroll
│   │   ├── hero_section.dart
│   │   ├── stats_bar.dart
│   │   ├── bento_grid.dart
│   │   ├── how_it_works.dart
│   │   ├── testimonials.dart
│   │   ├── pricing.dart
│   │   ├── cta_section.dart
│   │   └── footer.dart
│   ├── dashboard/
│   │   └── dashboard_screen.dart
│   ├── cases/
│   │   ├── cases_list_screen.dart
│   │   ├── case_detail_screen.dart
│   │   └── new_case_screen.dart
│   ├── hearings/
│   │   ├── hearings_list_screen.dart
│   │   ├── hearing_detail_screen.dart
│   │   └── new_hearing_screen.dart
│   ├── documents/
│   │   ├── upload_screen.dart
│   │   ├── document_list_screen.dart
│   │   └── document_detail_screen.dart
│   ├── auth/
│   │   ├── landing_screen.dart
│   │   └── login_screen.dart
│   └── profile/
│       └── profile_screen.dart
└── widgets/
    ├── floor_background.dart          # Atmospheric 3-layer bg
    ├── nav/
    │   ├── top_nav.dart               # Desktop/tablet floating pill
    │   └── bottom_nav.dart            # Mobile floating pill
    ├── buttons/
    │   ├── primary_button.dart
    │   ├── secondary_button.dart
    │   └── ghost_button.dart
    ├── cards/
    │   ├── card.dart                  # OrdrCard
    │   └── accent_card.dart           # OrdrAccentCard
    ├── badges/
    │   ├── hero_badge.dart
    │   ├── status_badge.dart
    │   └── ai_chip.dart
    ├── typography/
    │   └── eyebrow.dart              # Section eyebrow w/ ruled lines
    ├── indicators/
    │   ├── progress_bar.dart
    │   └── skeleton.dart
    ├── lists/
    │   └── list_row.dart
    ├── decorative/
    │   ├── timer_orb.dart
    │   └── ticker.dart               # Horizontal scroll ticker
    └── animation/
        └── reveal.dart               # Scroll-triggered entrance

test/
├── golden/                            # Screenshot golden tests
│   ├── landing_hero_test.dart
│   ├── buttons_test.dart
│   └── cards_test.dart
└── widget/
    └── component_tests.dart

integration_test/
└── visual_loop_test.dart              # Marionette visual loop tests
```

---

## PART 5 — THE VISUAL INSPECTION LOOP

This is the core feedback cycle. Run it after every significant change.
Repeat until all checks pass. Never commit without passing this loop.

### 5.1 — FULL VISUAL LOOP (Marionette + Dart MCP available)

```
LOOP START
  │
  ├─ STEP 1: LAUNCH
  │   flutter run --debug --target lib/main.dart
  │   Wait for: "A Dart VM Service on ... is available at: ws://..."
  │   Store VM URI.
  │
  ├─ STEP 2: SCREENSHOT CURRENT STATE
  │   [MCP: marionette.screenshot]
  │   Save as: screenshots/current_[timestamp].png
  │   Load web reference: screenshots/web_reference_[route].png
  │
  ├─ STEP 3: VISUAL DIFF ANALYSIS
  │   For each region of the screenshot, compare to web reference:
  │   □ Background atmosphere — does it have the gold radial glow?
  │   □ Typography — is Cormorant Garamond rendering for headlines?
  │   □ Accent color — is #C9A84C appearing in correct places only?
  │   □ Card surfaces — correct translucency and border?
  │   □ Button gradient — does the gold gradient match?
  │   □ Spacing — padding ≥28px inside cards?
  │   □ Radius — 22px on cards, pill on buttons?
  │
  ├─ STEP 4: WIDGET TREE INSPECTION
  │   [MCP: marionette.getWidgetTree or dart_mcp.analyzeFile]
  │   Check that widget hierarchy matches expected structure.
  │   Look for:
  │   □ OrdrFloorBackground wraps entire screen
  │   □ OrdrCard/OrdrAccentCard used (not plain Container)
  │   □ OrdrPrimaryButton used (not ElevatedButton with style)
  │   □ GoogleFonts.cormorantGaramond for headlines
  │   □ GoogleFonts.instrumentSans for UI
  │
  ├─ STEP 5: IDENTIFY DISCREPANCIES
  │   For each discrepancy, classify:
  │   [TOKEN]    → Wrong token value used
  │   [WIDGET]   → Wrong widget class used
  │   [LAYOUT]   → Wrong flex/grid structure
  │   [ANIM]     → Missing or wrong animation
  │   [SPACING]  → Wrong padding/margin
  │   [FONT]     → Wrong font or weight
  │
  ├─ STEP 6: FIX — ONE DISCREPANCY AT A TIME
  │   Fix only the highest-priority issue.
  │   Priority order: TOKEN > FONT > LAYOUT > WIDGET > SPACING > ANIM
  │   Apply fix. Save file.
  │
  ├─ STEP 7: HOT RELOAD
  │   [MCP: dart_mcp.hotReload or shell: 'r' to flutter process]
  │   Wait 500ms for UI to settle.
  │
  ├─ STEP 8: RE-SCREENSHOT
  │   [MCP: marionette.screenshot]
  │   Save as: screenshots/fix_[N]_[timestamp].png
  │
  ├─ STEP 9: VERIFY FIX
  │   Compare new screenshot to web reference.
  │   If discrepancy resolved: mark ✅, move to next
  │   If discrepancy remains: escalate fix, return to STEP 6
  │
  ├─ STEP 10: INTERACTION TEST
  │   [MCP: marionette.tap element="Get Started Free"]
  │   Verify: button animates (scale 0.97), navigates correctly
  │   [MCP: marionette.tap element="Start Free Trial"]
  │   [MCP: marionette.scroll direction=down amount=500]
  │   Screenshot each state.
  │
  └─ STEP 11: LOOP CONDITION
      If all visual checks pass AND all interactions work:
        → LOOP COMPLETE. Proceed to Part 6.
      Else:
        → Return to STEP 2.
      Maximum iterations: 20 before escalating to user.
```

### 5.2 — MCP Tool Reference

```dart
// In main.dart — Marionette initialization (DEBUG ONLY)
import 'package:flutter/foundation.dart';
import 'package:marionette_flutter/marionette_flutter.dart';

void main() {
  if (kDebugMode) {
    MarionetteBinding.ensureInitialized(
      MarionetteConfiguration(
        // Register all custom interactive widgets
        isInteractiveWidget: (type) => [
          OrdrPrimaryButton,
          OrdrSecondaryButton,
          OrdrGhostButton,
          OrdrCard,
          OrdrAccentCard,
          OrdrListRow,
        ].contains(type),
        // Extract text labels for targeting
        extractText: (widget) {
          if (widget is OrdrPrimaryButton) return widget.label;
          if (widget is OrdrSecondaryButton) return widget.label;
          if (widget is OrdrGhostButton) return widget.label;
          return null;
        },
        maxScreenshotSize: const Size(1440, 2560),
      ),
    );
  } else {
    WidgetsFlutterBinding.ensureInitialized();
  }
  runApp(const OrdrApp());
}
```

### 5.3 — ADB Fallback Screenshots (no Marionette)

If Marionette is unavailable, use ADB:

```bash
# Get connected device ID
adb devices

# Screenshot (Android)
adb -s DEVICE_ID shell screencap -p /sdcard/screen.png
adb -s DEVICE_ID pull /sdcard/screen.png screenshots/screen_$(date +%s).png

# Screenshot (iOS Simulator)
xcrun simctl io booted screenshot screenshots/screen_$(date +%s).png

# View logcat (filter for Flutter)
adb logcat -s flutter

# Hot reload via flutter CLI
# (keep flutter run process alive in background, send 'r' to stdin)
echo "r" | flutter run --debug
```

### 5.4 — Golden Test Approach (no live MCP)

```dart
// test/golden/landing_hero_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:golden_toolkit/golden_toolkit.dart';
import 'package:ordr_app/features/landing/hero_section.dart';
import 'package:ordr_app/core/theme/theme.dart';

void main() {
  testGoldens('LandingHeroSection - desktop', (tester) async {
    await loadAppFonts(); // loads google fonts in test
    await tester.pumpWidgetBuilder(
      const LandingHeroSection(),
      surfaceSize: const Size(1440, 900),
      wrapper: materialAppWrapper(theme: ordrTheme()),
    );
    await screenMatchesGolden(tester, 'landing_hero_desktop');
  });

  testGoldens('LandingHeroSection - mobile', (tester) async {
    await loadAppFonts();
    await tester.pumpWidgetBuilder(
      const LandingHeroSection(),
      surfaceSize: const Size(390, 844),
      wrapper: materialAppWrapper(theme: ordrTheme()),
    );
    await screenMatchesGolden(tester, 'landing_hero_mobile');
  });
}
```

Run golden tests:
```bash
# Generate baselines (first run)
flutter test --update-goldens test/golden/

# Compare against baselines (CI)
flutter test test/golden/
```

---

## PART 6 — RESPONSIVE LAYOUT SYSTEM

### 6.1 — Breakpoint Detection

```dart
// lib/core/layout/breakpoints.dart
import 'package:flutter/material.dart';

enum OrdrBreakpoint { mobile, tablet, desktop }

extension BreakpointExtension on BuildContext {
  double get screenWidth => MediaQuery.of(this).size.width;

  OrdrBreakpoint get breakpoint {
    final w = screenWidth;
    if (w < 768)  return OrdrBreakpoint.mobile;
    if (w < 1200) return OrdrBreakpoint.tablet;
    return OrdrBreakpoint.desktop;
  }

  bool get isMobile  => breakpoint == OrdrBreakpoint.mobile;
  bool get isTablet  => breakpoint == OrdrBreakpoint.tablet;
  bool get isDesktop => breakpoint == OrdrBreakpoint.desktop;
}

/// Responsive builder — mirrors CSS media queries exactly
class OrdrResponsive extends StatelessWidget {
  final Widget Function(BuildContext) mobile;
  final Widget Function(BuildContext)? tablet;
  final Widget Function(BuildContext) desktop;

  const OrdrResponsive({
    super.key,
    required this.mobile,
    this.tablet,
    required this.desktop,
  });

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(builder: (ctx, constraints) {
      if (constraints.maxWidth < 768) return mobile(ctx);
      if (constraints.maxWidth < 1200) return (tablet ?? desktop)(ctx);
      return desktop(ctx);
    });
  }
}
```

### 6.2 — Responsive Navigation

```dart
// lib/widgets/nav/adaptive_nav.dart
import 'package:flutter/material.dart';
import '../../core/layout/breakpoints.dart';
import 'top_nav.dart';
import 'bottom_nav.dart';

class OrdrAdaptiveNav extends StatelessWidget {
  final Widget child;
  final int currentIndex;
  final ValueChanged<int> onNavTap;

  const OrdrAdaptiveNav({
    super.key,
    required this.child,
    required this.currentIndex,
    required this.onNavTap,
  });

  @override
  Widget build(BuildContext context) {
    return OrdrResponsive(
      mobile: (_) => Scaffold(
        backgroundColor: Colors.transparent,
        body: child,
        bottomNavigationBar: OrdrBottomNav(
          currentIndex: currentIndex,
          onTap: onNavTap,
        ),
      ),
      desktop: (_) => Stack(
        children: [
          child,
          Positioned(
            top: 20,
            left: 0,
            right: 0,
            child: Center(child: OrdrTopNav(currentIndex: currentIndex)),
          ),
        ],
      ),
    );
  }
}
```

### 6.3 — Bento Grid in Flutter

```dart
// lib/widgets/cards/bento_grid.dart
import 'package:flutter/material.dart';
import '../../core/layout/breakpoints.dart';
import '../../core/theme/tokens.dart';

/// Maps CSS 12-column bento grid to Flutter
/// Desktop: full grid with span classes
/// Tablet: 6-column (each span halved)
/// Mobile: single column
class OrdrBentoGrid extends StatelessWidget {
  final List<BentoItem> items;
  const OrdrBentoGrid({super.key, required this.items});

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(builder: (ctx, constraints) {
      final w = constraints.maxWidth;

      if (w < 768) {
        // Mobile: single column, no spans
        return Column(
          children: items.map((item) => Padding(
            padding: const EdgeInsets.only(bottom: 14),
            child: item.child,
          )).toList(),
        );
      }

      // Tablet + Desktop: use Wrap with proportional widths
      final cols = w < 1200 ? 6 : 12;
      final cellW = (w - (14 * (cols - 1))) / cols;

      return Wrap(
        spacing: 14,
        runSpacing: 14,
        children: items.map((item) {
          final span = w < 1200
              ? (item.tabletSpan ?? (item.desktopSpan / 2).ceil())
              : item.desktopSpan;
          final itemWidth = (cellW * span) + (14 * (span - 1));
          return SizedBox(
            width: itemWidth,
            child: item.child,
          );
        }).toList(),
      );
    });
  }
}

class BentoItem {
  final int desktopSpan;    // out of 12
  final int? tabletSpan;    // out of 6 (defaults to desktopSpan/2)
  final Widget child;

  const BentoItem({
    required this.desktopSpan,
    this.tabletSpan,
    required this.child,
  });
}
```

---

## PART 7 — ANIMATION IMPLEMENTATION

### 7.1 — Page Entrance Sequence

```dart
// lib/widgets/animation/page_entrance.dart
import 'package:flutter/material.dart';
import '../../core/theme/tokens.dart';

/// Orchestrated page entrance — mirrors CSS staggered fadeUp
/// Usage: Wrap your screen's initState with this controller
class OrdrPageEntrance extends StatefulWidget {
  final Widget Function(BuildContext, Animation<double>) builder;

  const OrdrPageEntrance({super.key, required this.builder});

  @override
  State<OrdrPageEntrance> createState() => _OrdrPageEntranceState();
}

class _OrdrPageEntranceState extends State<OrdrPageEntrance>
    with SingleTickerProviderStateMixin {
  late final AnimationController _ctrl;

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 1200),
    );
    // Start after frame build — mirrors CSS animation-delay
    WidgetsBinding.instance.addPostFrameCallback((_) => _ctrl.forward());
  }

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return widget.builder(context, _ctrl);
  }
}

/// Staggered animation helper — mirrors animation-delay per element
/// delayFraction: 0.0 to 1.0 (fraction of total controller duration)
Animation<double> stagger(
  AnimationController ctrl,
  double delayFraction,
  double durationFraction,
) {
  return CurvedAnimation(
    parent: ctrl,
    curve: Interval(
      delayFraction,
      (delayFraction + durationFraction).clamp(0.0, 1.0),
      curve: OrdrTokens.springMedium,
    ),
  );
}

// CSS delay mapping → stagger fractions (total=1200ms):
// 200ms delay  → delayFraction = 0.17
// 350ms delay  → delayFraction = 0.29
// 500ms delay  → delayFraction = 0.42
// 650ms delay  → delayFraction = 0.54
// 800ms delay  → delayFraction = 0.67
// 900ms delay  → delayFraction = 0.75

// Usage in hero section:
/*
OrdrPageEntrance(
  builder: (ctx, ctrl) => Column(children: [
    FadeTransition(
      opacity: stagger(ctrl, 0.17, 0.25),
      child: SlideTransition(
        position: Tween(begin: Offset(0, 0.1), end: Offset.zero)
            .animate(stagger(ctrl, 0.17, 0.25)),
        child: const OrdrHeroBadge(),
      ),
    ),
    FadeTransition(
      opacity: stagger(ctrl, 0.29, 0.30),
      child: const HeroHeadline(),
    ),
  ]),
)
*/
```

### 7.2 — Floating Orb Animation

```dart
// lib/widgets/decorative/timer_orb.dart
import 'dart:math' as math;
import 'package:flutter/material.dart';
import '../../core/theme/tokens.dart';
import '../../core/theme/typography.dart';

class OrdrTimerOrb extends StatefulWidget {
  final String number;
  final String unit;

  const OrdrTimerOrb({
    super.key,
    required this.number,
    required this.unit,
  });

  @override
  State<OrdrTimerOrb> createState() => _OrdrTimerOrbState();
}

class _OrdrTimerOrbState extends State<OrdrTimerOrb>
    with SingleTickerProviderStateMixin {
  late final AnimationController _ctrl;
  late final Animation<double> _float;
  late final Animation<double> _rotate;

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 4),
    )..repeat(reverse: true);

    _float = Tween<double>(begin: 0, end: -8).animate(
      CurvedAnimation(parent: _ctrl, curve: Curves.easeInOut),
    );
    _rotate = Tween<double>(begin: 0, end: 3 * math.pi / 180).animate(
      CurvedAnimation(parent: _ctrl, curve: Curves.easeInOut),
    );
  }

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _ctrl,
      builder: (_, child) => Transform.translate(
        offset: Offset(0, _float.value),
        child: Transform.rotate(
          angle: _rotate.value,
          child: child,
        ),
      ),
      child: Container(
        width: 80,
        height: 80,
        decoration: BoxDecoration(
          gradient: OrdrTokens.accentGradientSimple,
          shape: BoxShape.circle,
          boxShadow: const [
            BoxShadow(
              color: Color(0x80C9A84C),
              blurRadius: 32,
              offset: Offset(0, 8),
            ),
          ],
        ),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text(
              widget.number,
              style: OrdrTextStyles.timerDisplay.copyWith(
                fontSize: 24,
                color: OrdrTokens.onAccent,
                fontWeight: FontWeight.w600,
              ),
            ),
            Text(
              widget.unit.toUpperCase(),
              style: OrdrTextStyles.label.copyWith(
                color: OrdrTokens.onAccent.withOpacity(0.70),
                letterSpacing: 1.0,
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

### 7.3 — Ticker Belt Animation

```dart
// lib/widgets/decorative/ticker.dart
import 'package:flutter/material.dart';
import '../../core/theme/tokens.dart';
import '../../core/theme/typography.dart';

class OrdrTicker extends StatefulWidget {
  final List<String> items;
  const OrdrTicker({super.key, required this.items});

  @override
  State<OrdrTicker> createState() => _OrdrTickerState();
}

class _OrdrTickerState extends State<OrdrTicker>
    with SingleTickerProviderStateMixin {
  late final AnimationController _ctrl;

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 24),
    )..repeat();
  }

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final doubled = [...widget.items, ...widget.items]; // duplicate for seamless loop
    return Container(
      height: 40,
      decoration: BoxDecoration(
        color: OrdrTokens.surfaceMuted,
        border: Border.symmetric(
          horizontal: BorderSide(
            color: Colors.white.withOpacity(0.05),
            width: 1,
          ),
        ),
      ),
      child: AnimatedBuilder(
        animation: _ctrl,
        builder: (_, __) {
          return LayoutBuilder(builder: (ctx, constraints) {
            return FractionalTranslation(
              translation: Offset(-_ctrl.value, 0),
              child: Row(
                children: doubled.map((item) => _TickerItem(text: item)).toList(),
              ),
            );
          });
        },
      ),
    );
  }
}

class _TickerItem extends StatelessWidget {
  final String text;
  const _TickerItem({required this.text});

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 16),
      child: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          Text(text, style: OrdrTextStyles.caption.copyWith(letterSpacing: 0.6)),
          const SizedBox(width: 16),
          Text('◆', style: TextStyle(color: OrdrTokens.accent.withOpacity(0.5), fontSize: 8)),
        ],
      ),
    );
  }
}
```

---

## PART 8 — MIGRATION EXECUTION ORDER

Follow this exact sequence. Do not skip steps. Check off each one.

```
PHASE 1 — FOUNDATION (do before any screen work)
□ 1.1  Create Flutter project with correct package name
□ 1.2  Add all dependencies to pubspec.yaml
□ 1.3  Create lib/core/theme/tokens.dart with ALL token values
□ 1.4  Create lib/core/theme/typography.dart with ALL text styles
□ 1.5  Create lib/core/theme/theme.dart with ThemeData
□ 1.6  Create lib/widgets/floor_background.dart
□ 1.7  Set up MCP tools (Marionette init in main.dart)
□ 1.8  Run flutter run — confirm app launches with dark floor background
□ 1.9  Run VISUAL LOOP STEP 1-3 on empty app — confirm atmosphere renders

PHASE 2 — CORE WIDGETS
□ 2.1  OrdrPrimaryButton — build, test press animation, screenshot
□ 2.2  OrdrSecondaryButton — build, test
□ 2.3  OrdrGhostButton — build, test
□ 2.4  OrdrCard — build, test hover lift
□ 2.5  OrdrAccentCard — build, confirm gold gradient
□ 2.6  OrdrHeroBadge — build, confirm pulse animation
□ 2.7  OrdrStatusBadge — build (positive, pending, neutral variants)
□ 2.8  OrdrAiChip — build
□ 2.9  OrdrSkeleton — build, confirm palette-toned shimmer (not grey)
□ 2.10 OrdrProgressBar — build animated + static variants
□ 2.11 OrdrEyebrow — build with ruled lines
□ 2.12 OrdrReveal — build, test scroll trigger
□ 2.13 Run VISUAL LOOP on component gallery screen

PHASE 3 — NAVIGATION
□ 3.1  OrdrTopNav — desktop floating pill with logo + links + CTA
□ 3.2  OrdrBottomNav — mobile 5-item floating pill with center orb
□ 3.3  OrdrAdaptiveNav — responsive shell
□ 3.4  Set up GoRouter with all routes from USER_FLOW_INTERACTION_MAP
□ 3.5  Test navigation on desktop, tablet, mobile
□ 3.6  Run VISUAL LOOP on navigation across breakpoints

PHASE 4 — LANDING PAGE SECTIONS (in scroll order)
□ 4.1  LandingHeroSection — badge, headline, sub, CTAs, trust row, mock
□ 4.2  OrdrTicker — belt with 8 items, doubled for loop
□ 4.3  LandingStatsBar — 4 stats with dividers
□ 4.4  LandingBentoGrid — all 9 bento cards in correct layout
□ 4.5  LandingHowItWorks — 3 steps with dividers
□ 4.6  LandingDocumentIntel — 2-col with pipeline mock
□ 4.7  LandingTestimonials — 3 cards with serif quotes
□ 4.8  LandingPricing — 3 plan cards, featured with accent
□ 4.9  LandingCta — centered with radial glow
□ 4.10 LandingFooter — 4-col grid + bottom bar
□ 4.11 Run FULL VISUAL LOOP on complete landing page
□ 4.12 Test scroll from top to bottom — confirm all reveals trigger

PHASE 5 — APP SCREENS (per USER_FLOW_INTERACTION_MAP)
□ 5.1  /login screen
□ 5.2  /onboarding screen
□ 5.3  /dashboard screen
□ 5.4  /cases list screen
□ 5.5  /cases/new multi-step flow
□ 5.6  /cases/[id] detail screen
□ 5.7  /hearings list screen
□ 5.8  /hearings/[id] detail screen
□ 5.9  /documents/upload screen
□ 5.10 /documents/[id] detail screen
□ 5.11 /reminders screen
□ 5.12 /search screen
□ 5.13 /profile screen
□ 5.14 /plans screen

PHASE 6 — QA & VISUAL VERIFICATION
□ 6.1  Run full visual loop on every screen at 3 breakpoints
□ 6.2  Run golden tests — generate baselines
□ 6.3  Test all 5 interaction states on every screen (loading/empty/error/success/processing)
□ 6.4  Test prefers-reduced-motion (disable all animations)
□ 6.5  Test at smallest supported size (360×640 Android)
□ 6.6  Test at largest supported size (1440px desktop web / iPad Pro)
□ 6.7  Confirm no hardcoded colors outside tokens.dart
□ 6.8  Confirm no hardcoded strings outside strings.dart
□ 6.9  Run flutter analyze — zero warnings
□ 6.10 Run flutter test — all golden tests pass
```

---

## PART 9 — COMMON MIGRATION PITFALLS & FIXES

| Problem | Symptom | Fix |
|---|---|---|
| Background appears solid black | No atmospheric glow visible | Ensure `OrdrFloorBackground` wraps `Scaffold.body`, not `Scaffold` |
| Fonts not loading | System font appears instead of Cormorant/Instrument | Call `GoogleFonts.config.allowRuntimeFetching = true` in main() or bundle fonts |
| Card has default Material elevation | White shadow or grey surface | Set `elevation: 0` and manage with `BoxDecoration.boxShadow` manually |
| Button doesn't scale on press | No tactile feedback | Use `GestureDetector` + `AnimationController`, not `ElevatedButton` |
| Gold color appears wrong | Orange or dull instead of `#C9A84C` | Verify `Color(0xFFC9A84C)` — Flutter ARGB order (Alpha first) |
| Ticker doesn't loop seamlessly | Jump at end | Ensure items list is EXACTLY doubled and animation translates by exactly 50% |
| Labels not uppercase | Mixed case text | Always call `OrdrTextStyles.labelText(str)` wrapper or `.toUpperCase()` |
| Shadows look harsh | High-contrast black box shadows | Use `rgba` alpha values matching web tokens exactly |
| Reveal not triggering | Elements stay invisible after scroll | Add `visibility_detector` package; confirm `VisibilityDetector` key is unique |
| Bento grid breaks on tablet | Cards overflow or misalign | Use `LayoutBuilder` + `Wrap` pattern from Part 6.3, not fixed `GridView` |
| Skeleton is grey | Generic grey shimmer shows | Use `shimmer` package with `baseColor: OrdrTokens.surfaceCard` |
| Border appears on card | Visible card outline | CSS `border: 1px solid rgba(255,255,255,0.07)` → `Border.all(color: Color(0x12FFFFFF), width: 1)` |
| Animation is janky | Stutter during card entrance | Switch to `transform` + `opacity` only. Never animate `height`, `width`, `padding` |
| Hot reload resets animations | Entrance plays again on every change | Use `mounted` guard in initState to only play once |

---

## PART 10 — VISUAL LOOP CHECKLIST (Per-screen)

Run this for EVERY screen before marking it complete.

```
VISUAL ACCURACY
□ Background: atmospheric gold radial glow visible (not solid black)
□ Navigation: correct variant for breakpoint (top/bottom)
□ Typography: Cormorant Garamond for headlines, Instrument Sans for UI
□ Accent: gold (#C9A84C) appears ONLY as full fill or tiny signal
□ Cards: correct surface color, 22px radius, 1px subtle border
□ Buttons: gold gradient on primary, neutral on secondary
□ Spacing: no element has internal padding < 28px
□ Labels: all uppercase, 10-11px, wide letter-spacing

ANIMATION ACCURACY
□ Entrance: elements stagger in on load (not all at once)
□ Scroll: reveal animations trigger on scroll
□ Hover: cards lift 3px on hover (desktop)
□ Press: buttons scale to 0.97 on press
□ Floating: timer orb (if present) floats up/down continuously
□ Ticker: scrolls continuously at correct speed (24s cycle)
□ Skeletons: shimmer uses palette tones, not grey

INTERACTION ACCURACY
□ Primary CTA tappable → correct navigation
□ All nav items tappable → correct routes
□ Back navigation works
□ Form inputs accept text with correct styling on focus
□ Loading states appear before data loads

RESPONSIVE ACCURACY
□ Desktop (1200px+): nav top, multi-column bento, full widths
□ Tablet (768-1199px): nav top (links hidden), 6-col bento
□ Mobile (<768px): nav bottom, single column, hero mock simplified

ACCESSIBILITY
□ Sufficient contrast on all text (primary on floor > 7:1)
□ Touch targets ≥ 44px on mobile
□ All images have semanticsLabel
□ Animations respect prefers-reduced-motion (test by disabling in device settings)
```

---

## APPENDIX A — Key Flutter/Web Equivalences

| Web CSS | Flutter Equivalent |
|---|---|
| `position: fixed` | `Positioned` inside `Stack`, or `Overlay` |
| `backdrop-filter: blur(24px)` | `BackdropFilter(filter: ImageFilter.blur(sigmaX: 24, sigmaY: 24))` |
| `display: flex; gap: 8px` | `Row/Column` + `SizedBox(width/height: 8)` or `gap` in `Wrap` |
| `overflow: hidden` | `ClipRRect` wrapping container |
| `::before / ::after` pseudo | Extra `Stack` children or `CustomPaint` |
| `transition: transform 0.3s` | `AnimationController` + `AnimatedBuilder` |
| `@keyframes fadeUp` | `Tween` + `CurvedAnimation` + `FadeTransition` + `SlideTransition` |
| `pointer-events: none` | `IgnorePointer(ignoring: true)` |
| `z-index: 100` | Widget order in `Stack` (last = highest) |
| `linear-gradient(135deg,...)` | `LinearGradient(begin: Alignment.topLeft, end: Alignment.bottomRight, colors: [...])` |
| `radial-gradient(...)` | `RadialGradient(center: Alignment(...), radius: ..., colors: [...])` |
| `border-radius: 22px` | `BorderRadius.circular(22)` |
| `box-shadow: 0 8px 32px rgba(0,0,0,0.28)` | `BoxShadow(color: Color(0x47000000), blurRadius: 32, offset: Offset(0, 8))` |
| `letter-spacing: 0.18em` | `letterSpacing: fontSize * 0.18` |
| `font-variant-numeric: tabular-nums` | `FontFeature.tabularFigures()` in `fontFeatures: [...]` |
| `white-space: nowrap` | `overflow: TextOverflow.visible` + `softWrap: false` |
| `max-width: 1100px; margin: 0 auto` | `Center(child: ConstrainedBox(constraints: BoxConstraints(maxWidth: 1100), child: ...))` |
| `clamp(52px, 8vw, 108px)` | `(screenWidth * 0.08).clamp(52.0, 108.0)` |
| `vh` units | `MediaQuery.of(context).size.height * fraction` |
| `vw` units | `MediaQuery.of(context).size.width * fraction` |
| `:hover` state | `MouseRegion(onEnter:..., onExit:...)` |
| `:active` state | `GestureDetector(onTapDown:..., onTapUp:...)` |
| `scroll-behavior: smooth` | `ScrollController` + `animateTo` with curve |
| `IntersectionObserver` | `VisibilityDetector` widget (visibility_detector package) |

---

## APPENDIX B — pubspec Dependencies Reference

```yaml
dependencies:
  google_fonts: ^6.1.0          # Cormorant Garamond + Instrument Sans
  flutter_svg: ^2.0.9           # SVG logo rendering
  go_router: ^13.0.0            # Navigation (mirrors web routes exactly)
  flutter_riverpod: ^2.4.0      # State management
  shimmer: ^3.0.0               # Skeleton loading (palette-toned)
  lottie: ^3.0.0                # Complex animations (optional)
  visibility_detector: ^0.4.0+2 # IntersectionObserver equivalent
  cached_network_image: ^3.3.0  # Network images with loading states
  intl: ^0.19.0                 # Date/number formatting
  marionette_flutter: ^1.0.0    # [DEBUG ONLY] Visual inspection MCP

dev_dependencies:
  flutter_test:
    sdk: flutter
  golden_toolkit: ^0.15.0       # Golden screenshot tests
  marionette_mcp: ^1.0.0        # [DEBUG ONLY] MCP server
  flutter_lints: ^3.0.0         # Lint rules
```

---

*ORDR Web-to-Flutter Migration Skill v1.0*
*Compatible with: Antigravity IDE, Cursor, Claude Code, Windsurf*
*MCP Requirements: Dart MCP Server (dart.dev/tools/mcp) + Marionette MCP*
*Last updated: 2026-02-20*
