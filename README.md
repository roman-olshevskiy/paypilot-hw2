# L03 · Automated Eval Suite — PayPilot · ДЗ №2

Автор: Roman Olshevskyi. Підготовка: 2026-10-04. Мандат: Ship it.
Статус: комплект ДЗ завершено; три живі daily-прогони та контроль відновлення виконано. Здача в LMS — посилання на main.
Репозиторій здачі: https://github.com/roman-olshevskiy/paypilot-hw2. Робоча гілка roman-olshevskyi/hw2; здача — main.

## Запуск

Кіт: https://github.com/sergeytkachenko/paypilot-l03-eval, commit 8108fc88f2cd39d193b09cb326b8463f8c439345.
Стенд: https://github.com/roman-olshevskiy/paypilot-stand, commit a876b7311d2b0a36bf4cb361cb4d27d5e211d5f2.
Покласти golden.jsonl з цього репозиторію у sets/ кіту, generate_golden.py — у корінь кіту.
Файли hw2_capture.py, hw2_runtime.py та tests/ — власне додаткове оточення; копіюються в кіт для збирання повних доказів.

Стенд має бути локальним, із live Anthropic. Ключ лише в локальному .env стенду.
Налаштування L03: STAND_DIR=../paypilot-stand, EVAL_STAND_URL=http://host.docker.internal:8000, JUDGE_MODEL=not-used.
Clock: 2026-09-15T10:00:00Z. Сам course runner не застосовує context.clock; capture adapter встановлює та перевіряє його після кожного reset.
Фактична модель усіх живих викликів за traces: claude-haiku-4-5-20251001.

З каталогу кіту:

~~~powershell
# Без моделі: регенерація та перевірка
docker compose run --rm -T eval python generate_golden.py --output sets/golden.jsonl --manifest evidence/generation-manifest.json
docker compose run --rm -T eval --set golden --dry-run
docker compose run --rm -T eval python -m unittest discover -s tests -p test_hw2_contract.py -v

# Повтор живої серії: використовувати тестовий запуск нижче
docker compose run --rm -T eval python hw2_capture.py --set golden --profile clean
docker compose run --rm -T eval python hw2_capture.py --set golden --profile lesson-03
~~~

hw2_capture.py делегує незмінному cli.py/runner.py; формат сирого course report не змінено.
Він додатково зберігає повні відповіді, request IDs, traces, input/output usage та model-call count;
відновлює попередні profile/extra defects/clock у finally. База після запуску залишається reset-станом; вихідний довільний стан БД не відновлюється.
Не запускати інші тести паралельно на цьому стенді.
Додаткові captures — evidence/runs/, сирі course reports — reports/.

set_hash: 92f5e7f20c48
Повний SHA256: 92f5e7f20c48a91a0fe03c5307abfd8f6fbd0345b595320d96525cde812130d6

## Скарги

| Скарга | Власний кейс | Клієнт / операція та уточнення | Oracle | Чому не готовий demo |
|---|---|---|---|---|
| C-01 | SWF-001 | CUS-0001; конкретизовано SWIFT 1000 EUR для перевірки сумарного тарифу | engine: fx.transfer_fee | SWIFT-кейсів у демо та генераторі кіту немає |
| C-11 | CMP-C11 | CUS-0009 Iryna, PharmaPlus TX-0902, fraud_card_not_present | engine: disputes.check | Саме ця операція та її eligibility не входять у кіт; не підміна TX-0701 |
| C-12 | CMP-C12 | CUS-0002, CloudServe TX-0201 на EUR-рахунку; TX-0202 на USD не вважаємо доведеним повторним списанням | engine: disputes.check | Новий transaction-specific eligibility case; не просте перейменування питання про «60» |
| C-14 | CMP-C14 | CUS-0002 як явний навчальний контекст; перелік усіх reason codes І duplicate window | engine: policy.DISPUTE_WINDOWS_DAYS | Нове комбіноване очікування; діагностику retrieval ця перевірка не покриває |
| C-17 | TON-HW2-001 | CUS-0002 / TX-0201 — конкретизована ситуація роздратованого клієнта | human | Нова рубрика tone/next step, release, не оцінена daily |

C-01/C-14/C-17 не містять усіх потрібних даних у скарзі; доданий навчальний контекст зазначено явно.
Друга купка triage кіту має лише три скарги, тому відбір розширено власною перевіркою контексту інших скарг.
C-03/DIS-002-N та C-09/LIM-001 уже представлені в демо і не зараховуються новими.
C-05 уже має 6000 EUR і відповідає FX-003; CMP-C05 — окремий edge-варіант на 2000 EUR, не відтворення нової скарги.
C-07 без даних початкового FX quote не оголошується точно відтвореним.
Детальна новизна кожного кейса — [походження](case-provenance.md).

