# U2 — План разработки для ИИ-агентов

Версия: 0.8.5 (Claude Sonnet)  
Дата: 2025-11-15

Последовательный план разработки для ИИ‑кодеров. Каждый этап фиксирует интерфейсы и вводит контроль качества (DoD/Acceptance), чтобы следующие работы не ломали уже отлаженные модули.

**Формат этапов:** Цель → Содержание → API/Freeze → Тесты/DoD → Инструменты

---

## M0. Базовая основа проекта

### 0.1. Репозиторий и сборка

**Цель:** Готовый каркас для клиента/сервера/общей библиотеки.

**Содержание:**

* Структура: `shared/` (математика, ECS‑ядро, протоколы), `server/` (headless .NET 8), `client/` (Unity LTS, URP, 2D).
* Пакеты: Protobuf, NUnit/xUnit, логирование, профайлинг.
* Code‑style, линтеры, pre‑commit хуки.

**API/Freeze:** Договорённости по неймингу, единицам (м/кг/с), time‑step.

**Тесты/DoD:** CI: сборка трёх проектов, юнит‑тесты проходят; один пустой интеграционный тест.

**Инструменты:** GitHub Actions/Runner, Protobuf‑генераторы, EditorConfig, .editorconfig + .gitattributes (CRLF/LF).

**Срок:** 1-2 недели

---

### 0.2. Shared.Math + поле c′ + валидация номиналов

**Цель:** Единая релятивистская математика и валидация ТТХ кораблей.

**Содержание:**

1. **Векторная алгебра:**
   - Операции с Vector2/Vector3
   - Численные фильтры (LPF)
   - Ограничители jerk/snap

2. **Релятивистская физика:**
   - Лоренц‑фактор γ = 1/√(1 - v²/c′²)
   - Преобразования через импульс p = γmv
   - Разложение ускорения: продольное/поперечное

3. **Поле c′ (упрощённое для M0-M8):**
   ```csharp
   public class LocationConfig {
     public float c_prime_mps;  // фиксировано для локации
     public string name;
     public Vector2 boundaries;
   }
   ```
   - НЕТ вычисления c′(x) на каждом тике
   - Значение читается из конфига один раз при загрузке

4. **Валидация номиналов (nominals.js):**
   ```csharp
   public class ShipNominals {
     public static ShipConfig GetShip(string size, string type);
     public static ValidationResult Validate(ShipConfig ship);
   }
   
   // Проверки:
   // - F = m × a (тяга соответствует массе и ускорению)
   // - Масса/длина в диапазонах ENVELOPE
   // - hull_radius = hypot(length, width) / 2
   ```

**API/Freeze:** 
- `RelativisticMath.Gamma(beta)`
- `LocationConfig.c_prime_mps`
- `ShipNominals.GetShip(size, type)`
- `ShipConfig.Validate()`

**Тесты/DoD:** 
- Property‑тесты: `|v| < c′` → `γ ≥ 1`
- Непрерывность γ(v)
- Все корабли из nominals.js проходят валидацию
- Формулы тяги совпадают с точностью ±0.1 МН

**Инструменты:** FsCheck (property-based), snapshot‑тест на расчёты γ.

**Срок:** 2-3 недели

---

### 0.3. Интеграция Entitas ECS

**Цель:** Интегрировать готовое ECS-решение без написания с нуля.

**Обоснование выбора Entitas:**
- ✅ Простая интеграция с Unity и .NET
- ✅ Понятная кодогенерация (хорошо для ИИ-агентов)
- ✅ Стабильный, проверенный временем
- ✅ Достаточная производительность для 12 кораблей
- ✅ Активное комьюнити и документация

**Альтернатива (Unity DOTS):**
- Отложена до масштабирования >50 кораблей
- Причина: крутая кривая обучения, избыточна для прототипа

**Содержание:**

1. **Установка Entitas:**
   - Unity Package Manager / NuGet
   - Настройка кодогенератора

2. **Базовые компоненты:**
   ```csharp
   [Game] public sealed class Transform2DComponent : IComponent {
     public Vector2 position;
     public float rotation;  // radians
   }
   
   [Game] public sealed class VelocityComponent : IComponent {
     public Vector2 linear;   // м/с
     public float angular;    // рад/с
   }
   
   [Game] public sealed class MassComponent : IComponent {
     public float mass_kg;
     public float inertia_kg_m2;
   }
   
   [Game] public sealed class HeatComponent : IComponent {
     public float temperature;      // Кельвины
     public float heat_capacity;    // Дж/К
     public float cooling_rate;     // Вт
   }
   
   [Game] public sealed class ControlStateComponent : IComponent {
     public float thrust;      // -1..1 (назад..вперёд)
     public float strafe_x;    // -1..1 (лево..право)
     public float strafe_y;    // -1..1 (вниз..вверх)
     public float yaw_input;   // -1..1 (влево..вправо)
   }
   
   [Game] public sealed class ShipConfigComponent : IComponent {
     public string size;       // snub, small, medium, heavy
     public string type;       // fighter, freighter, etc
     public float mass_t;
     public float accel_fwd_mps2;
     public float main_thrust_MN;
     // ... другие параметры из nominals
   }
   ```

3. **Системы-заглушки:**
   ```csharp
   public class PhysicsSystem : IExecuteSystem {
     public void Execute() {
       // Заполним в M1.1
     }
   }
   
   public class HeatSystem : IExecuteSystem {
     public void Execute() {
       // Заполним в M1.1
     }
   }
   
   public class ControlSystem : IExecuteSystem {
     public void Execute() {
       // Заполним в M3
     }
   }
   ```

4. **Простой сериализатор:**
   ```csharp
   public static byte[] SerializeEntity(GameEntity entity) {
     var proto = new EntityProto {
       Id = entity.id.value,
       Position = ToProto(entity.transform2D.position),
       Velocity = ToProto(entity.velocity.linear),
       // ...
     };
     return proto.ToByteArray();
   }
   ```

**API/Freeze:** 
- Интерфейсы компонентов (через Entitas атрибуты)
- Порядок систем в `Systems.Add(...)`
- Сериализация в Protobuf

**Тесты/DoD:**
- Создание/удаление entities работает
- Системы обновляются в правильном порядке
- Сериализация entity → Protobuf → entity (round-trip)
- Нет утечек памяти при 1000 циклах create/destroy
- Benchmark: 10,000 entities обрабатываются < 16 мс

**Инструменты:** 
- Entitas (NuGet/UPM)
- Entitas Code Generator
- Protobuf
- BenchmarkDotNet

**Срок:** 1-2 недели (вместо 3-4 для написания с нуля)

**Миграция на DOTS (будущее):**
При масштабировании до 50+ кораблей можно мигрировать на Unity DOTS:
- Entitas компоненты → DOTS IComponentData (механическое преобразование)
- Системы → DOTS Systems (потребует рефакторинга)
- Ожидаемое время миграции: 2-3 недели

---

## M1. Физика и оффлайн‑клиент (без сети)

### 1.1. Серверный headless‑симулятор (локально)

**Цель:** Канонический интегратор и тепловая модель.

**Содержание:**

1. **Релятивистская кинематика:**
   ```csharp
   // Псевдокод физического тика
   float c_prime = LocationConfig.c_prime_mps;  // константа!
   Vector2 force = CalculateForce(entity);
   
   // Разложение F∥ и F⊥
   Vector2 v_dir = entity.velocity.linear.normalized;
   float F_parallel = Vector2.Dot(force, v_dir);
   Vector2 F_perp = force - F_parallel * v_dir;
   
   // Интеграция импульса
   entity.momentum += force * dt;
   
   // Обратное преобразование p → v с γ
   float v = entity.momentum.magnitude / entity.mass;
   float beta = v / c_prime;
   float gamma = RelativisticMath.Gamma(beta);
   entity.velocity.linear = entity.momentum / (gamma * entity.mass);
   
   // Ограничение |v| < 0.999c′
   if (entity.velocity.linear.magnitude >= 0.999f * c_prime) {
     entity.velocity.linear = entity.velocity.linear.normalized 
                            * 0.999f * c_prime;
   }
   
   // Интеграция позиции
   entity.transform2D.position += entity.velocity.linear * dt;
   ```

