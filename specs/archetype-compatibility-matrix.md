# Archetype Compatibility Matrix

Maps personality framework values to Hartwell & Chen archetypes (12 families, 48 sub-archetypes).
Powers a set-intersection algorithm: specify framework values → intersect compatible sets → converge on archetype.

## Overview

The Hartwell & Chen archetype model (from *Archetypes in Branding*, 2012) defines 60 sub-archetypes
across 12 families. Each family has 5 sub-archetypes: 4 specialized variants plus the generic
family archetype itself (e.g., "Hero" as both the family name and the 5th sub-archetype within
the Hero family). Eidos uses the **48 specialized sub-archetypes only** (4 per family). The 12
generic family-name archetypes are omitted because they add no signal beyond what `ArchetypeFamily`
already provides — "Hero" as a sub-archetype tells the LLM nothing that `family: HERO` doesn't.
The specialized variants (Athlete, Liberator, Rescuer, Warrior) are where personality
differentiation lives.

The 12 families are grouped by 4 motivation quadrants:

| Quadrant | Drive | Families |
|---|---|---|
| **Independence** | Self-actualization | Innocent, Explorer, Sage |
| **Mastery** | Agency, power | Hero, Rebel, Magician |
| **Belonging** | Connection | Everyman, Lover, Jester |
| **Stability** | Structure | Caregiver, Creator, Sovereign |

---

## Section 1: Family-Level Compatibility

Each table maps a framework's values to compatible archetype families. The intersection
of all specified framework values yields the target archetype family.

### MBTI → Archetype Families

Based on PMAI × MBTI research (n=1,000+) and cognitive function stack analysis.

| MBTI | Compatible Families | Reasoning |
|---|---|---|
| INTJ | Sage, Magician, Sovereign | Ni-Te: strategic vision + systematic execution; knowledge-seeking with drive to implement |
| INTP | Sage, Explorer, Creator | Ti-Ne: deep analytical frameworks + possibility exploration; theoretical model-builders |
| ENTJ | Sovereign, Hero, Magician | Te-Ni: command + vision; natural organizers who drive transformation |
| ENTP | Magician, Rebel, Explorer | Ne-Ti: inventive challengers; see possibilities and dismantle conventions |
| INFJ | Magician, Sage, Caregiver | Ni-Fe: deep pattern insight + empathic drive; visionary guides |
| INFP | Creator, Innocent, Explorer | Fi-Ne: value-driven meaning-makers; authentic self-expression and idealism |
| ENFJ | Caregiver, Sovereign, Magician | Fe-Ni: empathic leadership; inspire and develop others toward a vision |
| ENFP | Explorer, Creator, Jester | Ne-Fi: enthusiastic possibility explorers; creative, spontaneous, values-driven |
| ISTJ | Sovereign, Everyman, Sage | Si-Te: methodical, reliable, evidence-based; guardians of established order |
| ISFJ | Caregiver, Everyman, Innocent | Si-Fe: dutiful protectors; quietly sustain community through service |
| ESTJ | Sovereign, Hero, Everyman | Te-Si: decisive administrators; enforce standards and get results |
| ESFJ | Caregiver, Everyman, Lover | Fe-Si: community builders; nurture harmony and uphold traditions |
| ISTP | Explorer, Hero, Rebel | Ti-Se: independent problem-solvers; hands-on mastery, cool under pressure |
| ISFP | Creator, Lover, Innocent | Fi-Se: aesthetic, present-moment awareness; quiet authenticity and sensory expression |
| ESTP | Hero, Jester, Rebel | Se-Ti: action-oriented risk-takers; thrive on challenge and improvisation |
| ESFP | Jester, Lover, Innocent | Se-Fi: spontaneous performers; bring energy, warmth, and enjoyment |

### Enneagram → Archetype Families

Based on practitioner consensus and motivational alignment.

