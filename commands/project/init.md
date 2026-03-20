# Initialize a new project

Initialize a project with the following information: $ARGUMENTS

## Git

Initialize the git repo. If the user didn't provide information about a remote repo, you must propose to create a new one and push the code.

## Code init

If the tech was not provided, ask the user. If at least one tech is provided you must:
1. Load appropriate skills for these techs,
2. Copy appropriate templates as defined by the loaded skills,
3. Ensure it is completely compliant with the requirements of the skills loaded.

If the user provides a tech for which no skill exists, mention it to the user and skip the code init part for that tech.

## Initial documentation

Create the initial documentation according to information provided and skills loaded. The initial documentation includes:
- CLAUDE.md for AI maintenance of the project,
- .agent_docs for additional AI information and maintain the CLAUDE.md efficient,
- README.md for main user documentations,
- docs folder for complete user documentation,
- specs folder with an empty BACKLOG.md for future feature tracking.

CLAUDE.md **must** contains at least the following process:
- Ensure that every modification is always commited and pushed if a remote repo exists.
- Ensure that every modification includes docs updates (CLAUDE.md + .agent_docs and README.md + docs)

## End of initialization workflow

Ensure to commit with "Initial commit" all the files prepared. Push if a remote repo exists.
