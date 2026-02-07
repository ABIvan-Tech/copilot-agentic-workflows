# Ultralight Orchestration

A minimal multi-agent system with an architect (Three), a coder, and a designer working together. The Three agent orchestrates all work by delegating to specialized subagents.

## Install Collection

[![Install in VS Code](https://img.shields.io/badge/VS_Code-Install_Collection-0098FF?style=flat-square&logo=visualstudiocode&logoColor=white)](vscode:chat-collection/install?url=https%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fultralight-orchestration.collection.yml)
[![Install in VS Code Insiders](https://img.shields.io/badge/VS_Code_Insiders-Install_Collection-24bfa5?style=flat-square&logo=visualstudiocode&logoColor=white)](vscode-insiders:chat-collection/install?url=https%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fultralight-orchestration.collection.yml)

## Agents

| Agent | Description | Install |
|-------|-------------|---------|
| [All Three](https://gist.github.com/burkeholland/0e68481f96e94bbb98134fa6efd00436#file-three-agent-md)<br />Agent that orchestrates work through subagents. | Sonnet, Codex, Gemini | [![Install in VS Code](https://img.shields.io/badge/VS_Code-Install-0098FF?style=flat-square&logo=visualstudiocode&logoColor=white)](vscode:chat-agent/install?url=https%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fthree.agent.md) [![Install in VS Code Insiders](https://img.shields.io/badge/VS_Code_Insiders-Install-24bfa5?style=flat-square&logo=visualstudiocode&logoColor=white)](vscode-insiders:chat-agent/install?url=https%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fthree.agent.md) |
| [Planner](https://gist.github.com/burkeholland/0e68481f96e94bbb98134fa6efd00436#file-three-agent-md)<br />Planning agent that generates comprehensive plans with steps. | Sonnet, Codex, Gemini | [![Install in VS Code](https://img.shields.io/badge/VS_Code-Install-0098FF?style=flat-square&logo=visualstudiocode&logoColor=white)](vscode:chat-agent/install?url=https%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fthree.agent.md) [![Install in VS Code Insiders](https://img.shields.io/badge/VS_Code_Insiders-Install-24bfa5?style=flat-square&logo=visualstudiocode&logoColor=white)](vscode-insiders:chat-agent/install?url=https%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fthree.agent.md) |
| [Coder](https://gist.github.com/burkeholland/0e68481f96e94bbb98134fa6efd00436#file-coder-agent-md)<br />Writes code following mandatory coding principles. | GPT-5.2-Codex | [![Install in VS Code](https://img.shields.io/badge/VS_Code-Install-0098FF?style=flat-square&logo=visualstudiocode&logoColor=white)](vscode:chat-agent/install?url=https%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fcoder.agent.md) [![Install in VS Code Insiders](https://img.shields.io/badge/VS_Code_Insiders-Install-24bfa5?style=flat-square&logo=visualstudiocode&logoColor=white)](vscode-insiders:chat-agent/install?url=https%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fcoder.agent.md) |
| [Designer](https://gist.github.com/burkeholland/0e68481f96e94bbb98134fa6efd00436#file-designer-agent-md)<br />Handles all UI/UX and design tasks. | Gemini 3 Pro | [![Install in VS Code](https://img.shields.io/badge/VS_Code-Install-0098FF?style=flat-square&logo=visualstudiocode&logoColor=white)](vscode:chat-agent/install?url=https%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fdesigner.agent.md) [![Install in VS Code Insiders](https://img.shields.io/badge/VS_Code_Insiders-Install-24bfa5?style=flat-square&logo=visualstudiocode&logoColor=white)](vscode-insiders:chat-agent/install?url=https%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fdesigner.agent.md) |

## How It Works

1. **Three** (the architect) receives your request and analyzes it
2. Plans are delegated to the **Planner** agent
3. Code implementation goes to the **Coder** agent  
4. UI/UX and design work goes to the **Designer** agent
5. Three integrates results and validates the final output

All work happens on feature branches - never on main.
