# Template: PlayMode cases

## When it applies

A plugin whose output has runtime behaviour an editor test
cannot observe — a state machine that must actually run, a
behaviour that fires on state entry — carries a small
PlayMode layer over the editor cases: the built structure
enters play mode, a parameter is driven, and the running
animator's state is asserted. What only the VRChat client
shows (synchronization, remote appearance) stays outside
every test; this layer covers the local runtime only.

The samples run against a document-local stand-in,
`EntryRecorder`, a state machine behaviour that records the
name of each state it enters into a static list. The test
assembly for PlayMode tests references the same assemblies
as the editor tests and runs under the PlayMode test
runner; the cases carry no test base, because the avatar is
assembled in play mode.

## Skeleton

```csharp
// [PlayMode: unit tests]
// Stand-ins to implement (document-local; contracts in the introduction): EntryRecorder
using System.Collections;
using NUnit.Framework;
using UnityEditor.Animations;
using UnityEngine;
using UnityEngine.TestTools;

public class PlayModeTests
{
    // PLAT-03 (platform and runtime): driving the condition parameter moves the running animator
    // to Active, and the state's behaviour fires on entry.
    [UnityTest]
    public IEnumerator DrivenParameterEntersTheActiveState()
    {
        var root = new GameObject("Avatar");
        var animator = root.AddComponent<Animator>();
        animator.runtimeAnimatorController = BuildTwoStateController();
        EntryRecorder.Entered.Clear();

        yield return null;
        Assert.IsTrue(animator.GetCurrentAnimatorStateInfo(0).IsName("Idle"));

        animator.SetFloat("NDMFComponent/Active", 1f);
        yield return null;
        yield return null;

        Assert.IsTrue(animator.GetCurrentAnimatorStateInfo(0).IsName("Active"));
        CollectionAssert.Contains(EntryRecorder.Entered, "Active");

        Object.Destroy(root);
    }

    static AnimatorController BuildTwoStateController()
    {
        var controller = new AnimatorController();
        controller.AddParameter("NDMFComponent/Active", AnimatorControllerParameterType.Float);
        controller.AddLayer("PlayMode");
        var machine = controller.layers[0].stateMachine;
        var idle = machine.AddState("Idle");
        var active = machine.AddState("Active");
        active.AddStateMachineBehaviour<EntryRecorder>();
        var forward = idle.AddTransition(active);
        forward.AddCondition(AnimatorConditionMode.Greater, 0.5f, "NDMFComponent/Active");
        machine.defaultState = idle;
        return controller;
    }
}
```
