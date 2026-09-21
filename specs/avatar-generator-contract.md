# Avatar Generator Contract

Specification for the blocks-ui agent avatar system. Covers two subsystems:
1. **Faceted personality selector** — interactive UI for building agent personality from any entry point
2. **Visual avatar generation** — producing distinctive agent avatars from converged personality data

---

## Section 1: The Archetype System

Agent personality is built on the Hartwell & Chen archetype model (*Archetypes in Branding*, 2012):
**4 motivation quadrants → 12 archetype families → 60 sub-archetypes**.

### Motivation Quadrants

| Quadrant | Drive | Families |
|---|---|---|
| **Independence** | Self-actualization | Innocent, Explorer, Sage |
| **Mastery** | Agency, power | Hero, Rebel, Magician |
| **Belonging** | Connection | Everyman, Lover, Jester |
| **Stability** | Structure | Caregiver, Creator, Sovereign |

### All 60 Sub-Archetypes

| Family | Sub-Archetypes | Key Characteristics |
|---|---|---|
| **Caregiver** | Angel, Guardian, Healer, Samaritan | Angel: unconditional service · Guardian: protective vigilance · Healer: restorative transformation · Samaritan: pragmatic immediate help |
| **Everyman** | Advocate, Networker, Servant, Citizen | Advocate: voice for the voiceless · Networker: relational bridge-builder · Servant: humble essential work · Citizen: reliable community contributor |
| **Creator** | Artist, Entrepreneur, Storyteller, Visionary | Artist: aesthetic emotional expression · Entrepreneur: venture and system building · Storyteller: narrative meaning-making · Visionary: future conception beyond constraints |
| **Innocent** | Child, Dreamer, Idealist, Muse | Child: wonder and fresh engagement · Dreamer: hope through imagination · Idealist: principled optimism · Muse: catalytic inspiration for others |
| **Explorer** | Adventurer, Generalist, Pioneer, Seeker | Adventurer: bold physical boundary-pushing · Generalist: versatile cross-domain bridging · Pioneer: trailblazing first-mover · Seeker: inward philosophical questioning |
| **Hero** | Athlete, Liberator, Rescuer, Warrior | Athlete: disciplined self-mastery · Liberator: systemic justice and emancipation · Rescuer: decisive crisis response · Warrior: strategic mission-focused courage |
| **Jester** | Clown, Entertainer, Provocateur, Shapeshifter | Clown: physical absurdist tension-breaker · Entertainer: charismatic performer · Provocateur: subversive satirical truth-teller · Shapeshifter: adaptive chameleonic context-reader |
| **Lover** | Companion, Hedonist, Matchmaker, Romantic | Companion: steadfast loyal presence · Hedonist: sensory celebration · Matchmaker: relational orchestrator · Romantic: passionate transcendent idealism |
| **Magician** | Alchemist, Engineer, Innovator, Scientist | Alchemist: mysterious hidden transformation · Engineer: systematic mechanism builder · Innovator: disruptive paradigm-breaker · Scientist: empirical rigorous discovery |
| **Rebel** | Activist, Gambler, Maverick, Reformer | Activist: organized movement for justice · Gambler: high-stakes instinctive risk · Maverick: independent convention-defiance · Reformer: principled constructive disruption |
| **Sage** | Detective, Mentor, Shaman, Translator | Detective: investigative evidence-driven truth · Mentor: developmental wisdom-sharing · Shaman: intuitive liminal insight · Translator: synthesis and cross-domain clarity |
| **Sovereign** | Ambassador, Judge, Patriarch, Ruler | Ambassador: diplomatic consensus-building · Judge: principled impartial evaluation · Patriarch: protective authoritative legacy · Ruler: commanding systemic order |

---

## Section 2: Compatibility Data

Six personality frameworks map to archetype families. Each framework value is associated with
a set of compatible archetype families. The intersection of all selected values yields the
target archetype.

Full data reference: `specs/archetype-compatibility-matrix.md`

### JSON Compatibility Structure

The following JSON structure contains all compatibility data needed by the faceted selector.
The UI team can consume this directly or transform it into their preferred format.

