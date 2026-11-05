# Modular Expressive Animation for LLM-Driven NPCs

**Independent Study, Fall 2026 · Pallavi Srinivas**

I want to see what happens when an LLM-driven NPC's internal state shows up in its body, and whether that movement actually changes an audience's belief in the character.

This repository contains **Kayla**, an autonomous NPC built in Unreal Engine 5 with Blueprints. Kayla's decisions come from a large language model (Claude), and her body expresses those decisions through layered animation: her arms, legs, and movement are chosen separately, so she can drink while running, think while standing still, or wander off to look at something that caught her eye.

---

## Research questions

- How do actions, expressions, and other visual aspects impact an audience's believability in an NPC?
- How can an LLM-driven NPC's internal state be translated into coordinated, expressive body movements using modular animations?

Right now the LLM drives the NPC to make logical, informed decisions. To make the game world more believable, I want animations that adapt to the NPC's emotions, needs, surroundings, and awareness. The goal is to explore what makes LLM-driven NPCs more engaging, believable, and perceptibly intelligent.

---

## Background: previous work (Independent Study: Spring 2026)

Last spring, my goal was to develop an NPC that could come up with philosophical thoughts, reason about her own needs, recognize objects in her environment, and make thoughtful decisions based on what she saw. The NPC moved to an actor location depending on which state the LLM decided she should enter.

That removed the need for a mechanically controlled NPC, and the choices made were intelligent and correct. However, all of her decisions played out as moves inside a state machine: the LLM picked one state, and a Behavior Tree ran a fixed routine for it (walk to the station, play one full-body animation, walk back home). **She could think, but she couldn't show what she was feeling.**

