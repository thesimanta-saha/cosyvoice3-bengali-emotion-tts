# EDA Summary — Full Dataset Analysis

## Pipeline Overview

| Stage | Files | Notes |
|-------|-------|-------|
| Raw segments | 4,253 | From 14 YouTube videos |
| After dedup/error removal | 4,252 | 1 corrupted/list JSON removed |
| Pure Bengali (no digits/English) | ~3,488 | Passed text purity filter |
| After quality + duration filter | **2,619** | Final gold training set |
| **Total rejection rate** | **38.4%** | 1,634 files removed |

---

## Raw Dataset (4,252 files)

### Gender Distribution
| Gender | Count | % |
|--------|-------|---|
| Male | 2,863 | 67.3% |
| Female | 1,301 | 30.6% |
| Other/Unknown | 88 | 2.1% |

### Audio Quality
| Level | Score Range | Count | % |
|-------|-------------|-------|---|
| High | 80–100 | 4,234 | 99.6% |
| Medium | 60–79 | 13 | 0.3% |
| Low | 0–59 | 5 | 0.1% |

### Multi-Speaker Detection
| Status | Count | % |
|--------|-------|---|
| Single speaker | 3,481 | 81.9% |
| Multi-speaker detected | 771 | 18.1% |

### Text Purity
| Category | Count | % |
|----------|-------|---|
| Pure Bengali | 2,383 | 56.0% |
| Contains digits (Bengali or ASCII) | 999 | 23.5% |
| Contains English characters | 870 | 20.5% |

### Transcription Agreement (Gemini vs Tugstugi)
| Match Type | Count | % |
|------------|-------|---|
| Perfect alignment | 2,383 | 56.0% |
| Minor mismatch (1–3 words) | ~1,200 | ~28.2% |
| Major mismatch (4+ words) | ~669 | ~15.7% |

### Top 10 Emotions (All Files)
| Rank | Emotion | Count |
|------|---------|-------|
| 1 | calm | 1,134 |
| 2 | serious | 968 |
| 3 | objective | 806 |
| 4 | confident | 327 |
| 5 | curious | 235 |
| 6 | introspective | 190 |
| 7 | insightful | 156 |
| 8 | wise | 128 |
| 9 | determined | 110 |
| 10 | hopeful | 95 |

### Top 10 Personas (All Files)
| Rank | Persona | Count |
|------|---------|-------|
| 1 | content_creator | 929 |
| 2 | intellectual | 701 |
| 3 | host | 454 |
| 4 | journalist | 181 |
| 5 | businessman | 89 |
| 6 | teacher | 78 |
| 7 | entrepreneur | 72 |
| 8 | activist | 55 |
| 9 | philosopher | 50 |
| 10 | doctor | 45 |

---

## Interference Analysis (All Files)

### Overlapping Speech
| Level | Score Range | Count | % |
|-------|-------------|-------|---|
| Clean | 0 | 2,891 | 68.0% |
| Low | 1–20 | 590 | 13.9% |
| High | 21–100 | 771 | 18.1% |

### Interruption Score
| Level | Score Range | Count | % |
|-------|-------------|-------|---|
| Clean | 0 | 3,010 | 70.8% |
| Low | 1–20 | 475 | 11.2% |
| High | 21–100 | 767 | 18.0% |

### Noise Confidence
| Level | Score Range | Count | % |
|-------|-------------|-------|---|
| Clean | 0 | 812 | 19.1% |
| Low | 1–20 | 2,988 | 70.3% |
| High | 21–100 | 452 | 10.6% |

---

## Final Gold Dataset (2,619 files)

### Gender Distribution
| Gender | Count | % |
|--------|-------|---|
| Male | 1,771 | 67.6% |
| Female | 848 | 32.4% |

### Quality (All Clean After Filter)
| Metric | Value |
|--------|-------|
| Multi-speaker | 0 files |
| Overlapping confidence > 20 | 0 files |
| Interruption score > 20 | 0 files |
| Audio quality high | 2,609 (99.6%) |
| Total duration | 4.12 hours |
| Average duration | ~5.7 seconds |
| Duration range | 3.0 – 15.0 seconds |
| Avg instruction length | 24.5 words |

---

## Key Observations

1. **Podcast audio is surprisingly high quality** — 99.6% of all raw segments scored ≥80 on the audio quality scale, suggesting YouTube podcasters use reasonably good microphones.

2. **Multi-speaker contamination is the biggest rejection cause** — 18.1% of segments had some secondary voice presence, far more than expected. Natural conversation always leaks.

3. **Pure Bengali is only 56% of podcast speech** — Modern Bengali content creators naturally mix English words (for technical terms) and digits (for statistics, years). This is a fundamental characteristic of contemporary spoken Bengali.

4. **Calm, serious, and objective dominate** — These top 3 emotions account for 55% of all data. Podcast conversations are inherently analytical and measured, not emotionally extreme. This creates a training data imbalance for high-energy emotions like `angry` or `energetic`.

5. **content_creator (929) vastly outnumbers all other personas** — Reflects the YouTube source: hosts/podcasters dominate. `intellectual` (701) and `host` (454) reinforce this. Rare personas like `doctor`, `lawyer`, or `scientist` have very few samples.