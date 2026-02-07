# Ultralight Orchestration

A minimal multi-agent system with a main entry point (All Three), a planner, a coder, and a designer working together. The Three agent orchestrates all work by delegating to specialized subagents using all three frontier large language models.

## Install Collection

[![Install in VS Code](https://img.shields.io/badge/VS_Code-Install_Collection-0098FF?style=flat-square&logo=visualstudiocode&logoColor=white)](vscode:chat-collection/install?url=https%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fultralight-orchestration.collection.yml)
[![Install in VS Code Insiders](https://img.shields.io/badge/VS_Code_Insiders-Install_Collection-24bfa5?style=flat-square&logo=visualstudiocode&logoColor=white)](vscode-insiders:chat-collection/install?url=https%3A%2F%2Fgist.githubusercontent.com%2Fburkeholland%2F0e68481f96e94bbb98134fa6efd00436%2Fraw%2Fultralight-orchestration.collection.yml)

## Agents

## How It Works

1. **All Three** receives your request and analyzes it (Sonnet 4.5)
2. Plans are delegated to the **Planner** agent (GPT-5.2)
3. Code implementation goes to the **Coder** agent (GPT-5.2-Codex)
4. UI/UX and design work goes to the **Designer** agent (Gemini 3 Pro)
5. All Three integrates results and validates the final output
