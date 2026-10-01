# Document 06: Flutter Studio & Kaisel Router Specification

## 🎨 1. Flutter Admin Studio Dashboard Architecture

The frontend is structured as a cross-platform Flutter application (`blogger_flutter`) that operates as both the **Admin Studio Command Center** for creators/vendors and the **Public Blog & Marketplace Reader** for visitors.

```
+-----------------------------------------------------------------------------------+
|                        FLUTTER ADMIN STUDIO (15-TAB NAVIGATION)                    |
+-----------------------------------------------------------------------------------+
| 1. Posts        : Manage articles, products, and services with JSON-LD composer.  |
| 2. Stats        : Traffic pageviews, unique visitors, referrers, and geo charts.  |
| 3. Comments     : Moderation queue (Published, Pending, Spam).                    |
| 4. Earnings     : AdSense impressions, clicks, estimated revenue, vendor payouts. |
| 5. Pages        : Static content pages (About, Contact, Store Catalog).           |
| 6. Blogger API  : Interactive tester for Google Blogger API v3 REST endpoints.    |
| 7. Theme        : Preset gallery, font family picker, layout variant switcher.    |
| 8. Layout       : Drag-and-drop gadget section manager (Header, Sidebar, Footer). |
| 9. Redirects    : Custom 301/302 URL forwarding manager.                         |
| 10. Atom/RSS    : Feed generator configuration.                                   |
| 11. Permissions : Member access control & invited author roles (RBAC).           |
| 12. Media       : Asset library upload manager.                                   |
| 13. Subscribers : Email newsletter follower list manager.                         |
| 14. Reading List: Subscribed blog feed reader.                                    |
| 15. Settings    : Basic settings, custom robots.txt, adult content warning.       |
+-----------------------------------------------------------------------------------+
```

---

## 🚦 2. Kaisel Router & Deep-Link Codec Engine (`KaiselConfigCodec`)

The app utilizes the **Kaisel Deep-Link Branch Router** to parse URIs into structured route stacks (`AppRoute`), support active branch navigation, and process global language query parameters (`gl`).