```json
{
  "quadrants": {
    "Independence": ["Innocent", "Explorer", "Sage"],
    "Mastery": ["Hero", "Rebel", "Magician"],
    "Belonging": ["Everyman", "Lover", "Jester"],
    "Stability": ["Caregiver", "Creator", "Sovereign"]
  },

  "frameworks": {
    "mbti": {
      "label": "Myers-Briggs Type",
      "values": {
        "INTJ": { "compatibleFamilies": ["Sage", "Magician", "Sovereign"] },
        "INTP": { "compatibleFamilies": ["Sage", "Explorer", "Creator"] },
        "ENTJ": { "compatibleFamilies": ["Sovereign", "Hero", "Magician"] },
        "ENTP": { "compatibleFamilies": ["Magician", "Rebel", "Explorer"] },
        "INFJ": { "compatibleFamilies": ["Magician", "Sage", "Caregiver"] },
        "INFP": { "compatibleFamilies": ["Creator", "Innocent", "Explorer"] },
        "ENFJ": { "compatibleFamilies": ["Caregiver", "Sovereign", "Magician"] },
        "ENFP": { "compatibleFamilies": ["Explorer", "Creator", "Jester"] },
        "ISTJ": { "compatibleFamilies": ["Sovereign", "Everyman", "Sage"] },
        "ISFJ": { "compatibleFamilies": ["Caregiver", "Everyman", "Innocent"] },
        "ESTJ": { "compatibleFamilies": ["Sovereign", "Hero", "Everyman"] },
        "ESFJ": { "compatibleFamilies": ["Caregiver", "Everyman", "Lover"] },
        "ISTP": { "compatibleFamilies": ["Explorer", "Hero", "Rebel"] },
        "ISFP": { "compatibleFamilies": ["Creator", "Lover", "Innocent"] },
        "ESTP": { "compatibleFamilies": ["Hero", "Jester", "Rebel"] },
        "ESFP": { "compatibleFamilies": ["Jester", "Lover", "Innocent"] }
      }
    },

    "enneagram": {
      "label": "Enneagram Type",
      "values": {
        "1": { "label": "Reformer", "compatibleFamilies": ["Hero", "Sovereign", "Sage"] },
        "2": { "label": "Helper", "compatibleFamilies": ["Caregiver", "Lover", "Everyman"] },
        "3": { "label": "Achiever", "compatibleFamilies": ["Hero", "Magician", "Sovereign"] },
        "4": { "label": "Individualist", "compatibleFamilies": ["Creator", "Rebel", "Lover"] },
        "5": { "label": "Investigator", "compatibleFamilies": ["Sage", "Explorer", "Magician"] },
        "6": { "label": "Loyalist", "compatibleFamilies": ["Everyman", "Caregiver", "Hero"] },
        "7": { "label": "Enthusiast", "compatibleFamilies": ["Jester", "Explorer", "Innocent"] },
        "8": { "label": "Challenger", "compatibleFamilies": ["Rebel", "Sovereign", "Hero"] },
        "9": { "label": "Peacemaker", "compatibleFamilies": ["Innocent", "Everyman", "Caregiver"] }
      }
    },

    "disc": {
      "label": "DISC Style",
      "values": {
        "D": { "label": "Dominance", "compatibleFamilies": ["Hero", "Rebel", "Magician", "Sovereign"] },
        "I": { "label": "Influence", "compatibleFamilies": ["Jester", "Lover", "Everyman", "Explorer"] },
        "S": { "label": "Steadiness", "compatibleFamilies": ["Caregiver", "Everyman", "Innocent", "Creator"] },
        "C": { "label": "Conscientiousness", "compatibleFamilies": ["Sage", "Sovereign", "Creator", "Explorer"] }
      }
    },

    "belbin": {
      "label": "Belbin Team Role",
      "values": {
        "Plant": { "compatibleFamilies": ["Creator", "Magician"] },
        "Shaper": { "compatibleFamilies": ["Hero", "Rebel", "Sovereign"] },
        "Monitor Evaluator": { "compatibleFamilies": ["Sage", "Sovereign"] },
        "Co-ordinator": { "compatibleFamilies": ["Sovereign", "Caregiver"] },
        "Teamworker": { "compatibleFamilies": ["Everyman", "Caregiver", "Lover"] },
        "Implementer": { "compatibleFamilies": ["Everyman", "Sovereign"] },
        "Completer-Finisher": { "compatibleFamilies": ["Sage", "Sovereign"] },
        "Specialist": { "compatibleFamilies": ["Sage", "Explorer"] },
        "Resource Investigator": { "compatibleFamilies": ["Explorer", "Jester", "Everyman"] }
      }
    },

    "bigFive": {
      "label": "Big Five Traits",
      "multiSelect": true,
      "values": {
        "highO": { "label": "High Openness", "compatibleFamilies": ["Creator", "Explorer", "Magician", "Rebel"] },
        "lowO": { "label": "Low Openness", "compatibleFamilies": ["Sovereign", "Everyman", "Caregiver"] },
        "highC": { "label": "High Conscientiousness", "compatibleFamilies": ["Sovereign", "Hero", "Sage"] },
        "lowC": { "label": "Low Conscientiousness", "compatibleFamilies": ["Jester", "Rebel", "Explorer"] },
        "highE": { "label": "High Extraversion", "compatibleFamilies": ["Jester", "Hero", "Lover", "Everyman"] },
        "lowE": { "label": "Low Extraversion", "compatibleFamilies": ["Sage", "Creator", "Innocent"] },
        "highA": { "label": "High Agreeableness", "compatibleFamilies": ["Caregiver", "Everyman", "Innocent", "Lover"] },
        "lowA": { "label": "Low Agreeableness", "compatibleFamilies": ["Rebel", "Sovereign", "Hero"] },
        "highN": { "label": "High Neuroticism", "compatibleFamilies": ["Creator", "Rebel", "Lover"] },
        "lowN": { "label": "Low Neuroticism", "compatibleFamilies": ["Sage", "Sovereign", "Innocent", "Everyman"] }
      }
    },

    "sdi": {
      "label": "SDI Conflict Style",
      "values": {
        "Blue": { "label": "Altruistic-Nurturing", "compatibleFamilies": ["Caregiver", "Innocent", "Lover"] },
        "Red": { "label": "Assertive-Directing", "compatibleFamilies": ["Hero", "Sovereign", "Rebel"] },
        "Green": { "label": "Analytic-Autonomizing", "compatibleFamilies": ["Sage", "Explorer", "Creator"] },
        "Hub": { "label": "Flexible-Cohering", "compatibleFamilies": ["Everyman", "Magician", "Jester"] }
      }
    }
  },

  "families": {
    "Caregiver": {
      "quadrant": "Stability",
      "drive": "Service, compassion, generosity",
      "subArchetypes": {
        "Angel": {
          "mbtiAffinity": ["INFJ", "ISFJ"],
          "enneagramAffinity": [2, 9],
          "keyCharacteristic": "Serves without expectation of return; radiates unconditional acceptance"
        },
        "Guardian": {
          "mbtiAffinity": ["ISTJ", "ISFJ", "ESTJ"],
          "enneagramAffinity": [6, 1],
          "keyCharacteristic": "Shields others from harm; enforces safety through vigilance"
        },
        "Healer": {
          "mbtiAffinity": ["INFJ", "INFP", "ISFP"],
          "enneagramAffinity": [2, 4],
          "keyCharacteristic": "Mends what is broken — emotional, physical, or systemic"
        },
        "Samaritan": {
          "mbtiAffinity": ["ESFJ", "ENFJ", "ISFJ"],
          "enneagramAffinity": [2, 6],
          "keyCharacteristic": "Practical help in the moment; shows up where needed"
        }
      }
    },
    "Everyman": {
      "quadrant": "Belonging",
      "drive": "Belonging, empathy, realism",
      "subArchetypes": {
        "Advocate": {
          "mbtiAffinity": ["ENFJ", "INFJ", "ENFP"],
          "enneagramAffinity": [1, 6],
          "keyCharacteristic": "Speaks up for those who cannot; bridges power gaps"
        },
        "Networker": {
          "mbtiAffinity": ["ENFP", "ESFJ", "ENTP"],
          "enneagramAffinity": [7, 3],
          "keyCharacteristic": "Weaves relationships; knows who to connect to whom"
        },
        "Servant": {
          "mbtiAffinity": ["ISFJ", "ISTJ"],
          "enneagramAffinity": [2, 9],
          "keyCharacteristic": "Does the unglamorous work that holds things together"
        },
        "Citizen": {
          "mbtiAffinity": ["ESTJ", "ISTJ", "ESFJ"],
          "enneagramAffinity": [6, 1],
          "keyCharacteristic": "Upholds community standards; reliable contributor to shared goals"
        }
      }
    },
    "Creator": {
      "quadrant": "Stability",
      "drive": "Innovation, expression, imagination",
      "subArchetypes": {
        "Artist": {
          "mbtiAffinity": ["ISFP", "INFP"],
          "enneagramAffinity": [4],
          "keyCharacteristic": "Creates from inner emotional truth; form matters as much as function"
        },
        "Entrepreneur": {
          "mbtiAffinity": ["ENTP", "ENTJ", "ESTP"],
          "enneagramAffinity": [3, 7],
          "keyCharacteristic": "Creates ventures and systems; sees market gaps and fills them"
        },
        "Storyteller": {
          "mbtiAffinity": ["ENFP", "INFP", "ENFJ"],
          "enneagramAffinity": [4, 7],
          "keyCharacteristic": "Shapes understanding through story; gives experience a structure"
        },
        "Visionary": {
          "mbtiAffinity": ["INTJ", "INFJ", "ENTP"],
          "enneagramAffinity": [5, 4],
          "keyCharacteristic": "Imagines what does not yet exist; sees beyond current constraints"
        }
      }
    },
    "Innocent": {
      "quadrant": "Independence",
      "drive": "Safety, optimism, simplicity",
      "subArchetypes": {
        "Child": {
          "mbtiAffinity": ["ESFP", "ENFP", "ISFP"],
          "enneagramAffinity": [7, 9],
          "keyCharacteristic": "Approaches experience with fresh eyes; pre-cynical engagement"
        },
        "Dreamer": {
          "mbtiAffinity": ["INFP", "INFJ"],
          "enneagramAffinity": [9, 4],
          "keyCharacteristic": "Lives partly in possibility; sustains hope through imagination"
        },
        "Idealist": {
          "mbtiAffinity": ["INFP", "ENFJ", "ENFP"],
          "enneagramAffinity": [1, 9],
          "keyCharacteristic": "Believes in what could be; holds the standard others have abandoned"
        },
        "Muse": {
          "mbtiAffinity": ["ENFP", "INFP", "ENFJ"],
          "enneagramAffinity": [4, 7],
          "keyCharacteristic": "Sparks creativity in others; their presence unlocks potential"
        }
      }
    },
    "Explorer": {
      "quadrant": "Independence",
      "drive": "Freedom, discovery, self-sufficiency",
      "subArchetypes": {
        "Adventurer": {
          "mbtiAffinity": ["ESTP", "ISTP", "ESFP"],
          "enneagramAffinity": [7, 8],
          "keyCharacteristic": "Pushes boundaries through direct experience; thrives on the unknown"
        },
        "Generalist": {
          "mbtiAffinity": ["ENTP", "ENFP", "INTP"],
          "enneagramAffinity": [7, 5],
          "keyCharacteristic": "Knows enough about everything to bridge specialties"
        },
        "Pioneer": {
          "mbtiAffinity": ["ENTJ", "ENTP", "INTJ"],
          "enneagramAffinity": [3, 7, 8],
          "keyCharacteristic": "Goes where none have gone; opens paths for others to follow"
        },
        "Seeker": {
          "mbtiAffinity": ["INFP", "INFJ", "INTP"],
          "enneagramAffinity": [5, 4],
          "keyCharacteristic": "Explores meaning, truth, and identity rather than geography"
        }
      }
    },
    "Hero": {
      "quadrant": "Mastery",
      "drive": "Mastery, courage, achievement",
      "subArchetypes": {
        "Athlete": {
          "mbtiAffinity": ["ESTP", "ISTP", "ESTJ"],
          "enneagramAffinity": [3, 1],
          "keyCharacteristic": "Excellence through relentless practice; pushes personal limits"
        },
        "Liberator": {
          "mbtiAffinity": ["ENFJ", "ENTJ", "ENFP"],
          "enneagramAffinity": [8, 1],
          "keyCharacteristic": "Frees others from oppressive systems; fights for collective freedom"
        },
        "Rescuer": {
          "mbtiAffinity": ["ESFJ", "ISFJ", "ESTJ"],
          "enneagramAffinity": [2, 6],
          "keyCharacteristic": "Acts decisively when others are in danger; runs toward the fire"
        },
        "Warrior": {
          "mbtiAffinity": ["ENTJ", "ESTJ", "INTJ"],
          "enneagramAffinity": [8, 3, 1],
          "keyCharacteristic": "Fights for a cause with discipline and determination"
        }
      }
    },
    "Jester": {
      "quadrant": "Belonging",
      "drive": "Joy, humor, spontaneity",
      "subArchetypes": {
        "Clown": {
          "mbtiAffinity": ["ESFP", "ESTP"],
          "enneagramAffinity": [7, 9],
          "keyCharacteristic": "Uses physicality and absurdity to break tension"
        },
        "Entertainer": {
          "mbtiAffinity": ["ESFP", "ENFP", "ESTP"],
          "enneagramAffinity": [7, 3],
          "keyCharacteristic": "Commands attention; makes every interaction a show"
        },
        "Provocateur": {
          "mbtiAffinity": ["ENTP", "ESTP", "INTJ"],
          "enneagramAffinity": [7, 8, 4],
          "keyCharacteristic": "Uses humor as a weapon; reveals truth through provocation"
        },
        "Shapeshifter": {
          "mbtiAffinity": ["ENFP", "ENTP", "INFJ"],
          "enneagramAffinity": [3, 7, 9],
          "keyCharacteristic": "Shifts persona to match the moment; elusive and versatile"
        }
      }
    },
    "Lover": {
      "quadrant": "Belonging",
      "drive": "Intimacy, passion, commitment",
      "subArchetypes": {
        "Companion": {
          "mbtiAffinity": ["ISFJ", "ISFP", "ESFJ"],
          "enneagramAffinity": [2, 6, 9],
          "keyCharacteristic": "Faithful presence; intimacy through constancy and reliability"
        },
        "Hedonist": {
          "mbtiAffinity": ["ESFP", "ISFP", "ESTP"],
          "enneagramAffinity": [7, 4],
          "keyCharacteristic": "Savours beauty, taste, texture; celebrates the physical world"
        },
        "Matchmaker": {
          "mbtiAffinity": ["ENFJ", "ESFJ", "ENFP"],
          "enneagramAffinity": [2, 7],
          "keyCharacteristic": "Sees potential connections between people; orchestrates unions"
        },
        "Romantic": {
          "mbtiAffinity": ["INFP", "ENFP", "INFJ"],
          "enneagramAffinity": [4, 2],
          "keyCharacteristic": "Love as transcendent force; seeks the extraordinary in connection"
        }
      }
    },
    "Magician": {
      "quadrant": "Mastery",
      "drive": "Transformation, vision, catalyst",
      "subArchetypes": {
        "Alchemist": {
          "mbtiAffinity": ["INFJ", "INTJ"],
          "enneagramAffinity": [5, 4],
          "keyCharacteristic": "Turns base material into gold; transformation through hidden process"
        },
        "Engineer": {
          "mbtiAffinity": ["INTJ", "INTP", "ENTJ"],
          "enneagramAffinity": [5, 1, 3],
          "keyCharacteristic": "Designs and builds the systems that make transformation possible"
        },
        "Innovator": {
          "mbtiAffinity": ["ENTP", "ENTJ", "ENFP"],
          "enneagramAffinity": [7, 3],
          "keyCharacteristic": "Creates new categories; makes the impossible suddenly obvious"
        },
        "Scientist": {
          "mbtiAffinity": ["INTJ", "INTP", "ISTJ"],
          "enneagramAffinity": [5, 1],
          "keyCharacteristic": "Discovers truth through systematic experimentation and evidence"
        }
      }
    },
    "Rebel": {
      "quadrant": "Mastery",
      "drive": "Liberation, disruption, revolution",
      "subArchetypes": {
        "Activist": {
          "mbtiAffinity": ["ENFJ", "ENFP", "ENTJ"],
          "enneagramAffinity": [1, 8],
          "keyCharacteristic": "Channels rebellion into organized movement for justice"
        },
        "Gambler": {
          "mbtiAffinity": ["ESTP", "ENTP"],
          "enneagramAffinity": [7, 8],
          "keyCharacteristic": "Lives on the edge; bets big and accepts consequences"
        },
        "Maverick": {
          "mbtiAffinity": ["ISTP", "INTP", "INTJ", "ENTP"],
          "enneagramAffinity": [5, 8, 4],
          "keyCharacteristic": "Does things their own way; ignores rules through irrelevance not malice"
        },
        "Reformer": {
          "mbtiAffinity": ["INTJ", "ENTJ", "INFJ"],
          "enneagramAffinity": [1, 8],
          "keyCharacteristic": "Breaks what doesn't work to build something better"
        }
      }
    },
    "Sage": {
      "quadrant": "Independence",
      "drive": "Knowledge, truth, understanding",
      "subArchetypes": {
        "Detective": {
          "mbtiAffinity": ["ISTJ", "INTJ", "ISTP"],
          "enneagramAffinity": [5, 6],
          "keyCharacteristic": "Uncovers truth through systematic investigation and evidence"
        },
        "Mentor": {
          "mbtiAffinity": ["ENFJ", "INFJ", "ENTJ"],
          "enneagramAffinity": [1, 2, 5],
          "keyCharacteristic": "Shares accumulated wisdom to develop others' potential"
        },
        "Shaman": {
          "mbtiAffinity": ["INFJ", "INFP", "INTP"],
          "enneagramAffinity": [5, 4, 9],
          "keyCharacteristic": "Accesses insight from unconventional or unseen sources"
        },
        "Translator": {
          "mbtiAffinity": ["INTP", "ENTP", "INTJ"],
          "enneagramAffinity": [5, 7],
          "keyCharacteristic": "Makes the complex accessible; bridges disciplines and audiences"
        }
      }
    },
    "Sovereign": {
      "quadrant": "Stability",
      "drive": "Control, order, leadership",
      "subArchetypes": {
        "Ambassador": {
          "mbtiAffinity": ["ENFJ", "ESFJ", "ENTJ"],
          "enneagramAffinity": [3, 2, 9],
          "keyCharacteristic": "Represents and negotiates between groups; builds consensus"
        },
        "Judge": {
          "mbtiAffinity": ["INTJ", "ISTJ", "ESTJ"],
          "enneagramAffinity": [1, 5, 6],
          "keyCharacteristic": "Weighs evidence and renders decisions; upholds standards impartially"
        },
        "Patriarch": {
          "mbtiAffinity": ["ESTJ", "ENTJ", "ISTJ"],
          "enneagramAffinity": [8, 1, 6],
          "keyCharacteristic": "Provides structure and security through established authority"
        },
        "Ruler": {
          "mbtiAffinity": ["ENTJ", "ESTJ"],
          "enneagramAffinity": [8, 3, 1],
          "keyCharacteristic": "Takes charge and creates order; exercises power to build and maintain systems"
        }
      }
    }
  }
}
```