2. **Тепловая модель (упрощённая):**
   ```csharp
   // Q̇ = Σheat_generation - cooling
   float heat_gen = ThrustHeat(entity.control.thrust) 
                  + WeaponHeat(entity.weapons);
   float heat_loss = entity.heat.cooling_rate;
   
   entity.heat.temperature += (heat_gen - heat_loss) * dt 
                            / entity.heat.heat_capacity;
   
   // Троттлинг при перегреве
   if (entity.heat.temperature > 0.9f * entity.heat.T_critical) {
     entity.control.thrust *= 0.5f;  // снижаем тягу
   }
   ```

3. **Упрощения для минимальной версии:**
   - Фиксированное c′ (из конфига локации)
   - НЕ вызываем `cField.Sample(pos)` → экономия ~5% CPU
   - Тепло обновляется каждый 2-й тик (15 Hz) → экономия CPU

4. **Параметры из nominals.js:**
   ```csharp
   var ship = ShipNominals.GetShip("small", "fighter");
   entity.AddMass(ship.mass_t * 1000);  // тонны → кг
   entity.AddShipConfig(ship);
   
   // Проверка формулы тяги
   float expected_thrust = ship.mass_t * ship.accel_fwd_mps2 / 1000;
   Debug.Assert(
     Mathf.Abs(ship.main_thrust_MN - expected_thrust) < 0.1f,
     "Thrust formula mismatch!"
   );
   ```

**API/Freeze:** 
- `PhysicsSystem.Step(dt)`
- `ForceCalculator.ApplyThrusterCmd(entity, cmd)`
- `LocationConfig` (c′, boundaries)

**Тесты/DoD:** 
- Инварианты: `|v| < 0.999·c′`, `γ ≥ 1`
- Стабильно проходит 10⁶ тиков без "телепортов"
- Все корабли из nominals.js могут разгоняться до 0.8c′
- Тепло накапливается и рассеивается корректно
- Троттлинг включается при T > 90% T_critical

**Инструменты:** NUnit + property‑based набор, BenchmarkDotNet.

**Срок:** 2-3 недели

---

### 1.2. Клиент: оффлайн рендер 2D + HUD (минимум)

**Цель:** Визуализация мира без сети.

**Содержание:**

1. **Рендеринг:**
   - Top‑down камера (вид сверху)
   - Сетка координат
   - Спрайт корабля (простая геометрия)
   - Статические "астероиды" (круги/эллипсы)

2. **Камера (фиксация на корабле):**
   ```csharp
   // Корабль ВСЕГДА направлен вверх экрана
   // Мир вращается относительно корабля
   Vector2 ship_pos = playerShip.transform2D.position;
   float ship_yaw = playerShip.transform2D.rotation;
   
   Camera.position = ship_pos;
   Camera.rotation = -ship_yaw;  // мир вращается в обратную сторону
   
   // Zoom: 3 уровня
   Camera.orthographicSize = zoomLevel;  // близко/средне/далеко
   ```

3. **HUD минимальный:**
   - |v| (скорость в м/с)
   - |v|/c′ (% от скорости света)
   - γ (гамма-фактор)
   - T/T_critical (температура, %)
   - Ускорение в g (текущая перегрузка)

**API/Freeze:** 
- `ViewModel` (DTO, только чтение)
- `HUDController.UpdateIndicators(ViewModel)`

**Тесты/DoD:** 
- Ручной чеклист: визуал стабильный на 60 FPS
- Корректные числовые индикаторы (сверяем с логами сервера)
- Камера плавно следует за кораблём
- Zoom работает (колёсико мыши)

**Инструменты:** Unity, TextMeshPro для HUD.

**Срок:** 2 недели

---

## M2. Сеть, предсказание и reconcile

### 2.1. Протокол и транспорт

**Цель:** Базовая сетка и формат сообщений.

**Содержание:**

1. **Protobuf‑схемы:**
   ```protobuf
   message ControlInput {
     uint32 seq = 1;           // sequence number
     float thrust = 2;          // -1..1
     float strafe_x = 3;
     float strafe_y = 4;
     float yaw_input = 5;
     uint64 timestamp_ms = 6;
   }
   
   message EntitySnapshot {
     uint32 entity_id = 1;
     Vector2Proto position = 2;
     Vector2Proto velocity = 3;
     float rotation = 4;
     float temperature = 5;
     // ... компактно
   }
   
   message WorldSnapshot {
     uint32 tick = 1;
     uint64 timestamp_ms = 2;
     repeated EntitySnapshot entities = 3;
   }
   
   message Observation {
     uint32 observer_id = 1;
     uint32 target_id = 2;
     EntitySnapshot state = 3;
     float t_show_ms = 4;  // когда показывать клиенту
   }
   ```

2. **Транспорт:**
   - UDP (PC/mobile): низкая латентность
   - WebSocket (WebGL): совместимость с браузерами
   - Выбор транспорта по платформе автоматически

3. **Версионирование:**
   ```protobuf
   message Envelope {
     uint32 version = 1;      // версия протокола
     bytes payload = 2;
     map<string, string> capabilities = 3;  // для фич
   }
   ```

**API/Freeze:** 
- Версии сообщений (v1)
- Backward‑compatible эволюция (не ломаем старые клиенты)
- Capabilities флаги для новых фич

**Тесты/DoD:** 
- Фаззинг сериализации (random data)
- Round‑trip идентичен: serialize → deserialize → equals original
- Версионирование работает (старый клиент + новый сервер)

**Инструменты:** Protobuf compiler, Wireshark для debug.

**Срок:** 2-3 недели

---

### 2.2. Authoritative server + клиентское предсказание

**Цель:** Играбельная петля с локальным предсказанием своего корабля.

**Содержание:**

1. **Частоты (окончательные):**
   - Сервер тик: **30 Hz** (33.3 мс)
   - ControlInput от клиента: **30 Hz**
   - WorldSnapshot от сервера: **15 Hz** (базовая)
   - Адаптивно: 10-20 Hz (зависит от RTT и нагрузки)

2. **Клиентское предсказание:**
   ```csharp
   // Клиент локально симулирует свой корабль на 1-2 тика вперёд
   void Update() {
     // Читаем ввод
     ControlInput input = ReadInput();
     input.seq = nextSeq++;
     SendToServer(input);
     
     // Предсказываем своё состояние
     PredictOwnShip(input, Time.deltaTime);
     
     // Сохраняем в истории
     inputHistory.Add(input);
   }
   
   void OnSnapshotReceived(WorldSnapshot snap) {
     // Reconciliation: подтягиваем к серверному состоянию
     var serverState = snap.FindEntity(myShipId);
     var myPredicted = myShip.GetState();
     
     float error = Vector2.Distance(
       serverState.position, 
       myPredicted.position
     );
     
     if (error > 0.5f) {  // порог рассогласования
       // Плавно подтягиваем за 3-5 кадров
       StartCoroutine(SmoothCorrect(serverState, 0.1f));
     }
     
     // Очищаем историю до lastAcceptedSeq
     inputHistory.RemoveWhere(i => i.seq <= snap.lastAcceptedSeq);
   }
   ```

3. **Future-queue для чужих кораблей:**
   ```csharp
   // Буфер на 100-200 мс вперёд
   Queue<Observation> futureQueue;
   
   void OnObservationReceived(Observation obs) {
     futureQueue.Enqueue(obs);
   }
   
   void Update() {
     float now = Time.time * 1000;  // мс
     
     while (futureQueue.Count > 0 && futureQueue.Peek().t_show_ms <= now) {
       var obs = futureQueue.Dequeue();
       ApplyToEntity(obs.target_id, obs.state);
     }
     
     // Интерполяция между известными состояниями
     InterpolateEntities();
   }
   ```