The architecture diagrams for both versions are on [Figma](https://www.figma.com/board/KmBNkNpDsjQy4lyBqSFI9k/Game-Assemblies-Lab--System-Architecture?node-id=0-1&t=QWCxtFbDbG4vNxzV-1).

---

## Inspiration

After my SWE internship at Halo Studios this summer, I became even more interested in AI-powered NPCs. I had the chance to talk with Brie Chin-Deyerle, one of the main developers of the bots in Halo Infinite. She described how the team had to recalibrate how characters interacted and made decisions so they'd be more believable to the player, and what she sees as the caveats of using this new technology.

That pushed me toward building a character that uses visuals to show what she might be feeling. Other references I've looked at include Halo Infinite, the Forza Horizon series, and research on Counter-Strike bots.

### The gap in current AI systems

Traditional game AI uses discrete state machines or full-body clip triggers. When an LLM is added on top, its rich, high-level output usually still gets squeezed into one rigid animation per decision, so the body never matches the nuance of the thought.

### The opportunity

Shift from selecting one full-body animation to **modular, multi-layered body-part control**. Instead of the LLM choosing a single clip, the LLM's decision drives independent skeletal regions at the same time.

### Abstractions of human expression

I'm particularly excited about demonstrating the complexities of the human expression. Each part of the body can carry its own "thought": an arm might still be holding the last idea while the torso shifts into a new one and the legs settle into something else. My Mixamo-based demo tests this pipeline with upper- and lower-body layers. The long-term goal is to blend different "thought processes" across the whole body, so the character reads as one person without feeling choppy or state-machine-like.

---

## Meet Kayla

Kayla is a shopaholic born and raised in Beverly Hills. She loves her life and lounging around, but her true calling is being an FBI agent. She sounds a little silly, but she is actually very thoughtful and philosophical. Her personality lives in the server's prompt, so every decision she makes and every thought she has stays in character. For example:

> *"Okay, the horse is just standing there with those big eyes, like it saw everything and is too cool to say so. Honestly, that's the most reliable witness on this whole street."*

---

## How it works

Every ~10 seconds, Kayla makes a new decision. One decision flows through the system like this:

```
BTTask_Vision → Flask server → Claude → Flask server → AutoGenFunc → Behavior Tree + Animation Blueprint
     ↑                                                                                      │
     └──────────────────────────────── repeats every ~10 seconds ───────────────────────────┘
```


### 1. Vision: `BTTask_Vision`

A Behavior Tree task that registers tagged objects in Kayla's surroundings (for example `Fountain`, `Food`, `Bed`, `Horse`, `Plant`) and stores them in the `VisibleObjects` Blackboard key. These are the things she can notice, need, or be curious about.

### 2. Sending her state: `QueryLLM`

A function in the AI Controller (`AIC_Enemy_Base`) that reads her current needs (Thirst, Hunger, Sleep) and visible objects from the Blackboard and sends them as plain text to the local Flask server, for example:

```
Current state: Thirst=4.2, Hunger=70.0, Sleep=55.0.
Nearby objects: Plant, Fountain, Bed, Food, Horse
```

It skips sending a request while Kayla is asleep, so a new decision can't wake her up mid-sleep.

### 3. The translator: Flask server (`server.py`)

A small Python server on port 5000 that sits between Unreal and Claude. It:

- **Turns numbers into words.** `Thirst=4` becomes "desperately thirsty" and `Hunger=70` becomes "full". Earlier versions sent raw numbers, and Claude sometimes misread which direction they went (for example treating Hunger 105 as "starving" when it actually meant full).
- **Adds her personality and the rules** for how to reply.
- **Adds memory** of her last five thoughts and actions, so she doesn't repeat herself.
- **Calls Claude** through the Anthropic API.
- **Cleans up the reply** so Unreal always receives a valid answer: unknown words become safe defaults, `Fountain` is mapped to `Water`, stray labels are removed, and so on.
- **Logs every decision** to `decisions.csv` for later analysis.

### 4. The decision: Claude

Claude replies with one line containing five parts, one per "channel" of Kayla's body:

```
MOVE | TARGET | UPPER | LOWER | THOUGHT
Go   | Horse  | Think | Idle  | Okay, the horse is just standing there...
```

| Channel | Controls | Options |
|---|---|---|
| **MOVE** | Whether and how she travels | Stay, Wander (slow stroll to a random nearby spot), Go (runs to a target) |
| **TARGET** | Where she goes | Water, Food, Bed, Home, or any tagged object she can see |
| **UPPER** | What her arms do (works even while running) | None, Drink, Eat, Think |
| **LOWER** | Her leg stance once she stops | Idle, Drink, Eat, Think |
| **THOUGHT** | What she's thinking | One or two in-character sentences |

Because the channels are independent, Claude can combine them freely, for example thinking with her arms while her legs settle into a different stance, or drinking while running to the next place.

### 5. Acting it out: `AutoGenFunc`

A function in the AI Controller that receives the server's reply, splits it on `|`, and acts on it:

- Sets **`MoveMode`** and the running or wandering speed.
- Sets **`LowerPose`** for the legs.
- Plays the **upper-body montage** (`AM_Drink`, `AM_Eat`, `AM_Think`) on the character.
- **Resets the matching need** to 100 when she drinks or eats.
- Looks up **`TargetLocation`**: a fixed location for Water, Food, Bed, and Home, an actor found by tag for objects, or a random reachable point for Wander.
- Sets **`PendingAction = Sleep`** when she heads to bed.
- Prints her **thought** on screen.

### 6. Moving her legs: Behavior Tree (`BT_Enemy_Base`)

The Behavior Tree no longer decides what Kayla does. It only handles physical movement:

| Branch | Runs when | What it does |
|---|---|---|
| **Move** | `MoveMode` is not None | Stands up, runs to `TargetLocation`, takes the chosen lower-body pose on arrival |
| **Sleep** | `PendingAction` is Sleep | Walks to the bed, lies down for a set time, resets Sleep, then clears the flag |
| **Stay** | Fallback | Takes the chosen lower-body pose and waits |

### 7. Layered animation: `ABP_Mixamo`

The Animation Blueprint combines three layers into the final pose:

1. **State machine** (Idle, Running, Sleep): handles walking, running, and lying down, based on her speed and the `DesiredAction` Blackboard key.
2. **Leg layer** (`LowerGroup.LowerBody` slot): a Layered Blend Per Bone filtered to `LeftUpLeg` and `RightUpLeg` replaces the legs with a looping lower-body montage (`AM_Drink_Lower`, `AM_Eat_Lower`, `AM_Think_Lower`).
3. **Arm layer** (`DefaultGroup.UpperBody` slot): a Layered Blend Per Bone split at `Spine1` replaces the torso and arms with an upper-body montage, while the legs keep doing whatever they were doing.

This is what lets one animation play on her arms while another plays on her legs.

### 8. Needs

Thirst, Hunger, and Sleep are stored in the Blackboard and drop over time in `UpdateNeeds` (a random amount each update, clamped between 0 and 100). A low number means urgent. Drinking, eating, or sleeping resets the matching need to 100.

---

## Version 1 vs version 2

| | Version 1 (Spring 2026) | Version 2 (Fall 2026) |
|---|---|---|
| Claude's reply | `ACTION \| THOUGHT` | `MOVE \| TARGET \| UPPER \| LOWER \| THOUGHT` |
| Who decides | The Behavior Tree, which picks a branch for the action | Claude composes the whole body |
| AutoGenFunc | Only parses the reply | Parses and acts: animations, needs, speed, target |
| Behavior Tree | Four branches (Hunger, Thirst, Sleep, Wander), each a fixed routine | Three branches (Move, Sleep, Stay) that only move her legs |
| Animation | One full-body clip per state | Layered: arms and legs animated separately |
| Afterwards | Always walked back home | Goes wherever she decides next |
| Destinations | Four fixed stations | Stations plus any tagged object she's curious about |

### Problems solved along the way

- **Gliding.** In version 1, the action animation started as soon as Claude replied, so Kayla slid across the floor in the drink pose. Fixed by separating `PendingAction` ("what she's about to do") from `DesiredAction` ("what she's doing right now").
- **Misread needs.** Claude sometimes interpreted the raw numbers backwards. Fixed by having the server describe needs in words.
- **Repetition.** Kayla kept choosing the same action or thought. Fixed by giving Claude memory of her recent thoughts and actions.
- **Waking up mid-sleep.** New decisions cleared the sleep animation. Fixed by skipping requests while she sleeps and letting the Sleep branch clear the flag itself.
- **Needs never returning.** A sign error in `UpdateNeeds` made needs climb past 100, so she never got needy again. Fixed with subtraction and a 0–100 clamp.

---

## Evaluation: LLM vs rules

To measure what the LLM actually adds, the server has a **rules-only baseline**. Setting `MODE = "rules"` in `server.py` replaces Claude with simple hand-written logic: go to whatever need is most urgent, otherwise wander. Unreal doesn't know the difference, so both modes run through exactly the same pipeline.

Every decision in both modes is logged to `decisions.csv`. Running `analyze.py` summarizes each mode:

- **Different movement + arm combinations:** how varied her behavior is
- **Urgent need handled:** how often she dealt with her most urgent need
- **Same choice twice in a row:** how repetitive she is
- **Curiosity visits:** how often she went to an object for no need-related reason

*(Add final results here after the LLM and rules runs.)*

---

## Running the project

### Requirements
- Unreal Engine 5
- Python 3 with Flask and the Anthropic SDK: `pip install flask anthropic`
- An Anthropic API key

### Steps
1. Open `server.py` and replace `KEY` with your API key.
2. Start the server from the project folder:
   ```
   python server.py
   ```
   Wait for `Running on http://127.0.0.1:5000`.
3. Open the Unreal project and press **Play**. New decisions appear in the server terminal every ~10 seconds.
4. After a session, summarize the log:
   ```
   python analyze.py
   ```

### Configuration (top of `server.py`)

| Setting | Meaning |
|---|---|
| `MODEL` | The Claude model used (default `claude-haiku-5-5`, the fastest and cheapest) |
| `MODE` | `"llm"` for Claude, `"rules"` for the hand-written baseline |
| `UPPERS`, `LOWERS` | The animation names Claude may choose; each must match a montage in Unreal |
| `ALIASES` | Maps object names to places or tags, e.g. `"Fountain": "Water"` |
| `URGENT_WHEN_LOW` | Whether each need counts as urgent when its number is low |

---

## Files

| File / asset | Purpose |
|---|---|
| `server.py` | Flask server: builds the prompt, calls Claude, cleans the reply, logs decisions, rules baseline |
| `analyze.py` | Summarizes `decisions.csv` for each mode |
| `decisions.csv` | Created automatically: one row per decision (time, mode, model, needs, objects, five choices, thought) |
| `AIC_Enemy_Base` | AI Controller: `QueryLLM`, `AutoGenFunc`, `UpdateNeeds` |
| `BT_Enemy_Base` / `BB_Enemy_Base` | Behavior Tree and Blackboard |
| `BTTask_Vision`, `BTTask_SetName`, `BTTask_PrintString`, `BTTask_LowerPose` | Custom Behavior Tree tasks |
| `ABP_Mixamo` | Animation Blueprint with the layered upper/lower-body setup |
| `AM_Drink`, `AM_Eat`, `AM_Think` | Upper-body montages |
| `AM_Drink_Lower`, `AM_Eat_Lower`, `AM_Think_Lower` | Looping lower-body montages |
| `BP_Enemy_Base` | Kayla's character Blueprint |

---

## Limitations

- The animations are Mixamo clips, so the lower-body stances from drinking, eating, and thinking look fairly similar to standing idle.
- Kayla has a small set of gestures, so once her needs are met, her arm choices are limited.
- With only a few tagged objects in the level, her curiosity tends to rotate between the same ones.
- Each decision depends on a network call to the API, so there is a short delay between decisions.

---

## Next steps

- **Mocap primitives.** Record atomic movement primitives with a dedicated mocap performer and recombine them in Unreal, replacing the Mixamo clips. I've already recruited my MoCap muse.
- **Emotion layer.** Blend emotional states (tired, anxious, happy) over the whole body with an adjustable intensity.
- **Per-region "thought processes."** Let each body region hold its own state, so an arm can still carry the last thought while the torso and legs move on.
- **More expressive range.** More gestures and distinct lower-body poses (sitting, kneeling, leaning).
- **Audience study.** Show viewers different versions and measure how believable they find Kayla, to answer the first research question directly.
- **Broader game-AI knowledge.** Continue learning traditional game AI architectures alongside LLM-driven approaches.