---

## Section 3: Faceted Selection Interaction Model

The personality selector is a **multi-entry faceted filter**. Users can start from any
dimension — archetype, MBTI, Enneagram, DISC, Belbin, Big Five, or SDI. Every selection
narrows all other dimensions via set intersection.

### Entry Points

All entry points are equally valid. The system supports:

- **Pick an archetype family** (e.g., Sage) → narrows all framework values to Sage-compatible
- **Pick a sub-archetype** (e.g., Detective) → locks the family, narrows frameworks further
- **Pick an MBTI type** (e.g., INTJ) → narrows compatible families and other frameworks
- **Pick an Enneagram type** → further narrows
- **Pick from any other framework** → further narrows
- **Pick adjectives** (after archetype convergence) → fine-grained refinement

### Interaction Rules

1. **Initial state:** all 60 archetypes available, all framework values available
2. **User selects** a value from ANY dimension
3. **System computes intersection:**
   - Which archetype families are still compatible with ALL selections?
   - Which framework values in OTHER dimensions remain compatible?
4. **UI updates:** incompatible values are greyed out (not hidden — users should see what was eliminated)
5. **Repeat** until converged
6. **Adjective selection** becomes available once an archetype is converged

### Selection Algorithm

```
State:
  selections: Map<Framework, Value>         // user's choices so far
  candidateFamilies: Set<Family>            // families compatible with all selections
  candidateArchetypes: Set<SubArchetype>    // sub-archetypes in candidate families
  availableValues: Map<Framework, Set<Value>>  // non-greyed values per framework

afterSelection(framework, value, state):
  1. state.selections[framework] = value

  2. // Recompute candidate families: intersect compatible families for ALL selections
     candidateFamilies = ALL_FAMILIES
     for each (fw, val) in state.selections:
       candidateFamilies = candidateFamilies ∩ compatibleFamilies(fw, val)

  3. // If candidate families is empty → CONFLICT
     if candidateFamilies.isEmpty():
       return CONFLICT {
         conflicting: identifyConflictingPair(state.selections),
         suggestion: "Remove one of the conflicting selections"
       }

  4. // Compute candidate sub-archetypes from remaining families
     candidateArchetypes = all sub-archetypes in candidateFamilies

  5. // If single family remains, apply sub-archetype distinction rules
     if candidateFamilies.size == 1:
       candidateArchetypes = filterBySubArchetypeAffinity(
         candidateArchetypes, state.selections
       )

  6. // Compute available values for each unselected framework
     for each unselected framework fw:
       availableValues[fw] = { v : compatibleFamilies(fw, v) ∩ candidateFamilies ≠ ∅ }

  7. return state

filterBySubArchetypeAffinity(candidates, selections):
  // Within a single family, narrow using MBTI and Enneagram affinity
  if selections.has("mbti"):
    preferred = candidates where mbtiAffinity includes selections["mbti"]
    if preferred.nonEmpty(): candidates = preferred
  if selections.has("enneagram"):
    preferred = candidates where enneagramAffinity includes selections["enneagram"]
    if preferred.nonEmpty(): candidates = preferred
  return candidates
```

