# entservices-maintenancemanager

Following plugins are moved to dedicated repositories as per below table:

| Plugin | Repository |
| --- | --- |
| firmwaredownload | https://github.com/rdkcentral/entservices-firmwaredownload |
| firmwareupdate | https://github.com/rdkcentral/entservices-firmwareupdate |

Any further code changes need to come from the above repositories on the `develop` branch.
Ongoing release changes for `8.0`, `8.1`, `8.2`, `8.3`, and `8.4` branches should still use this repository.

Default development branch: develop.

## Plugin Documentation

- MaintenanceManager: [MaintenanceManager/README.md](MaintenanceManager/README.md)
- Packager: [Packager/README.md](Packager/README.md)

## Agents and Skills

- Detailed usage guide: [.github/agents/README.md](.github/agents/README.md)
- Skills documentation: [.github/skills/README.md](.github/skills/README.md)
- OpenSpec directory guide: [openspec/README.md](openspec/README.md)

### Custom Agents

- Maintenance Architect Agent: [.github/agents/maintenance-architect.agent.md](.github/agents/maintenance-architect.agent.md)
- Maintenance Code Implementer: [.github/agents/maintenance-code-implementer.agent.md](.github/agents/maintenance-code-implementer.agent.md)

### Custom Skills

- diagnose: [.github/skills/diagnose/SKILL.md](.github/skills/diagnose/SKILL.md)
- grill-me-with-docs: [.github/skills/grill-me-with-docs/SKILL.md](.github/skills/grill-me-with-docs/SKILL.md)
