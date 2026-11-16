# Template: prioritized branch state machine

## When it applies

Several mutually exclusive branches (menu selection, custom
conditions, per-hand gestures, idle) share one layer, and one
branch outranks the others whenever its condition holds. Layer
order cannot express this: the branches live in one state machine.

## The structure

Priority is built from transition registration order plus
accumulated exit conditions:

1. The branches form an ordered list, highest priority first; the
   default branch (idle) is last and unconditional.
2. Entry transitions are added to the root state machine in that
   order, each carrying its branch's enter conditions. The
   animator evaluates entry transitions in registration order, so
   the highest-priority branch whose conditions hold is entered.
3. Each non-default branch's exit conditions are: any one of its
   own enter conditions no longer holding, OR the enter
   conditions of any higher-priority branch holding. Exiting a
   low branch therefore happens as soon as a higher branch
   becomes available, not only when the low branch's own input
   ends.
4. Inside a branch, each state exits back to the root state
   machine — one transition on the branch parameter no longer
   equalling the state's value, plus one transition per branch
   exit condition — so priority is re-evaluated at the root on
   every change instead of being decided inside the branch.
5. Every transition has no exit time and one shared duration.

## Skeleton

```csharp
// [Editor: generator]
// branches: highest priority first. EnterConditions are the
// branch's own conditions; the last branch is the default and
// has none.
void BuildBranches(AnimatorStateMachine root, IReadOnlyList<Branch> branches)
{
    for (var i = 0; i < branches.Count; i++)
    {
        var branch = branches[i];
        var entry = root.AddEntryTransition(branch.Machine);
        if (i < branches.Count - 1)
            foreach (var condition in branch.EnterConditions)
                entry.AddCondition(condition.Mode, condition.Threshold,
                                   condition.Parameter);

        // Exit: one transition per enter condition, carrying its
        // inverse — the branch is left as soon as ANY of its
        // conditions stops holding.
        foreach (var condition in branch.EnterConditions)
        {
            var exitOne = root.AddStateMachineExitTransition(branch.Machine);
            var inverse = Inverse(condition);
            exitOne.AddCondition(inverse.Mode, inverse.Threshold,
                                 inverse.Parameter);
        }
        for (var higher = 0; higher < i; higher++)
        {
            var exitToHigher = root.AddStateMachineExitTransition(branch.Machine);
            foreach (var condition in branches[higher].EnterConditions)
                exitToHigher.AddCondition(condition.Mode, condition.Threshold,
                                          condition.Parameter);
        }
    }
}
```

Note: each cause is its own transition (transitions are ORed);
the conditions within one transition are ANDed — a higher
branch's exit transition therefore carries that branch's whole
enter set, while the branch's own exit is one transition per
condition. The inverse of a condition swaps its mode (If↔IfNot,
Greater↔Less, Equals↔NotEqual). The threshold is kept for a
Float parameter — at exactly the threshold neither side fires —
and shifts by one for an Int parameter, so the boundary value
satisfies exactly one side: Greater t inverts to Less t + 1,
Less t to Greater t − 1.

The state-side half of the wiring — every state inside a branch
exits back to the root, so priority is re-evaluated there:

```csharp
// [Editor: generator]
// branch.Parameter is the parameter the branch's states are
// selected by (menu index, gesture value); StateValue returns
// the value that selects this state. branch.ExitConditions are
// the branch's exit conditions from the structure above (its own
// enter conditions inverted, plus every higher branch's enter
// conditions, as separate transitions).
static void WireStateExits(
    AnimatorState state, Branch branch,
    AnimatorStateMachine root, float duration)
{
    var byValue = state.AddTransition(root);
    byValue.hasExitTime = false;
    byValue.duration = duration;
    byValue.AddCondition(AnimatorConditionMode.NotEqual,
                         branch.StateValue(state), branch.Parameter);

    foreach (var condition in branch.ExitConditions)
    {
        var exit = state.AddTransition(root);
        exit.hasExitTime = false;
        exit.duration = duration;
        exit.AddCondition(condition.Mode, condition.Threshold, condition.Parameter);
    }
}
```