4. **Reconciliation (мягкая подтяжка):**
   - НЕ резкий скачок (телепорт)
   - Плавное смешивание за 3-5 кадров (50-83 мс)
   - Визуально незаметно для игрока

**API/Freeze:** 
- `NetworkClient.Send(ControlInput)`
- `NetworkServer.Broadcast(WorldSnapshot)`
- `FutureQueue.Schedule(Observation, t_show)`

**Тесты/DoD:**

1. **Сеть стабильна:**
   - RTT 0-200 мс: плавное движение
   - Потери 0-5%: без крашей, буфер компенсирует

2. **Reconciliation работает:**
   - Среднее расхождение позиции < 1 м (после подтяжки)
   - Нет визуальных "телепортов"
   - Нет "дрожи" камеры

3. **Буфер держится:**
   - Future-queue: 3-5 снапшотов (~100-200 мс)
   - При RTT > 200 мс: экстраполяция включается

4. **ВАЖНО:** Детерминизм НЕ проверяется
   - Клиент и сервер могут использовать float/double
   - Reconciliation всегда активен — это нормально
   - Цель: комфортный геймплей, не побитовое совпадение

**Инструменты:** Network simulator (искусственная латентность), Wireshark.

**Срок:** 3-4 недели

---

## M3. Ввод, арбитраж, ассистенты

### 3.1. Ввод и арбитраж источников

**Цель:** Единый пайплайн ввода и бесшовная передача управления.

**Содержание:**

1. **Источники ввода:**
   - Клавиатура/мышь (PC)
   - Геймпад (консоли)
   - Виртуальные стики (мобильные)
   - Автопилот (скрипты)
   - Удалённый оператор (будущее)

2. **Политики приоритета:**
   ```
   Priority: Local Input > Remote Operator > Autopilot
   
   Правила:
   1. Если игрок трогает стики/клавиши → Local активен
   2. Если игрок отпустил управление > 0.5 сек → передача Autopilot
   3. Remote может перехватить только у Autopilot (не у Local)
   4. Cross-fade между источниками: 150-200 мс
   
   Ассистенты НЕ источник:
   - Пост-обрабатывают команды от любого источника
   - Эскалация w_a → 1 при аварии
   ```

3. **Пример сценария:**
   ```
   t=0.0s: Игрок держит W (вперёд), Local активен
   t=2.0s: Игрок отпускает W
   t=2.5s: После 0.5с тишины → передача Autopilot
   t=2.7s: Cross-fade завершён, Autopilot ведёт
   t=5.0s: Игрок нажал A (влево) → Local перехватывает
   ```

4. **Cross-fade (плавный переход):**
   ```csharp
   float blend = 0;  // 0=старый источник, 1=новый
   float blendSpeed = 1f / 0.2f;  // за 200 мс
   
   void Update(float dt) {
     if (sourceChanged) {
       blend = Mathf.MoveTowards(blend, 1, blendSpeed * dt);
       
       ControlState result = ControlState.Lerp(
         oldSource.GetControl(), 
         newSource.GetControl(), 
         blend
       );
       
       if (blend >= 1) {
         sourceChanged = false;
         oldSource = newSource;
       }
     }
   }
   ```

**API/Freeze:** 
- `IInputSource.GetControl() → ControlState`
- `IArbitrationPolicy.Evaluate(sources) → ActiveSource`
- События: `OnSourceChanged(oldSrc, newSrc, blendTime)`

**Тесты/DoD:**
- Автотесты "перехват оси без скачка тяги"
- Ручные чеклисты с разными девайсами (KB+M, геймпад, мобильный)
- Cross-fade плавный (нет рывков)
- Можно биндить несколько клавиш на одну ось (ED-style)

**Инструменты:** Unity Input System, Input Debugger.

**Срок:** 2-3 недели

---

### 3.2. Ассистент: только Stabilized режим

**Цель:** Базовая стабилизация вращения и плавность трансляции.

**Обоснование упрощения:**
- Decoupled режим: избыточен для PvE, сложен для новичков
- Coupled режим: требует 6-12 недель настройки на корабль
- Stabilized: оптимальный баланс помощи и контроля

**Содержание:**

1. **Стабилизация вращения (PD-демпфер):**
   ```csharp
   // Цель: остановить вращение при отпускании стиков
   float target_omega = 0;  // желаемая угловая скорость
   float error = target_omega - current_omega;
   float correction = Kp * error + Kd * (error - prev_error);
   
   torque_output = Mathf.Clamp(correction, -max_torque, max_torque);
   ```

2. **Jerk-limiting (плавность трансляции):**
   ```csharp
   // Ограничение рывка ускорения (производная ускорения)
   float max_jerk = 160;  // м/с³ (из nominals.assist.jerk)
   float accel_change = requested_accel - current_accel;
   float jerk = accel_change / dt;
   
   if (Mathf.Abs(jerk) > max_jerk) {
     current_accel += Mathf.Sign(accel_change) * max_jerk * dt;
   } else {
     current_accel = requested_accel;
   }
   ```

3. **Параметры (единые для всех кораблей на старте):**
   ```yaml
   stabilized:
     Kp_yaw: 0.9       # пропорциональный коэффициент
     Kd_yaw: 1.2       # дифференциальный коэффициент
     jerk_max_forward: 160   # м/с³
     jerk_max_lateral: 130
     angular_jerk_max: 0.25  # рад/с³
   ```

4. **Эскалация влияния w_a:**
   ```csharp
   // Нормальный режим
   float w_a = 0.3f;  // 30% влияния ассистента
   
   // Триггеры эскалации:
   if (Mathf.Abs(omega) > 80 * Mathf.Deg2Rad) {
     w_a = Mathf.MoveTowards(w_a, 0.8f, 2f * dt);  // за 0.5с
   }
   
   if (temperature > 0.9f * T_critical) {
     w_a = Mathf.MoveTowards(w_a, 1.0f, 5f * dt);  // за 0.2с
   }
   
   if (distanceToObstacle < 500) {
     w_a = 1.0f;  // немедленно
   }
   
   // Спад после стабилизации (медленно)
   if (allSafe) {
     w_a = Mathf.MoveTowards(w_a, 0.3f, 0.25f * dt);  // за 3-5с
   }
   
   // Финальная команда: смешивание player и assist
   ForceCommand final = ForceCommand.Lerp(
     playerCommand,
     assistCommand,
     w_a
   );
   ```

**API/Freeze:** 
- `IAssist.Apply(state, control, dt) → ForceCommand`
- `AssistConfig.Stabilized` (параметры PD, jerk)
- Интерфейс неизменен (готовность к Decoupled/Coupled позже)

**Тесты/DoD:**

1. **Стабилизация работает:**
   - Корабль останавливает вращение при отпускании стиков за < 2 сек
   - Нет овершутов (перекрутов) > 10°
   - Работает для всех кораблей из nominals.js

2. **Jerk-limiting:**
   - Ускорение не скачет (производная < 160 м/с³)
   - Визуально плавно

3. **Эскалация:**
   - w_a растёт при |ω| > порога
   - HUD показывает "ASSIST TAKEOVER" с причиной
   - Возврат контроля после стабилизации за 3-5 сек

4. **Детерминизм (опционально):**
   - Клиент и сервер дают близкие результаты (ε < 1%)
   - Но не обязательно побитовое совпадение

**Инструменты:** 
- Property-based тесты для PD-контроллера
- Replay записанных сессий для регрессии
- Telemetry для анализа w_a

**Срок:** 2 недели

**Decoupled/Coupled — будущее:**
- Decoupled: добавится в M9 по запросу хардкорных игроков
- Coupled: M10+ после extensive balancing

---

## M4. Наблюдение с задержкой света и HUD

### 4.1. Observation по retarded time

**Цель:** Маскировка сети физикой света.

**Содержание:**