## Генератор

generate_golden.py походить від generate_from_engines.py: розширено fx_cases plan, додано complaint/limit сценарії та авторські human-рубрики.
Очікування engine-кейсів обчислено реальним offline запуском чистих модулів app.engines, а не переписано з лекції.
CUSTOMERS/операції читаються з seed; transfer limits враховують усі settled outgoing транзакції, як у check_limits.
Для engine_call записано відтворювані вирази; результати кожного виклику збережено в [generation manifest](evidence/generation-manifest.json).

Додані межі:
- CUS-0007: 1000 / 1001 EUR, FX-006/007 та spread-пари.
- CUS-0001: 380 / 381 EUR після використаних 120 EUR, FX-008/009 та spread-пари.
- CUS-0002: 200 / 201 EUR після використаних 800 EUR, FX-010/011.
- SWF-002: USD1000 з fee у EUR — валюта як нова межа тарифного обчислення.

Критерій відбору: потрібні граничні пари, відмінні observable (сума/спред), нові complaint-сценарії,
контролі eligibility та daily/monthly fields; зайві готові FX-комбінації не додаються тільки заради кількості.
12 kit-сценаріїв зберігають свою ідентичність added_in=l03; параметри/числа перевірено чистими модулями.
CMP-003 (balance/DB) та CMP-006 (вузький product phrase) виключено з демо-піднабору, щоб не називати їх повним engine/catalog oracle.

Знахідки: на межі allowance спред нульовий, на один EUR понад нею — на всю суму. Результат узгоджується з правилом all-or-nothing.
Перевірка flags у C-11/C-12 очікує True: для цих конкретних дат обидві операції в межах своїх різних reason-code windows.
Додавання «60» у тексті не є доказом correct action; тому для eligibility обрано tool_result_flag.

35 кейсів: 12 kit + 18 власних обчислених + 5 власних human = 23 власних.
Daily: 30 кейсів ×1; release: 5 human ×5, вони не входять до трьох daily-прогонів.
Джерела: complaint=7, engine=12, edge=16. Oracle: engine=29, corpus=1, human=5.
Рівні: 1/2/3/4/7. Severity 100%; critical=7, high=28.
Точної новизни лише за відмінністю input недостатньо: [case-provenance.md](case-provenance.md) описує зміну параметра або observable.

## Прогони

| Прогін | Профіль | Результат | Сирий звіт |
|---|---|---|---|
| Baseline | clean | 30/30 | [golden-clean-20261004T000956.json](reports/golden-clean-20261004T000956.json) |
| Дефектна система | lesson-03 | 14/30 | [golden-lesson-03-20261004T001243.json](reports/golden-lesson-03-20261004T001243.json) |
| Власний рядок промпту | clean | 30/30 | [golden-clean-20261004T001507.json](reports/golden-clean-20261004T001507.json) |
| Контроль після відновлення | clean | 1/1 | [golden-clean-20261004T001927.json](reports/golden-clean-20261004T001927.json) |

