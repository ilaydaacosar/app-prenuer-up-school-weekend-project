[TECH_PRD_FitMealAI_MVP.md](https://github.com/user-attachments/files/28440460/TECH_PRD_FitMealAI_MVP.md)
# TEKNIK URUN GEREKSINIM DOKUMANI (TECH PRD)

## 1) Urun Bilgisi

- **Proje Adi:** FitMeal AI - Smart Nutrition & Meal Planner
- **Asama:** Faz 1 - MVP (Minimum Viable Product)
- **Platform:** Web Application (Responsive)
- **Durum:** Onaylandi
- **Dokuman Sahibi:** Technical Product Owner

FitMeal AI, kullanicilarin beslenme hedeflerine gore (kilo verme, kas kazanimi, saglikli yasam) gunluk ve haftalik yemek plani olusturmasini saglayan AI destekli bir sistemdir.

---

## 2) Is Hedefleri ve Basari Metrikleri

- **Kuzey Yildizi Metrigi:** Haftalik olusturulan yemek plani sayisi
- **Kullanici Metrigi:** Gunluk aktif kullanici (DAU)
- **MVP Basari Hedefi (Faz 1):**
  - Ilk 30 gunde en az 100 kayitli kullanici
  - DAU/MAU >= %20
  - Plan olusturma basari orani >= %95

---

## 3) Kapsam (Faz 1)

### Dahil

1. Kullanici girisi (email tabanli basit oturum)
2. Hedef secimi (lose_weight, gain_muscle, healthy_living)
3. Gunluk/haftalik yemek plani olusturma
4. Gunluk kalori hesaplama
5. AI tabanli yemek onerisi
6. Alisveris listesi olusturma (plan bazli)

### Kapsam Disi

- Premium uyelik
- Fitness tracking entegrasyonu
- Wearable cihaz baglantisi

---

## 4) Teknik Yigin ve Mimari

- **Frontend:** React
- **Backend:** Flask (Python)
- **Veritabani:** PostgreSQL
- **AI/ML:** Scikit-learn
- **Optimizasyon:** PuLP
- **Hosting:** Vercel (Frontend) + Render (Backend)

### Yüksek Seviye Akis

1. Kullanici hedef ve profil bilgilerini girer
2. Backend kalori ihtiyacini hesaplar
3. Oneri/optimizasyon modulu uygun yemek kombinasyonunu olusturur
4. Plan DB'ye kaydedilir ve UI'da sunulur

---

## 5) Veri Modeli (PostgreSQL)

> Tum tablolar icin `created_at`, `updated_at` alanlari zorunludur.

### `users`

- `id` UUID PK
- `email` VARCHAR(255) UNIQUE NOT NULL
- `goal` VARCHAR(32) NOT NULL CHECK (goal IN ('lose_weight', 'gain_muscle', 'healthy_living'))
- `age` INT NULL
- `height_cm` FLOAT NULL
- `weight_kg` FLOAT NULL
- `gender` VARCHAR(16) NULL CHECK (gender IN ('male', 'female', 'other'))

### `foods`

- `id` UUID PK
- `name` VARCHAR(255) UNIQUE NOT NULL
- `calories` FLOAT NOT NULL CHECK (calories >= 0)
- `protein` FLOAT NOT NULL CHECK (protein >= 0)
- `carbs` FLOAT NOT NULL CHECK (carbs >= 0)
- `fat` FLOAT NOT NULL CHECK (fat >= 0)

### `meal_plans`

- `id` UUID PK
- `user_id` UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE
- `date` DATE NOT NULL
- `total_calories` FLOAT NOT NULL DEFAULT 0
- `target_calories` FLOAT NOT NULL DEFAULT 0
- UNIQUE (`user_id`, `date`)

### `meal_items`

- `id` UUID PK
- `meal_plan_id` UUID NOT NULL REFERENCES meal_plans(id) ON DELETE CASCADE
- `food_id` UUID NOT NULL REFERENCES foods(id)
- `meal_type` VARCHAR(16) NOT NULL CHECK (meal_type IN ('breakfast', 'lunch', 'dinner', 'snack'))
- `portion` FLOAT NOT NULL DEFAULT 1 CHECK (portion > 0)

### `recommendation_logs` (MVP icin opsiyonel ama onerilir)

- `id` UUID PK
- `user_id` UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE
- `input_context` JSONB NOT NULL
- `recommended_food_ids` JSONB NOT NULL
- `created_at` TIMESTAMP NOT NULL DEFAULT NOW()

---

## 6) API Uc Noktalari (REST/JSON)

Base path: `/api`

### 6.1 `POST /generate-plan`

Kullanicinin hedefi ve profil verilerine gore plan olusturur.

**Request**

```json
{
  "user_id": "uuid",
  "date": "2026-04-18"
}
```

**Response 200**

```json
{
  "meal_plan_id": "uuid",
  "target_calories": 2200,
  "total_calories": 2175,
  "items": [
    { "meal_type": "breakfast", "food_id": "uuid", "food_name": "Oatmeal", "portion": 1.0, "calories": 350 }
  ]
}
```

### 6.2 `GET /meal-plan`

Belirli bir gunun planini getirir.

**Query params:** `user_id`, `date`

**Response 200**

```json
{
  "meal_plan_id": "uuid",
  "date": "2026-04-18",
  "target_calories": 2200,
  "total_calories": 2175,
  "items": []
}
```

### 6.3 `POST /calculate-calories`

BMR ve hedefe gore gunluk kalori ihtiyacini hesaplar.

**Request**

```json
{
  "weight_kg": 78,
  "height_cm": 178,
  "age": 29,
  "gender": "male",
  "goal": "lose_weight"
}
```

**Response 200**

```json
{
  "bmr": 1732.5,
  "daily_calorie_target": 2000
}
```

### 6.4 `POST /recommend-foods`

Kullanicinin hedefi ve gecmisine gore yemek onerir.

**Request**

```json
{
  "user_id": "uuid",
  "limit": 10
}
```

**Response 200**

```json
{
  "recommendations": [
    { "food_id": "uuid", "name": "Grilled Chicken Salad", "score": 0.91 }
  ]
}
```

### Hata Kodlari

- `400`: Gecersiz istek
- `404`: Kayit bulunamadi
- `409`: Cakisan plan (user_id + date)
- `500`: Sunucu hatasi

---

## 7) Cekirdek Is Kurali: Kalori Hesabi

Mifflin-St Jeor yaklasimi baz alinmistir:

- Erkek: `BMR = 10w + 6.25h - 5a + 5`
- Kadin: `BMR = 10w + 6.25h - 5a - 161`
- Diger: `BMR = 10w + 6.25h - 5a`

Burada:
- `w = weight_kg`
- `h = height_cm`
- `a = age`

Hedef bazli ayarlama:
- `lose_weight`: `target = BMR - 300`
- `gain_muscle`: `target = BMR + 250`
- `healthy_living`: `target = BMR`

`target` alt limiti guvenlik icin 1200 kcal (MVP kural seti) olarak uygulanir.

---

## 8) Ekranlar ve Kullanici Akisi

1. **Login Screen**
   - Email ile giris
2. **Goal Selection Screen**
   - Hedef secimi ve temel profil bilgileri
3. **Dashboard**
   - Gunluk hedef kalori, kalan kalori, ozet plan
4. **Meal Plan Screen**
   - Ogun bazli liste ve toplam makrolar
5. **Shopping List Screen**
   - Haftalik planin birlesik malzeme listesi

---

## 9) User Stories ve Acceptance Criteria

### EPIC 1: Meal Planning

**US 1.1 - Generate Meal Plan**  
Bir kullanici olarak gunluk yemek plani olusturmak istiyorum ki hedefime uygun beslenebileyim.

- **Given:** Kullanici hedefini ve profil bilgilerini girmistir
- **When:** Plan olusturma islemi tetiklenir
- **Then:** Sistem gunluk ogun listesi ve toplam kaloriyi dondurur

### EPIC 2: Calorie Tracking

**US 2.1 - Track Calories**  
Bir kullanici olarak tuketimime gore kalori durumumu gormek istiyorum.

- **Given:** Kullanici plan veya yemek secmistir
- **When:** Kalori hesaplama islemi yapilir
- **Then:** Sistem BMR ve gunluk hedef kaloriyi gosterir

### EPIC 3: AI Recommendation

**US 3.1 - Food Recommendation**  
Bir kullanici olarak bana uygun yemek onerileri almak istiyorum.

- **Given:** Kullanici hedefi ve temel gecmisi vardir
- **When:** Oneri servisi cagrilir
- **Then:** Sistem skorlu uygun yemek listesi dondurur

---

## 10) Non-Functional Requirements (NFR)

- API p95 yanit suresi: `< 500 ms` (hesaplama endpointleri haric)
- Uygulama erisilebilirlik seviyesi: WCAG AA temel kontroller
- Guvenlik:
  - Giris dogrulamasi (input validation)
  - Email tekilligi
  - SQL injection korumasi (ORM + parametrik sorgu)
- Loglama: request_id ile izlenebilir API logu
- Hata izleme: merkezi hata kaydi (Sentry benzeri)

---

## 11) Test Stratejisi (MVP)

- **Backend Unit Tests**
  - Kalori hesap fonksiyonu
  - Hedef bazli kalori ayari
  - Oneri skorlamasi temel kontrol
- **API Integration Tests**
  - `POST /generate-plan` mutlu yol + validation hatalari
  - `GET /meal-plan` plan var/yok senaryolari
- **Frontend Smoke Tests**
  - Login -> Goal -> Dashboard temel akis

Minimum hedef: kritik akislarda `%80` test coverage (backend service katmani).

---

## 12) Release Plan (Faz 1)

- **Sprint 1:** DB + Auth + Kalori hesaplama
- **Sprint 2:** Plan olusturma + yemek listesi UI
- **Sprint 3:** AI onerileri + alisveris listesi + stabilizasyon

Canliya cikis kriterleri:
- Kritik bug sayisi = 0
- Temel API testleri yesil
- p95 performans hedefi saglanmis

---

## 13) Gelecek Gelistirmeler

- Mobil uygulama (iOS/Android)
- Kisisellestirilmis AI ogrenme dongusu
- Diyetisyen paneli ve manuel plan override
- Fitness/veri entegrasyonlari (HealthKit, Google Fit, wearable)
