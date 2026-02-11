# Project Overview

This repository contains the **LIMA Mudlib**, a library/framework for running text MUDs on the **FluffOS** driver.

## What it is

- A modular LPC mudlib focused on natural-language command parsing, in-game usability features, and admin tooling.
- Distributed with scripts to rebuild the bundled FluffOS submodule and launch the game server.

## Key repository areas

- `lib/`: core mudlib content (commands, daemons, secure objects, standard inheritables, domains, etc.).
- `adm/dist/`: operational scripts (`rebuild`, `run`) and runtime configuration assets for compiling/running FluffOS with this mudlib.
- `bin/`: prebuilt helper binaries (with recommendation to rebuild locally).
- `resources/`: historical/auxiliary assets.

## Notable capabilities (from project docs)

- Centralized natural-language parsing (Zork-like commands).
- Featureful wizard shells (globbing, piping/redirection).
- Intermud-3 support.
- Menu-driven administrative tools.
- Emphasis on player UX (channels, menus, news, socials).

## How to run (high level)

From the repository root:

1. `cd adm/dist`
2. `./rebuild`
3. `./run`

The rebuild script updates/init FluffOS submodules, compiles the driver, installs binaries/headers, and updates runtime paths in `config.mud`.
