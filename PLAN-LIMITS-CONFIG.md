# Anthropic Plan Limits Configuration

## Overview

Das Usage Dashboard berechnet jetzt automatisch den Verbrauch gegen deine Anthropic Pro Plan Limits aus den Session-Transcripts.

## Wie es funktioniert

1. **Datenquelle**: Session-Transcripts (JSON) aus `~/.openclaw/agents/main/sessions/`
2. **Berechnung**: Token-Verbrauch pro Modellklasse für die letzten 5h und 7d
3. **Anzeige**: Fortschrittsbalken im Dashboard (grün/gelb/rot je nach Auslastung)

## Konfiguration

Bearbeite die folgenden Werte in `usage-dashboard-generic.py`:

### Plan Limits Budgets

```python
ANTHROPIC_PLAN_LIMITS = {
    "pro": {
        "5h": {
            "sonnet_4x": {"spend_budget": 50.0},      # ← ADJUST
            "opus_4x": {"spend_budget": 50.0},        # ← ADJUST
            "haiku_45": {"spend_budget": 50.0},       # ← ADJUST
            "total": {"spend_budget": 100.0},         # ← ADJUST
        },
        "7d": {
            "sonnet_4x": {"spend_budget": 200.0},     # ← ADJUST
            "opus_4x": {"spend_budget": 200.0},       # ← ADJUST
            "haiku_45": {"spend_budget": 200.0},      # ← ADJUST
            "total": {"spend_budget": 500.0},         # ← ADJUST
        }
    }
}
```

## Werte finden

1. Gehe zu https://console.anthropic.com/settings/limits
2. Suche nach "Plan Limits" oder "Usage Limits"
3. Notiere die Spend-Budgets für 5h und 7d Fenster
4. Trage die Werte in die Konfiguration oben ein

## Modellklassen

Das Dashboard gruppiert Modelle nach Klasse:

- **Sonnet 4.x**: claude-sonnet-4-6, claude-sonnet-4-5
- **Opus 4.x**: claude-opus-4-6
- **Haiku 4.5**: claude-haiku-4-5

## Reset-Zeiten

- **5h Window**: Rollendes Fenster (reset ab letztem API-Call + 5h)
- **7d Window**: Jeden Samstag 17:00 CET

## Farben

- 🟢 **Grün** (<70%): Alles OK
- 🟡 **Gelb** (70-89%): Warnung
- 🔴 **Rot** (≥90%): Nah am Limit / 100% = Limit erreicht

## Environment Variable

Zum Umschalten zwischen verschiedenen Plan-Tiers:

```bash
export ANTHROPIC_PLAN="pro"  # Standard
export ANTHROPIC_PLAN="enterprise"  # (falls definiert)
```

## Ausgabe

Das Dashboard generiert zwei Seiten automatisch:

1. **Berechnete Limits** (aus Transcripts): Immer verfügbar
2. **Live API Limits** (via API-Key): Nur mit `ANTHROPIC_API_KEY` vorhanden

Falls kein API-Key konfiguriert ist, zeigt das Dashboard nur die berechneten Limits.
