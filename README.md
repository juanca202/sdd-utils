# Utils

![version](https://img.shields.io/badge/version-1.0.0-blue) ![license](https://img.shields.io/badge/license-MIT-green)

**Utils** es un plugin de skills de utilidad general para agentes de IA. Incluye:

- [alm-install](skills/alm-install/SKILL.md) — instala, configura o repara el MCP local de Azure DevOps o Jira en Claude Code / Cursor.
- [project-create](skills/project-create/SKILL.md) — crea un proyecto nuevo fusionando una plantilla del equipo según el stack (Angular, Next.js).
- [prompt-validate](skills/prompt-validate/SKILL.md) — valida prompts dirigidos a agentes de IA contra reglas de redacción efectiva.

## Instalación

Instala el plugin completo, no skills sueltos. Es compatible con cualquier agente que soporte el estándar abierto [Agent Plugins](https://agent-plugins.org/), entre ellos:

**Cursor:**

```
/add-plugin https://github.com/juanca202/sdd-utils
```

**VS Code:**

Paleta de comandos (`Ctrl/Cmd+Shift+P`) → `Chat: Install Plugin From Source` → pega `https://github.com/juanca202/sdd-utils`

**Kiro:**

Panel de Powers → `Add Custom Power` → `Import power from GitHub` → pega `https://github.com/juanca202/sdd-utils`

**Claude Code** (tiene su propio sistema de plugins, no usa el estándar Agent Plugins):

```
/plugin marketplace add juanca202/sdd-devkit
/plugin install utils@juanca202
```
