.. _doc_signals_versus_methods:

When to use signals versus methods
==================================

Signals update external entities that a thing *has happened*, to allow for
reactions. Methods are instructions to *do* something.

Programmers should observe OOP best practices by ensuring that node hierarchies
are self-contained systems, as previously discussed. To that end, nodes should
usually not reference their ancestors in any way (unless you know what you are
doing). Making nodes dependent on their placement within the SceneTree leads to
component-like behavior that restricts the flexibility of the node system.

To preserve this dynamic in situations where a sub-scene needs to trigger
behavior in a separate scene (parent, sibling, etc.), one can do any of
the following:

1. Emit a signal.
    - Implementation:
        1. Define a signal on the sub-scene's root node.
        2. Have an ancestor node connect the sub-scene's signal to the desired
        object's method.
        3. Have the sub-scene emit the signal to notify that the triggering
        behavior has occurred.
    - Explanation:
        - The sub-scene has no awareness/dependencies on the surrounding
          system, so it can easily be moved into a different scene and re-used.
          If the state of the surrounding system changes, it will have no
          effect on the sub-scene's functionality.
        - Note that the name of the signal should indicate
          that an event took place, NOT that a particular action should
          execute. This preserves the flexibility of the system by allowing for
          conceptual consistency should one later realize that additional
          reactions are needed.
2. Expose an ancestor-initialized variable to use in a method call.
    - Implementation:
        1. Define a Node variable on the sub-scene's root node.
        2. Have an ancestor node assign a value to the sub-scene's variable.
        3. Use the variable in a method call.
            a. If the object's data is needed, then pass the variable into a
               method call.
            b. If the object's functions are needed, then call the method on
               the variable.
    - Explanation:
        - Again, the sub-scene has no dependencies, this time because the
          ancestor is the one responsible for setting up the value.
        - Note that the sub-scene's root node script logic must take into
          account the possibility of the variable not being initialized
          properly. Either an `assert(var)` should be used (to error out),
          or methods should be wrapped with `if not var: return`, or the
          equivalent, to fail silently.
