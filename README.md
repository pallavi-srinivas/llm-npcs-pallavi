# Modular Expressive Animation for LLM-Driven NPCs

Independent Study, Fall 2026. Pallavi Srinivas.

I want to see what happens when an LLM-driven NPC’s internal state shows up in its body, and whether that movement actually changes an audience’s believability in the character.

## Previous Research + Work

Last spring, as a researcher, my goal was to develop an NPC that could concoct philosophical thoughts and intelligently deduce its needs, and eventually recognize objects in its environment and make thoughtful decisions based on what it saw. The NPC moves to an actor location depending on the state the LLM has thought to enter, based on those philosophical thoughts.

That eliminated the need for a first-person POV to control the NPC, and the choices it made were intelligent and correct. But there was still no way for the NPC to gauge exactly what was in the room around it, and all of its decisions played out as moves inside a state machine. It could think. It couldn’t show me what it was feeling.

The system architecture from that work is on [Figma](https://www.figma.com/board/KmBNkNpDsjQy4lyBqSFI9k/Game-Assemblies-Lab--System-Architecture?node-id=0-1&t=QWCxtFbDbG4vNxzV-1).

## Project Pitch

After my SWE internship at Halo Studios this summer, I got even more intrigued by AI-powered NPCs. I had the chance to talk with Brie Chin-Deyerle, one of the main developers of the bots in Halo Infinite. She described how the team had to recalibrate how characters interacted and made decisions so they’d be more believable to the player, and what she sees as the caveats of using this new technology.

That pushed me toward building a dynamic character that uses visuals to display the emotions it might be feeling. Upon receiving the LLM prompts, I want my NPC to move in an abstract, dance-like fashion, with emotions that blend together. That should make the way it moves feel more natural, and it gives me a way to observe how movement affects an audience’s perception of the NPC.

Some examples I’ve been looking at: Halo Infinite, the Forza Horizon series, and research on Counter-Strike.

## Research Question

- How do actions, expressions, and other visual aspects impact an audience’s believability in an NPC?
- How can an LLM-driven NPC's internal state be translated into coordinated, expressive body movements using modular mocap animations?

Right now the LLM drives the NPC to make logical, informed decisions. To make the game world more believable, I want animations that dynamically adapt to the NPC’s emotions, environmental factors, awareness, surroundings, and more. With this final deliverable, I hope to explore what makes LLM-driven NPCs more engaging, believable, and perceptibly intelligent.

## Pitch + Mood Board

**The Gap in Current AI Systems.** Traditional game AI uses discrete state machines or full-body clip triggers. High-level LLM outputs often lack fluid, expressive body language.

**Core Opportunity.** Shift from selecting rigid full-body animations to modular, multi-layered body part controllers. Instead of the LLM choosing a single static animation clip, the LLM state drives independent skeletal regions simultaneously.

**Mocap Integration.** Record atomic movement primitives with a dedicated mocap performer, then recombine them dynamically in Unreal Engine.

## Abstractions of Human Expression

This is the part I’m most excited about. My experiment addresses the abstractions of human expression, with different limbs experiencing different “emotions.” Each skeletal region can carry its own consciousness: an arm might still be holding the last thought while the torso shifts into a new state and the legs settle into something else.

In my demo so far, I used Mixamo animations to depict different states. That helped me test the pipeline, but it’s still one clip per state. What I’m after is blending different “thought processes” across the body, so the character reads as one person without feeling choppy or state machine-like.

## Technical Goals

- Learn to use mocap to capture someone’s movements and generate animations as quickly as possible. I’ve already recruited my MoCap muse.
- Research the best way to employ these animations within the existing environment.
- Learn to blend animations and different “thought processes” without them looking choppy or state machine-like.
- Expand my knowledge of traditional game AI architectures, so I pick up skills I can apply at a larger game studio.
