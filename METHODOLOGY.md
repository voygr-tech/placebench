# PlaceBench methodology

How the questions, answer keys, grades and scores in this repository were made. The short version is in the [README](README.md). A paper is in preparation.

## Questions

`data/questions.jsonl` holds the 150 scored questions. Each record has:
- `id`, `kind`, `city`, `country`, `language`;
- `stated_time`: the day and time the question is set at;
- `question`: as it was asked;
- `question_en`: an English translation, for the 46 questions asked in another language;
- `follow_up`: for 4 questions, a scripted second turn;
- `asked_three_times`: for the 20 questions used to measure consistency.

| Kind | Questions | Typical request |
|---|---:|---|
| Finding a place nearby | 54 | The nearest pharmacy, ATM or supermarket open at the stated time |
| Place details | 41 | Opening hours, entry rules, what a place offers |
| Planning a route or a time | 33 | The last train, when to leave, whether a plan fits |
| Booking and contact details | 11 | A phone number, an email address, how to buy tickets |
| Sharing with friends | 11 | A message or a link for a group |

The booking and contact questions ask for information only. No assistant was asked to call, book or buy anything.

Every question names the place, so the answer doesn't depend on the device's location. 115 questions also name a day and a time between 15 and 21 November 2026, so the answer doesn't depend on the device's clock either. The time is chosen so that it matters: close to a closing time, on a day the place is shut, after the last train. The other 35 hold on any day.

In a first version, all five personal assistants answered 68 questions correctly. We replaced 30 of those with questions of the same kind that need a fact and a step of reasoning, so the set is harder than a random sample of everyday questions. Ask Maps wasn't part of that filter.

## Answer keys and ground truth

`data/keys.jsonl` holds one key per question:
- `answer`;
- `counts_as_right` and `counts_as_wrong`, the grading rules;
- `sources`;
- `follow_up_answer`, where the question has a follow-up.

Facts come from sources in this order:
1. The place's own website, booking pages and notices.
2. The operator's timetable for trains, metros and buses.
3. The place's Google Maps listing, for hours and status where the place has no page of its own.
4. Other public sources: OpenStreetMap, official registers and licence notices, and dated press.

Where sources disagree, the place's own page wins.

27 keys cite a Google Maps listing or Google transit directions, usually next to the place's own page or the operator's timetable. One key, for pharmacy hours in Hpa-An, rests on Google listings alone. Points with and without those 27 questions:

| | Dots | Muse | Claude Duet | Instinct | Grok Bot | Ask Maps |
|---|---:|---:|---:|---:|---:|---:|
| All 150 questions | 92 | 83 | 79 | 76 | 73 | 49 |
| Without the 27 | 94 | 85 | 78 | 86 | 80 | 54 |
| Only the 27 | 86 | 74 | 82 | 29 | 41 | 24 |

Dots is first and Ask Maps last either way. The middle four change order, but they can't be separated in either case.

Walking and driving distances and times come from OpenStreetMap routing (OSRM). Public transport comes from the operator's timetable, and from Google Maps transit directions for a door-to-door time. Keys allow a margin around a time or distance, usually about 20% or a few minutes, stated in the key.

Keys were drafted by research agents with web access and checked against their sources. Every key was written twice, independently, and disagreements were settled against the sources. The keys hold for the week the questions name; hours and timetables move after that.

## Closed and moved places

A place an assistant recommends counts as closed, moved or renamed only when a public source shows it: a closure or move notice, the operator's own list, a licence suspension, a dated press report, or Google's "permanently closed". A listing that still shows the place open doesn't outweigh such evidence.

A closed place lowers the answer's grade: usually to wrong when it was the main recommendation, and to partly right or a small error when it was a backup or one of several options. This happened 12 times across the six assistants.

## Asking the assistants

Each assistant was asked every question through its own consumer app between 11 and 14 November 2026, before the week the questions ask about. The score uses the first answer to each question.

The 20 questions marked `asked_three_times` were asked twice more, to measure whether an answer holds. They were fixed before any answers were collected. Instinct keeps one continuous conversation and Dots keeps memory between chats, so their repeats can draw on earlier answers.

## Grading

Claude Opus 5.5 grades each answer against its key. Where an official page shows the key to be incomplete, it grades on the facts and cites the page. We hand-verified places to calibrate the judge. As a cross-check, OpenAI's gpt-6.1-sol regraded 60 first answers, 10 per assistant, without seeing the original grades or browsing. It made the same right-or-not call on 48. In 11 of the 12 disagreements it was the stricter grader.