### Full Code Implementation Spec
```dart
part of 'router.dart';

final class AppStackCodec implements KaiselConfigCodec<AppRoute> {
  static const _homeBranch = 0;
  static const _storesBranch = 1;
  static const _labelBranch = 2;
  static const _settingsBranch = 3;

  final bool isOnboardingFirstRoute;
  final AppearanceSettingsBloc appearanceSettingsBloc;
  final AppDependencies appDependencies;

  const AppStackCodec({
    required this.isOnboardingFirstRoute,
    required this.appearanceSettingsBloc,
    required this.appDependencies,
  });

  @override
  KaiselConfig<AppRoute>? decode(Uri uri) {
    debugPrint('🔥 DECODE URI = $uri');

    _applyGlobalLanguage(uri);

    final segments = uri.pathSegments
        .where((segment) => segment.isNotEmpty)
        .toList(growable: false);

    return switch (segments) {
      // Root / Landing
      [] => _rootConfig(),
      ['onboarding'] => _onboardingConfig(),

      // Global Search & Labels
      ['search', 'label', final label] => _globalSearchLabelConfig(
        Uri.decodeComponent(label),
      ),
      ['search'] => _globalSearchConfig(
        query: uri.queryParameters['q'],
      ),

      // Blog-Scoped Search & Labels
      ['blog', final blogId, 'search'] => _blogSearchConfig(
        blogId,
        query: uri.queryParameters['q'],
      ),
      ['blog', final blogId, 'search', 'label', final label] =>
        _blogSearchLabelConfig(blogId, Uri.decodeComponent(label)),

      // Core Unified Blog Routes
      ['blog', final blogId] => _blogDetailConfig(blogId),
      ['blog', final blogId, 'post', final postId] => _postConfig(
        blogId,
        postId,
      ),
      ['blog', final blogId, 'page', final pageId] => _blogPageConfig(
        blogId,
        pageId,
      ),

      // Settings Branch Routes
      ['settings'] => _settingsConfig(),
      ['settings', 'general'] => _generalSettingsConfig(),
      ['settings', 'appearance'] => _appearanceConfig(),
      ['settings', 'notifications'] => _notificationsConfig(),
      ['settings', 'privacy'] => _privacyConfig(),
      _ => null,
    };
  }

  KaiselConfig<AppRoute> _rootConfig() {
    return isOnboardingFirstRoute ? _homeConfig() : _onboardingConfig();
  }

  KaiselConfig<AppRoute> _onboardingConfig() {
    return KaiselConfig(mainStack: const [OnboardingRoute()]);
  }

  KaiselConfig<AppRoute> _homeConfig() {
    return KaiselConfig(
      mainStack: const [MainShellRoute()],
      nestedState: KaiselShellConfig(
        activeBranch: _homeBranch,
        activeBranchStack: const [HomeRoot()],
      ),
    );
  }

  KaiselConfig<AppRoute> _globalSearchConfig({String? query}) {
    return KaiselConfig(mainStack: [GlobalSearchRoute(query: query ?? '')]);
  }

  KaiselConfig<AppRoute> _globalSearchLabelConfig(String label) {
    return KaiselConfig(mainStack: [GlobalSearchLabelRoute(label: label)]);
  }

  KaiselConfig<AppRoute> _blogSearchConfig(String blogId, {String? query}) {
    return KaiselConfig(
      mainStack: [BlogSearchRoute(blogId: blogId, query: query ?? '')],
    );
  }

  KaiselConfig<AppRoute> _blogSearchLabelConfig(String blogId, String label) {
    return KaiselConfig(
      mainStack: [BlogSearchLabelRoute(blogId: blogId, label: label)],
    );
  }

  KaiselConfig<AppRoute> _blogDetailConfig(String blogId) {
    return KaiselConfig(
      mainStack: const [MainShellRoute()],
      nestedState: KaiselShellConfig(
        activeBranch: _homeBranch,
        activeBranchStack: [const HomeRoot(), BlogDetailRoute(blogId)],
      ),
    );
  }

  KaiselConfig<AppRoute> _postConfig(String blogId, String postId) {
    return KaiselConfig(mainStack: [BlogPostRoute(blogId, postId)]);
  }

  KaiselConfig<AppRoute> _blogPageConfig(String blogId, String pageId) {
    return KaiselConfig(mainStack: [BlogPageRoute(blogId, pageId)]);
  }

  KaiselConfig<AppRoute> _settingsConfig() {
    return KaiselConfig(
      mainStack: const [MainShellRoute()],
      nestedState: KaiselShellConfig(
        activeBranch: _settingsBranch,
        activeBranchStack: const [SettingsMasterRoute()],
      ),
    );
  }

  KaiselConfig<AppRoute> _generalSettingsConfig() {
    return KaiselConfig(
      mainStack: const [MainShellRoute()],
      nestedState: KaiselShellConfig(
        activeBranch: _settingsBranch,
        activeBranchStack: const [SettingsMasterRoute(), GeneralSettingRoute()],
      ),
    );
  }

  KaiselConfig<AppRoute> _appearanceConfig() {
    return KaiselConfig(
      mainStack: const [MainShellRoute()],
      nestedState: KaiselShellConfig(
        activeBranch: _settingsBranch,
        activeBranchStack: const [
          SettingsMasterRoute(),
          AppearanceSettingRoute(),
        ],
      ),
    );
  }

  KaiselConfig<AppRoute> _notificationsConfig() {
    return KaiselConfig(
      mainStack: const [MainShellRoute()],
      nestedState: KaiselShellConfig(
        activeBranch: _settingsBranch,
        activeBranchStack: const [
          SettingsMasterRoute(),
          NotificationsSettingRoute(),
        ],
      ),
    );
  }

  KaiselConfig<AppRoute> _privacyConfig() {
    return KaiselConfig(
      mainStack: const [MainShellRoute()],
      nestedState: KaiselShellConfig(
        activeBranch: _settingsBranch,
        activeBranchStack: const [SettingsMasterRoute(), PrivacySettingRoute()],
      ),
    );
  }

  @override
  Uri encode(KaiselConfig<AppRoute> config) {
    final uri = switch ((config.mainStack.lastOrNull, config.nestedState)) {
      (OnboardingRoute(), _) => Uri(path: '/onboarding'),
      (BlogRoute(:final blogId), _) => Uri(path: '/blog/$blogId'),
      (PostRoute(:final blogId, :final postId), _) => Uri(
        path: '/blog/$blogId/post/$postId',
      ),
      (MainShellRoute(), final KaiselShellConfig shell) =>
        switch (shell.activeBranch) {
          _homeBranch => _encodeHome(shell.activeBranchStack),
          _settingsBranch => _encodeSettings(shell.activeBranchStack),
          _ => Uri(path: '/'),
        },
      _ => Uri(path: '/'),
    };

    return uri;
  }

  Uri _encodeHome(List<KaiselRoute> stack) {
    if (stack.isEmpty) return Uri(path: '/');
    return Uri(path: '/');
  }

  Uri _encodeSettings(List<KaiselRoute> stack) {
    if (stack.isEmpty) return Uri(path: '/settings');

    return switch (stack.last) {
      AppearanceSettingRoute() => Uri(path: '/settings/appearance'),
      GeneralSettingRoute() => Uri(path: '/settings/general'),
      NotificationsSettingRoute() => Uri(path: '/settings/notifications'),
      PrivacySettingRoute() => Uri(path: '/settings/privacy'),
      SettingsMasterRoute() => Uri(path: '/settings'),
      _ => Uri(path: '/settings'),
    };
  }

  /// Global Language Localization Event Dispatcher
  void _applyGlobalLanguage(Uri uri) {
    final globalLanguage = uri.queryParameters['gl'];
    if (globalLanguage == null || globalLanguage.isEmpty) return;

    final languageCode = globalLanguage.split('_').first.toLowerCase();
    final supported = AppLocalizations.supportedLocales.any(
      (locale) => locale.languageCode.toLowerCase() == languageCode,
    );

    if (!supported) return;

    appearanceSettingsBloc.add(
      SetLocaleEvent(
        Locale.fromSubtags(languageCode: languageCode),
      ),
    );
  }
}
```

