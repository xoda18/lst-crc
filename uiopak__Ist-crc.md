# Карточка анализа — uiopak/lst-crc

> СТАТУС: ЗАПОЛНЕНО.

Правила: факт без `файл:строки` — не факт; не уверен — пиши `unclear` и в Open questions;
`declares dependency ≠ uses it`; anti-pattern и framework gap — в РАЗНЫХ блоках (см. codebook внизу шаблона).

---

## Tier 1 — Обязательное ядро

### 1.1 Идентификация и provenance

```text
Repo:            uiopak/lst-crc
URL:             https://github.com/uiopak/lst-crc
Commit / branch: main @ bd1a712 (из клона в my-review-of-repos/lst-crc)
Дата анализа:    — (полный проход не выполнен)
Аналитик:        —
Метод доступа:   локальный клон (my-review-of-repos/lst-crc) + вторичные карточки docs-llm-analytics.
                 Глубоко проверена только тема test-bridge (см. §2.2, §3.4); сквозной проход не делался.
Grep-сигнатуры:  bridge: "LstCrcUiTestBridge", "LstCrcUiTestBridgeRemote", "lstcrc.testing", "service<"
```

### 1.2 Классификация

```text
Заявленная категория: Both
Verified категория:   не верифицировано независимо (в готовых карточках стоит Both)
Совпадает:            —
Coexistence type:     —
Confidence:           Low (нет собственного сквозного анализа)
```

### 1.3 Verification chain

```text
1. Dependency:      ✅ Robot: build.gradle.kts:102 testImplementation(libs.bundles.remoterobot)
                       + libs.versions.toml:33-34,57; robot-server: build.gradle.kts:628 robotServerPlugin()
                    ✅ Starter: build.gradle.kts:136 testFramework(TestFrameworkType.Starter, "uiTestImplementation")
2. Source set:      ✅ Starter: build.gradle.kts:52-55 create("uiTest") (classpath += main+test output)
                    ✅ Robot: стандартный src/test (robot-стек)
3. Gradle task:     ✅ Robot: register<Test>("uiTest") :540 + runIdeForUiTests :601/:634
                       + readiness :331 (uiTestReady) + teardown :393 (stopUiTestProcesses)
                    ✅ Starter: register<Test>("starterUiTest") :556 + starterPerformanceTest :572
4. Test classes:    ✅ Robot: 5× *UiTest.kt в src/test/.../plugin + gutters/VisualTrackerManagerBehaviorTest.kt
                    ✅ Starter: 8 файлов в src/uiTest/.../starter (5× *StarterUiTest + performance + base + project)
5. CI / local launch: ✅ Robot: run-ui-tests.yml — runIdeForUiTests (:35/:46) + uiTest (:38/:47) + teardown (:40)
                    ✅ Starter: run-starter-ui-tests.yml:52 — ./gradlew starterUiTest (+ upload результатов :74)
6. Real assertions: ✅ Robot: 141 assert-хитов; напр. VisualTrackerManagerBehaviorTest.kt:24 assertEquals(Range.INSERTED, …)
                    ✅ Starter: 54 assert-хита; напр. LstCrcVisualStarterUiTest.kt:22 assertTrue(…gutter range…)

Где обрывается:     не обрывается — ОБЕ цепочки полные, от dependency до CI-run и до real assertions
Что это значит:     confidence High. Каждое звено подтверждено кодом/CI; assertions проверяют реальное
                    состояние (gutter ranges, диапазоны трекеров), а не «клики без проверок».
                    Триангуляция build + test-код + CI сходится по обоим стекам; ложных срабатываний нет (2.3).
```

### 1.4 Резюме и главные findings

```text
Резюме:        Один плагин тестируется двумя стеками параллельно — legacy RemoteRobot (src/test) и
               Starter/Driver (src/uiTest), с двумя подписанными CI-workflow и in-IDE мостом (@Service).
Главный вывод: Эталонный dual stack с документированной миграцией robot→Starter. Единственный реальный
               gap перехода — нет callJs (нельзя инлайнить in-IDE код в тест), severity НИЗКАЯ (обходится мостом).
Что НЕЛЬЗЯ overclaim:  направление списания legacy не доказано (git log не гонялся); «мост = gap» неверно
               (~82% его методов — convenience, достижимы штатно). Сквозной ревью не делался.
```