| Grade | Points | Meaning |
|---|---:|---|
| Right | 1 | Right on everything asked; nothing added would mislead |
| Small error | 0.75 | Right on what was asked, with a slip that wouldn't change what the user does |
| Partly right | 0.25 | Part of the answer holds, or it rests on a claim nobody can check |
| Couldn't confirm | 0 | No firm answer, but says so and points to the right source |
| No answer | 0 | Nothing usable |
| Wrong | −1 | A confident answer that a source contradicts |

A grade of wrong needs a fact that a source contradicts. A claim nobody can check either way isn't wrong. An assistant isn't marked down for leaving out something the user didn't ask for.

## Scores

**Points** is the mean of the first answers' points, times 100. It can go below zero: a confident wrong answer costs more than no answer.

**Fully right** is the share of first answers graded right.

**Right 3 of 3** counts the repeated questions an assistant got right on all three tries. We also report pass^k, the chance that k tries at a question are all right, estimated from the three tries:

| | Dots | Muse | Claude Duet | Instinct | Grok Bot | Ask Maps |
|---|---:|---:|---:|---:|---:|---:|
| pass^1 | 95% | 78% | 75% | 58% | 62% | 73% |
| pass^2 | 90% | 68% | 68% | 35% | 48% | 65% |
| pass^3 | 85% | 60% | 65% | 25% | 40% | 60% |

pass^1 averages over all three tries, so it can differ from the first try alone.

**Time** runs from sending the question to the end of the final message, over the 146 questions without a follow-up. We report the median and the 95th percentile (nearest rank). Grok Bot sometimes replies within seconds that it's looking and sends the answer minutes later; we time the answer. Its times come from 53 answers; the other 93 have no recorded time.

**Intervals** are 95% bootstrap intervals over questions, from 4,000 resamples with a fixed seed. A pair of assistants is separated when the interval of their difference leaves out zero. We don't adjust for multiple comparisons.

## Recomputing the scores

`data/grades.jsonl` holds every grade. This prints each assistant's points, fully right, right 3 of 3 and pass^k, matching the tables above:

```python
import json
from collections import defaultdict
from math import comb

POINTS = {"right": 1, "small_error": 0.75, "partly_right": 0.25,
          "couldnt_confirm": 0, "no_answer": 0, "wrong": -1}
grades = [json.loads(line) for line in open("data/grades.jsonl")]

for name in ["Dots", "Muse", "Claude Duet", "Instinct", "Grok Bot", "Ask Maps"]:
    first = [g for g in grades if g["assistant"] == name and g["ask"] == 1]
    score = 100 * sum(POINTS[g["grade"]] for g in first) / len(first)
    tries = defaultdict(list)
    for g in grades:
        if g["assistant"] == name:
            tries[g["id"]].append(g["grade"] == "right")
    repeated = [t for t in tries.values() if len(t) == 3]
    pass_k = [100 * sum(comb(sum(t), k) / comb(3, k) for t in repeated) / len(repeated) for k in (1, 2, 3)]
    print(f"{name:12} points {score:.0f}  fully right {100 * sum(g['grade'] == 'right' for g in first) / len(first):.0f}%"
          f"  right 3/3 {sum(all(t) for t in repeated)}  pass^1-3 " + "/".join(f"{p:.0f}" for p in pass_k))
```

## Limitations

- This is one run, in November 2026. Consumer assistants change often.
- An LLM grades the answers. We calibrated it on hand-verified places and cross-checked it with a model from another vendor, but its grades aren't human grades. The judge is a Claude model, and one of the assistants, Claude Duet, is built on Claude.
- The 20 repeated questions are a small sample.
- Grok Bot's times rest on a subset of answers that isn't random.
- Replacing questions that all five personal assistants answered correctly lowers every absolute score. Ask Maps wasn't part of that filter.
- We wrote the keys before 15 November from what the sources published for that week, and didn't re-check them during the week.
- 27 keys cite Google Maps, and one rests on it alone (see Answer keys).

## Citation

The paper is in preparation for arXiv. Until it's out, cite the repository:

```bibtex
@misc{placebench2026,
  title  = {PlaceBench: Evaluating AI Assistants on Everyday, Time-Bound Questions about Real Places},
  author = {{VOYGR}},
  year   = {2026},
  month  = nov,
  url    = {https://github.com/voygr-tech/placebench}
}
```