### Convergence States

| State | Condition | UI Behavior |
|---|---|---|
| **Multiple families** | 2+ families in candidateFamilies | Show remaining families grouped by quadrant; show available framework values |
| **Single family** | 1 family, 2+ sub-archetypes | Show sub-archetype cards with descriptions; let user pick or add more frameworks |
| **Converged** | 1 sub-archetype | Show archetype card; enable adjective selection |
| **Choice needed** | 2-3 sub-archetypes after all filters | Present cards: "Your profile matches Detective or Translator — which fits?" |
| **Conflict** | Empty intersection | Highlight conflicting selections in red; suggest which to revise |

### Conflict Detection

When the intersection is empty, identify the minimal conflicting pair:

```
identifyConflictingPair(selections):
  for each pair (a, b) in selections:
    if compatibleFamilies(a) ∩ compatibleFamilies(b) == ∅:
      return (a, b)
  // If no pair conflicts alone, it's a three-way conflict
  return findMinimalConflictingSubset(selections)
```

Display: `"INTJ (→ Sage, Magician, Sovereign) conflicts with Enneagram 7
(→ Jester, Explorer, Innocent) — no shared archetype family"`

---

## Section 4: Adjective System

After archetype convergence, users can add 1-2 adjectives for fine-grained personality
refinement. Adjectives modify within the archetype's character space — they cannot
contradict the archetype's core identity.