### 1.5 Open questions

```text

```

---

## Tier 2 — Чек-лист измерений (доски «Структура» + «Билд» + «Костыли» + gaps)

### 2.1 Build

```text
Проверенные файлы: build.gradle.kts, settings.gradle.kts, gradle.properties, libs.versions.toml, .github/workflows, .run

Зависимости:
- Starter: testFramework(TestFrameworkType.Starter) на uiTestImplementation (:136)
- Robot: bundles.remoterobot на testImplementation (:102) + robotServerPlugin() (:628)

Версии:
- IntelliJ Platform Gradle Plugin: v2
- IDE + variant: IU (dev, local.ide.path) / IC (fallback) — IdeProductProvider.IU/IC
- Build-number: sinceBuild=251 vs platformVersion=2026.1 → drift (вопрос coverage, не баг)
- Starter/Driver: через platform-плагин + JetBrains ide-starter maven repo

Starter vs Driver (роли): Starter = lifecycle (runIdeWithDriver); Driver = client (@Remote/service<>() к мосту). Роли РАЗДЕЛЕНЫ.
Локальный запуск: .run configs есть (Run UI Tests, Run Starter Tests, Start IDE with Robot Server).
RemDev: нет — monolith only (см. 2.4).
DI / service lookup: service<LstCrcUiTestBridgeRemote>() (starter); RemoteRobot fixtures (robot).
```

### 2.2 Structure

```text
Заготовленные модули / prepared projects:
- starter: LstCrcStarterProject — on-disk git + LocalProjectInfo (поддержка multi-root/worktree).
- robot: IdeUiTestProjectManager — создание проекта инъекцией через runJs (симптом GAP A, см. 3.4).

Base classes / helper layer:
- robot: LstCrcUiTestSupport + pages/ (6 фикстур) + steps/PluginUiTestSteps + RemoteRobotExtension.
- starter: LstCrcStarterUiTestBase + @Service-мост (см. Test bridge ниже).

Публичные методы в UI-элементах:
- отдельно не инвентаризировано; основной добавленный test-API = @Service-мост (см. Test bridge), а не методы на самих UI-компонентах.

Использование IDE/API-методов:
- ПРОВЕРЕНО (bridge): из 62 методов моста ~82% операций достижимы штатно через service/@Remote
  (13 напрямую + 38 «можно, но много RPC»); тесты реально дёргают service<LstCrcUiTestBridgeRemote>().
- МЁРТВЫЙ КОД: 4 метода моста не вызываются НИКОГДА — openFindInFilesDialog, openRepoComparisonDialog,
  findDialogScopeOptionsSnapshot, dismissTransientUi. Для диалогов автор перешёл на штатный SDK:
  driver.invokeAction("FindInPath"), findInPathPopup{...}, ideFrame{ actionButton{...}.click() }.

Test bridge / debug API:
- где лежит + попадает ли в normal plugin artifact + исключается ли из production build:
  (ПРОВЕРЕНО: testing/LstCrcUiTestBridge.kt = @Service (light service, НЕ в plugin.xml) в src/main, 62 метода;
   build.gradle.kts:221-236 исключает testing/** из publish по умолчанию, включает для starterUiTest;
   @Remote-прокси starter/remote/LstCrcUiTestBridgeRemote.kt, вызывается service<>() в starter-тестах.
   НЕ использует штатный test-fixtures механизм — самодельный exclude из релизного JAR.)

Покрытые product areas:
- gutters, VCS-scopes (created/modified/moved/deleted/changed), tool window (табы), status widget,
  settings (click-actions/gutter/includeHead), dialogs (branch selection, Find-in-Files), Git/multi-root.
  Inspections / persistence — n/a.
```

### 2.3 Ложные срабатывания