| Enneagram | Compatible Families | Reasoning |
|---|---|---|
| Type 1 (Reformer) | Hero, Sovereign, Sage | Principled improvement; driven to correct and perfect; moral compass |
| Type 2 (Helper) | Caregiver, Lover, Everyman | Relational generosity; worth through service to others |
| Type 3 (Achiever) | Hero, Magician, Sovereign | Success-driven adaptability; shape-shift to excel; image-conscious |
| Type 4 (Individualist) | Creator, Rebel, Lover | Authentic self-expression; intensity of feeling; resist the ordinary |
| Type 5 (Investigator) | Sage, Explorer, Magician | Knowledge mastery; observe before engaging; conserve resources |
| Type 6 (Loyalist) | Everyman, Caregiver, Hero | Security through belonging or courage; anticipate threats; loyal to trusted group |
| Type 7 (Enthusiast) | Jester, Explorer, Innocent | Freedom and stimulation; reframe pain as possibility; optimistic diversification |
| Type 8 (Challenger) | Rebel, Sovereign, Hero | Strength and control; protect the vulnerable; refuse to be dominated |
| Type 9 (Peacemaker) | Innocent, Everyman, Caregiver | Harmony and unity; merge with environment; resist conflict |

### DISC → Archetype Families

Based on motivation quadrant alignment. DISC maps to broad groups — less discriminating than MBTI or Enneagram.

| DISC Style | Compatible Families | Reasoning |
|---|---|---|
| D (Dominance) | Hero, Rebel, Magician, Sovereign | Results-driven, direct, decisive; agency and power orientation |
| I (Influence) | Jester, Lover, Everyman, Explorer | Enthusiastic, collaborative, optimistic; connection and inspiration |
| S (Steadiness) | Caregiver, Everyman, Innocent, Creator | Patient, reliable, supportive; stability and harmony orientation |
| C (Conscientiousness) | Sage, Sovereign, Creator, Explorer | Analytical, precise, quality-focused; independence and accuracy |

### Belbin → Archetype Families

Derived from team role descriptions and behavioral parallels. No published research — conceptual alignment.

| Belbin Role | Compatible Families | Reasoning |
|---|---|---|
| Plant | Creator, Magician | Creative, unorthodox problem-solver; generates novel ideas independently |
| Shaper | Hero, Rebel, Sovereign | Challenges the team to improve; driven, dynamic, thrives under pressure |
| Monitor Evaluator | Sage, Sovereign | Sober, strategic, discerning; sees all options and judges accurately |
| Co-ordinator | Sovereign, Caregiver | Clarifies goals, promotes team decision-making, delegates effectively |
| Teamworker | Everyman, Caregiver, Lover | Cooperative, perceptive, diplomatic; averts friction and builds cohesion |
| Implementer | Everyman, Sovereign | Disciplined, reliable, efficient; turns ideas into practical actions |
| Completer-Finisher | Sage, Sovereign | Painstaking, conscientious; ensures delivery to standard |
| Specialist | Sage, Explorer | Dedicated, self-starting, single-minded; provides rare knowledge |
| Resource Investigator | Explorer, Jester, Everyman | Extrovert who explores external opportunities and develops contacts |

### Big Five → Archetype Families

Each pole (high/low) filters compatible families. Multiple poles intersect.

| Big Five Pole | Compatible Families | Reasoning |
|---|---|---|
| High Openness | Creator, Explorer, Magician, Rebel | Curiosity, imagination, unconventional thinking |
| Low Openness | Sovereign, Everyman, Caregiver | Practical, conventional, tradition-respecting |
| High Conscientiousness | Sovereign, Hero, Sage | Organized, disciplined, achievement-oriented |
| Low Conscientiousness | Jester, Rebel, Explorer | Spontaneous, flexible, unstructured |
| High Extraversion | Jester, Hero, Lover, Everyman | Sociable, assertive, energetic |
| Low Extraversion | Sage, Creator, Innocent | Reflective, reserved, independent |
| High Agreeableness | Caregiver, Everyman, Innocent, Lover | Cooperative, trusting, empathic |
| Low Agreeableness | Rebel, Sovereign, Hero | Challenging, competitive, skeptical |
| High Neuroticism | Creator, Rebel, Lover | Emotionally intense, sensitive, vigilant |
| Low Neuroticism | Sage, Sovereign, Innocent, Everyman | Emotionally stable, calm, resilient |