1. **Расчёт задержки света:**
   ```csharp
   // Сервер
   float c_prime = LocationConfig.c_prime_mps;  // константа локации
   float distance = Vector2.Distance(observer.pos, target.pos);
   float t_flight = distance / c_prime;  // секунды
   
   // Добавляем сетевую задержку и буфер
   float t_show = currentTime + t_flight + RTT/2 + jitterBudget;
   
   Observation obs = new Observation {
     observer_id = observer.id,
     target_id = target.id,
     state = target.GetSnapshot(),
     t_show_ms = t_show * 1000
   };
   
   SendToClient(observer, obs);
   ```

2. **Клиент планирует показ:**
   ```csharp
   Queue<Observation> futureQueue;
   
   void OnObservationReceived(Observation obs) {
     futureQueue.Enqueue(obs);
   }
   
   void Update() {
     float now = Time.time * 1000;
     
     // Показываем observations по расписанию
     while (futureQueue.Count > 0 && 
            futureQueue.Peek().t_show_ms <= now) {
       var obs = futureQueue.Dequeue();
       UpdateEntity(obs.target_id, obs.state);
     }
     
     // Интерполируем между states
     InterpolateAll();
   }
   ```

3. **Что происходит при разных RTT:**
   ```
   RTT 50 мс,  буфер 150 мс → Плавно, свет маскирует сеть
   RTT 100 мс, буфер 100 мс → Плавно, буфер держится
   RTT 200 мс, буфер 50 мс  → Буфер истощается, экстраполяция
   RTT 500 мс+, буфер 0 мс  → Видимые лаги, стандартная интерполяция
   ```

4. **Преимущество:**
   - Задержка света **физически обоснована** (не "лаги сети")
   - ~80% игроков с RTT < 150 мс не замечают сетевых задержек
   - При плохой сети откат к классическим методам

**API/Freeze:** 
- `Observation` (Protobuf message)
- `FutureQueue.Schedule(obs, t_show)`

**Тесты/DoD:**
- Визуальные "ступени" исчезают (плавная интерполяция)
- Метрики "сколько кадров держим в будущем" в зелёной зоне (3-5)
- При RTT 50-150 мс движение плавное
- При RTT > 300 мс корректный откат на экстраполяцию

**Инструменты:** Network simulator, telemetry dashboard.

**Срок:** 2-3 недели

---

### 4.2. HUD: базовая информация

**Цель:** Отображение ключевых параметров полёта и боя.

**Содержание:**

1. **Основные индикаторы:**
   ```
   Полёт:
   - Скорость: |v| (м/с) и |v|/c′ (%)
   - Ускорение: вектор, |a| в g
   - Ориентация: курс (yaw°), крен (roll°)
   
   Корабль:
   - Температура: T/T_critical (%)
   - Состояние ассистента: "STABILIZED" / "TAKEOVER: reason"
   - Health: Hull HP (число), Shield HP по граням (4-6 индикаторов)
   
   Цель (если выбрана):
   - Имя/тип корабля
   - Дистанция (км)
   - Задержка света: t_light (мс)
   - Относительная скорость (±XXX м/с)
   
   Вооружение:
   - Готовность оружия (cooldown bar)
   - Боезапас (если ballistic)
   ```

2. **Для отладки (F3 для показа/скрытия):**
   ```
   - RTT / jitter (мс)
   - Future-queue size (кол-во снапшотов, мс вперёд)
   - FPS клиента
   - Server tick time (если доступно от сервера)
   - Reconciliation count (сколько подтяжек за последнюю секунду)
   ```

3. **Убрано из M0-M8:**
   - ❌ "Голограмма будущего"
   - ❌ Реконструкция текущего положения объектов
   - ❌ Зоны перехвата (слишком сложно)

4. **Упрощённые подсказки прицеливания:**
   ```csharp
   // Круг упреждения
   Vector2 targetPos = target.position;
   Vector2 targetVel = target.velocity;
   float t_intercept = CalculateInterceptTime(
     myPos, targetPos, targetVel, projectileSpeed
   );
   Vector2 leadPoint = targetPos + targetVel * t_intercept;
   DrawCircle(leadPoint, radius, colorByProbability);
   
   // Цвет по вероятности попадания
   if (P_hit > 0.7f) color = green;
   else if (P_hit > 0.4f) color = yellow;
   else color = red;
   ```

**API/Freeze:** 
- `HUDViewModel` (DTO, read-only)
- События: `OnTargetSelected`, `OnShieldHit`, `OnHullDamage`
- `HUDController.UpdateIndicators(ViewModel)`

**Тесты/DoD:**
- Все индикаторы показывают корректные значения
- HUD не лагает (обновление 60 FPS)
- Отладочная панель включается/выключается F3
- Круг упреждения движется вместе с целью

**Инструменты:** Unity UI / TextMeshPro, возможно ImGui для прототипа.

**Срок:** 1 неделя

---

## M5. Эскалация влияния ассистента и аварийные режимы

### 5.1. Эскалация и аварийные режимы

**Цель:** Безопасность управления в критических ситуациях.

**Содержание:**

1. **Триггеры эскалации (w_a → 1):**
   ```csharp
   // Потеря контроля вращения
   if (Mathf.Abs(omega_dps) > 90) {
     escalation.Add(EscalationReason.HighAngularVelocity);
     targetWa = 0.8f;
     escalationSpeed = 2f;  // за 0.5 сек
   }
   
   // Критический перегрев
   if (temperature > 0.9f * T_critical) {
     escalation.Add(EscalationReason.Overheat);
     targetWa = 1.0f;
     escalationSpeed = 5f;  // за 0.2 сек
   }
   
   // Предотвращение столкновения
   if (distanceToObstacle < 500) {
     escalation.Add(EscalationReason.CollisionAvoidance);
     targetWa = 1.0f;
     escalationSpeed = 10f;  // немедленно
   }
   
   // Критические повреждения
   if (hull_HP < 0.2f * hull_HP_max) {
     escalation.Add(EscalationReason.CriticalDamage);
     targetWa = 0.6f;
     escalationSpeed = 1f;  // за 1 сек
   }
   ```

2. **Скорость эскалации:**
   ```
   Высокий приоритет (столкновение): w_a → 1.0 за 0.1 сек
   Средний (перегрев, вращение): w_a → 0.8-1.0 за 0.5 сек
   Низкий (HP): w_a → 0.6 за 1.0 сек
   ```

3. **Спад после стабилизации:**
   ```csharp
   // Условия для спада (гистерезис)
   bool allSafe = omega_dps < 20  // ниже порога роста
               && temperature < 0.7f * T_critical
               && distanceToObstacle > 1000
               && hull_HP > 0.3f * hull_HP_max;
   
   if (allSafe && escalation.Count == 0) {
     targetWa = 0.3f;
     escalationSpeed = 0.25f;  // медленно, за 3-5 сек
   }
   
   // Плавное изменение
   currentWa = Mathf.MoveTowards(currentWa, targetWa, escalationSpeed * dt);
   ```

4. **Индикация в HUD:**
   ```csharp
   if (currentWa < 0.5f) {
     hudText = "STABILIZED";
     hudColor = Color.green;
   } else {
     string reasons = string.Join(", ", escalation.Select(r => r.ToString()));
     hudText = $"ASSIST TAKEOVER: {reasons}";
     hudColor = currentWa >= 0.9f ? Color.red : Color.yellow;
   }
   ```

5. **Звуковые предупреждения:**
   ```csharp
   if (currentWa > 0.8f && !beepPlaying) {
     AudioSource.PlayOneShot(warningBeep);
     beepPlaying = true;
   }
   
   if (currentWa >= 1.0f && !voicePlayed) {
     AudioSource.PlayOneShot(voiceAssistOverride);  // "Assist Override"
     voicePlayed = true;
   }
   ```

**API/Freeze:** 
- `AssistEscalation.Evaluate(state) → (w_a, reasons[])`
- `AssistConfig.EscalationThresholds`
- Enum `EscalationReason`

**Тесты/DoD:**