```text
Dead dependency:                                          нет
Template / scaffold leftover:                             нет
Commented-out / disabled:                                 нет — 0
Fork / clone noise:                                       нет
Итог: найденное реально используется?                     да
```

### 2.4 Реальность исполнения

```text
CI реально прогоняет UI-тесты (не только buildPlugin): НЕТ на per-commit. UI-тесты (robot и starter)
  запускаются только вручную (workflow_dispatch). Основной CI build.yml (push→main + PR) UI НЕ гоняет —
  только buildPlugin + check(unit) + verifyPlugin + Qodana. Вывод: тесты реальные, но не в постоянной петле
  (на Confidence не влияет — тесты существуют и рабочие).
Trigger:
  - run-ui-tests.yml (robot): workflow_dispatch (:11-12) — вручную
  - run-starter-ui-tests.yml (starter): workflow_dispatch (:10-11) — вручную
  - build.yml: push→main + pull_request (:16-21); release.yml: release published (:6-8)
Test filtering: @Disabled = 0 (см. 2.3); tags не используются; фильтров тестов в CI нет.
RemDev: отсутствует во всех файлах — только монолитная IDE (IdeProductProvider.IC/IU). JMX 17777/RPC 24000 —
  Driver↔IDE, не Gateway. → monolith only, split/RemDev не тестируется.
Launch model (ПРОВЕРЕНО по build.gradle.kts + CI-workflow, две модели):
  - Robot: отдельный процесс IDE + robot-server по HTTP (robot-server.auto.run=false → сервер стартует
    отдельно через runIdeForUiTests); задача uiTestReady ждёт его готовности по сети (GET / + POST /js/execute).
    В CI это 5 РУЧНЫХ шагов (детальная таблица «кто делает» — в 3.5): фон runIdeForUiTests (run-ui:36) →
    readiness uiTestReady poll :8082 (:380-395) → отдельный процесс uiTest (:398-406) → teardown (:433-445).
  - Starter: IDE поднимается самим тестом (local.ide.path + path.to.build.plugin), управление по JMX:17777 / RPC:24000.
    В CI это 1 шаг: ./gradlew starterUiTest (run-starter:52) — runIdeWithDriver() сам запускает/ждёт/гасит IDE.
Environment:
  - Robot: 3 OS (ubuntu/mac/win, fail-fast:false); Xvfb (Linux); JDK21 corretto; disk-cleanup; FFmpeg.
  - Starter: 1 OS (ubuntu); Xvfb; JDK21 zulu. (Мелочь: distribution рассинхрон corretto vs zulu.)
Artifacts on failure:
  - Robot: video (FFmpeg, run-ui:408-414) + sandbox logs build/idea-sandbox/system/log (:416-422) + report upload (:424-431).
  - Starter: ide-starter diagnostics (run-starter:54-64) + allure-results + test-results upload (:66-76).
```

### 2.5 Зрелость и качество тестов

```text
Wait strategy: в основном waitFor/waitForIndicators (зрело). Thread.sleep=5 (robot 2, starter 3): 3 — опрос ФС/git (ок),
  🔴 2 — timing-ассерты «клик ещё не сработал», флак на CI: LstCrcSettingsStarterUiTest.kt:136 + LstCrcSettingsUiTest.kt:211.
  🔴 глотание в waitUntil (LstCrcStarterUiTestBase.kt:368) маскирует ошибки под таймаут (см. 3.4).
Операционная ЦЕНА запуска реальной IDE (не костыли — цена; build.gradle.kts):
  maxParallelForks=1 (одна IDE → нет параллели), heap до 4g (IDE тяжёлая), таймауты больше на CI (240/900 vs 90/600),
  --add-opens (доступ к JDK-модулям). Признак брутальности/дороговизны обоих UI-стеков.
```

### 2.6 Костыли / workarounds

