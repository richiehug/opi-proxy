# Offline Image Archive

The public distribution repo does not store the Docker image tar in git.
GitHub rejects large binary files above the repository file size limit, and the release image can exceed that limit.

Primary install path:

- pull the versioned image from GHCR with `docker compose pull`

Offline path:

- download the matching public release asset, for example `opi-proxy-1.0.0-linux-arm64.tar` on Raspberry Pi
- place it in `image/` and load it with `docker load -i image/<archive-name>.tar`