# Dominion-AntiDup622

A fork of Dominion 4.8.4 that closes the end portal duplication bug on Paper servers.

It is not meant to be a new plugin, just a patch: the plugin is renamed to `Dominion-AntiDup622` so you can tell it apart from upstream, the data folder is still `plugins/Dominion/`, and the config is the same one. On an existing server you can just swap the jar.

## What the bug actually is

When a player inserts the last eye of ender, Paper does things in this order:

1. It writes the portal blocks into the middle 3x3 — whatever fell out of the container is already lying on the ground at that point;
2. Only then does it fire `BlockMultiPlaceEvent`.

Dominion checks the claim on step 2. If it cancels the event, Paper rolls the blocks back — but the drops are still on the floor and the container contents come back with it. Put a shulker box in, pop in an eye of ender, and you have two of everything.

Paper considers that ordering intentional and there is no API that lets you abort the placement before the blocks get swapped. So blocking it "at the moment of activation" is a dead end.

## What the patch does

It moves the check to `BlockPlaceEvent`:

- Inside a complete but **unactivated** end portal (12 frames, all facing inward), nothing can be placed in the middle 3x3. Cancel the event and Paper never replaces the blocks, so there are no drops and nothing to duplicate.
- An activated portal (there is an `END_PORTAL` block in the 3x3) is let through as before — sand duper style machines keep working.
- A cancelled place does not consume the item; the player just gets a message.
- `BlockMultiPlaceEvent` is explicitly excluded from the claim check, so portal creation itself is never cancelled.
- If one player gets blocked 3 times inside 10 seconds, they cannot place anything around that portal for the next 10 seconds (the frame plus the ring of blocks touching it). Keeps people from sitting there clicking.
- Five seconds after start-up the plugin sweeps loaded chunks, and keeps sweeping every chunk that loads, to clear blocks that older builds of this bug left inside unactivated portals. Container contents are dropped first, then the block is removed.

## Differences from upstream

- Plugin name is `Dominion-AntiDup622`; main class, command, permission nodes and PlaceholderAPI identifiers are unchanged.
- The data folder is still `plugins/Dominion/`. The plugin points the data folder back at that path when it enables, so existing configs and databases do not need to move.
- One extra config section, `end-portal-protection`. The config version stays at upstream's `3`, so an old `config.yml` only gains the missing section instead of being regenerated.
- Everything else is upstream 4.8.4.

## Configuration

`plugins/Dominion/config.yml`:

```yaml
end-portal-protection:
  # master switch
  enable: true
  # clean up blocks left inside unactivated portals
  clean-existing-structures: true
  # how long the placement restriction lasts after a player gets blocked repeatedly
  cooldown-seconds: 10
```

Messages live under `end-portal-protection-handler-text` in the language files; all four bundled languages (en_us, zh_cn, zh_tw, jp_jp) have entries.

## Install

1. Back up `plugins/Dominion/` first.
2. Remove the old Dominion jar and drop this one in.
3. Run Java 21 or newer. The full build contains the 1.21.9 modules, so Java 17 fails with `UnsupportedClassVersionError` — the upstream full build behaves the same way, it is not caused by this patch.

## Building from source

The `api/` directory contains a snapshot of DominionAPI 4.8.4 (commit `f40955f0`), so you do not need to fetch submodules.

```
gradlew shadowJar -PBuildFull=true
```

JDK 17 and JDK 25 are required (the `v26` module needs 25). If Gradle cannot find them, point `org.gradle.java.installations.paths` in `gradle.properties` at your installations.

## Known limitations

- Only the standard structure is detected: 12 frames, correct facings, empty corners. Weird portals are ignored — they usually cannot be activated either.
- Only tested on Paper 1.20.1. Newer versions should fire the same events in the same order, but I have not run them.
- The in-game feel (messages, the cooldown after repeated attempts) cannot be automated; only the server side detection logic is covered by tests. If something is off, open an issue with the server version and the steps to reproduce.

## License

MIT, same as upstream. Dominion is a LunaDeerMC project; this fork only changes the parts described above.