```text
Только реальные находки (пустой шаблон убран). Классификация: gap → 3.4 [Framework gap];
anti-pattern → 3.4 [Anti-pattern]. Систематический инвентарь по репо НЕ делался.

Brittleness / lifecycle workarounds (отдельная ось — НЕ callJs-gap и НЕ convenience):
- Starter: `notCompatibleWithConfigurationCache` — config-cache ломает starter discovery (build.gradle.kts). Обход хрупкости Starter.
- Robot: `stopUiTestProcesses` — киллер сирот-процессов runIdeForUiTests/shared-ui-ide (build.gradle.kts). Обход слабости lifecycle Robot.

Остальные найденные костыли/anti-pattern вынесены в 3.4 (byXpath-по-тексту, god-test, глотание в waitUntil).
```

---

## Tier 3 — Миграция, framework gaps, синтез

### 3.1–3.2 Starter / Robot usage — свёрнуто

Детали обоих стеков уже покрыты в **1.3** (verification chain) и **3.4** (findings/gaps), не дублируем.
Кратко: Starter = lifecycle (`runIdeWithDriver`) + Driver-клиент (`@Remote`/`service<>()` к мосту);
Robot = отдельный процесс IDE + robot-server :8082 + `callJs`/`byXpath`.

### 3.3 Migration evidence

```text
Migration found: да (dual stack, документировано)
Evidence:
  - docs-comment: run-starter-ui-tests.yml:1-7 — автор ПРЯМО ссылается на legacy run-ui-tests.yml и
    формулирует отличие: "Unlike the legacy Remote Robot UI tests (run-ui-tests.yml), IDE Starter tests
    launch and manage their own IDE instance internally via runIdeWithDriver(), so there is no need to
    start the IDE separately." → сам автор фиксирует переход.
  - CI changes: два отдельных подписанных workflow (robot vs starter), оба workflow_dispatch.
  - build setup: dual config — RemoteRobot для src/test + Starter для src/uiTest (см. 1.3).
  - added files: src/uiTest/.../starter (8 файлов) параллельно robot-стеку src/test.
История: точный timeline (когда появился каждый стек, удаляются ли robot-тесты) НЕ собран — shallow clone,
  git log -S не прогонялся.
Интерпретация: dual stack с явными признаками направленной миграции (комментарий-мотивация); списание legacy — unclear.
Confidence: High по факту dual stack + документированной миграции; Medium по «активному списанию robot».
```

### 3.4 Findings — раздельные блоки (НЕ смешивать anti-pattern и gap)

```text
[Pattern] «Новый стек покрывает то, чего нет в robot»
Pattern: 2 фичи есть только на Starter — Performance и MultiRoot/worktree.
Evidence:
  - LstCrcStarterPerformanceTest.kt:48-108 — measureTime{} + пороги (<10s/<60s/<30s) через мост.
    На robot бессмысленно: каждый вызов = HTTP round-trip → замер = сетевой шум.
  - LstCrcMultiRootStarterUiTest.kt:20-33 — несколько git-репо + linked worktree
    (initializeGitRepositoryAt/createLinkedWorktree). Нужна программная on-disk подготовка
    (есть у LstCrcStarterProject); у robot-project-prep такого API нет.
Why it matters: пример «где новый подход уже лучше» — Starter практически включает то,
  что на robot невозможно (perf) или мучительно (multi-root).
Confidence: Performance = Med-High (HTTP-overhead — реальная причина); MultiRoot = Med (Starter enables).
Open questions: robot физически не мог или автор просто не бэкпортил на старый стек — не проверено (нужна история).
```

```text
[Anti-pattern] Локаторы по user-facing тексту (robot) — ЕДИНСТВЕННЫЙ самостоятельный anti-pattern
Anti-pattern: byXpath по переводимому/видимому тексту (@text / @visible_text / @accessiblename / @title).
Evidence:
  - LstCrcInteractionUiTest.kt:94 — @visible_text='1 difference' (ломается и от языка, и от числа → "2 differences")
  - ActionMenuFixture.kt:13 — @text=$text (пункт меню по надписи); DialogFixture.kt:28 — @title
  - PluginUiTestSteps.kt:401/406 — "Don't ask again" / "Add Tab" (@accessiblename)
Почему это anti-pattern: локатор по надписи ломок к локали / версии IDE / плюрализации.
Нормальный способ: локатор по @class (стабилен) — как GitChangesViewFixture.kt:14 (@class='LstCrcChangesBrowser').
Impact: флейки/поломки при смене языка или версии.
Confidence: High.
ВАЖНО (дедуп): runJs-инъекция для создания проекта СЮДА НЕ входит — это симптом GAP A (см. [Framework gap]),
  а не отдельный anti-pattern. Раньше ошибочно дублировалось в обоих блоках.
```

