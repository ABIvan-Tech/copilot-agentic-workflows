# Ultralight Orchestration

A minimal multi-agent system with a main ageint (All Three), a planner, a coder, and a designer working together. The Three agent orchestrates all work by delegating to specialized subagents.

You must install all agents in this collection for proper functionality.

## Items in this Collection

| Title | Type | Description |
| ----- | ---- | ----------- |
| [Three](https://gist.github.com/burkeholland/0e68481f96e94bbb98134fa6efd00436#file-three-agent-md)<br />[![Install in VS Code](https://img.shields.io/badge/VS_Code-Install-0098FF?style=flat-square&logo=visualstudiocode&logoColor=white)](https://aka.ms/awesome-copilot/install/agent?url=vscode%3Achat-agent%2Finstall%3Furl%3Dhttps%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fthree.agent.md)<br />[![Install in VS Code Insiders](https://img.shields.io/badge/VS_Code_Insiders-Install-24bfa5?style=flat-square&logo=visualstudiocode&logoColor=white)](https://aka.ms/awesome-copilot/install/agent?url=vscode-insiders%3Achat-agent%2Finstall%3Furl%3Dhttps%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fthree.agent.md) | Agent | Architect agent that orchestrates work through subagents (Sonnet, Codex, Gemini) |
| [Planner](https://gist.github.com/burkeholland/0e68481f96e94bbb98134fa6efd00436#file-planner-agent-md)<br />[![Install in VS Code](https://img.shields.io/badge/VS_Code-Install-0098FF?style=flat-square&logo=visualstudiocode&logoColor=white)](https://aka.ms/awesome-copilot/install/agent?url=vscode%3Achat-agent%2Finstall%3Furl%3Dhttps%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fplanner.agent.md)<br />[![Install in VS Code Insiders](https://img.shields.io/badge/VS_Code_Insiders-Install-24bfa5?style=flat-square&logo=visualstudiocode&logoColor=white)](https://aka.ms/awesome-copilot/install/agent?url=vscode-insiders%3Achat-agent%2Finstall%3Furl%3Dhttps%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fplanner.agent.md) | Agent | Creates detailed implementation plans by researching the codebase and consulting documentation |
| [Coder](https://gist.github.com/burkeholland/0e68481f96e94bbb98134fa6efd00436#file-coder-agent-md)<br />[![Install in VS Code](https://img.shields.io/badge/VS_Code-Install-0098FF?style=flat-square&logo=visualstudiocode&logoColor=white)](https://aka.ms/awesome-copilot/install/agent?url=vscode%3Achat-agent%2Finstall%3Furl%3Dhttps%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fcoder.agent.md)<br />[![Install in VS Code Insiders](https://img.shields.io/badge/VS_Code_Insiders-Install-24bfa5?style=flat-square&logo=visualstudiocode&logoColor=white)](https://aka.ms/awesome-copilot/install/agent?url=vscode-insiders%3Achat-agent%2Finstall%3Furl%3Dhttps%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fcoder.agent.md) | Agent | Writes code following mandatory coding principles (GPT-5.2-Codex) |
| [Designer](https://gist.github.com/burkeholland/0e68481f96e94bbb98134fa6efd00436#file-designer-agent-md)<br />[![Install in VS Code](https://img.shields.io/badge/VS_Code-Install-0098FF?style=flat-square&logo=visualstudiocode&logoColor=white)](https://aka.ms/awesome-copilot/install/agent?url=vscode%3Achat-agent%2Finstall%3Furl%3Dhttps%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fdesigner.agent.md)<br />[![Install in VS Code Insiders](https://img.shields.io/badge/VS_Code_Insiders-Install-24bfa5?style=flat-square&logo=visualstudiocode&logoColor=white)](https://aka.ms/awesome-copilot/install/agent?url=vscode-insiders%3Achat-agent%2Finstall%3Furl%3Dhttps%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fdesigner.agent.md) | Agent | Handles all UI/UX and design tasks (Gemini 3 Pro) |

## Usage

### All Three (Sonnet 4.5)

The orchestrator agent that receives requests and delegates work. It:
- Analyzes requests and gathers context
- Delegates planning to the Planner agent
- Delegates code implementation to the Coder agent
- Delegates UI/UX work to the Designer agent
- Integrates results and validates final output

### Planner (GPT-5.2)

Creates comprehensive implementation plans by researching the codebase, consulting documentation, and identifying edge cases. Use when you need a detailed plan before implementing a feature or fixing a complex issue.

### Coder (GPT-5.2-Codex)

Writes code following mandatory principles including structure, architecture, naming conventions, error handling, and regenerability. Always uses context7 MCP Server for documentation.

### Designer (Gemini 3 Pro)

Focuses on creating the best possible user experience and interface designs with emphasis on usability, accessibility, and aesthetics.