### SDI → Archetype Families

Based on Strength Deployment Inventory conflict motivation styles.

| SDI Style | Compatible Families | Reasoning |
|---|---|---|
| Blue (Altruistic-Nurturing) | Caregiver, Innocent, Lover | Concern for others' welfare; open, supportive, helpful |
| Red (Assertive-Directing) | Hero, Sovereign, Rebel | Concern for task accomplishment; direct, competitive, action-oriented |
| Green (Analytic-Autonomizing) | Sage, Explorer, Creator | Concern for process and order; cautious, methodical, self-reliant |
| Hub (Flexible-Cohering) | Everyman, Magician, Jester | Flexible motivation; adapts style to situation; bridges perspectives |

---

## Section 2: Sub-Archetype Distinction Rules

Within each family, sub-archetypes are distinguished by specific trait combinations.
After family-level intersection selects a family, these rules select the sub-archetype.

### Caregiver Family

| Sub-archetype | Distinguishing Traits | MBTI Affinity | Enneagram | Key Characteristic |
|---|---|---|---|---|
| Angel | Selfless, transcendent, unconditional | INFJ, ISFJ | 2, 9 | Serves without expectation of return; radiates unconditional acceptance |
| Guardian | Protective, watchful, boundary-setting | ISTJ, ISFJ, ESTJ | 6, 1 | Shields others from harm; enforces safety through vigilance |
| Healer | Restorative, empathic, transformative | INFJ, INFP, ISFP | 2, 4 | Mends what is broken — emotional, physical, or systemic |
| Samaritan | Pragmatic, community-serving, responsive | ESFJ, ENFJ, ISFJ | 2, 6 | Practical help in the moment; shows up where needed |

**Distinguishing axis:** Angel = unconditional/spiritual; Guardian = protective/preventive; Healer = restorative/transformative; Samaritan = practical/immediate.

### Everyman Family

| Sub-archetype | Distinguishing Traits | MBTI Affinity | Enneagram | Key Characteristic |
|---|---|---|---|---|
| Advocate | Voice for others, principled representation | ENFJ, INFJ, ENFP | 1, 6 | Speaks up for those who cannot; bridges power gaps |
| Networker | Connective, socially fluent, bridge-builder | ENFP, ESFJ, ENTP | 7, 3 | Weaves relationships; knows who to connect to whom |
| Servant | Humble, dutiful, quietly essential | ISFJ, ISTJ | 2, 9 | Does the unglamorous work that holds things together |
| Citizen | Responsible, civic-minded, participatory | ESTJ, ISTJ, ESFJ | 6, 1 | Upholds community standards; reliable contributor to shared goals |

**Distinguishing axis:** Advocate = voice/justice; Networker = connection/social; Servant = humility/duty; Citizen = responsibility/participation.

### Creator Family

| Sub-archetype | Distinguishing Traits | MBTI Affinity | Enneagram | Key Characteristic |
|---|---|---|---|---|
| Artist | Aesthetic, expressive, emotionally driven | ISFP, INFP | 4 | Creates from inner emotional truth; form matters as much as function |
| Entrepreneur | Inventive, opportunistic, building-oriented | ENTP, ENTJ, ESTP | 3, 7 | Creates ventures and systems; sees market gaps and fills them |
| Storyteller | Narrative, meaning-making, communicative | ENFP, INFP, ENFJ | 4, 7 | Shapes understanding through story; gives experience a structure |
| Visionary | Future-seeing, paradigm-shifting, conceptual | INTJ, INFJ, ENTP | 5, 4 | Imagines what does not yet exist; sees beyond current constraints |