```text
[Anti-pattern] God-test в robot (LstCrcFileScopeUiTest.kt)
Anti-pattern: один тест testFileOperations (100+ строк) проверяет несколько независимых вещей:
  rename/delete/create + scope-содержимое + includeHeadInScopes. При падении одного ассерта
  падает вся цепочка — неясно, что именно сломано.
Evidence: LstCrcFileScopeUiTest.kt:19 — 1 тест против 6 изолированных в LstCrcFileScopeStarterUiTest.kt.
Почему anti-pattern: robot IDE висит одна на всю сессию (runIdeForUiTests & в CI) → дробить дёшево.
  Нет технической причины держать всё в одном тесте.
Нормальный способ: 6 изолированных тестов — как на Starter-стороне (testPermanentHeadTab…,
  testIncludeHeadInScopes…, testFindDialog…, testDeletedColor…). Starter платит дороже (IDE на каждый тест),
  но всё равно дробит — ради читаемости и изоляции.
Confidence: High.
```

```text
[Anti-pattern] 🔴 Глотание исключений в waitUntil (LstCrcStarterUiTestBase.kt:368)
runCatching(condition).getOrDefault(false) → любая ошибка bridge-RPC превращается в тихий ретрай до таймаута.
Затрагивает весь starter-suite: реальная поломка выглядит как «не дождались». Poll-интервал 500мс.
Confidence: verify (один прогон агентов).
```

