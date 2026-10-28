# claude-cloud-memory

A Claude Code plugin marketplace with one plugin, [`axon-memory`](plugins/axon-memory/README.md). It lets cloud Claude sessions (Claude Code on the web, the desktop app's cloud sessions and project threads) recall from and retain into a self-hosted Hindsight memory node through a public HTTPS door. The key stays outside the VM as a network secret.

The plugin does nothing unless the cloud environment sets `AXON_DOMAIN=personal`, an `https://` `AXON_MEMORY_URL` and `AXON_MEMORY_BANK`. Setup is in the [plugin's README](plugins/axon-memory/README.md#set-it-up).

This repository is temporary. The plugin is a copy of the one in [axon](https://github.com/LorDHaZarD/axon), and this repository will be archived once axon publishes its own marketplace.
