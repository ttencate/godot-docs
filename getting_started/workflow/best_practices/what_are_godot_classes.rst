.. _doc_what_are_godot_classes:

What are Godot classes really?
==============================

Godot offers two main means of creating types: scripts and scenes.
Both of these represent a "class" since Godot revolves around
Object-Oriented design. *How* they do this may not be clear to beginner
or intermediate users though.

Godot Engine provides classes out-of-the-box (like
:ref:`Node <class_Node>`), but user-created types are not actually classes.
Instead they are resources that tell the engine a sequence of initializations
to perform on an engine class.

Godot's internal classes have methods that register a class's data with
a :ref:`ClassDB <class_ClassDB>`. This database provides runtime access to
class information (also called "reflection"). Things stored in the ClassDB
include, among other things...

- properties
- methods
- constants
- signals

Furthermore, this ClassDB is what Objects actually check against when
performing any operation. Access a property? Call a method? Emit a signal?
It will check the database's records (and the records of the Object's base
types) to see if the Object supports the operation. Every C++ Object defines
a static `_bind_methods()` function that describes what C++ content it
registers to the database and how.

So, if the engine provides all of this data at startup, then how does
a user define their own data? It'd be nice if users could define a custom
set of data to be appended to an object's data. That way, users could inject
their own properties and methods into the engine's Object query requests.

*This* is what a :ref:`Script <class_Script>` is. Objects check their attached
script before the database, so scripts can even override methods.
If a script defines a `_get_property_list()` method, that data is appended to
the list of properties the Object fetches from the ClassDB. The same holds
true for other declarative code.

This can lead to some users' confusion when they see a script as being
a class unto itself. In reality, the engine just auto-instantiates the
base engine class and then adds the script to that object. This then allows
the Object to defer to the Script's content where the engine logic deems
appropriate.

A problem does present itself though. As the size of Objects increases,
the scripts' necessary size to create them grows much, much larger.
Creating node hierarchies demonstrates it as each individual Node's logic
could be several hundred lines of code in length.

let's see an example of creating a single Node as a child.

.. tabs::
  .. code-tab:: gdscript GDScript
    
    # main.gd
    extends Node
    
    var child # define a variable to store a reference to the child

    func _init():
        child = Node.new() # construct the child
        child.name = "Child" # change its name
        child.script = preload("child.gd") # give it custom features
        child.set_owner(self) # serialize this node if self is saved
        add_child(child) # add 'child' as a child of self

  .. code-tab:: csharp

        // Main.cs
        using System;
        using Godot;

        namespace ExampleProject {
            public class Main : Resource
            {
                public Node Child { get; set; }

                public Main()
                {
                    Child = new Node();
                    Child.Name = "Child";
                    Child.Script = ResourceLoader.load("child.gd") as Script;
                    Child.SetOwner(this);
                    AddChild(Child);
                }
            }
        }

Notice that only two pieces of declarative code are involved in
the creation of this child node: the variable declaration and
the constructor declaration. Everything else about the child
must be setup using imperative code. However, script code is
much slower than engine C++ code. Each change must make a separate
call to the scripting API which means a lot of C++ "lookups" within
data structures to find the corresponding logic to execute.

To help offload the work, it would be convenient if one could batch up
all operations involved in creating and setting up node hierarchies. The
engine could then handle the construction using its fast C++ code, and the
script code would be free from the perils of imperative code.

*This* is what a scene (:ref:`PackedScene <class_PackedScene>`) is: a
resource that provides an advanced "constructor" serialization which is
offloaded to the engine for batch processing.

Now, why is any of this important to scene organization? Because one must
understand that scenes *are* objects. One often pairs a scene with
a scripted root node that makes use of the sub-nodes. This means that the
scene is often an extension of the script's declarative code.

It helps to define...

- what objects are available to the script?
- how are they organized?
- how are they initialized?
- what connections to each other do they have, if any?

As such, many Object-Oriented principles which apply to "programming", i.e.
scripts, *also* apply to scenes. Some scripts are designed to only work
in one scene (which are often bundled into the scene itself). Other scripts
are meant to be re-used between scenes.

**Regardless, the scene is always an extension of the root script, and can
therefore be interpreted as a part of the class.**
Most of the points covered in this tutorial will build on this point, so
keep it in mind as we proceed.

How to structure a scene effectively
------------------------------------

One of the biggest things to consider in OOP is maintaining
focused, singular-purpose classes with
:ref:`loose coupling <https://en.wikipedia.org/wiki/Loose_coupling>`
to other parts of the codebase. This keeps the size of objects small (for
maintainability) and improves their reusability so that there's no need to
rewrite concepts that have already been written (DRY principle).