```text
[Framework gap] — консолидировано, 3 gap'а (A/B/C)

GAP A [Сильный] — у robot НЕТ built-in подготовки проекта.
  Нет аналога LocalProjectInfo/GitHubProject → проект создаётся инъекцией внутреннего API через runJs
  (ProjectManagerEx и т.п., IdeUiTestProjectManager.kt). Starter даёт из коробки (LocalProjectInfo +
  LstCrcStarterProject — on-disk git). runJs-инъекция = СИМПТОМ этого gap'а, НЕ отдельный anti-pattern
  (это и есть дедуп). Правильно: готовить проект на диске + открыть папку. Confidence: High.

GAP B [Структурный, узкий] — нельзя прислать в IDE НОВЫЙ произвольный композитный/атомарный JVM-код (детально ниже).
Итог: структурное ограничение SDK реально — нельзя прислать в IDE новый произвольный композитный/атомарный
        JVM-код; чтобы удержать один замок на многошаговую логику, нужен скомпилированный метод в IDE.
        Но мост как целое — в основном convenience (≈82% методов достижимы штатно); genuine gap виден
        только в его атомарных методах. Детализация confidence — внизу блока.
Что в SDK достижимо штатно (НЕ gap):
  • инстанс-методы/сервисы → @Remote / service<>() ;
  • статические методы → utility<>() (напр. VfsUtil.saveText — запись одного файла без моста);
  • ПРОИЗВОЛЬНЫЙ статический метод по имени → %importCall/%call через PlaybackRunnerService.runScript
    (perf-commands generalCommandChain.kt:606-611; starter-driver PlaybackRunnerService.kt) — server-side рефлексия,
    args строки, возврат String?;
  • JS в IDE → JCEF callJs/runJs (driver-sdk JCefUI.kt: @Remote JcefComponentWrapper) — но только web-DOM
    встроенного браузера, НЕ JVM/PSI/сервисы;
  • готовые файловые команды %addFile / %renameFile / %deleteFile / %openFile (~120 команд).
Что реально НЕЛЬЗЯ (остаточный узкий gap): прислать НОВЫЙ произвольный блок JVM-кода/лямбду на исполнение
        в IDE. И @Remote, и %call зовут только УЖЕ СКОМПИЛИРОВАННЫЕ методы. Плюс withWriteAction крутит блок
        client-side (DriverImpl.withContext: this.code()) → многошаговой атомарной транзакции из теста не собрать.
        Поэтому СВОЮ атомарную композитную логику приходится компилировать в IDE (это и есть мост).
Но и это мягко: с %call (вызвать существующий статик) и %addFile/%deleteFile файловые методы моста, вероятно,
        можно было обойти БЕЗ компиляции → склоняет их к convenience, а не к gap. НЕ проверено на этом репо.
Evidence: perf-commands/generalCommandChain.kt:606-611 (%call); starter-driver/PlaybackRunnerService.kt (runScript);
        driver-sdk/JCefUI.kt (JCEF callJs); driver-client DriverImpl.kt/RemoteCall.java (Invoker-транспорт без кода);
        LstCrcUiTestBridge.kt:452-506 (мост). Всё — sources.jar v261.25134.95.
Дополнительный пример gap'а — рефлексия приватного поля (in-process):
  robot:   IdeaFrame.kt:1116 — callJs с getDeclaredField("visualTrackers")+setAccessible(true) — код летит в IDE динамически;
  starter: LstCrcUiTestBridge.kt:896 — та же рефлексия, но скомпилированный Kotlin в мосте, зовётся через @Remote.
  Причина: нет штатного API для gutter-трекеров → рефлексия обязательна → robot шлёт её через callJs,
  Driver не умеет → код компилируется в мост заранее. Пара тестов: LstCrcVisualUiTest.kt vs LstCrcVisualStarterUiTest.kt.
Дополнительный пример gap'а — глубокое внутреннее состояние сервиса (testBranchComparisonUpdatesModifiedScope):
  robot:   LstCrcBranchComparisonUiTest.kt:62-144 — 80-строчный callJs читает ProjectActiveDiffDataService
           (activeBranchName, modifiedFilePaths, createdFilePaths, deletedFilePaths), создаёт экземпляры
           scope-классов через рефлексию (getPluginClass+newInstance) и проверяет что каждый файл попадает
           в правильный scope (created/modified/moved/deleted). Всё одним callJs-блоком, server-side.
  starter: аналога нет. Мост не открывает ProjectActiveDiffDataService (activeBranchName, filePaths).
           Добавить возможно — но требует 5+ новых методов моста к этому сервису. Ни одного из них в
           LstCrcUiTestBridge.kt нет.
  Вывод:   именно этот тест не перенесён на Starter (из 4-х непереведённых тестов BranchComparison) — не
           потому что «не дошли руки», а потому что нет подходящего API моста. Ярко показывает цену отсутствия
           callJs: на robot — 80 строк JS и готово; на Starter — надо расширять мост.
  Confidence: High по факту отсутствия (тест в robot есть, в Starter нет, мост не покрывает сервис).
Confidence (по трём РАЗНЫМ утверждениям, не одной строкой):
  • структурное ограничение SDK — «нельзя прислать новый произвольный композитный/атомарный JVM-код;
    для одного замка на свою многошаговую логику нужен скомпилированный helper» → HIGH
    (прочитан транспорт без поля кода, withContext client-side, весь SDK 8 модулей; %call — только
    существующий статик, JCEF callJs — только браузер).
  • «мост КАК ЦЕЛОЕ = framework gap» → LOW (≈82% методов = convenience, достижимо штатно).
  • «атомарные методы моста writeProjectFile/rename/delete требуют helper из-за этого ограничения» →
    MEDIUM-HIGH (структурный факт применим; НЕ проверено лишь, заменимы ли именно они на %addFile/%deleteFile/%call).
Открытый вопрос: можно ли writeProjectFile/rename/delete заменить на %call + %addFile/%deleteFile? Если да — это convenience.
```

### 3.5 Migration Pattern Map

Терсно. Детали/evidence по gap-строкам — в §3.4. Пустые строки = ещё не анализировались.

