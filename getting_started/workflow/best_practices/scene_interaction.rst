.. _doc_scene_interaction:

When and how to have scenes interact with each other
====================================================

Two scenes walk into a bar. One of them says, "I need to interact with you."

What's the punchline? Well, there are a number of ways to have scenes interact.
Each approach is different, and some adhere to OOP design more so than others.

..note: Keep aware that each of these strategies works with relationships
between individual nodes, not just scenes. After all, scenes are really just
fancy "constructor" functions for a root node's script that help set up its
operating environment.

Let's break down a developer's options:

1. Parent-child relationship
    - Traditional approach. Most common. Simplest. Cleanest.
    - Should ensure that the child scene has no knowledge of the parent scene.
      This implies that either...
        1. The child scene has additional behavior that *optionally* triggers
           off of the parent's existence. This is otherwise referred to as a
           "component" node.
        2. The child scene exposes data which its parent then uses without its
           awareness.
            - Example: Area2D uses CollisionShape2D children.
        3. The child relies on data or functions from the parent, in which
           case one uses a signal or an exposed property to connect anonymousl
           with its parent.
            - Note that this solution is the "signal vs. method" topic from
              before.
2. Lateral relationship (i.e. sibling, cousin, etc.)
    - Sometimes necessary within one's project. Similar, but more disjointed
      than the parent-child relationship.
    - Should ensure that the scenes minimize any direct dependencies on each
      other. References to each other should be mediated by an ancestor.
    - The implementation examples are the same as a parent-child relationship,
      except rather than the two scenes needing to use a child name or
      `get_parent()` to access one another...
            1. a NodePath should be exported, configured in the editor,
                and then used to initialize a variable, or...
            2. a variable should be declared and an ancestor node can then
                assign a value to it.
3. Delegate data or features to an Object that is loaded by both scenes.
    - Occasionally useful.
    - Automatically preserves anonymiity between scenes via a third-party.
    - As...
        1. Resource: Unfortunately, results in I/O file access any time a user
           mutates the data (slower, outdated if accessed by multiple threads).
        2. Reference: no I/O problems, and users can safely pass it around
           without worrying that a deletion in place A will result in an error
           in place B.
        3. Node: no I/O problems, but with added benefit that it can integrate
           with the tree. Deletion/Memory leaks are a risk factor to manage.
4. Use an Autoload or otherwise persisted Node on the outer edges of the
   SceneTree to manage the data shared between various nodes.
    - Convenient, but incredibly dangerous to one's project.
    - Creates the same problems that
      :ref:`global variables <http://wiki.c2.com/?GlobalVariablesAreBad>` do.
    - Pierces scenes' internal sub-structures, i.e. their "private" data, to
      allow for invasive access between all nodes.