### Adjective Catalog

Each sub-archetype defines valid and invalid adjectives. The UI shows only valid adjectives
after convergence.

#### Caregiver Family

| Sub-archetype | Valid Adjectives | Invalid Adjectives |
|---|---|---|
| Angel | gentle, serene, selfless, patient, radiant, forgiving, compassionate, transcendent | aggressive, cynical, calculating, ruthless |
| Guardian | vigilant, steadfast, protective, disciplined, reliable, firm, alert | reckless, chaotic, negligent, indifferent |
| Healer | empathic, restorative, intuitive, nurturing, transformative, patient, holistic | destructive, callous, impatient, clinical |
| Samaritan | pragmatic, responsive, generous, grounded, warm, community-minded, tireless | detached, theoretical, self-serving, aloof |

#### Everyman Family

| Sub-archetype | Valid Adjectives | Invalid Adjectives |
|---|---|---|
| Advocate | principled, passionate, persistent, courageous, empathic, articulate, just | apathetic, compliant, self-interested |
| Networker | gregarious, perceptive, resourceful, warm, strategic, inclusive, energetic | isolated, rigid, secretive, antisocial |
| Servant | humble, reliable, devoted, quiet, essential, steadfast, thorough | proud, showy, demanding, self-promoting |
| Citizen | responsible, dependable, civic-minded, fair, consistent, dutiful, cooperative | anarchic, selfish, unreliable, reckless |