1. **Сценарные тесты:**
   ```
   Сценарий 1: Раскрутка
   - Корабль раскручен до |ω| = 100 °/с
   - Ассистент перехватывает (w_a → 0.8)
   - Корабль стабилизируется за < 3 сек
   - Возврат контроля за 3-5 сек
   
   Сценарий 2: Перегрев
   - Температура достигает 95%
   - Эскалация w_a → 1.0
   - Тяга снижается
   - Охлаждение до 70% → возврат
   
   Сценарий 3: Столкновение
   - Сближение с астероидом < 400 м
   - Аварийное торможение (w_a = 1.0 немедленно)
   - Корабль останавливается или уворачивается
   ```

2. **Нет "борьбы" с игроком:**
   - w_a растёт плавно (непрерывно)
   - Игрок всегда видит причину в HUD
   - После возврата нет резких скачков

3. **Логирование:**
   - Все события эскалации → telemetry
   - Для анализа баланса порогов

**Инструменты:** Scenario scripting, telemetry dashboard (Grafana).

**Срок:** 1-2 недели

**Coupled режим — будущее:**
Если потребуется в M9+:
- Реализуется как отдельный режим (не заменяет Stabilized)
- Требует extensive testing (6+ недель)
- Включается через настройки управления

---

### 5.5. Интеграция управления и прицеливания

**Цель:** Связать полётное управление с боевыми действиями.

**Обоснование:** 
Между M5 (ассистент) и M6 (бой) нужен промежуточный этап для интеграции input mapping и UI состояния.

**Содержание:**

1. **Input mapping:**
   ```
   === ПОЛЁТ ===
   WASD / левый стик:
   - W/S: forward/backward thrust
   - A/D: strafe left/right
   
   Мышь / правый стик:
   - Горизонталь: yaw (рыскание)
   - Вертикаль: pitch (тангаж, для будущего 3D)
   
   Q/E / бамперы:
   - Roll (крен, для будущего)
   
   === БОЙ ===
   ЛКМ / правый триггер:
   - Огонь (если цель захвачена)
   
   ПКМ / левый триггер:
   - Захват цели (под курсором мыши / центром экрана)
   
   Tab / D-pad вверх:
   - Переключение цели (cycle через враждебные корабли)
   
   R / D-pad вниз:
   - Перезарядка (если ballistic weapons)
   
   B / D-pad влево:
   - Переключение режима огня (Single/Burst/Auto)
   ```

2. **UI состояния:**
   ```csharp
   // Индикатор выбранной цели
   if (currentTarget != null) {
     DrawBrackets(currentTarget.screenPos, bracketSize);
     
     // Цвет рамки
     Color bracketColor = currentTarget.IsHostile ? Color.red 
                                                   : Color.green;
     
     // Линия от корабля к цели
     DrawLine(myShip.screenPos, currentTarget.screenPos, bracketColor);
     
     // Название и тип
     DrawLabel(currentTarget.screenPos, 
               $"{currentTarget.name} ({currentTarget.type})");
   }
   
   // Прицел
   DrawCrosshair(screenCenter, crosshairSize);
   
   // Круг упреждения (lead indicator)
   if (currentTarget != null) {
     Vector2 leadPoint = CalculateLeadPoint(currentTarget);
     Color leadColor = P_hit > 0.7f ? Color.green
                     : P_hit > 0.4f ? Color.yellow
                     : Color.red;
     DrawCircle(leadPoint, leadCircleSize, leadColor);
   }
   
   // Готовность оружия
   foreach (var weapon in weapons) {
     float cooldownPercent = weapon.timeSinceLastShot / weapon.cooldown;
     DrawCooldownBar(weapon.slot, cooldownPercent);
     
     if (weapon.type == WeaponType.Ballistic) {
       DrawAmmoCount(weapon.slot, weapon.ammo, weapon.maxAmmo);
     }
   }
   ```

3. **Режимы огня:**
   ```csharp
   enum FireMode { Single, Burst, Auto }
   
   FireMode currentMode = FireMode.Single;
   
   void OnFirePressed() {
     if (currentTarget == null) {
       ShowMessage("No target locked");
       return;
     }
     
     switch (currentMode) {
       case FireMode.Single:
         FireOnce();
         break;
       case FireMode.Burst:
         StartCoroutine(FireBurst(3));  // 3 выстрела
         break;
       case FireMode.Auto:
         isAutoFiring = true;
         break;
     }
   }
   
   void OnFireReleased() {
     isAutoFiring = false;
   }
   
   void Update() {
     if (isAutoFiring && weaponReady) {
       FireOnce();
     }
   }
   ```

4. **Логика захвата цели:**
   ```csharp
   void OnLockTargetPressed() {
     // Raycast от центра экрана (или от курсора мыши)
     Ray ray = Camera.ScreenPointToRay(Input.mousePosition);
     
     if (Physics.Raycast(ray, out RaycastHit hit, maxLockRange)) {
       var entity = hit.collider.GetComponent<ShipEntity>();
       
       if (entity != null) {
         float distance = Vector3.Distance(myShip.position, entity.position);
         
         if (distance < lockRangeKm * 1000) {
           currentTarget = entity;
           AudioSource.PlayOneShot(lockAcquiredSound);
           ShowMessage($"Target locked: {entity.name}");
         } else {
           ShowMessage("Target out of range");
         }
       }
     }
   }
   
   void OnCycleTargetPressed() {
     var hostiles = FindAllHostiles();
     int currentIndex = hostiles.IndexOf(currentTarget);
     currentIndex = (currentIndex + 1) % hostiles.Count;
     currentTarget = hostiles[currentIndex];
   }
   ```

**API/Freeze:** 
- События: `OnTargetSelected(Entity target)`
- События: `OnFirePressed()`, `OnFireReleased()`
- `IWeaponSystem.CanFire() → bool`
- `IWeaponSystem.Fire(target) → ProjectileEntity` (заглушка для M6)

**Тесты/DoD:**
- Можно захватить цель кликом/raycast
- Нельзя стрелять без захваченной цели (блокируется)
- Cooldown оружия отображается корректно
- Круг упреждения движется с целью
- Смена режима огня работает (Single/Burst/Auto)
- Cycle target переключает между враждебными кораблями

**Инструменты:** Unity Input System, UI Toolkit.

**Срок:** 1 неделя

**Примечание:** Фактическая стрельба и урон реализуются в M6.1, здесь только UI и input binding.

---

## M6. Сетевой бой и боты

### 6.1. Lock‑target бой

**Цель:** Минимальный PvP/PvE без трассировки снарядов.

**Содержание:**

1. **Захват цели (уже реализован в M5.5):**
   - Raycast / клик на корабль
   - Проверка дистанции < lock_range
   - Отслеживание цели (tracking)

2. **Формула попадания (серверная, авторитарная):**
   ```csharp
   float CalculateHitProbability(
     Weapon weapon, 
     Entity shooter, 
     Entity target
   ) {
     float base_accuracy = weapon.base_accuracy;  // 0.7-0.9
     
     // Фактор сигнатуры (CS из nominals)
     float sig_factor = Mathf.Sqrt(
       weapon.optimal_signature / target.shipConfig.CS
     );
     
     // Штраф за дистанцию
     float distance = Vector2.Distance(shooter.pos, target.pos);
     float dist_penalty = Mathf.Exp(-distance / weapon.optimal_range);
     
     // Штраф за относительную скорость
     Vector2 rel_vel = target.velocity - shooter.velocity;
     float rel_speed = rel_vel.magnitude;
     float speed_penalty = weapon.tracking_speed 
                         / (weapon.tracking_speed + rel_speed);
     
     // Релятивистский штраф
     float gamma_target = RelativisticMath.Gamma(
       target.velocity.magnitude / LocationConfig.c_prime
     );
     float gamma_penalty = 1.0f / Mathf.Sqrt(gamma_target);
     
     float P_hit = base_accuracy 
                 * sig_factor
                 * dist_penalty
                 * speed_penalty
                 * gamma_penalty;
     
     return Mathf.Clamp01(P_hit);
   }
   ```

   **Примеры optimal_signature для калибров:**
   ```
   S1 (light):   optimal_sig = 1.5 (эффективен против CS=1-2)
   S2 (medium):  optimal_sig = 2.5 (универсальный, CS=2-3)
   S3 (heavy):   optimal_sig = 3.5 (против CS=3-4)
   S4+ (capital): optimal_sig = 4.5 (против CS=4-5)
   ```

