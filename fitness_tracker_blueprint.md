# Personal Nutrition & Workout Tracker — Project Blueprint

**Status:** Initial planning baseline (version 0.1)  
**Goal:** A private, cross-device app for detailed nutrition analysis and highly customizable workout logging.  
**Primary user:** An individual tracking their own nutrition and training.  
**Development approach:** Build a responsive web app first; add installability/PWA features and offline capability deliberately after the core system is reliable.

## 1. Product principles

1. **Accuracy before intelligence.** Every nutrient value must have a traceable data source, unit, and known/unknown state. Never silently equate missing data to zero.
2. **Fast daily logging.** Repeating meals and workouts should take very few interactions, especially on a phone.
3. **Customizable.** Users can choose tracked nutrients, targets, exercises, and preferred units.
4. **Private and portable.** Restrict records to the authorized user, allow exports, and back up data.
5. **Progressive development.** Ship a small useful first version before AI, wearables, or complex recommendations.

## 2. Recommended implementation stack

- **Frontend:** Next.js with TypeScript, App Router, and Tailwind CSS.
- **Backend and database:** Supabase (PostgreSQL, Auth, row-level security).
- **Deployment:** Choose an appropriate Next.js-compatible host once the local application works.
- **App form factor:** Responsive web app for phone and computer; potentially installable as a PWA.
- **Data sources:** USDA FoodData Central as primary nutrient data; optionally Open Food Facts for products; published reference-intake authorities for nutrient targets.
- **Code storage:** Private GitHub repository with Git commits and small, testable changes.
- **Automated tests:** Unit tests for nutrition and exercise calculations; database authorization tests; end-to-end tests for core user flows.

**Design decision to defer:** Whether to use a local-first/offline-sync architecture. Do not promise full offline operation in MVP unless it is explicitly prioritized, because reliable sync/conflict resolution is a separate engineering project.

## 3. MVP scope (version 1)

### A. Account, settings, and measurements

- Sign in and keep diary information private.
- Preferences: metric/imperial units, timezone, date display.
- Basic profile: optionally weight for weight-dependent targets and calculations.
- Body measurements log: date, body weight, optional notes.
- Edit/remove historical entries.
- Export data to CSV/JSON.

### B. Food diary and nutrient engine

- Search and select foods from a source-backed food library.
- Enter food amount using grams, with optional recognized serving sizes.
- Save diary items for a date and meal (breakfast, lunch, dinner, snack, or custom).
- Save favorite foods and repeat previous entries.
- Calculate nutrients from amounts and normalized reference portions.
- Display daily totals for energy, macros, vitamins, minerals, essential amino acids, and individual fatty acids when available.
- Track **lysine** and **alpha-linolenic acid (ALA)** as first-class nutrients.
- Create a recipe with ingredients and their weights; allocate nutrition per specified serving or final recipe yield.
- Show daily totals, reference values, percentage of target, food contributors, and data completeness indicators.
- Allow manually defined foods with explicit values and explicitly unknown fields.

### C. Nutrient targets and analysis

- Store each nutrient with canonical identifier, name, aliases, quantity/unit, and nutrient category.
- Store reference targets separately from foods, with: intake type (e.g., RDA, AI, EAR, or other), value, unit, source, jurisdiction, applicable population, effective/publication date, and notes.
- Support configurable per-user target overrides without rewriting reference data.
- Support body-weight-based targets (e.g., mg per kg per day) where supported by a selected reference.
- Store upper intake levels (UL) only when defined by a reputable source; do not manufacture them.
- Clearly distinguish intake targets, tolerable upper limits, and optional athletic-performance goals.
- If a food lacks a nutrient record, mark that contribution as unknown rather than zero.
- Display **reported amount** and **coverage/completeness**. Warn that low values with incomplete data do not necessarily indicate insufficient intake.
- Support mass unit conversion (g, mg, µg) and carefully map fatty acid and amino acid names.
- Do not infer clinical deficiency or provide personalized medical diagnoses from self-entered dietary records.

### D. Workout diary