#### Creator Family

| Sub-archetype | Valid Adjectives | Invalid Adjectives |
|---|---|---|
| Artist | expressive, sensitive, authentic, aesthetic, emotional, original, intense | conformist, insensitive, utilitarian |
| Entrepreneur | inventive, opportunistic, bold, driven, pragmatic, adaptive, ambitious | passive, risk-averse, complacent |
| Storyteller | narrative, eloquent, engaging, imaginative, empathic, evocative, perceptive | inarticulate, literal, dry, monotone |
| Visionary | prophetic, conceptual, revolutionary, far-sighted, paradigm-shifting, bold | myopic, conventional, incremental, timid |

#### Innocent Family

| Sub-archetype | Valid Adjectives | Invalid Adjectives |
|---|---|---|
| Child | curious, open, spontaneous, playful, trusting, wonder-filled, unselfconscious | jaded, cynical, calculating, guarded |
| Dreamer | imaginative, gentle, hopeful, ethereal, contemplative, soft, wistful | harsh, pragmatic, cynical, grounded |
| Idealist | principled, optimistic, reform-minded, earnest, aspirational, steadfast | cynical, nihilistic, apathetic, corrupt |
| Muse | inspiring, catalytic, luminous, awakening, magnetic, ethereal, enchanting | dulling, discouraging, draining, mundane |

#### Explorer Family

| Sub-archetype | Valid Adjectives | Invalid Adjectives |
|---|---|---|
| Adventurer | bold, fearless, restless, physical, daring, energetic, spontaneous | timid, sedentary, cautious, routine |
| Generalist | versatile, curious, broad, adaptable, connecting, resourceful, eclectic | narrow, rigid, specialist, dogmatic |
| Pioneer | trailblazing, determined, visionary, courageous, independent, first-mover | follower, cautious, derivative, timid |
| Seeker | questioning, philosophical, introspective, searching, contemplative, deep | superficial, certain, unreflective, shallow |

#### Hero Family

| Sub-archetype | Valid Adjectives | Invalid Adjectives |
|---|---|---|
| Athlete | disciplined, competitive, relentless, focused, precise, driven, resilient | lazy, undisciplined, careless, quitting |
| Liberator | just, passionate, systemic, courageous, empowering, revolutionary, fierce | oppressive, complacent, passive, conformist |
| Rescuer | brave, decisive, responsive, selfless, alert, protective, crisis-ready | cowardly, hesitant, indifferent, negligent |
| Warrior | strategic, courageous, disciplined, mission-focused, resolute, fierce, loyal | cowardly, unfocused, undisciplined, disloyal |

#### Jester Family

| Sub-archetype | Valid Adjectives | Invalid Adjectives |
|---|---|---|
| Clown | physical, absurd, disarming, silly, spontaneous, warm, self-deprecating | serious, dignified, uptight, pretentious |
| Entertainer | charismatic, performative, magnetic, energetic, dazzling, dramatic, fun | boring, withdrawn, subdued, forgettable |
| Provocateur | subversive, sharp, satirical, irreverent, incisive, daring, witty | deferential, gentle, agreeable, conformist |
| Shapeshifter | adaptive, elusive, perceptive, versatile, mercurial, chameleonic, fluid | rigid, predictable, transparent, fixed |

#### Lover Family

| Sub-archetype | Valid Adjectives | Invalid Adjectives |
|---|---|---|
| Companion | loyal, steady, present, devoted, reliable, warm, attentive | fickle, absent, unreliable, cold |
| Hedonist | sensual, appreciative, indulgent, luxurious, aesthetic, celebratory, rich | ascetic, denying, spartan, puritanical |
| Matchmaker | perceptive, connective, generous, intuitive, orchestrating, social, nurturing | isolating, oblivious, selfish, divisive |
| Romantic | passionate, intense, idealistic, devoted, expressive, poetic, yearning | cold, pragmatic, detached, indifferent |

#### Magician Family

| Sub-archetype | Valid Adjectives | Invalid Adjectives |
|---|---|---|
| Alchemist | mysterious, transformative, patient, secretive, profound, intuitive, deep | superficial, transparent, hasty, mundane |
| Engineer | systematic, precise, building, methodical, logical, structural, elegant | chaotic, sloppy, intuitive, disorganized |
| Innovator | disruptive, bold, inventive, unconventional, energetic, visionary, daring | conservative, cautious, derivative, incremental |
| Scientist | empirical, rigorous, hypothesis-driven, meticulous, objective, patient, systematic | sloppy, subjective, impulsive, careless |

#### Rebel Family

| Sub-archetype | Valid Adjectives | Invalid Adjectives |
|---|---|---|
| Activist | organized, passionate, principled, relentless, mobilizing, justice-driven | apathetic, complacent, passive, unjust |
| Gambler | daring, instinctive, high-stakes, fearless, impulsive, audacious, reckless | cautious, calculating, safe, risk-averse |
| Maverick | independent, unconventional, self-directed, original, nonconformist, resourceful | conformist, obedient, predictable, dependent |
| Reformer | principled, constructive, determined, strategic, improvement-focused, bold | destructive, nihilistic, passive, aimless |

