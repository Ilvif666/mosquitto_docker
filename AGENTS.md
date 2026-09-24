# mosquitto_docker guide

- Each `mosquitto_<index>` is a broker assigned to one cabinet index and is shared persistent infrastructure.
- Active native mapping is defined by `/etc/smartward/instances/<instance>.env` (`BROKER_PORT`). Do not delete a broker merely because its backend is currently stopped.
- Storage-only cabinets may stop their broker through Docker Manager; their broker data and compose entry must remain recoverable.
- Preserve passwords, retained messages/data volumes, logs, and `sw_net`. Do not globally restart brokers while testing an individual cabinet.
