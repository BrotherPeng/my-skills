# My Skills (AI Agentic Capabilities)

This repository serves as a centralized collection of **Custom Skills** designed to enhance and specialize the capabilities of AI coding assistants. Each skill is a self-contained module that provides specific context, rules, and behavioral guidelines to optimize the AI's performance in various scenarios.

## 📂 Repository Structure

Skills are organized logically within the `skills/` directory:

```text
my-skills/
├── README.md           # Repository documentation
└── skills/
    └── [skill-name]/   # Individual skill directory
        ├── SKILL.md    # Core skill definition, rules, and instructions
        ├── scripts/    # (Optional) Helper scripts and automation utilities
        └── resources/  # (Optional) Templates, assets, and reference files
```

## 🚀 Available Skills

Currently, the following skills are maintained in this repository:

### 1. language-style-guide
- **Directory**: `skills/language-style-guide`
- **Description**: Enforces strict adherence to **Simplified Chinese (简体中文)** for all interactions, code comments, and documentation.
- **Key Features**:
  - Ensures professional technical writing standards.
  - Standardizes mixed English-Chinese typography (e.g., spacing format).
  - Guarantees that code comments and documentation are native-quality and consistent.
  - Prevents "machine translation" style responses.

## 🛠️ Usage

To utilize these skills with your AI assistant:

1. **Import/Mount**: Ensure this repository is accessible to your agent's workspace.
2. **Context Loading**: The agent should read the `SKILL.md` of the relevant skill to internalize the instructions.
3. **Execution**: The agent will then operate according to the defined protocols (e.g., referencing specific documentation standards, running provided scripts, etc.).

## 📝 Contribution

Feel free to add new skills by creating a new directory under `skills/` and adding a detailed `SKILL.md` that defines the persona, tools, and workflows for that specific capability.
