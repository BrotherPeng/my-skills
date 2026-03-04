---
name: "flutter-getx-architecture"
description: "Guidelines and rules for the GetX-based feature-first folder structure, module composition (View, Controller, Binding, Widgets), component splitting boundaries, get_cli route/state conventions, and strategy for GetX Fat Controller anti-bloat/decomposition (> 300 lines)."
version: "2.1.0"
---

# Flutter GetX Architecture & Folder Structure Guidelines (GetX Pattern)

> This guideline is based on the official [get_cli](https://github.com/jonataslaw/get_cli) tool and the recommended architecture from [getx_pattern](https://kauemurakami.github.io/getx_pattern/), adapted and supplemented for practical projects. All new pages, feature additions, and UI refactoring operations **MUST** strictly adhere to these guidelines.

---

## 1. Core Directory Architecture

```text
lib/
├── app/
│   ├── modules/                     # ★ Core business code organized by feature (Feature-First)
│   │   └── {module_name}/           # Specific module (e.g., home, trade, search)
│   │       ├── bindings/            # Dependency injection for the current module (Binding)
│   │       │   └── {module_name}_binding.dart
│   │       ├── controllers/         # Business logic for the current module (Controller)
│   │       │   └── {module_name}_controller.dart
│   │       ├── views/               # Main UI view for the current module (View)
│   │       │   └── {module_name}_view.dart
│   │       └── widgets/             # Private UI components specific to the current module (Widgets)
│   │           ├── {module_name}_banner_widget.dart
│   │           └── {module_name}_product_list_widget.dart
│   │
│   ├── data/                        # Data layer: responsible for data fetching and persistence
│   │   ├── models/                  # Globally shared data models (reusable across modules)
│   │   ├── providers/               # Data providers (Dio wrappers, API requests, local storage ops)
│   │   └── services/                # Background persistent services (exist with App lifecycle, e.g., AuthService)
│   │
│   └── routes/                      # Route management (automatically maintained by get_cli)
│       ├── app_pages.dart           # GetPage list: Mapping between route path ↔ View ↔ Binding
│       └── app_routes.dart          # Route path constants (part of app_pages.dart)
│
├── core/                            # Core layer: Global infrastructure (no business logic)
│   ├── theme/                       # Themes and styles (app_theme.dart, app_colors.dart)
│   ├── values/                      # Constants (strings, enums, regex, etc.)
│   └── utils/                       # Utility classes (date formatting, encryption, etc.)
│
├── components/                      # Global common UI components reusable across modules
│   ├── custom_button.dart
│   └── loading_dialog.dart
│
└── main.dart                        # Application entry point
```

---

## 2. Module Structure — The Four Pillars (Strict Enforcement)

When `get_cli` uses the `get create page:name` command to generate a module, it automatically creates `bindings`, `controllers`, and `views` subfolders. Based on this, this project **additionally mandates** a `widgets/` directory, forming a closed loop of Four Pillars.

| No. | Directory/File | Responsibility | Key Constraints |
|---|---|---|---|
| 1 | `views/{name}_view.dart` | **UI Skeleton**: Assembles the page, references widgets. | ❌ NO network requests, dialog logic, or non-pure UI code in the View. |
| 2 | `widgets/` | **Module's Private Component Library**: Large independent areas split from the page. | One file per large area, named `{module}_{area}_widget.dart`. |
| 3 | `controllers/{name}_controller.dart` | **Business Brain**: API calls, form validation, state management. | Use `.obs` for reactive state, keep variables private with public getters where possible. |
| 4 | `bindings/{name}_binding.dart` | **Dependency Management Hub**: Injects the Controller. | Use `Get.lazyPut()` to ensure instantiation only when used. |

### 2.1 When to create an independent Module?
- When it has **independent business logic** (e.g., needs its own Controller to manage independent state).
- When it is a **main trunk page** switched via bottom navigation bar / drawer (e.g., Home, Community, Profile).
- When a sub-feature's logic is complex enough that it shouldn't be stuffed into a single Widget of the parent module (e.g., `publish_product` under `trade`).

### 2.2 Widget Splitting Rules
**Core Principle**: Prevent a single View file from getting too long (consider splitting if it exceeds 200 lines) to keep the view layer clean.

- Before building a page, analyze the design mockup to identify **relatively independent large areas**.
- Extract these areas into independent Widget files and place them in the sibling `widgets/` directory.
- The View layer is only responsible for composing these Widgets like building blocks.

**Naming Convention**: `{module_name}_{area_name}_widget.dart`

**Example (Using Home Module as an example):**
```text
home/
├── bindings/home_binding.dart
├── controllers/home_controller.dart
├── views/home_view.dart              ← Only the skeleton, references the widgets below
└── widgets/
    ├── home_header_widget.dart        ← Top Logo + Search Bar
    ├── home_top_menu_widget.dart      ← Icon menus like Leaderboard / Activity
    ├── home_banner_widget.dart        ← Carousel Banner
    ├── home_announcement_widget.dart  ← Announcement Bar
    ├── home_big_cards_widget.dart     ← Large functional cards
    ├── home_category_tabs_widget.dart ← Business category Tab bar
    └── home_product_list_widget.dart  ← Content waterfall list
```

---

## 3. `get_cli`-Based Coding Standards

### 3.1 View Layer: Must extend `GetView<T>`
The View template generated by `get_cli` defaults to extending `GetView<XxxController>`. This project **strictly follows** this convention:

```dart
// ✅ CORRECT: Extends GetView, automatically holds the controller reference
class HomeView extends GetView<HomeController> {
  const HomeView({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          // Directly use controller to access business logic
          Obx(() => Text(controller.userName.value)),
        ],
      ),
    );
  }
}

// ❌ INCORRECT: Manual Get.find, loses type safety
class HomeView extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final controller = Get.find<HomeController>(); // Redundant and non-standard
    ...
  }
}
```

### 3.2 Controller Layer: `GetxController` + Reactive State
```dart
class HomeController extends GetxController {
  // --- Reactive State (Recommended: private variables + public getters) ---
  final _isLoading = true.obs;
  bool get isLoading => _isLoading.value;

  final _records = <RecordModel>[].obs;
  List<RecordModel> get records => _records;

  // --- Lifecycle ---
  @override
  void onInit() {
    super.onInit();
    _loadData();  // Fetch data on page initialization
  }

  @override
  void onClose() {
    // Release resources: cancel subscriptions, close streams, etc.
    super.onClose();
  }

  // --- Business Methods ---
  Future<void> _loadData() async {
    _isLoading.value = true;
    try {
      _records.value = await DataProvider.getRecords();
    } finally {
      _isLoading.value = false;
    }
  }
}
```

### 3.3 Binding Layer: `Bindings` Dependency Injection
```dart
class HomeBinding extends Bindings {
  @override
  void dependencies() {
    // Lazy injection: instantiated when entering the route, automatically destroyed when leaving
    Get.lazyPut<HomeController>(() => HomeController());
  }
}
```

### 3.4 Routing Layer: Named Routes (Mandatory)

**This project uses `part` files to split route definitions, which is exactly consistent with the structure generated by `get_cli`.**

`app_routes.dart` (as a part file of `app_pages.dart`):
```dart
part of 'app_pages.dart';

abstract class Routes {
  Routes._();
  static const HOME = _Paths.HOME;
  static const SEARCH = _Paths.SEARCH;
}

abstract class _Paths {
  static const HOME = '/home';
  static const SEARCH = '/search';
}
```

`app_pages.dart`:
```dart
import 'package:get/get.dart';

part 'app_routes.dart';

class AppPages {
  AppPages._();
  static const INITIAL = Routes.HOME;

  static final routes = [
    GetPage(
      name: _Paths.HOME,
      page: () => const HomeView(),
      binding: HomeBinding(),
    ),
  ];
}
```

**Route Navigation Rules (Strict Rules):**
| Scenario | Correct Syntax | Forbidden Syntax |
|---|---|---|
| Navigate to a new page | `Get.toNamed(Routes.SEARCH)` | ~~`Get.to(() => SearchView())`~~ |
| Navigate and replace current page | `Get.offNamed(Routes.HOME)` | ~~`Get.off(() => HomeView())`~~ |
| Navigate and clear stack | `Get.offAllNamed(Routes.HOME)` | ~~`Get.offAll(() => HomeView())`~~ |
| Pass arguments | `Get.toNamed(Routes.DETAIL, arguments: {'id': '123'})` | ~~Hardcoded path strings~~ |
| Retrieve arguments | `final args = Get.arguments as Map<String, dynamic>;` | — |
| Return with result | `Get.back(result: data)` | — |

---

## 4. Component Ownership: Global vs Module Private

| Global Components `lib/components/` | Module Private Components `modules/*/widgets/` |
|---|---|
| Zero business coupling, can be directly reused in another project. | Dependent on the current module's specific Model / Controller / Route logic. |
| e.g., Gradient buttons, network images, Loading dialogs, Empty state pages. | e.g., Specific business cards (depends on `BusinessModel`), custom category Tab bars. |

---

## 5. Data Layer (`data/`) Responsibility Division

| Subdirectory | Responsibility | Lifecycle |
|---|---|---|
| `models/` | Dart data models corresponding to backend APIs. Shared across modules. | None (Pure data structure) |
| `providers/` | The ONLY place allowed to make HTTP requests or read/write local storage. | Created on demand |
| `services/` | Singleton services that start with the APP and reside permanently in the background. | `Get.put(XxxService(), permanent: true)` |

**Criteria**: If a Model is referenced by more than two modules, it belongs in `data/models/`, not as a module's private asset.

---

## 6. `get_cli` Common Commands Quick Reference

```bash
# Create a new module (automatically generates bindings + controllers + views + registers route)
get create page:trade

# Create a sub-module under an existing module
get create page:publish_product on trade

# Generate Controller only
get create controller:payment on trade

# Generate View only
get create view:payment_success on trade

# Generate Model from JSON
get generate model on home with assets/models/user.json

# Generate i18n translation files
get generate locales assets/locales

# Sort imports and format
get sort
```

**`pubspec.yaml` Optional Configurations:**
```yaml
# File name separator (default is underscore, can be changed to dot)
get_cli:
  separator: "_"         # my_controller_controller.dart (if dot is selected: my_controller.controller.dart)

# Whether to generate sub-folders (default true)
get_cli:
  sub_folder: false      # Flat structure, do not generate bindings/ controllers/ views/ subdirectories
```

> ⚠️ This project maintains the **default `sub_folder: true`** configuration, which means retaining `bindings/`, `controllers/`, and `views/` subfolders within each module.

---

## 7. Implementation Workflow (3 Steps for New Features)

1. **Define Data (Data Layer)**
   - After confirming the backend API, handle the response entity class in `app/data/models/`.
   - Encapsulate the request methods in `app/data/providers/`.

2. **Create Module (Module Layer)**
   - Use `get create page:xxx` or manually create the `bindings/`, `controllers/`, and `views/` trio.
   - **Immediately** register the route in `routes/app_routes.dart` and `routes/app_pages.dart`.
   - Call the Provider written in step 1 within the Controller.

3. **Build UI (View + Widget Layer)**
   - Analyze the design mockup and **split the page into large areas within the `widgets/`** folder.
   - The View layer acts only as a skeleton, assembling Widgets like building blocks.
   - Use or create common basic components from `lib/components/`.
   - All event callbacks → `controller.doSomething()`, do not handle logic in the View.

---

## 8. Complex Page Splitting and Fat Controller Decomposition Guide

To address the bloating of a single `GetxController` (Fat Controller) caused by complex pages, we define **5 dimensions of decoupling strategies and a decision tree**.

**Conditions triggering this guideline:**
1. A single Controller being written or refactored is approaching or exceeds 300~500 lines.
2. The page contains multiple independent heavy business logics (e.g., home page simultaneously containing a carousel, complex cascading lists, and dialog logic).
3. Discovering a massive amount of network requests and data transformation logic directly written in the Controller.

### 🎯 Core Splitting Decision Tree

When encountering the need to add complex logic or refactor a giant Controller, please follow these priorities to think and execute:

#### 🥇 Priority 1: Sink data processing logic to Provider / Repository
- **Scenario**: The Controller becomes huge because it's filled with HTTP request code, JSON to Model code, data cleansing/restructuring algorithms, and local storage read/writes.
- **Action**: Peel off all `dio.get/post` and complex algorithms to the `app/data/providers/` layer or independent UseCase classes.
- **Architectural Effect**: The Controller returns to its essence as a "state dispatcher", keeping only: Call interface ➡️ Get Model (or exception) ➡️ Modify `.obs` variables to respond to UI.

#### 🥈 Priority 2: Return pure UI state to native `StatefulWidget`
- **Scenario**: Unnecessary global/cross-component states are introduced merely to implement pure visual interactions. For example: password field clear/obscure text toggle, expand/collapse panel animation state, input field focus feedback.
- **Action**: Do NOT create variables like `isPasswordVisible.obs` in the GetxController! Wrap the corresponding building block as a `StatefulWidget` and use `setState()` to manage its internal temporary closed-loop state.
- **Architectural Effect**: Greatly reduces meaningless `obs` variables in the Controller, avoiding permanent memory residency.

#### 🥉 Priority 3: Horizontal Slicing - Split out dedicated Sub-Controllers by UI Block
- **Scenario**: The current page is an aggregation page pieced together from several **unrelated** core business blocks (e.g., Home page's [Top Nav] + [Search Bar] + [Carousel] + [Waterfall]).
- **Action**:
  1. Don't handle everything in `HomeController`. The main Controller only does external framework coordination.
  2. For specific areas (like the Carousel), create a new `HomeBannerController`.
  3. Inject it in the current module's Binding file (`Get.lazyPut(() => HomeBannerController())`).
  4. The corresponding `home_banner_widget.dart` extends `GetView<HomeBannerController>` to implement self-contained data.
- **Architectural Effect**: Achieves logical isolation of in-page components, facilitating concurrent development by multiple people without code conflicts.

#### 🏅 Priority 4: Deep Slicing - Extract into Sub-modules
- **Scenario**: A piece of complex logic not only has massive internal state but also involves **independent page navigation flows, independent deep routing, or full-screen dialogs** (e.g., a "Publish Form" page popping up in a business flow).
- **Action**: Use the get_cli command `get create page:XXX on YYY`. This will generate a new module with its own independent Routing, Bindings, Controllers, and Views lifecycle inside the existing parent module (YYY).
- **Architectural Effect**: Completely destroys massive feature branch states when the route is popped from the stack, preventing memory leaks.

#### 🎖️ Priority 5: Extract into Global Service or Component
- **Scenario**: The complex logic is **globally available** and frequently reused in multiple places. For example: A globally floating player control bar, IM long connection message listener, global data settlement.
- **Action**:
  - Pure headless service (logic): Extract into a `GetxService` and inject it globally with `permanent: true`.
  - Component with global UI presentation: Place in the top-level `lib/components/`, letting it mount or create its own partial Controller internally to be self-sufficient.
- **Architectural Effect**: Avoids writing these shared logics everywhere in the Controllers of various independent pages.