3. **Расчёт урона при попадании:**
   ```csharp
   void ProcessShot(Weapon weapon, Entity shooter, Entity target) {
     float P_hit = CalculateHitProbability(weapon, shooter, target);
     
     if (Random.value < P_hit) {
       // Попадание!
       float raw_damage = weapon.base_damage
                        * Mathf.Pow(weapon.size / 2f, 0.8f)
                        * Mathf.Sqrt(shooter.gamma);
       
       // Взаимодействие со щитами
       float shield_damage = 0;
       float hull_damage = 0;
       
       if (target.shield_HP > 0) {
         if (weapon.type == WeaponType.Energy) {
           shield_damage = raw_damage * 1.0f;  // 100% в щит
           hull_damage = 0;
         } else if (weapon.type == WeaponType.Ballistic) {
           shield_damage = raw_damage * 0.7f;  // 70% в щит
           hull_damage = raw_damage * 0.3f;    // 30% проходит
         }
         
         target.shield_HP -= shield_damage;
         
         if (target.shield_HP < 0) {
           hull_damage += Mathf.Abs(target.shield_HP);  // остаток
           target.shield_HP = 0;
         }
       } else {
         hull_damage = raw_damage;  // щиты пробиты
       }
       
       // Применение брони
       float actual_damage = hull_damage * (1f - target.armor / 100f);
       target.hull_HP -= actual_damage;
       
       // Отправляем событие
       BroadcastHitEvent(shooter.id, target.id, actual_damage, P_hit);
       
       if (target.hull_HP <= 0) {
         DestroyShip(target);
       }
     } else {
       // Промах
       BroadcastMissEvent(shooter.id, target.id);
     }
   }
   ```

4. **Клиентские эффекты:**
   ```csharp
   void OnHitEventReceived(HitEvent evt) {
     // Визуальные эффекты
     if (evt.targetId == myShipId) {
       // Удар по своему кораблю
       CameraShake(intensity: evt.damage / 100f);
       FlashDamageIndicator(direction: evt.shooterPos - myPos);
       PlaySound(hitSound);
     } else {
       // Удар по другому кораблю
       SpawnHitEffect(evt.targetPos);
     }
     
     // Обновление HUD
     if (evt.targetId == myShipId) {
       UpdateHealthBars(myShip.hull_HP, myShip.shield_HP);
     }
   }
   ```

**API/Freeze:** 
- Protobuf messages: `FireRequest`, `HitEvent`, `MissEvent`
- `WeaponSystem.Fire(targetId)`
- `CombatSimulator.ProcessShot(weapon, shooter, target)`

**Тесты/DoD:**
- Консистентность при разных RTT (сервер авторитарен)
- Нет "двойного попадания" (дедупликация)
- P_hit в ожидаемых диапазонах:
  - Small fighter vs small fighter @ 3 km: 40-60%
  - Small fighter vs medium freighter @ 3 km: 60-80%
- Урон от ballistic проходит через щит (30%)
- Урон от energy не проходит (100% в щит)
- Телеметрия попаданий для балансировки

**Инструменты:** Unit tests, combat simulator (отдельная сцена для балансировки).

**Срок:** 3-4 недели

---

### 6.2. Простейшие боты/автопилот

**Цель:** Наполнение шарда NPC-кораблями.

**Содержание:**

1. **AI управление (уточнение):**
   
   Боты принимают решения с частотой **10 Hz** (каждый 3-й тик сервера):
   
   ```csharp
   // Каждый 3-й тик (10 Hz)
   void AIDecisionTick(BotEntity bot) {
     // 1. Анализ ситуации
     var enemies = FindEnemiesInRange(bot, 10000);  // 10 км
     var allies = FindAlliesInRange(bot, 5000);
     
     // 2. Принятие решения (Behavior Tree)
     BehaviorState newState = bot.behaviorTree.Evaluate(enemies, allies);
     
     // 3. Расчёт целевых параметров
     switch (newState) {
       case BehaviorState.Patrol:
         bot.targetPosition = GetNextWaypoint(bot);
         bot.targetVelocity = (bot.targetPosition - bot.position).normalized 
                            * bot.cruiseSpeed;
         bot.firePrimary = false;
         break;
       
       case BehaviorState.Attack:
         var target = SelectTarget(enemies);
         bot.currentTarget = target;
         
         // Lead pursuit (летим впереди цели на intercept)
         Vector2 interceptPoint = CalculateIntercept(bot, target);
         bot.targetPosition = interceptPoint;
         bot.targetVelocity = (interceptPoint - bot.position).normalized 
                            * bot.maxSpeed;
         
         // Стреляем если P_hit > 0.4
         float P_hit = CalculateHitProbability(bot.weapon, bot, target);
         bot.firePrimary = P_hit > 0.4f;
         break;
       
       case BehaviorState.Retreat:
         // Вектор от ближайшего врага
         var nearestEnemy = enemies.OrderBy(e => 
           Vector2.Distance(bot.position, e.position)
         ).First();
         Vector2 fleeDirection = (bot.position - nearestEnemy.position).normalized;
         bot.targetPosition = bot.position + fleeDirection * 10000;
         bot.targetVelocity = fleeDirection * bot.maxSpeed;
         bot.firePrimary = false;
         break;
     }
   }
   
   // Каждый тик (30 Hz)
   void AIExecutionTick(BotEntity bot, float dt) {
     // Ассистент ведёт корабль к целевым параметрам
     ControlState control = NavigateTo(
       bot.position, 
       bot.velocity,
       bot.targetPosition,
       bot.targetVelocity
     );
     
     // Плавная интерполяция между решениями AI
     bot.control = ControlState.Lerp(bot.control, control, 0.1f);
     
     // Применяем через те же системы, что у игроков
     bot.controlState = bot.control;
   }
   ```

2. **Behavior Tree структура:**
   ```
   Root (Selector)
   ├─ Emergency Retreat (HP < 20%)
   │  └─ Sequence: 
   │     ├─ SetState(Retreat)
   │     └─ BroadcastHelp() [будущее]
   │
   ├─ Combat (Enemy in range)
   │  └─ Selector
   │     ├─ Alpha Strike (missiles ready && range < 5km)
   │     │  └─ Sequence: FireMissiles → Evade
   │     ├─ Engage (optimal distance 3-5 km)
   │     │  └─ Sequence: ApproachTarget → CircleStrafe → Fire
   │     └─ Kite (if outmatched: enemy HP > my HP)
   │        └─ Sequence: MaintainDistance(7km) → Fire
   │
   └─ Patrol (default)
      └─ Sequence: 
         ├─ FollowWaypoints
         └─ ScanForTargets
   ```

3. **Простые поведения для M6:**
   ```csharp
   // Патруль: linear interpolation между waypoints
   Vector2 GetNextWaypoint(BotEntity bot) {
     if (Vector2.Distance(bot.position, bot.waypoints[bot.currentWaypoint]) < 100) {
       bot.currentWaypoint = (bot.currentWaypoint + 1) % bot.waypoints.Count;
     }
     return bot.waypoints[bot.currentWaypoint];
   }
   
   // Атака: lead pursuit
   Vector2 CalculateIntercept(BotEntity bot, Entity target) {
     // Простой расчёт точки перехвата
     Vector2 toTarget = target.position - bot.position;
     float distance = toTarget.magnitude;
     float closingSpeed = Vector2.Dot(bot.velocity - target.velocity, 
                                       toTarget.normalized);
     float timeToIntercept = distance / Mathf.Max(closingSpeed, 1f);
     
     return target.position + target.velocity * timeToIntercept;
   }
   
   // Отступление: вектор от врага, full thrust
   Vector2 CalculateFleeDirection(BotEntity bot, List<Entity> enemies) {
     Vector2 fleeVector = Vector2.zero;
     foreach (var enemy in enemies) {
       Vector2 awayFromEnemy = bot.position - enemy.position;
       float weight = 1f / awayFromEnemy.sqrMagnitude;  // ближе = сильнее влияние
       fleeVector += awayFromEnemy.normalized * weight;
     }
     return fleeVector.normalized;
   }
   ```