These OOP best practices have *several* ramifications for the best practices
in scene structure and script usage.

**If at all possible, one should design scenes to have no dependencies.**
That is, one should create scenes where everything they need is kept
within themselves.

If a scene must interact with an external context, experienced developers
recommend the use of 
:ref:`Dependency Injection <https://en.wikipedia.org/wiki/Dependency_injection>`.
This technique involves having a high-level API define the exact
dependencies of the low-level API. Why do this? Because classes which rely on
their external environment can inadvertantly trigger bugs and unexpected
behavior.


To accomplish this, one must expose data
and then rely on it to be initialized by a parent context:

1. Connect to a signal. Extremely safe, but should use only to "respond" to
   behavior, not start it.
       
       ..tabs::
         ..code-tab:: gdscript GDScript
         
         # Parent
         $Child.connect("signal_name", object_with_method, "method_on_the_object")

         # Child
         emit_signal("signal_name") # Triggers parent-defined behavior

         ..code-tab:: csharp

         // Parent
         GetNode("Child").Connect("SignalName", ObjectWithMethod, "MethodOnTheObject");

         // Child
         EmitSignal("SignalName") // Triggers parent-defined behavior

2. Call a method. Used to start behavior.
       
       ..tabs::
         ..code-tab:: gdscript GDScript
         
         # Parent
         $Child.method_name = "do"

         # Child
         call(method_name) # Call parent-defined method (which child must own)

         ..code-tab:: csharp

         // Parent
         GetNode("Child").Set("MethodName", "Do");

         // Child
         Call(MethodName); // Call parent-defined method (which child must own)

3. Initialize a :ref:`FuncRef <class_FuncRef>` property. Safer than a method
   as ownership of the method is unnecessary. Used to start behavior.
       
       ..tabs::
         ..code-tab:: gdscript GDScript
         
         # Parent
         $Child.func_property = funcref(object_with_method, "MethodOnTheObject")

         # Child
         func_property.call_func() # Call parent-defined method (can come from anywhere)

         ..code-tab:: csharp

         // Parent
         GetNode("Child").Set("FuncProperty", GD.FuncRef(ObjectWithMethod, "MethodOnTheObject"));

         // Child
         FuncProperty.CallFunc(); // Call parent-defined method (can come from anywhere)

4. Initialize a Node or other Object reference.
       
       ..tabs::
         ..code-tab:: gdscript GDScript
         
         # Parent
         $Child.target = self

         # Child
         print(target) # Use parent-defined node

         ..code-tab:: csharp

         // Parent
         GetNode("Child").Set("Target", this);

         // Child
         GD.Print(Target); // Use parent-defined node

5. Initialize a NodePath.
       
       ..tabs::
         ..code-tab:: gdscript GDScript
         
         # Parent
         $Child.target_path = ".."

         # Child
         get_node(target_path) # Use parent-defined NodePath

         ..code-tab:: csharp

         // Parent
         GetNode("Child").Set("TargetPath", NodePath(".."));

         // Child
         GetNode(TargetPath); // Use parent-defined NodePath

These options hide the source of accesses from the child node. This in turn
keeps the child **loosely coupled** to its environment. One can re-use it
in another context without any additional changes to its API.

One should favor keeping data in-house (internal to a scene) though as placing
a dependency on an external context, even a loosely coupled one, still means
that the node will expect something in its environment to be true.

Writing code that relies on external documentation for one to use it safely
is error-prone by default. To avoid creating and maintaining such
documentation, one converts the dependent node ("child" above)
into a tool script that implements
:ref:`_get_configuration_warning() <class_Node__get_configuration_warning>`.
Returning a non-empty string from it will make the Scene dock generate a
warning icon with the string as a tooltip by the node. This is the same icon
that appears for nodes such as the
:ref:`Area2D <class_Area2D>` node when it has no child
:ref:`CollisionShape2D <class_CollisionShape2D>` nodes defined. The editor
then self-documents the scene through the script code. No separately managed
documentation is necessary.

A GUI like this can better inform project users of critical information about
a Node. Does it have external dependencies? Have those dependencies been
satisfied? Other programmers, and especially designers and writers, will need
clear instructions in the messages telling them what to do to configure it.

So, why do all of this complex switcharoo work? Well, because scenes operate
best when they operate alone. If unable to work alone, then working with
others anonymously (with minimal hard depenedencies, i.e. loose coupling).
If the inevitable changes made to a class cause it to interact with other
scenes differently, things break down. A change to one class could
inadvertantly have damaging effects on other classes. Scripts and scenes,
as extensions of the engine classes, abide by these same OOP principles.