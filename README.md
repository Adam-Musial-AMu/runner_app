# 🏃 Half-Marathon Time Predictor

Aplikacja do szacowania czasu ukończenia półmaratonu na podstawie danych dostępnych
**przed startem biegu** (pre-race).

Projekt obejmuje pełny pipeline:
- czyszczenie i przygotowanie danych,
- trenowanie i walidację modeli,
- wersjonowanie artefaktów,
- aplikację inferencyjną w Streamlit,
- ekstrakcję danych wejściowych z tekstu użytkownika przy użyciu LLM,
- monitoring jakości ekstrakcji (Langfuse).

---

## 🎯 Cel projektu

Celem projektu jest realistyczna estymacja czasu ukończenia półmaratonu
w oparciu o minimalny zestaw informacji, który zawodnik może podać przed startem biegu.

Projekt **świadomie unika data leakage** – wykorzystywane są wyłącznie cechy,
które są znane przed rozpoczęciem zawodów.

---

## 🧠 Wytrenowane modele

W ramach projektu wytrenowano dwa komplementarne modele predykcyjne.

### PRE_RACE_5K
Model bazowy, wykorzystywany gdy dostępne są tylko podstawowe dane:
- płeć,
- wiek,
- czas uzyskany na dystansie 5 km.

Model ten:
- działa przy minimalnych wymaganiach wejściowych,
- zapewnia stabilną predykcję,
- osiąga średni błąd bezwzględny (MAE) ok. **5 minut** na danych testowych z roku 2024.

---

### PRE_RACE_10K
Model rozszerzony, używany gdy użytkownik poda dodatkowo czas na dystansie 10 km:
- płeć,
- wiek,
- czas na 5 km,
- czas na 10 km.

Dodatkowa informacja o dłuższym dystansie pozwala:
- lepiej odwzorować tempo zawodnika,
- zmniejszyć błąd predykcji względem wariantu 5 km.

Aplikacja automatycznie wybiera ten model, jeśli dane wejściowe są dostępne.

---

## 📊 Walidacja i interpretowalność

- Modele walidowane są **czasowo**:
  - trening na danych z 2023 roku,
  - test na danych z 2024 roku.
- Zapewnia to realistyczną ocenę generalizacji w przyszłych edycjach biegu.
- Analiza istotności cech potwierdza, że:
  - dominującą rolę odgrywają czasy na 5 km i 10 km,
  - wiek pełni rolę korekcyjną,
  - płeć i rok mają marginalny wpływ.

Takie zachowanie modeli jest zgodne z wiedzą dziedzinową.

---

## 📦 Artefakty modelu

Każdy model posiada komplet artefaktów:

- **model `.pkl`** – wytrenowany model predykcyjny,
- **`schema.json`** – kontrakt danych wejściowych (typy, zakresy, wymagane pola),
- **`metadata.json`** – metryki jakości, zakresy danych treningowych, kontekst,
- **`latest.json`** – wskazanie aktualnej wersji modelu używanej przez aplikację.

Dzięki temu:
- modele mogą być aktualizowane bez zmiany kodu aplikacji,
- możliwy jest łatwy rollback lub A/B testing.

---

## 🧩 Aplikacja Streamlit

Aplikacja Streamlit:
- przyjmuje **jedno pole tekstowe** jako wejście,
- wykorzystuje model językowy (OpenAI) do ekstrakcji danych do postaci JSON,
- waliduje dane zgodnie z `schema.json`,
- informuje użytkownika o brakujących danych,
- automatycznie dobiera właściwy model (5K / 10K),
- prezentuje wynik wraz z informacją o średnim błędzie modelu.

W przypadku braku dostępu do LLM stosowany jest fallback oparty o wyrażenia regularne.

---

## 🔍 Monitoring LLM (Langfuse)

Ekstrakcja danych wejściowych przez LLM jest logowana do Langfuse, co umożliwia:
- analizę poprawności ekstrakcji,
- monitoring błędów,
- iteracyjne doskonalenie promptów.

---

## 🛠️ Stack technologiczny

- **Python 3.10**
- **PyCaret 3.3.2**
- **scikit-learn**
- **Streamlit**
- **OpenAI SDK**
- **Langfuse**
- **pandas / numpy / matplotlib**

---

## 🚀 Uruchomienie lokalne

pip install -r requirements.txt
streamlit run app.py

## App flow

┌──────────────┐
│  Streamlit   │
│  start app   │
└──────┬───────┘
       │
       ▼
┌────────────────────┐
│ User wpisuje tekst │
└──────┬─────────────┘
       │ klik
       ▼
┌──────────────────────────┐
│ Ekstrakcja danych        │
│ - LLM (@observe)         │
│ - regex fallback         │
└──────┬───────────────────┘
       │
       ▼
┌──────────────────────────┐
│ Anti-hallucination guard │
│ - brak 5km → None        │
│ - brak 10km → None       │
└──────┬───────────────────┘
       │
       ▼
┌──────────────────────────┐
│ Wybór modelu             │
│ AUTO / 5K / 10K          │
└──────┬───────────────────┘
       │
       ▼
┌──────────────────────────┐
│ Build DataFrame (1 row)  │
└──────┬───────────────────┘
       │
       ▼
┌──────────────────────────┐
│ Pandera VALIDATION       │
│ (schema.json)            │
└───┬───────────────┬──────┘
    │ OK            │ ERROR
    ▼               ▼
┌────────────┐   ┌──────────────┐
│ Predict    │   │ UI error     │
│ PyCaret    │   │ st.stop()    │
└────┬───────┘   └──────────────┘
     │
     ▼
┌──────────────────────────┐
│ Wynik + tempo + MAE      │
└──────────────────────────┘