**Distinguishing axis:** Artist = aesthetic expression; Entrepreneur = venture building; Storyteller = narrative meaning; Visionary = future conception.

### Innocent Family

| Sub-archetype | Distinguishing Traits | MBTI Affinity | Enneagram | Key Characteristic |
|---|---|---|---|---|
| Child | Open, wonder-filled, unselfconscious | ESFP, ENFP, ISFP | 7, 9 | Approaches experience with fresh eyes; pre-cynical engagement |
| Dreamer | Imaginative, gentle, hope-oriented | INFP, INFJ | 9, 4 | Lives partly in possibility; sustains hope through imagination |
| Idealist | Principled, optimistic, reform-minded | INFP, ENFJ, ENFP | 1, 9 | Believes in what could be; holds the standard others have abandoned |
| Muse | Inspiring, catalytic, awakening | ENFP, INFP, ENFJ | 4, 7 | Sparks creativity in others; their presence unlocks potential |

**Distinguishing axis:** Child = openness/wonder; Dreamer = imagination/hope; Idealist = principle/reform; Muse = inspiration/catalyst.

### Explorer Family

| Sub-archetype | Distinguishing Traits | MBTI Affinity | Enneagram | Key Characteristic |
|---|---|---|---|---|
| Adventurer | Bold, sensation-seeking, physical | ESTP, ISTP, ESFP | 7, 8 | Pushes boundaries through direct experience; thrives on the unknown |
| Generalist | Broad, versatile, connecting disparate domains | ENTP, ENFP, INTP | 7, 5 | Knows enough about everything to bridge specialties |
| Pioneer | Trailblazing, first-mover, frontier-pushing | ENTJ, ENTP, INTJ | 3, 7, 8 | Goes where none have gone; opens paths for others to follow |
| Seeker | Inward-searching, philosophical, questioning | INFP, INFJ, INTP | 5, 4 | Explores meaning, truth, and identity rather than geography |

**Distinguishing axis:** Adventurer = external/physical; Generalist = breadth/versatility; Pioneer = first-mover/trailblazing; Seeker = internal/philosophical.

### Hero Family

| Sub-archetype | Distinguishing Traits | MBTI Affinity | Enneagram | Key Characteristic |
|---|---|---|---|---|
| Athlete | Disciplined, competitive, self-mastery | ESTP, ISTP, ESTJ | 3, 1 | Excellence through relentless practice; pushes personal limits |
| Liberator | Justice-driven, emancipating, systemic | ENFJ, ENTJ, ENFP | 8, 1 | Frees others from oppressive systems; fights for collective freedom |
| Rescuer | Responsive, brave, crisis-oriented | ESFJ, ISFJ, ESTJ | 2, 6 | Acts decisively when others are in danger; runs toward the fire |
| Warrior | Strategic, courageous, mission-focused | ENTJ, ESTJ, INTJ | 8, 3, 1 | Fights for a cause with discipline and determination |

**Distinguishing axis:** Athlete = self-mastery/competition; Liberator = systemic justice; Rescuer = crisis response; Warrior = strategic mission.

### Jester Family

| Sub-archetype | Distinguishing Traits | MBTI Affinity | Enneagram | Key Characteristic |
|---|---|---|---|---|
| Clown | Physical, slapstick, disarming | ESFP, ESTP | 7, 9 | Uses physicality and absurdity to break tension |
| Entertainer | Performative, charismatic, audience-aware | ESFP, ENFP, ESTP | 7, 3 | Commands attention; makes every interaction a show |
| Provocateur | Subversive, satirical, boundary-testing | ENTP, ESTP, INTJ | 7, 8, 4 | Uses humor as a weapon; reveals truth through provocation |
| Shapeshifter | Adaptive, chameleonic, context-reading | ENFP, ENTP, INFJ | 3, 7, 9 | Shifts persona to match the moment; elusive and versatile |