#### Sage Family

| Sub-archetype | Valid Adjectives | Invalid Adjectives |
|---|---|---|
| Detective | analytical, meticulous, persistent, methodical, observant, evidence-driven, skeptical | gullible, sloppy, inattentive, impulsive |
| Mentor | wise, patient, developmental, nurturing, experienced, guiding, generous | withholding, impatient, dismissive, selfish |
| Shaman | intuitive, liminal, mysterious, deep, unconventional, visionary, otherworldly | superficial, conventional, literal, materialistic |
| Translator | clear, synthesizing, bridging, accessible, versatile, articulate, patient | obscure, confusing, narrow, jargon-laden |

#### Sovereign Family

| Sub-archetype | Valid Adjectives | Invalid Adjectives |
|---|---|---|
| Ambassador | diplomatic, representative, tactful, persuasive, consensus-building, graceful | confrontational, divisive, tactless, rude |
| Judge | impartial, principled, evaluative, fair, discerning, measured, authoritative | biased, unprincipled, arbitrary, capricious |
| Patriarch | protective, authoritative, legacy-minded, stable, provider, traditional, firm | neglectful, weak, irresponsible, abandoning |
| Ruler | commanding, decisive, systemic, powerful, order-creating, visionary, controlling | passive, indecisive, chaotic, powerless |

### Adjective Visual Effects

Adjectives modify the avatar within the archetype's visual space:

| Adjective Type | Visual Effect |
|---|---|
| Intensity adjectives (fierce, gentle, bold, subtle) | Expression intensity — brow tension, mouth set, eye focus |
| Temperament adjectives (warm, cool, measured, passionate) | Color temperature and saturation of palette accents |
| Energy adjectives (restless, calm, driven, contemplative) | Pose dynamism — static vs. implied motion |
| Precision adjectives (meticulous, precise, methodical, sharp) | Line quality — clean edges, geometric elements |
| Organic adjectives (intuitive, flowing, natural, earthy) | Shape language — softer curves, organic textures |

---

## Section 5: Visual Avatar Generation Contract

### Input Schema

After faceted selection completes, the avatar generator receives this payload:

```json
{
  "archetype": {
    "family": "Sage",
    "subArchetype": "Detective",
    "quadrant": "Independence",
    "drive": "Knowledge, truth, understanding",
    "keyCharacteristic": "Uncovers truth through systematic investigation and evidence"
  },
  "adjectives": ["meticulous", "persistent"],
  "derivedFrameworks": {
    "mbti": "INTJ",
    "enneagram": { "type": 5, "label": "Investigator" },
    "disc": "C",
    "belbin": "Monitor Evaluator",
    "bigFive": {
      "openness": "high",
      "conscientiousness": "high",
      "extraversion": "low",
      "agreeableness": "low",
      "neuroticism": "low"
    },
    "sdi": "Green"
  },
  "canonicalAxes": {
    "socialOrientation": { "term": "independent", "weight": 1.0 },
    "ruleFollowing": { "term": "strict", "weight": 1.0 },
    "riskAppetite": { "term": "cautious", "weight": 1.0 },
    "autonomy": { "term": "autonomous", "weight": 1.0 },
    "conflictMode": { "term": "avoiding", "weight": 1.0 }
  }
}
```

### What Drives Visual Generation

| Data | Visual Role | Priority |
|---|---|---|
| `archetype.family` | Base visual template — overall character archetype aesthetic | **Primary** |
| `archetype.subArchetype` | Template variant — distinguishing visual details within the family | **Primary** |
| `adjectives` | Expression and styling refinements within the template | **Secondary** |
| `canonicalAxes` | Expression geometry — mouth curve, eye openness, brow position | **Tertiary** |
| `derivedFrameworks` | Tooltip / detail display only — NOT used for visual generation | **Display only** |

### Visual Template Guidelines

Each archetype family should have a distinct visual language:

| Family | Visual Language | Color Tendency |
|---|---|---|
| Caregiver | Warm, soft, open posture, gentle expression | Warm earth tones, soft greens |
| Everyman | Approachable, grounded, friendly, unremarkable distinction | Neutral tones, comfortable mid-range |
| Creator | Artistic, expressive, unique styling, creative elements | Bold contrasts, saturated accents |
| Innocent | Bright, clear, open eyes, youthful quality | Light palette, pure colors |
| Explorer | Dynamic, forward-leaning, windswept quality, outdoor elements | Natural earth tones, sky blues |
| Hero | Strong, determined, upright posture, direct gaze | Bold primary colors, strong contrast |
| Jester | Animated, asymmetric, playful features, unexpected elements | Bright, unexpected color combinations |
| Lover | Warm, inviting, expressive eyes, soft features | Rich warm tones, deep reds, golds |
| Magician | Mysterious, intense gaze, otherworldly quality, depth | Deep purples, midnight blues, metallic |
| Rebel | Edgy, defiant expression, sharp angles, unconventional | Dark palette with sharp accent colors |
| Sage | Contemplative, focused gaze, scholarly elements, depth | Cool blues, silver, parchment tones |
| Sovereign | Commanding, symmetrical, authoritative posture, regal | Rich golds, deep blues, formal tones |

Sub-archetypes differentiate within the family template:

| Example | Differentiation |
|---|---|
| Sage → Detective | Sharper, more scrutinizing gaze; investigative quality |
| Sage → Mentor | Warmer, more open expression; teaching quality |
| Sage → Shaman | More mysterious, liminal quality; unconventional elements |
| Sage → Translator | Clearer, more bridging expression; accessible quality |

### Canonical Axes → Expression Geometry

