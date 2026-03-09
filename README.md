# [AI and Close Reading] Data Dictionary

## Overview

This repository contains the data processing pipeline for the experiment in the paper *["What Does AI Do for Cultural Interpretation? A Randomized Experiment on Close Reading Poems with Exposure to AI Interpretation"](https://doi.org/10.1145/3772318.3791727)*. It is intended for the use of the [study interface](https://github.com/xxxxbrandieeee/close-reading-interface) for understanding what data the interface collects and processing it into a structured dataset.

### Repository Structure

```
├── README.md                        ← This file (data dictionary)
└── processing/
    └── data_processing.ipynb        ← Notebook: raw JSON logs → xlsx
```

### Note

The raw JSON interaction logs contain identifiable participant IDs (Prolific worker IDs) and are not included here. The data dictionary below documents the raw JSON structure to explain what the interface collects and how the processed columns are derived.

---

## Raw Log Structure

The study interface produces one JSON object per participant. Each object has top-level keys corresponding to pages of the study interface, plus metadata fields. The three randomized poems are fixed across participants and map to page suffixes as follows:

- `_1` → `love_poem`
- `_2` → `dusting`
- `_3` → `english_b`

Pages 4–6 repeat for each poem (in randomized order), while all other pages appear once.

```
{
  "page1":    { ... },   // Online consent
  "page2":    { ... },   // User ID
  "page3":    { ... },   // Instruction check
  "page8":    { ... },   // Demographics
  "page4_1":  { ... },   // First reading: love_poem
  "page5_1":  { ... },   // Interpretation tasks: love_poem
  "page6_1":  { ... },   // Post-task ratings: love_poem
  "page4_2":  { ... },   // First reading: dusting
  ...                    // (same structure for _2 and _3)
  "page7":    { ... },   // Debrief and study feedback
  "isRefresh": true      // (optional) Browser refresh detected
}
```

The processing notebook (`processing/data_processing.ipynb`) parses these JSON objects and flattens them into the columns documented below.

---

## Data Dictionary

### Global Fields

#### Browser Refresh Detection

A top-level field `isRefresh` indicates whether the participant refreshed the browser during the study, which would restart the session. It is used as a data quality check — participants whose logs include this field were excluded from analysis.

| Column Name | Description | Type |
|---|---|---|
| `is_refresh` | Whether the participant refreshed the page during the study | Boolean |

#### Page Leave Tracking

Each page may include a `leave_page` list logging when the participant left and returned to the tab.

**Raw JSON example:**
```json
"page1": {
  "leave_page": [
    { "type": "leave",  "time": 1751823014352 },
    { "type": "return", "time": 1751823017720 },
    { "type": "leave",  "time": 1751823028383 },
    { "type": "return", "time": 1751823031244 }
  ],
  "value": "I agree to participate in the research.",
  "time": 1751823025673
}
```

**Processed columns:**

| Column Name | Description | Type |
|---|---|---|
| `leaved_pageN_times` | Number of times the participant left page N (N = 1, …, 12) | Integer |
| `leaved_pageN_duration` | Total duration (ms) away from page N (N = 1, …, 12) | Integer |

**Processing logic:**
- Each log is a list of alternating `leave` and `return` events with timestamps.
- Unpaired events are ignored for duration calculations.
- Durations are calculated as `return_time - leave_time`, summed per page.

---

### Page 1: Consent

Records whether the participant agreed to the study's consent form and when they submitted the page.

**Raw JSON example:**
```json
"page1": {
  "value": "I agree to participate in the research.",
  "time": 1751138126624
}
```

**Processed columns:**

| Column Name | Description | Type |
|---|---|---|
| `consent_given` | Whether the participant agreed to the consent form | Boolean |
| `page1_submission_time` | Timestamp (Unix ms) when the consent page was submitted | Integer |

**Processing logic:**
- `consent_given`: Set to `TRUE` if `value` exactly matches `"I agree to participate in the research."`; otherwise `FALSE`.

---

### Page 2: Participant ID

Captures the participant's identifier (worker ID) and submission timestamp.

**Raw JSON example:**
```json
"page2": {
  "value": "<worker_id>",
  "time": 1751138331919
}
```

**Processed columns:**

| Column Name | Description | Type |
|---|---|---|
| `participant_id` | Participant's unique ID | String |
| `page2_submission_time` | Timestamp (Unix ms) when the ID page was submitted | Integer |

---

### Page 3: Instruction Check

Confirms that participants acknowledged the study instructions.

**Raw JSON example:**
```json
"page3": {
  "3_1": "I understand what is close reading and I will read each instruction carefully.",
  "3_2": "I understand the procedure and estimated time of the study.",
  "3_3": "I understand that I will not go back or refresh the webpage at any point in this study.",
  "3_4": "I understand that I must only use the web interface provided and no external sources.",
  "time": 1753799851246
}
```

**Processed columns:**

| Column Name | Description | Type |
|---|---|---|
| `instruction_checks_passed` | Whether all instruction checkboxes were acknowledged | Boolean |
| `page3_submission_time` | Timestamp (Unix ms) when the instruction check was submitted | Integer |

**Processing logic:**
- `instruction_checks_passed`: `TRUE` if `page3_submission_time` exists and is non-empty; `FALSE` otherwise.

---

### Page 8: Demographics

Collects demographic information and baseline measures of close reading and LLM experience.

**Raw JSON example:**
```json
"page8": {
  "info": [
    { "title": "How familiar are you with LLMs...", "answer": "Very familiar, I have technical knowledge of what they are and how they work" },
    { "title": "How often do you use LLMs...",      "answer": "Sometimes, about 3–4 times a month" },
    { "title": "Overall, how do you feel about LLMs...", "answer": "Somewhat negative" },
    { "title": "What is your age?",                 "answer": "45–54" },
    { "title": "What is the highest degree of education you have completed?...", "answer": "Professional degree" },
    { "title": "What is your major or field of study?...", "answer": ["Information and Communication Technologies (ICTs)", "Health and welfare"] },
    { "title": "What is your gender identity?",     "answer": "Female" },
    { "title": "How would you describe your race and ethnicity?", "answer": "Black or African American" },
    { "title": "In your everyday life, how often do you read poems?", "answer": "Sometimes, about 3–4 times a month" },
    { "title": "In your everyday life, how much do you enjoy reading poetry?", "answer": "I enjoy it quite a bit" },
    { "title": "Have you ever taken any college-level humanities courses...?", "answer": "A lot (e.g., multiple courses or extensive coursework specifically focused on humanities)" },
    { "title": "How much coursework have you completed that is related to poetry?", "answer": "A moderate amount (e.g., a full unit or a single course focused on poetry)" }
  ],
  "time": 1751860260316
}
```

**Processed columns:**

| Column Name | Original Question | Type |
|---|---|---|
| `llm_familiarity` | How familiar are you with LLMs and LLM-infused applications such as ChatGPT, Copilot, and Claude? | String |
| `llm_use_frequency` | How often do you use LLMs and LLM-infused applications such as ChatGPT, Copilot, and Claude? | String |
| `llm_attitude` | Overall, how do you feel about LLMs and LLM-infused applications such as ChatGPT, Copilot, and Claude? | String |
| `age` | What is your age? | String |
| `education` | What is the highest degree of education you have completed? | String |
| `major` | What is your major or field of study? (Select all that apply.) | String |
| `gender` | What is your gender identity? | String |
| `race` | How would you describe your race and ethnicity? | String |
| `everyday_poem_reading` | In your everyday life, how often do you read poems? | String |
| `everyday_poem_enjoyment` | In your everyday life, how much do you enjoy reading poetry? | String |
| `humanities_course_experience` | Have you ever taken any college-level humanities courses (e.g., literature, philosophy, history)? | String |
| `poetry_course_experience` | How much coursework have you completed that is related to poetry? | String |
| `page8_submission_time` | Timestamp (Unix ms) when Page 8 was submitted | Integer |

#### Author–Participant Identity Match Variables

Based on `gender` and `race`, the following binary variables are constructed for each poem. Poem authorship: *Love Poem* (white female), *Dusting* (Black female), *Theme for English B* (Black male).

| Column Name | Description |
|---|---|
| `love_poem_gender_match` | 1 if participant's gender is female, else 0 |
| `love_poem_race_match` | 1 if participant's race is white, else 0 |
| `dusting_gender_match` | 1 if participant's gender is female, else 0 |
| `dusting_race_match` | 1 if participant's race is Black, else 0 |
| `english_b_gender_match` | 1 if participant's gender is male, else 0 |
| `english_b_race_match` | 1 if participant's race is Black, else 0 |

---

### Poem Order

The three poems are presented in randomized order across participants. Page suffixes `_1`, `_2`, `_3` indicate presentation order; each suffix always maps to the same poem.

| Column Name | Description | Type |
|---|---|---|
| `poem_order` | Semicolon-separated list of poem names in the order shown to the participant (e.g., `"love_poem;english_b;dusting"`) | String |

---

### Pages 4_X: First Reading

Captures participants' initial reading of each poem. Logs fine-grained cursor interactions including line-level hovering, panel hovering, and submission time.

**Raw JSON example:**
```json
"page4_2": {
  "poemItemHover": [
    { "index": 18, "time": 1751875805986, "text": "For this infernal, endless chore," },
    { "index": 16, "time": 1751875805994, "text": "make sweeping circles." }
  ],
  "poemScroll": {},
  "left_content_hover": [
    { "time": 1751875809849, "type": "enter" },
    { "time": 1751875810432, "type": "leave" }
  ],
  "right_content_hover": [
    { "time": 1751875805869, "type": "enter" },
    { "time": 1751875806069, "type": "leave" }
  ],
  "time": 1751875814070
}
```

**Processed columns (per poem):**

| Column Name | Description | Type |
|---|---|---|
| `[poem_name]_first_reading_line_hover` | Poem line indices hovered over, with repeat counts where applicable (e.g., `6, 10(2), 16`) | String |
| `[poem_name]_reading_instruction_hover_duration` | Total time (ms) hovering over the left instruction panel | Integer |
| `[poem_name]_first_reading_hover_duration` | Total time (ms) hovering over the right content panel (poem) | Integer |
| `page4_X_submission_time` | Timestamp (Unix ms) when the page was submitted | Integer |

**Processing logic:**
- `[poem_name]_first_reading_line_hover`: Group `poemItemHover` events by `index`, count occurrences. Format as comma-separated string: single hovers as `index`, repeated hovers as `index(n)`. Sort by ascending line index.
- Hover durations (`left_content_hover`, `right_content_hover`): Sum durations of paired `enter`/`leave` events; ignore unpaired events.

---

### Pages 5_X: Interpretation Tasks

The main task page. Participants write three stylistic examples and corresponding explanations in six textboxes. The system logs behavioral data including copy actions, input duration, hover patterns, AI tab viewing, and AI text selections.

#### Textbox Mapping

| Index | Label | Extracted Column Suffix |
|---|---|---|
| 0 | First example | `first_example` |
| 1 | Explanation (1st) | `first_explanation` |
| 2 | Second example | `second_example` |
| 3 | Explanation (2nd) | `second_explanation` |
| 4 | Third example | `third_example` |
| 5 | Explanation (3rd) | `third_explanation` |

**Raw JSON example:**
```json
"page5_3": {
  "info": [
    { "title": "First example:",
      "value": "The phrase that stood out to me was \"So will my page be colored that I write?\"",
      "placeholder": "The word/phrase that stuck out to me was ________________." },
    { "title": "Explanation:",
      "value": "The line stands out to me because it evokes the image of a blank white sheet of paper...",
      "placeholder": "The poet is using the term to convey the idea that ________________..." },
    { "title": "Second example:",
      "value": "The line \"I hear New York, too\" gives us insight into the poet's voice or perspective.",
      "placeholder": "The line/phrase ________________ gives us some insight into the perspective the poet is trying to create." },
    { "title": "Explanation:",
      "value": "This line immediately follows a mention of Harlem, which is a part of New York...",
      "placeholder": "This line/phrase expresses the idea that ________________..." },
    { "title": "Third example:",
      "value": "The poetic/stylistic element that stood out to me is the way he groups a lot of words together, specifically in groups of three.",
      "placeholder": "The poetic/stylistic element that stood out to me was ________________." },
    { "title": "Explanation:",
      "value": "This element adds a musicality to the work - like a beat almost...",
      "placeholder": "This element works by ________________..." }
  ],
  "copy_text": {},
  "answer_input": {
    "0": [
      { "start_time": 1754705885942, "title": "First example:" },
      { "end_time":   1754705908307, "title": "First example:" }
    ],
    "1": [
      { "start_time": 1754705908308, "title": "Explanation:" },
      { "end_time":   1754706013216, "title": "Explanation:" }
    ]
  },
  "ai_select_text": [],
  "poemItemHover": [
    { "index": 10, "time": 1754705786145, "text": "The steps from the hill lead down into Harlem," },
    { "index": 9,  "time": 1754705786156, "text": "I am the only colored student in my class." }
  ],
  "question_hover": [ { "time": 1754705773961, "type": "enter" }, { "time": 1754705786139, "type": "leave" } ],
  "poem_hover":     [ { "time": 1754705786145, "type": "enter" }, { "time": 1754705804118, "type": "leave" } ],
  "ai_hover":       [ { "time": 1754705772784, "type": "enter" }, { "time": 1754705772863, "type": "leave" } ],
  "answer_hover":   [ { "time": 1754705772680, "type": "enter" }, { "time": 1754705772776, "type": "leave" } ],
  "rest_area":      [ { "type": "enter", "time": 1754705772616 }, { "type": "leave", "time": 1754705772680 } ],
  "question_total": 14.727,
  "poem_total":     81.672,
  "ai_total":       52.793,
  "answer_total":   256.659,
  "rest_area_total_time": 78.044,
  "input_time_0":      [ { "type": "start", "time": 1754705885942 }, { "type": "end", "time": 1754705903157 } ],
  "stop_input_time_0": [ { "type": "start", "time": 1754705903157 }, { "type": "end", "time": 1754705908307 } ],
  "input_total_time_0": 17.215,
  "input_total_time_1": 93.526,
  "input_total_time_2": 26.113,
  "input_total_time_3": 65.011,
  "input_total_time_4": 34.71,
  "input_total_time_5": 77.01,
  "tap1": [
    { "type": "start", "time": 1754705772595 },
    { "type": "end",   "time": 1754705819031 },
    { "type": "start", "time": 1754705852129 },
    { "type": "end",   "time": 1754705878694 }
  ],
  "tap2": [
    { "type": "start", "time": 1754705822371 },
    { "type": "end",   "time": 1754705826355 },
    { "type": "start", "time": 1754705827475 },
    { "type": "end",   "time": 1754705847785 }
  ],
  "tap3": [
    { "type": "start", "time": 1754705819031 },
    { "type": "end",   "time": 1754705822371 },
    { "type": "start", "time": 1754705826355 },
    { "type": "end",   "time": 1754705827475 }
  ],
  "tab_total_time1": 280.418,
  "tab_total_time2": 193.477,
  "tab_total_time3": 15.149,
  "hide_total_time": 0,
  "time": 1754706261639
}
```

**Processed columns (per poem):**

##### Final Text Answers

| Column Name | Description | Type |
|---|---|---|
| `[poem_name]_[textbox_name]_answer` | Final submitted answer for each textbox (from `info[index].value`) | String |

##### Copy Behavior

| Column Name | Source | Description | Type |
|---|---|---|---|
| `[poem_name]_[textbox_name]_copied_text` | `copy_text[index]` | All strings copied into each textbox, joined with `\|` | String |

##### Input Behavior

| Column Name | Source | Description | Type |
|---|---|---|---|
| `[poem_name]_[textbox_name]_num_input` | `input_time_[index]` | Number of distinct input events (start–end pairs) per textbox | Integer |
| `[poem_name]_[textbox_name]_input_duration` | `input_total_time_[index]` | Total active input time (s), pre-computed | Float |
| `[poem_name]_[textbox_name]_num_pause` | `stop_input_time_[index]` | Number of distinct pause events (start–end pairs) per textbox | Integer |
| `[poem_name]_[textbox_name]_pause_duration` | `stop_input_time_[index]` | Total pause duration (ms), summed from start–end pairs | Integer |
| `[poem_name]_[textbox_name]_input_sequence` | `input_time_[index]` | Order in which textboxes were first typed into, separated by `\|` (based on first `start_time` per textbox) | String |

##### AI Tab Viewing (AI conditions only)

The AI panel displays one interpretation at a time; participants click tabs to switch between them. `tap1`, `tap2`, and `tap3` each log start–end events for when the participant was actively viewing that tab. `tab_total_time1/2/3` are the pre-computed totals. `hide_total_time` records the total time (s) the AI panel was hidden.

Note that in the AI-Single condition, only `tap1` and `tab_total_time1` are populated; `tap2`, `tap3`, and their totals will be zero or absent.

| Column Name | Source | Description | Type |
|---|---|---|---|
| `[poem_name]_ai_tab[index]_num_view` | `tap[index]` | Number of times the participant switched to AI answer tab `index` (1, 2, or 3) | Integer |
| `[poem_name]_ai_tab[index]_duration` | `tab_total_time[index]` | Total time (s) spent viewing AI answer tab `index`, pre-computed | Float |
| `[poem_name]_ai_hide_duration` | `hide_total_time` | Total time (s) the AI panel was hidden, pre-computed | Float |

##### AI Text Selection

| Column Name | Source | Description | Type |
|---|---|---|---|
| `[poem_name]_ai_select_text` | `ai_select_text` | Text selected/highlighted within the AI panel, joined with `\|` | String |

##### Poem Line Hover

| Column Name | Source | Description | Type |
|---|---|---|---|
| `[poem_name]_interpreting_line_hover` | `poemItemHover` | Poem line indices hovered over during the task, formatted as `index` or `index(n)` for multiple hovers | String |

##### Panel Hover Counts and Durations

Hover events are logged for five areas of the interface. Counts reflect the number of `enter` events; durations are summed from paired `enter`/`leave` events.

| Column Name | Source | Description | Type |
|---|---|---|---|
| `[poem_name]_question_num_entries` | `question_hover` | Count of entries into the task question panel | Integer |
| `[poem_name]_ai_num_entries` | `ai_hover` | Count of entries into the AI panel | Integer |
| `[poem_name]_poem_num_entries` | `poem_hover` | Count of entries onto the poem text | Integer |
| `[poem_name]_answer_area_num_entries` | `answer_hover` | Count of entries into the answer area | Integer |
| `[poem_name]_rest_area_num_entries` | `rest_area` | Count of entries into the remaining screen area | Integer |
| `[poem_name]_question_hover_total_duration` | `question_hover` | Total time (ms) hovering over the task question panel | Integer |
| `[poem_name]_ai_hover_total_duration` | `ai_hover` | Total time (ms) hovering over the AI panel | Integer |
| `[poem_name]_poem_hover_total_duration` | `poem_hover` | Total time (ms) hovering over the poem text | Integer |
| `[poem_name]_answer_area_hover_total_duration` | `answer_hover` | Total time (ms) hovering over the answer area | Integer |
| `[poem_name]_rest_area_hover_total_duration` | `rest_area` | Total time (ms) hovering over the remaining screen area | Integer |
| `[poem_name]_hover_total_duration` | — | Sum of all five hover duration columns above | Integer |

##### Pre-Computed Total Durations

These fields are recorded directly in the raw logs (in seconds).

| Column Name | Source | Description | Type |
|---|---|---|---|
| `[poem_name]_question_total_duration` | `question_total` | Total time (s) over task question panel | Float |
| `[poem_name]_ai_total_duration` | `ai_total` | Total time (s) over AI panel | Float |
| `[poem_name]_poem_total_duration` | `poem_total` | Total time (s) over poem panel | Float |
| `[poem_name]_answer_area_total_duration` | `answer_total` | Total time (s) over answer area | Float |
| `[poem_name]_rest_area_total_duration` | `rest_area_total_time` | Total time (s) over remaining screen area | Float |
| `[poem_name]_cursor_activity_total_duration` | — | Sum of all five pre-computed duration fields above | Float |

##### Additional

| Column Name | Description | Type |
|---|---|---|
| `page5_X_submission_time` | Timestamp (Unix ms) when the page was submitted | Integer |

---

### Pages 6_X: Post-Task Ratings

After completing the interpretation tasks for each poem, participants rate their experience and (in AI conditions) their use of and satisfaction with the AI.

**Raw JSON example:**
```json
"page6_3": {
  "info": [
    { "title": "After close reading, how difficult was this poem for you?",
      "answer": "Neither easy nor difficult" },
    { "title": "After close reading, to what degree did this poem resonate with you?...",
      "answer": "My response to the poem was neutral (I didn't feel anything)" },
    { "title": "Why did you feel this way about how the poem resonated with you after close reading?...",
      "answer": "It did not really resonate with me..." },
    { "title": "After close reading, how much did you enjoy reading the poem?",
      "answer": "Neither enjoyable nor unenjoyable" },
    { "title": "Why did you feel this way about your enjoyment of reading the poem after close reading?...",
      "answer": "It didn't ring a bell with me..." },
    { "title": "After close reading, how confident are you in your ability to interpret the poem?",
      "answer": "I feel slightly able" },
    { "title": "Why do you feel this way about your confidence in your ability to interpret the poem after close reading?...",
      "answer": "It was understandable and I thought I got its meaning." },
    { "title": "Did you use the AI for interpreting this poem?",
      "answer": "Yes" },
    { "title": "How helpful was the AI for you when interpreting the poem?",
      "answer": "Slightly helpful" }
  ],
  "time": 1753801517010
}
```

**Processed columns (per poem):**

| Column Name | Description | Scale | Type |
|---|---|---|---|
| `[poem_name]_close_reading_difficulty` | Perceived difficulty after close reading | 1–7 | Integer |
| `[poem_name]_close_reading_appreciation` | Degree to which the poem resonated after close reading | 1–7 | Integer |
| `[poem_name]_close_reading_appreciation_rationale` | Open-ended rationale for appreciation rating | — | String |
| `[poem_name]_close_reading_enjoyment` | Enjoyment of the poem after close reading | 1–7 | Integer |
| `[poem_name]_close_reading_enjoyment_rationale` | Open-ended rationale for enjoyment rating | — | String |
| `[poem_name]_close_reading_self_efficacy` | Confidence in ability to interpret the poem after close reading | 1–7 | Integer |
| `[poem_name]_close_reading_self_efficacy_rationale` | Open-ended rationale for self-efficacy rating | — | String |
| `[poem_name]_close_reading_ai_use` | Whether the participant used the AI for this poem (AI conditions only) | — | String |
| `[poem_name]_close_reading_ai_helpfulness` | Perceived helpfulness of the AI for this poem (AI conditions only) | — | String |
| `page6_X_submission_time` | Timestamp (Unix ms) when the page was submitted | — | Integer |

---

### Page 7: Debrief and Study Feedback

Collects any prior familiarity with the poems and post-study reflection on task approaches. This page appears after all three poem loops.

**Raw JSON example:**
```json
"page7": {
  "info": [
    { "title": "Before participating in this study, had you read any of these poems? Please select all that apply.",
      "answer": ["Love Poem", "Dusting"] },
    { "title": "In the study, you read and interpreted three poems: Love Poem, Dusting, Theme for English B. When interpreting each poem, what's your approach?",
      "answer": "I used the AI's answer to check my first impression..." },
    { "title": "How did you feel about your experience engaging in close reading during this study? Did anything feel confusing, challenging, or particularly interesting? Did you take away anything new or surprising about the skill of close reading — whether about how it works, how you approach it, or how it might apply to your everyday life?",
      "answer": "No issues." }
  ],
  "time": 1752616998096
}
```

**Processed columns:**

| Column Name | Description | Type |
|---|---|---|
| `poem_read_prior_participation` | Which poems, if any, the participant had read before the study | String |
| `task_approach` | Post-study reflection on task approaches | String |
| `study_feedback` | Feedback on engaging in close reading | String |
| `page7_submission_time` | Timestamp (Unix ms) when the debrief page was submitted (from `page7.time`) | Integer |

---

## Citation

```bibtex
@inproceedings{zhi2026what,
  author    = {Zhi, Jiayin and Long, Hoyt and So, Richard Jean and Lee, Mina},
  title     = {What Does AI Do for Cultural Interpretation? A Randomized Experiment on Close Reading Poems with Exposure to AI Interpretation},
  booktitle = {Proceedings of the 2026 CHI Conference on Human Factors in Computing Systems},
  series    = {CHI '26},
  year      = {2026},
  location  = {Barcelona, Spain},
  publisher = {ACM},
  address   = {New York, NY, USA},
  pages     = {1--18},
  doi       = {10.1145/3772318.3791727}
}
```

## Contact

Jiayin Zhi (jzhi@uchicago.edu)