**Distinguishing axis:** Clown = physical/absurdist; Entertainer = performative/charismatic; Provocateur = subversive/satirical; Shapeshifter = adaptive/chameleonic.

### Lover Family

| Sub-archetype | Distinguishing Traits | MBTI Affinity | Enneagram | Key Characteristic |
|---|---|---|---|---|
| Companion | Loyal, present, steady devotion | ISFJ, ISFP, ESFJ | 2, 6, 9 | Faithful presence; intimacy through constancy and reliability |
| Hedonist | Sensory, pleasure-seeking, appreciative | ESFP, ISFP, ESTP | 7, 4 | Savours beauty, taste, texture; celebrates the physical world |
| Matchmaker | Connective, perceptive about others' bonds | ENFJ, ESFJ, ENFP | 2, 7 | Sees potential connections between people; orchestrates unions |
| Romantic | Passionate, idealizing, emotionally intense | INFP, ENFP, INFJ | 4, 2 | Love as transcendent force; seeks the extraordinary in connection |

**Distinguishing axis:** Companion = steadfast presence; Hedonist = sensory pleasure; Matchmaker = relational orchestration; Romantic = passionate idealism.

### Magician Family

| Sub-archetype | Distinguishing Traits | MBTI Affinity | Enneagram | Key Characteristic |
|---|---|---|---|---|
| Alchemist | Transformative, mysterious, process-oriented | INFJ, INTJ | 5, 4 | Turns base material into gold; transformation through hidden process |
| Engineer | Systematic, building, mechanism-focused | INTJ, INTP, ENTJ | 5, 1, 3 | Designs and builds the systems that make transformation possible |
| Innovator | Disruptive, novel, paradigm-breaking | ENTP, ENTJ, ENFP | 7, 3 | Creates new categories; makes the impossible suddenly obvious |
| Scientist | Empirical, hypothesis-driven, rigorous | INTJ, INTP, ISTJ | 5, 1 | Discovers truth through systematic experimentation and evidence |

**Distinguishing axis:** Alchemist = mysterious transformation; Engineer = systematic building; Innovator = disruptive novelty; Scientist = empirical rigor.

### Rebel Family

| Sub-archetype | Distinguishing Traits | MBTI Affinity | Enneagram | Key Characteristic |
|---|---|---|---|---|
| Activist | Cause-driven, organized, systemic change | ENFJ, ENFP, ENTJ | 1, 8 | Channels rebellion into organized movement for justice |
| Gambler | Risk-embracing, high-stakes, instinctive | ESTP, ENTP | 7, 8 | Lives on the edge; bets big and accepts consequences |
| Maverick | Independent, convention-defying, self-directed | ISTP, INTP, INTJ, ENTP | 5, 8, 4 | Does things their own way; ignores rules not through malice but irrelevance |
| Reformer | Principled disruption, improvement-focused | INTJ, ENTJ, INFJ | 1, 8 | Breaks what doesn't work to build something better; constructive rebellion |

**Distinguishing axis:** Activist = organized movement; Gambler = high-stakes risk; Maverick = independent convention-defiance; Reformer = principled improvement.

### Sage Family

| Sub-archetype | Distinguishing Traits | MBTI Affinity | Enneagram | Key Characteristic |
|---|---|---|---|---|
| Detective | Analytical, evidence-driven, investigative | ISTJ, INTJ, ISTP | 5, 6 | Uncovers truth through systematic investigation and evidence |
| Mentor | Teaching, guiding, developmental | ENFJ, INFJ, ENTJ | 1, 2, 5 | Shares accumulated wisdom to develop others' potential |
| Shaman | Intuitive, liminal, deep-pattern | INFJ, INFP, INTP | 5, 4, 9 | Accesses insight from unconventional or unseen sources |
| Translator | Synthesizing, bridging, clarifying | INTP, ENTP, INTJ | 5, 7 | Makes the complex accessible; bridges disciplines and audiences |

