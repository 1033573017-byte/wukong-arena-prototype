# Wukong Arena — Combat Prototype

A playable browser souls-like combat prototype inspired by Black Myth: Wukong.

## Core loop
Stamina-based action combat. Attacks and dodges drain stamina; a perfectly-timed parry refunds stamina and opens a combo window.

## First deliverable
One arena, one boss, the full stamina–parry–combo loop, playable in the browser.

## Architecture
- TypeScript + Vite, Canvas 2D rendering, fixed 60Hz timestep
- Separate state machines: player FSM and boss AI FSM with telegraphed attack strings
- Frame-data driven combat: every move defined in JSON (startup / active / recovery frames, stamina cost, parry window)
- Stamina manager: drain on attacks/dodges, refund on perfect parries, combo window timer
- Input buffering so parries feel fair
- Built in slices with a Devin Bot: one PR per system (core loop, stamina, boss AI, tuning passes), deployed as a static site

## Roadmap
New bosses, weapons and levels built on top of the core loop, documented as a devlog series.