The 5 canonical axes provide consistent expression adjustments across all archetypes:

| Axis | Expression Mapping |
|---|---|
| `socialOrientation` | independent → more closed/self-contained expression · collaborative → more open/welcoming |
| `ruleFollowing` | strict → precise, angular features · flexible → softer, more relaxed features |
| `riskAppetite` | cautious → narrower eyes, slight tension · bold → wider eyes, confident expression |
| `autonomy` | autonomous → more self-contained posture · directed → more attentive/receptive posture |
| `conflictMode` | competing → more assertive jaw/brow · avoiding → softer, more withdrawn · accommodating → open, yielding |

These are secondary adjustments — the archetype template is the dominant visual driver.

---

## Section 6: Example Flows

### Flow 1: Start from Archetype

```
Step 1: User browses archetype families
  UI: 12 family cards grouped by 4 quadrants
  All framework values available (nothing greyed)

Step 2: User selects "Sage" family
  UI: 4 sub-archetype cards shown (Detective, Mentor, Shaman, Translator)
  Framework values narrow:
    MBTI available:    INTJ, INTP, ISTJ (others greyed)
    Enneagram:         1, 5 (others greyed)
    DISC:              C (others greyed)
    Belbin:            Monitor Evaluator, Specialist, Completer-Finisher (others greyed)
    Big Five:          highO, highC, lowE, lowN (complementary poles greyed)
    SDI:               Green (others greyed)

Step 3: User selects "Detective" sub-archetype
  UI: Archetype card expanded with full description
  Adjective selector appears with valid options:
    [analytical] [meticulous] [persistent] [methodical] [observant]
    [evidence-driven] [skeptical]
  Framework values further narrow to Detective affinity:
    MBTI suggested:    ISTJ, INTJ, ISTP
    Enneagram:         5, 6

Step 4: User selects adjectives "meticulous, persistent"
  UI: Avatar preview generates with Detective template + adjective modifiers

Step 5: User confirms
  Output: Full payload sent to avatar generator
```

### Flow 2: Start from MBTI

```
Step 1: User selects MBTI = INTJ
  UI: Compatible families highlighted: Sage, Magician, Sovereign
  Remaining archetypes: 12 (4 per family)
  Other frameworks narrow:
    Enneagram:         1, 3, 5, 8 (families overlap → union of compatible)
    DISC:              D, C
    Belbin:            Plant, Shaper, Monitor Evaluator, Co-ordinator,
                       Completer-Finisher, Specialist
    Big Five:          highO, highC, lowE, lowA, lowN
    SDI:               Red, Green

Step 2: User selects Enneagram = 5
  Families: Sage ∩ {Sage, Explorer, Magician} + INTJ ∩ {Sage, Magician, Sovereign}
  Intersection: {Sage, Magician}
  Remaining archetypes: 8
  DISC narrows:        C (the only value compatible with both Sage and Magician
                       while excluding Hero/Rebel)

Step 3: User selects DISC = C
  compatible(DISC-C) = {Sage, Sovereign, Creator, Explorer}
  Intersection with {Sage, Magician}: {Sage}
  Family converged: Sage
  Remaining sub-archetypes: 4 (Detective, Mentor, Shaman, Translator)

Step 4: Sub-archetype distinction
  INTJ affinity: Detective ✓, Translator ✓
  E5 affinity: Detective ✓, Translator ✓
  Both match → PRESENT CHOICES:
    "Detective — investigative, evidence-driven, uncovers truth systematically"
    "Translator — synthesizing, bridging, makes the complex accessible"

Step 5: User selects Detective
  Adjective selector appears
  User picks "analytical"
  Avatar generates
```

### Flow 3: Start from Belbin

```
Step 1: User selects Belbin = Plant
  compatible(Plant) = {Creator, Magician}
  Remaining archetypes: 8
    Creator:  Artist, Entrepreneur, Storyteller, Visionary
    Magician: Alchemist, Engineer, Innovator, Scientist
  Frameworks narrow:
    MBTI:      INTJ, INTP, ENTJ, ENTP, INFJ, INFP, ENFP, ISFP (union of
               Creator- and Magician-compatible)
    Enneagram: 3, 4, 5, 7 (union)
    DISC:      D, C, S (union — Creator adds S, Magician adds D)
    SDI:       Red, Green (union)

Step 2: User selects MBTI = ENTP
  compatible(ENTP) = {Magician, Rebel, Explorer}
  Intersection with {Creator, Magician}: {Magician}
  Family converged: Magician
  Remaining sub-archetypes: Alchemist, Engineer, Innovator, Scientist

Step 3: Sub-archetype distinction
  ENTP affinity: Innovator ✓ (others: INFJ, INTJ, INTP)
  → Converged: Innovator

  Adjective selector shows:
    [disruptive] [bold] [inventive] [unconventional] [energetic]
    [visionary] [daring]

Step 4: User selects "bold, inventive"
  Avatar generates with Magician/Innovator template + adjective modifiers
```

---

## Appendix: Implementation Notes

### Data Loading

The JSON structure in Section 2 is the complete compatibility dataset. Load it at
application startup. The data is static — it does not change at runtime.

### Performance

The intersection algorithm operates on small sets (max 12 families, max 60 archetypes).
All operations are O(1) lookups and O(n) intersections where n ≤ 60. No optimization
needed — compute on every selection change.

### State Management

The faceted selector state is a pure function of the selections. No history needed —
recompute available values from scratch on every selection/deselection. This makes
undo trivial: remove the selection, recompute.

### Deselection

Users can deselect a previously chosen value. The system recomputes from the remaining
selections, widening the available space. Previously greyed values may become available
again.

### Mobile Considerations

On small screens, consider a step-by-step wizard flow instead of showing all dimensions
simultaneously. The algorithm is the same — just present one dimension at a time with
remaining candidate count shown.