**Distinguishing axis:** Detective = investigative evidence; Mentor = developmental guidance; Shaman = intuitive depth; Translator = synthesis and clarity.

### Sovereign Family

| Sub-archetype | Distinguishing Traits | MBTI Affinity | Enneagram | Key Characteristic |
|---|---|---|---|---|
| Ambassador | Diplomatic, representative, bridge-building | ENFJ, ESFJ, ENTJ | 3, 2, 9 | Represents and negotiates between groups; builds consensus |
| Judge | Principled, evaluative, fair-minded | INTJ, ISTJ, ESTJ | 1, 5, 6 | Weighs evidence and renders decisions; upholds standards impartially |
| Patriarch | Protective authority, legacy-building | ESTJ, ENTJ, ISTJ | 8, 1, 6 | Provides structure and security through established authority |
| Ruler | Commanding, systemic, order-creating | ENTJ, ESTJ | 8, 3, 1 | Takes charge and creates order; exercises power to build and maintain systems |

**Distinguishing axis:** Ambassador = diplomatic representation; Judge = principled evaluation; Patriarch = protective authority; Ruler = commanding order.

---

## Section 3: Intersection Algorithm

### Input

A set of personality framework values. Any subset of:

```
mbti:         one of 16 types
enneagram:    one of 9 types
disc:         one of D, I, S, C
belbin:       one of 9 roles
bigFive:      one or more of 10 poles (highO, lowO, highC, lowC, highE, lowE, highA, lowA, highN, lowN)
sdi:          one of Blue, Red, Green, Hub
```

### Algorithm

```
1. candidates = ALL_60_ARCHETYPES

2. For each specified framework value:
     a. Look up compatible_families(value) from the Section 1 tables
     b. Filter candidates to only those whose family is in compatible_families
     c. If candidates is empty → CONFLICT (see step 5)

3. If |candidates| == 1:
     → CONVERGED: return the single archetype

4. If |candidates| > 1 and all candidates are in the SAME family:
     → Use Section 2 sub-archetype distinction rules:
       a. Check MBTI affinity column — prefer sub-archetypes matching the specified MBTI
       b. Check Enneagram column — prefer sub-archetypes matching the specified Enneagram
       c. If still ambiguous (2-3 candidates) → PRESENT CHOICES to user
       d. If exactly 1 match → CONVERGED

5. If |candidates| > 1 across MULTIPLE families:
     → Insufficient specification. Report remaining families and ask for
       another framework value to narrow further.

6. If candidates is empty (CONFLICT):
     → Identify which two framework values produced disjoint family sets
     → Report: "MBTI INTJ (→ Sage, Magician, Sovereign) conflicts with
       Enneagram 7 (→ Jester, Explorer, Innocent) — no shared family"
     → User must revise one value
```

### Convergence Examples

**Example 1: Quick convergence**
```
Input:  MBTI=INTJ, Enneagram=5
Step 1: compatible(INTJ) = {Sage, Magician, Sovereign}
Step 2: compatible(E5)   = {Sage, Explorer, Magician}
Intersection of families = {Sage, Magician}
Candidates = {Detective, Mentor, Shaman, Translator, Alchemist, Engineer, Innovator, Scientist}

Add: DISC=C
Step 3: compatible(DISC-C) = {Sage, Sovereign, Creator, Explorer}
Intersection of families  = {Sage}
Candidates = {Detective, Mentor, Shaman, Translator}

Sub-archetype rules (INTJ + E5):
  Detective: INTJ ✓, E5 ✓
  Mentor:    INTJ ✗ (ENFJ/INFJ), E5 ✓
  Shaman:    INTJ ✗ (INFJ/INFP), E5 ✓
  Translator: INTJ ✓, E5 ✓

Two candidates: Detective, Translator
→ PRESENT CHOICES: "Your profile matches Detective (investigative, evidence-driven)
  or Translator (synthesizing, bridging). Which fits better?"
```