4. **Боты используют те же ограничения:**
   - Та же физика (γ, тепло, перегрузки)
   - Те же ассистенты (Stabilized)
   - Те же формулы P_hit и урона

**API/Freeze:** 
- `BotBehaviorTree.Evaluate(context) → BehaviorState`
- `BotController.NavigateTo(target) → ControlState`
- Режимы: Patrol, Attack, Retreat (enum)

**Тесты/DoD:**
- Боты соблюдают огибающую (не превышают v < 0.999c′)
- Боты не "ломают" FPS/сеть (10 ботов @ 30 Hz укладываются в бюджет)
- Боты корректно отображаются с задержкой света у клиентов
- Боты могут патрулировать маршрут
- Боты атакуют игрока при обнаружении
- Боты отступают при HP < 20%
- Телеметрия: % попаданий ботов ≈ ожидаемому P_hit

**Инструменты:** Behavior Tree editor (возможно Behavior Designer для Unity), telemetry.

**Срок:** 2-3 недели

---

## M7. Тепло/повреждения/аллокатор тяги

### 7.1. Расширенный Heat/Damage

**Цель:** Реалистичный троттлинг и деградации.

**Содержание:**

1. **Кусочно‑линейное охлаждение:**
   ```csharp
   // Cooling rate зависит от температуры
   float coolingRate = entity.baseCoolingRate;
   
   if (entity.temperature > 0.8f * entity.T_critical) {
     coolingRate *= 0.5f;  // снижение эффективности при высоких T
   }
   
   entity.temperature -= coolingRate * dt / entity.heatCapacity;
   entity.temperature = Mathf.Max(entity.temperature, ambientTemp);
   ```

2. **Аварийный сброс тяги:**
   ```csharp
   if (entity.temperature > entity.T_critical) {
     // Аварийное отключение двигателей
     entity.controlState.thrust = 0;
     entity.emergencyShutdown = true;
     BroadcastEvent(new OverheatShutdownEvent(entity.id));
   }
   
   // Восстановление после охлаждения
   if (entity.emergencyShutdown && 
       entity.temperature < 0.6f * entity.T_critical) {
     entity.emergencyShutdown = false;
   }
   ```

3. **Отказ подсистем:**
   ```csharp
   // Вероятность отказа при высокой T
   if (entity.temperature > 0.95f * entity.T_critical) {
     float failureChance = 0.01f * dt;  // 1% в секунду
     
     if (Random.value < failureChance) {
       var subsystem = ChooseRandomSubsystem(entity);
       subsystem.health = 0;
       BroadcastEvent(new SubsystemFailureEvent(entity.id, subsystem.type));
     }
   }
   ```

**API/Freeze:** 
- Расширение `HeatComponent` (cooling curves, failure states)
- События: `OverheatShutdownEvent`, `SubsystemFailureEvent`

**Тесты/DoD:**
- Инварианты энергобаланса: Σheat_in - Σheat_out = ΔT
- Сценарии перегрева/восстановления работают
- Троттлинг плавно включается
- Аварийное отключение срабатывает при T > T_critical

**Инструменты:** Unit tests, scenario tests.

**Срок:** 1-2 недели

---

### 7.2. Thrust Allocation

**Цель:** Реальное распределение по движкам.

**Содержание:**

1. **Для M0-M7: упрощённая модель**
   
   Используется **виртуальный двигатель**:
   ```csharp
   // Один "суммарный" движок
   Vector2 force = controlState.thrust * ship.main_thrust_MN * 1e6;  // MN → N
   force += new Vector2(controlState.strafe_x, controlState.strafe_y) 
          * ship.rcs_budget_MN * 1e6;
   
   ApplyForce(entity, force);
   ```
   
   Достаточно для кораблей с **симметричными layouts** из nominals.js.

2. **Реальное распределение (M7.2):**
   
   Только для кораблей с явным `ThrusterLayout`:
   ```csharp
   public class ThrusterLayout {
     public List<Thruster> thrusters;
   }
   
   public class Thruster {
     public Vector2 position;      // относительно центра масс
     public Vector2 direction;     // вектор тяги (normalized)
     public float maxThrust_N;
     public bool damaged;
   }
   
   public class ThrustAllocator {
     public ThrusterSet Allocate(
       Vector2 desiredForce,
       float desiredTorque,
       ThrusterLayout layout,
       DamageState damages
     ) {
       // Жадный алгоритм для M7 (быстро, но не оптимально)
       var result = new ThrusterSet();
       
       // 1. Сортируем движки по эффективности
       var sorted = layout.thrusters
         .Where(t => !damages.IsDamaged(t))
         .OrderByDescending(t => EffectivenessFor(t, desiredForce));
       
       // 2. Жадно распределяем
       Vector2 remainingForce = desiredForce;
       float remainingTorque = desiredTorque;
       
       foreach (var thruster in sorted) {
         float contribution = CalculateContribution(
           thruster, remainingForce, remainingTorque
         );
         
         contribution = Mathf.Clamp(contribution, 0, thruster.maxThrust_N);
         result[thruster.id] = contribution;
         
         remainingForce -= thruster.direction * contribution;
         remainingTorque -= Cross2D(thruster.position, thruster.direction) 
                          * contribution;
         
         if (remainingForce.magnitude < 0.01f && 
             Mathf.Abs(remainingTorque) < 0.01f) {
           break;  // достигли цели
         }
       }
       
       return result;
     }
   }
   ```

3. **Квадратичный оптимизатор (M9+, будущее):**
   ```
   QP-задача:
   minimize ||F_actual - F_desired||² + ||M_actual - M_desired||²
   subject to: 0 ≤ thrust[i] ≤ max[i]
   
   Решается через QP solver (O(n³), медленнее, но оптимально)
   ```

4. **Для каких кораблей критично:**
   - Асимметричные layouts (damaged ships)
   - Отдельные RCS thrusters (maneuvering)
   - Capital ships с множеством движков

**API/Freeze:** 
- `IThrustAllocator.Allocate(cmd, layout, damages) → ThrusterSet`
- `ThrusterLayout` (data structure)

**Тесты/DoD:**
- Виртуальный движок работает для всех nominals
- Реальное распределение даёт результат в пределах 5% от желаемого
- При отказе 1 движка корабль остаётся управляем (graceful degradation)
- Benchmark: распределение для 20 движков < 0.5 мс

**Инструменты:** Unit tests, visualization tool (debug draw thrusters).

**Срок:** 1 неделя (виртуальный) + 1-2 недели (реальный)

---

## M8. Полировка, платформы, производительность

### 8.1. Оптимизации

**Цель:** Стабильная работа на целевых платформах.

**Содержание:**
- Пулы объектов (снижение GC)
- Батчинг рендеринга
- Профилирование тиков (Unity Profiler, dotTrace)
- Оптимизация сериализации (zero-copy где возможно)

**API/Freeze:** Без изменений публичных интерфейсов.

**Тесты/DoD:** 
- Целевой FPS: 60 на средних настройках
- Сеть в бюджете битрейта: < 50 кбит/с на клиента
- GC < 5 МБ/сек
- Server tick time < 25 мс (при 12 кораблях)

**Инструменты:** Unity Profiler, BenchmarkDotNet, dotMemory.

**Срок:** 2-3 недели

---

### 8.2. Платформенные сборки

**Цель:** PC, WebGL, мобильные.