Усі звіти мають set_hash 92f5e7f20c48. Три основні виконали ті самі 30 daily-кейсів, четвертий — лише FX-007.
П'ять release/human кейсів не запускалися і не оцінювалися.
Прогноз зафіксовано окремим [комітом 1768082](https://github.com/roman-olshevskiy/paypilot-hw2/commit/1768082aba930dd6a08fea499c3a967de57d1222)
2026-10-04 00:12:01 UTC; третій звіт створено о 00:15:07 UTC.
Точний рядок та початковий прогноз збережено нижче без виправлення заднім числом.

Фактична чутливість до цієї зміни: **0/30**, pass→fail=0, fail→pass=0; прогноз восьми падінь не справдився.
Усі вісім відповідей містять правильну final_amount після спреду; наприклад FX-007 назвав 1078.25 USD.
Рядок додано до базового файла, але модель не виконала його заборону показувати final_amount.
Це спостереження одного запуску, не доказ стійкості до довільних змін промпту.
FX-002 залишився green попри хибне пояснення часткового allowance: числовий oracle перевіряє суму, не весь текст.

Профіль lesson-03: 16 падінь, ознаки всіх п'яти дефектів у payload:
D19 — DIS-006 (90 замість 60 днів); D20 — FX-007-S (1.5% замість 0.9%);
D21 — FX-007 (spread лише на понадлімітний 1 EUR); D22 — LIM-003 (daily як monthly);
D26 — DIS-007 (eligible=true при compliance_hold).
[Матриця доказів](evidence/defect-analysis.json) містить request IDs і фактичні значення.
Дефекти ввімкнено одночасно; ізольованих mutant-прогонів не було, тому це не п'ять незалежних оцінок detection rate.

Усі дії зі стендом виконано реальними Python unittest:
test_preflight_hw2.py і test_live_hw2.py; live-тест запускає Docker eval через hw2_capture.py,
який делегує незмінному runner. Успішний unittest означає завершений прогін і збережені докази;
lesson-03 всередині має 16 quality failures.

Для повтору з каталогу кіту, після копіювання власних support-файлів і evidence/forecast-commit.json:

~~~powershell
$env:HW2_RUN_PROFILE='clean'
$env:HW2_RUN_LABEL='baseline-clean'
python -m unittest discover -s tests -p test_live_hw2.py -v
$env:HW2_RUN_PROFILE='lesson-03'
$env:HW2_RUN_LABEL='defective-lesson03'
python -m unittest discover -s tests -p test_live_hw2.py -v
$env:HW2_RUN_PROFILE='clean'
$env:HW2_RUN_LABEL='prompt-changed-clean'
$env:HW2_PROMPT_LINE=(Get-Content evidence/forecast-commit.json -Raw | ConvertFrom-Json).line
python -m unittest discover -s tests -p test_live_hw2.py -v
Remove-Item Env:HW2_PROMPT_LINE
$env:HW2_ONLY='FX-007'
$env:HW2_RUN_LABEL='restored-clean-control'
python -m unittest discover -s tests -p test_live_hw2.py -v
Remove-Item Env:HW2_ONLY
~~~

Python для host-тесту має бути реальним інтерпретатором, не Windows Store alias; Docker eval — Python 3.12.15.
Runtime base prompt відновлено побайтово: before/after SHA256 однакові.
Зовнішній prompt-guard snapshot може містити попередні D04/D05/D25, які відновлюються між тестами;
фактичні clean/lesson-03 snapshots — before.json/during.json у кожному каталозі runs, профіль кожного запиту також є в trace.
Не трактувати version label base.v1 як відсутність доданого рядка.

Чистий клон кіту: loader dry-run пройшов із тим самим hash, offline acceptance/guards — 6/6 passed. [Журнал](evidence/clean-clone-offline-tests.txt).

## Докази й вартість

[Автоматичний аналіз](evidence/run-analysis.json), [повні captures](evidence/runs/),
[походження кейсів](case-provenance.md), [залишкові ризики](residual-risk.md),
[покажчик здачі](SUBMISSION.md), [підготовка](evidence/preparation-manifest.json).
Підготовчі manifests фіксують стан до live-серії; остаточні результати наведено у run-analysis.json.

Три daily: 90 chat requests, 185 model calls, input=438084, output=20890, total=458974 tokens.
Разом із контролем: 91 chat requests, 187 model calls, input=442697, output=21153, total=463850.
Captures узгоджуються із сумою tokens кожного raw report.
Ціна Haiku 4.5: $1/M input та $5/M output ([офіційний прайс](https://platform.claude.com/docs/en/about-claude/pricing)).
Оцінка: $0.542534 за три daily, $0.005928 за контроль, **$0.548462 разом**.
Формула: (input + 5 × output) / 1000000. Judge-викликів немає.
Це оцінка за usage, без звірки billing console та окремої cache-тарифікації; вона не є підтвердженим рахунком.

## Прогноз перед третім прогоном

Рядок англійською:

~~~text
For currency conversion answers, state only the pre-spread gross amount as the amount received and do not repeat the after-spread final amount.
~~~

Прогноз: FX-004, FX-002, FX-003, FX-005, FX-007, FX-009, FX-011 і CMP-C05 мають впасти.
Підстава: чистий quote_fx поверне правильний after-spread final_amount, але текст під впливом нового рядка
покаже лише gross_amount; tool_grounded_numeric потребує того самого правильного числа й у відповіді.
FX-006, FX-008, FX-010 мають пройти: при нульовому спреді gross_amount дорівнює final_amount.
Spread-field checks, SWIFT, limits та dispute checks прямо не змінюються.
Очікуємо щонайменше одну реальну регресію; якщо прогноз не справдиться, пояснення запишемо після запуску.
Датасет: set_hash 92f5e7f20c48, 30 daily-кейсів. Прогноз записано до зміни runtime-промпту й третього report.