Только строки с evidence. Остальные операции (открытие проекта, установка plugin, поиск UI,
ожидания/indexing, dialogs, diagnostics) — **не анализировались, вне скоупа**.

| Операция | Legacy robot | Starter/Driver equivalent | Conf |
|---|---|---|---|
| запуск IDE / CI-runtime | 5 ручных шагов: фон-IDE + robot-server :8082 + readiness poll + отдельный тест-процесс + teardown | 1 шаг: runIdeWithDriver() — фреймворк сам запускает/связывает/ждёт/гасит | High |
| доступ к IDE API/состоянию | robot.callJs/runJs | @Remote + driver.invokeAction + нативный UI API | Med-High |
| произвольная композитная/атомарная логика | callJs (свой JS) | штатного пути нет → helper (мост); %call/JCEF не заменяют — см. §3.4 | High* |

\* High — что ограничение SDK реально; про convenience-нюанс и открытый вопрос см. §3.4.

**Запуск IDE (модель):** robot = 5 ручных шагов в CI (запуск фон-IDE → связь :8082 → readiness poll →
отдельный тест-процесс → teardown); starter = 1 шаг (`runIdeWithDriver` сам всё делает). При миграции шаги
1-2-3-5 просто перестаёшь писать. (Детальная таблица «кто делает» — в `notes.md`.)

### 3.6 Синтез

```text
Priority / research-value score (0–10): 10/10
Обоснование score: самый явно документированный dual stack корпуса — те же фичи в обоих фреймворках,
  два подписанных CI-workflow, мост + зафиксированный callJs-gap. Идеальный apples-to-apples для миграции.
Interesting angle: единственный репо, где gap «нет callJs в Driver» виден предметно — мост существует
  именно чтобы его обойти; тест testBranchComparisonUpdatesModifiedScope не перенесён из-за отсутствия API моста.
Самое сильное evidence: два раздельных CI-workflow с авторским комментарием о переходе + полная verification chain (1.3).
```

### 3.7 Вопросы автору

```text
[Вопрос 1] Почему recovery-тест только в Starter?
Вопрос: testMissingBranchComparisonTargetRecoversToHeadAndShowsWarning (и два похожих теста —
  testMissingCommitComparisonTargetDoesNotRecoverToHeadOrWarn, testRepositoryComparisonToolbarDialogAllowsChangingComparison)
  написаны ТОЛЬКО на Starter, аналогов в robot-стеке нет. Robot технически мог бы сделать то же самое
  через callJs (IdeaFrame.kt:1064 — updateTabComparisonMap; NotificationsManager читается через callJs).
  Вопрос: это намеренное решение («recovery-сценарии удобнее на Starter, не стали дублировать»), или
  тесты добавили когда Starter уже был и просто написали туда, или robot-версия планируется/не нужна?
Почему нельзя найти самому: намерение автора в публичном коде/истории не отражено;
  тест появился без PR-описания, объясняющего выбор.
Что меняет в выводе: если намеренно (robot неудобен для state-heavy сценариев) → это convenience-аргумент
  в пользу Starter, подтверждает «Starter расширяет, а не только переносит»; если «просто написали туда» →
  слабее как аргумент о возможностях Starter.
Приоритет: средний
```

```text
[Вопрос 2] 4 теста из LstCrcBranchComparisonUiTest не перенесены на Starter. Из них 2 технически
  простые (keyboard-ввод). testBranchComparisonUpdatesModifiedScope требует расширения моста —
  ProjectActiveDiffDataService не открыт; robot делает это через callJs.
  Вопрос: простые — «не дошли руки» или дублирование не нужно? testBranchComparisonUpdatesModifiedScope
  останется robot-only или планируется расширить мост?
Почему нельзя найти самому: намерение не отражено в истории/PR.
Что меняет: «не дошли» → незавершённая миграция; «сознательно» → подтверждает callJs-gap.
Приоритет: средний
```

---

_Черновые рабочие заметки по этому репо — в `notes.md` (рядом)._