---

## 🌍 3. Global Language (`gl`) Parameter Processing

When external users open links containing global language query parameters (e.g. `https://example.com/blog/102/post/product-888?gl=fr_FR`), the router's `_applyGlobalLanguage` interceptor:
1. Extracts `gl` parameter (`fr_FR`).
2. Parses language code (`fr`).
3. Checks against `AppLocalizations.supportedLocales`.
4. Dispatches `SetLocaleEvent(Locale('fr'))` to `AppearanceSettingsBloc`.
5. Triggers instant app-wide translation without page reloads.

---

## 🖥️ 4. Universal JSON-LD Visual Renderer & Schema Inspector

```
+-----------------------------------------------------------------------------------+
|                        UNIVERSAL JSON-LD FRONTEND RENDERER                        |
+-----------------------------------------------------------------------------------+
|  1. Type Inspection     : Examines `@type` field in post JSON-LD payload.         |
|  2. View Dispatcher     : Directs payload to matching view builder component.      |
|  3. Fallback Inspector  : Interactive Schema Inspector tree view for custom types.|
+-----------------------------------------------------------------------------------+
```

When rendering arbitrary Schema.org payloads that do not have a hardcoded widget, the **Schema Inspector Tree** renders a visual collapsible tree node structure allowing users to inspect properties interactively.