**Example 2: Immediate convergence**
```
Input:  MBTI=ESTP, Enneagram=7
Step 1: compatible(ESTP) = {Hero, Jester, Rebel}
Step 2: compatible(E7)   = {Jester, Explorer, Innocent}
Intersection = {Jester}
Candidates = {Clown, Entertainer, Provocateur, Shapeshifter}

Sub-archetype rules (ESTP + E7):
  Clown:       ESTP ✓, E7 ✓
  Entertainer: ESTP ✓, E7 ✓
  Provocateur: ESTP ✓, E7 ✓ (but also E8)
  Shapeshifter: ESTP ✗ (ENFP/ENTP), E7 ✓

Three candidates: Clown, Entertainer, Provocateur
→ PRESENT CHOICES with descriptions
```

**Example 3: Conflict detection**
```
Input:  MBTI=ISFJ, Enneagram=8
Step 1: compatible(ISFJ) = {Caregiver, Everyman, Innocent}
Step 2: compatible(E8)   = {Rebel, Sovereign, Hero}
Intersection = {} (empty)
→ CONFLICT: "ISFJ (→ Caregiver, Everyman, Innocent) has no overlap with
  Enneagram 8 (→ Rebel, Sovereign, Hero). This combination is contradictory."
```

### Partial Specification

The algorithm works with any number of inputs:

| Inputs specified | Typical result |
|---|---|
| 1 framework value | 2-4 compatible families (~8-20 candidates) |
| 2 framework values | 1-2 compatible families (~4-8 candidates) |
| 3 framework values | 1 family (~4 candidates, sub-archetype rules apply) |
| 4+ framework values | 1 archetype (converged) or conflict detected |

Users can stop at any level of specificity. An agent with only family-level
resolution ("you are a Sage") is valid — the sub-archetype adds optional refinement.

### Reverse Direction: Archetype → Framework Values

The same compatibility tables work in reverse. Given a selected archetype:

1. Look up all framework values where the archetype's family appears in
   their compatible set
2. These are the valid/suggested framework values for that archetype
3. Present as defaults — user can accept or override (override may shift
   the archetype)

```
Selected: Detective (Sage family)

Suggested values:
  MBTI:      ISTJ, INTJ, ISTP (from sub-archetype affinity)
  Enneagram: 5, 6 (from sub-archetype affinity)
  DISC:      C (Sage-compatible)
  Belbin:    Monitor Evaluator, Specialist, Completer-Finisher (Sage-compatible)
  Big Five:  High O, High C, Low E, Low N (Sage-compatible poles)
  SDI:       Green (Sage-compatible)
```

---

## Appendix: Complete Family Membership

For quick reference — all 60 archetypes by family.

| Family | Sub-Archetypes |
|---|---|
| Caregiver | Angel, Guardian, Healer, Samaritan |
| Everyman | Advocate, Networker, Servant, Citizen |
| Creator | Artist, Entrepreneur, Storyteller, Visionary |
| Innocent | Child, Dreamer, Idealist, Muse |
| Explorer | Adventurer, Generalist, Pioneer, Seeker |
| Hero | Athlete, Liberator, Rescuer, Warrior |
| Jester | Clown, Entertainer, Provocateur, Shapeshifter |
| Lover | Companion, Hedonist, Matchmaker, Romantic |
| Magician | Alchemist, Engineer, Innovator, Scientist |
| Rebel | Activist, Gambler, Maverick, Reformer |
| Sage | Detective, Mentor, Shaman, Translator |
| Sovereign | Ambassador, Judge, Patriarch, Ruler |