- Custom exercise library: name, category, primary/secondary muscle groups, equipment, unit type.
- Reusable workout templates with ordered exercises.
- Log workout sessions and individual sets.
- Fields per set: repetitions, external load, optional RPE/RIR, set type (warm-up, working, failure, etc.), and notes.
- Support bodyweight-only and duration/distance exercise variants.
- View previous performance while logging the same exercise.
- Show training history and simple personal records (explicit definitions for what counts as a record).
- Basic weekly statistics: session count, sets per muscle group, and load-volume trends with transparent formulas.

### E. Responsive UI

- Dashboard: latest weight, today's logged foods, basic macro/nutrient coverage, planned/last workout.
- Food Diary: easy daily entry, favorites, search, meal sections.
- Nutrition Report: selectable day/week, nutrient categories, source/coverage information.
- Food/Recipe Library: search and edit custom items.
- Workout Log: fast set entry and recent history.
- Routines: create/edit templates.
- Progress: body-weight and workout trend charts.
- Settings: targets, units, data export and account.

## 4. Features deferred until after MVP

- Camera barcode scanning and nutrition label capture.
- Advanced progression recommendations, periodization, deloads, and rest timers.
- Native Apple Health / Android Health Connect / wearable integration.
- AI-assisted entry and food substitutions.
- Correlation analysis between diet and performance.
- Complex local-first offline sync, conflict resolution, and push notifications.
- Multiple user roles and social/shared features.

## 5. Initial data architecture

Suggested **domain entities** (not yet final SQL):

| Domain | Entities | Main relationships |
|---|---|---|
| Identity | UserProfile, UserPreference | Owned by authenticated user |
| Food | Food, FoodPortion, FoodNutrient, DataSource | FoodNutrient joins Food and Nutrient |
| Nutrients | Nutrient, NutrientAlias, ReferenceTarget, UserTargetOverride | Targets reference nutrient and relevant population |
| Diary | DiaryEntry, MealType | DiaryEntry references Food or Recipe and quantity |
| Recipes | Recipe, RecipeIngredient | Ingredient references food and weight; handle yields |
| Exercise | Exercise, ExerciseMuscle, Routine, RoutineExercise | Routines contain ordered exercises |
| Training | WorkoutSession, WorkoutExercise, ExerciseSet | Session contains exercises and sets |
| Progress | BodyMeasurement | Dated measurements owned by user |
| Operations | ImportJob, SourceProvenance | Track food data updates and source IDs |

Key data requirements:

- Every user-owned table has a `user_id` and enforced row-level security.
- Food data may include a shared source-backed catalog plus user-specific custom foods.
- Source-backed foods should retain external dataset name and source food identifier.
- Nutrient amounts should be normalized to an unambiguous basis (for example, per 100 g of edible portion), with conversion tests.
- Unknown nutrient values are represented as missing, not numerical zero.
- Recipes must define whether values use raw input weights, cooked yield, or serving quantities.
- Do not overwrite historical diary nutrition unexpectedly after importing newer database values; choose and document a versioning/snapshot policy.
- Track UTC timestamps and a user-specific local diary date.
- Keep database schema migrations in version control.

## 6. Example calculation rules

**Food nutrient contribution:** `nutrient_per_100g × consumed_grams / 100`. Compute only where the relevant value is known. The absence of a measurement is not a measurement of zero.

**Daily reported intake:** Sum known contributions from each logged item; show missing data/coverage for the foods whose values were unavailable.

**Percentage of target:** `reported_amount / applicable_target × 100`, only when a valid target is available. Identify the reference type and include context when completeness is low.

**Body-weight-dependent target:** `reference_mg_per_kg × selected_body_weight_kg`; note which body weight was used and which reference was chosen.

**Workout volume load:** For conventional external-weight sets, `reps × external_weight`, summed across selected sets. Clearly label the metric as *volume load*; it is not interchangeable with hard sets or muscle stimulus.

**Unit conversion:** Normalize internally. Display user's preferred unit with rounding rules. Preserve sufficient precision for trace nutrient amounts.

## 7. UX/interaction priorities

