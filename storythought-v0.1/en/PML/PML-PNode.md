## PNode

Used for handling AGI and task node interactions. PNode is a neural task node that, while lacking complex Thought capability, can rapidly accept AGI model task templates and efficiently execute computation.

### PNode Executes AGI Model-Planned Template Tasks

AGI models delegate tasks not requiring Thought to PNodes for execution. PNodes execute tasks according to the Prompt Program output by the AGI model.

### PNode Protocol

Protocol for AGI models to control PNode execution states, such as interrupt handling, template task updates, etc.


Ref.

[PNode](github.com/prompt-lang/storythought/pnode)