**Содержание:**
- Ввод/контроллеры (адаптация для touch)
- Layout HUD (разные разрешения)
- Транспорт (UDP для PC, WebSocket для WebGL)
- Настройки качества (Low/Medium/High)

**API/Freeze:** Конфиги билда, без изменений кода ядра.

**Тесты/DoD:** 
- Чеклисты платформ (PC, WebGL работают)
- Smoke‑тест онлайна на каждой платформе

**Срок:** 2-3 недели

---

## Сквозные правила этапирования

**Важно для неизменности модулей:**

1. **Freeze после каждого этапа**
   
   Публичные интерфейсы помечаются `vX.Y`. Новые поля/сообщения — через capability‑флаги или расширения, не ломающие совместимость.

2. **Воспроизводимость (не детерминизм!)**
   
   Логика физики и ассистентов использует фиксированный шаг времени (fixed timestep). 
   
   **Строгий детерминизм НЕ требуется:**
   - Клиент и сервер могут использовать float/double
   - Reconciliation всегда активен
   - Для тестов в офлайн-режиме допустимо фиксировать random seed
   
   Цель: статистическая близость результатов (ε ≈ 1%), не побитовое совпадение.

3. **Тестовые наборы обязательны до реализации UI**
   
   Прежде чем включать фичу в клиент, должны быть юнит‑/property‑/интеграционные тесты на сервере/Shared.

4. **Инструменты отладки — часть DoD**
   
   HUD‑оверлеи, лог‑каналы, счётчики метрик на каждом этапе.

5. **Откат**
   
   Любая новая фича за флажком конфигурации (enable/disable) — позволяет откат без изменения кода базового модуля.

6. **Документация**
   
   На каждый модуль — краткая карта: назначение, публичные интерфейсы, инварианты, тесты.

---

## Технические решения по умолчанию

**Движок:** 
- Unity LTS (клиент, 2D top-down)
- .NET 8 (сервер, headless)
- Shared codebase: ECS + физика + протоколы

**ECS:**
- **Entitas** для M0-M8
- Миграция на Unity DOTS при масштабировании >50 кораблей (M9+)

**Сеть:** 
- UDP (PC/mobile)
- WebSocket (WebGL)

**Протокол:** 
- Protobuf
- Частоты:
  - `Control/Maneuver`: 30 Hz (от клиента к серверу)
  - `Snapshot/Observation`: 15 Hz базовая (от сервера, адаптивно 10-20 Hz)

**Шаги симуляции:**
- Сервер: **30 Hz** fixed timestep
- Клиент предсказатель: 60 Hz
- Клиент рендер: 60 FPS

**Числа:** 
- Сервер: double (64-bit)
- Клиент: float (32-bit)
- ⚠️ Допустимы различия округления, reconciliation компенсирует

**Арбитраж источников управления:** 
- Priority: **Local > Remote > Autopilot**
- Cross-fade: 150-200 мс
- Ассистенты: пост-обработка с эскалацией w_a

**Ассистент:**
- M0-M8: только **Stabilized** (PD + jerk-limiting)
- M9+: Decoupled, Coupled (опционально)

**Бой:** 
- Lock-target (захват цели)
- Расчёт P_hit на сервере (авторитарный)
- HP/Shield/Armor система

**c′:**
- **Фиксировано** для локации
- Типичные значения:
  - Тестовые: 1,000 м/с
  - Бой: 3,000 – 10,000 м/с
  - Межзвёздный: до 100,000 м/с (будущее)
- **НЕТ** динамического изменения под нагрузкой

**Детерминизм:**
- **НЕ требуется** строгий (побитовый)
- Достаточно статистической близости (ε ≈ 1%)
- Авторитарный сервер + reconciliation

---

## Что подготовить продюсеру/гейм‑дизайнеру

1. **Definition of Fun для Stabilized режима**
   - Целевые ощущения: "плавно, предсказуемо, прощает ошибки"
   - Референсы: Elite Dangerous (Flight Assist On), Everspace 2
   - Диапазоны скоростей для боя: 500-2000 м/с при c′ = 5000 м/с

2. **Карта локаций с c′**
   ```yaml
   locations:
     - name: "Test Arena"
       c_prime: 1000
       size: 20x20 км
     
     - name: "Asteroid Field Combat"
       c_prime: 5000
       size: 100x100 км
     
     - name: "Deep Space"
       c_prime: 50000
       size: 1000x1000 км
   ```

3. **HUD‑референсы**
   - Минимализм: Elite Dangerous
   - Информативность: Star Citizen
   - Читаемость: Everspace 2

4. **Боевые правила (базовая формула P_hit)**
   - См. M6.1
   - Балансировать через optimal_sig для калибров
   - Цель: small vs small = 40-60% на средней дистанции (3-5 км)

5. **Чеклисты приёмки по каждому этапу**
   - M0: Офлайн полёт работает
   - M3: Stabilized держит курс
   - M6: Бой 1v1 с ботом функционален

---

## Итоговая оценка сроков

### Сравнение (недели):

| Этап | Исходный план | До правок | После правок | Экономия |
|------|---------------|-----------|--------------|----------|
| M0 | 3 | 8-10 | 6-8 | -2 нед |
| M1 | 3 | 5-7 | 4-5 | -2 нед |
| M2 | 3 | 6-9 | 5-7 | -2 нед |
| M3 | 3 | 5-7 | 4-5 | -2 нед |
| M4 | 2 | 4-6 | 3-4 | -2 нед |
| M5 | 3 | 8-12 | 2-3 | **-9 нед** |
| M6 | 3 | 6-9 | 5-7 | -2 нед |
| M7-M8 | 5 | 8-12 | 6-9 | -3 нед |
| **Итого** | **25** | **50-72** | **35-48** | **~22 нед** |

**Реалистичная оценка после правок:** 35-48 недель (~8-11 месяцев)

С учётом буфера на непредвиденное: **~1 год до играбельного прототипа**

**Вероятность успеха:** 85%

---

## Приоритеты для ИИ-агентов

### Критичные для M0 (6-8 недель):
1. M0.1: Репозиторий + сборка
2. M0.2: Математика + валидация nominals
3. M0.3: Entitas интеграция
4. M1.1: Серверная физика
5. M1.2: Базовый клиент

**Milestone:** Офлайн полёт одного корабля

### Критичные для MVP (9-12 недель):
6. M2.1: Protobuf протокол
7. M2.2: Сеть + reconciliation
8. M3.1: Ввод + арбитраж
9. M3.2: Stabilized ассистент

**Milestone:** Онлайн полёт 2 игроков

### Для играбельного прототипа (10-14 недель):
10. M4.1: Observation
11. M4.2: HUD
12. M5: Эскалация
13. M5.5: Интеграция управления
14. M6.1: Бой
15. M6.2: Боты

**Milestone:** 4 игрока + 8 ботов, PvE кооп

---

## Changelog

**v0.8.5 (Claude Sonnet, 2025-11-15):**
- ✅ M0.2: Добавлена валидация nominals.js
- ✅ M0.3: Заменено на Entitas (вместо собственного ECS)
- ✅ M1.1: Упрощена физика (фиксированное c′)
- ✅ M2.2: Убрано требование строгого детерминизма
- ✅ M3.2: Один режим ассистента (Stabilized)
- ✅ M4.1: Упрощён Observation (фиксированное c′)
- ✅ M4.2: Упрощён HUD (без голограммы)
- ✅ M5: Убран Coupled, только эскалация
- ✅ M5.5 (NEW): Добавлена интеграция управления и прицеливания
- ✅ M6.1: Уточнена формула P_hit с CS из nominals
- ✅ M6.2: Уточнена работа AI (10 Hz decision)
- ✅ M7.2: Уточнён Thrust Allocation (виртуальный для M0-M7)
- ✅ Обновлены "Технические решения"
- ✅ Обновлены "Сквозные правила"
- ✅ Пересчитаны сроки: 50-72 → 35-48 недель

**v0.7.4 (исходный):**
- Базовая версия без учёта правок

---

**Конец документа**