- On mobile, prioritize one-thumb food search and set logging.
- One-tap repeat foods, meals, workout templates and previous exercise values.
- Let users choose dense nutrient tables or simple progress summaries.
- Use a neutral “unavailable data” state, not a red deficiency warning, for incompletely populated nutrients.
- Offer accessible text labels, sensible error messages, and keyboard navigation on desktop.

## 8. Engineering / security guardrails

- Keep API keys and private tokens out of commits; use environment configuration.
- Review Supabase row-level security, and test that account A cannot access account B's entries.
- Avoid exposing privileged server keys in browser/client bundles.
- Validate inputs and units on server boundaries.
- Respect third-party API terms, attribution requirements and request limits.
- Do not use model-generated numbers as verified nutrient data.
- Avoid premature microservices or heavy abstractions: keep the early app understandable.
- Use automated tests to detect numerical and authorization regressions.

## 9. Stepwise delivery plan

### Milestone 0 — Planning and skeleton
- Review this spec and identify contradictions/unknowns.
- Create initial routes, navigation, reusable UI components and placeholder data.
- Run the app locally on desktop and a mobile-width viewport.
- Set up linting, formatting and minimal automated tests.

**Acceptance:** Main pages render and navigation works at narrow and wide screen sizes.

### Milestone 1 — Authentication, data schema, and security
- Implement login.
- Write SQL migrations for the smallest required tables.
- Enforce row-level security and test user isolation.

**Acceptance:** User can sign in; user records persist privately; cross-user requests are denied.

### Milestone 2 — Food logging engine
- Seed a small set of known example foods with source and nutrient records.
- Implement add/edit/delete diary entries, servings, conversions and totals.
- Expose raw nutrient values and source/data-coverage metadata.
- Integrate USDA food search/import as a separate, tested boundary.

**Acceptance:** Two known foods logged at specified gram amounts produce correct totals for lysine and ALA; a missing nutrient does not appear as zero.

### Milestone 3 — Nutrition references, recipes and reports
- Configure reference targets, user overrides and body-weight-dependent values.
- Implement recipe composition and portions.
- Show daily/weekly summary with data quality.

**Acceptance:** Reports calculate totals, target percentages and unknown-value warnings correctly; target sources remain visible.

### Milestone 4 — Workout tracking
- Implement exercise library, templates, sessions and set-level logging.
- Show past values and simple history reports.

**Acceptance:** A full workout can be created, completed, edited and reopened on another device after sync.

### Milestone 5 — Progress, polish and release
- Add weight/workout charts, export, responsive refinements and backup strategy.
- Verify accessibility, authorization, data correctness and failure handling.
- Deploy and test phone/desktop sign-in and synchronization.

**Acceptance:** The user can perform core nutrition and workout workflows on both phone and computer, with consistent saved data.

## 10. Tasks Codex should not do without explicit approval

- Publish or deploy the app publicly.
- Generate nutrition reference values from guesswork.
- Remove source attribution or missing-value distinctions.
- Add medical/clinical claims based on dietary logs.
- Introduce paid external services or expose personal logs to external AI APIs.
- Apply destructive database migrations to real data.

## 11. First Codex task (planning only)

> Read `fitness_tracker_blueprint.md` as the product specification. Do not implement features yet. Examine the current repository and produce: (1) a proposed Next.js + Supabase architecture, (2) a minimal database schema with privacy and nutrient provenance in mind, (3) an implementation plan split into small independently testable tasks, (4) a list of assumptions or decisions that need approval, and (5) concrete acceptance tests for the nutrition calculation engine. Create `docs/architecture.md` and `docs/tasks.md` with your findings. Do not add live API keys, deploy, or apply database migrations. Summarize what you need approved before writing application code.

## 12. Useful official documentation

- Codex setup: https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan
- Next.js: https://nextjs.org/docs/app/getting-started/installation
- Supabase and Next.js: https://supabase.com/docs/guides/getting-started/quickstarts/nextjs
- USDA FoodData Central: https://fdc.nal.usda.gov/api-guide
