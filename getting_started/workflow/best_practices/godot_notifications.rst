.. _doc_godot_notifications:

Godot notifications
===================

Every Object in Godot implements a
:ref:`_notification <class_Object__notification>` method. Its purpose is to
allow the Object to respond to a variety of engine-level callbacks that may
relate to it. For example, if the engine tells a
:ref:`CanvasItem <class_CanvasItem>` to "draw", it will call
``_notification(NOTIFICATION_DRAW)``.

Some of these notifications, like draw, are useful to override in scripts. So
much so that Godot exposes many of them with dedicated functions:

- ``_ready()`` : NOTIFICATION_READY
- ``_enter_tree()`` : NOTIFICATION_ENTER_TREE
- ``_exit_tree()`` : NOTIFICATION_EXIT_TREE
- ``_process(delta)`` : NOTIFICATION_PROCESS
- ``_physics_process(delta)`` : NOTIFICATION_PHYSICS_PROCESS
- ``_input()`` : NOTIFICATION_INPUT
- ``_unhandled_input()`` : NOTIICATION_UNHANDLED_INPUT
- ``_draw()`` : NOTIFICATION_DRAW

What users might *not* realize is that notifications exist for types other
than Node alone:

- :ref:`Object::NOTIFICATION_POSTINITIALIZE <class_Object_NOTIFICATION_POSTINITIALIZE>`:
  a callback that triggers during object initialization. Not accessible to scripts.

- :ref:`Object::NOTIFICATION_PREDELETE <class_Object_NOTIFICATION_PREDELETE>`:
  a callback that triggers just before the engine deletes an Object, i.e. a
  'destructor'.
  
- :ref:`MainLoop::NOTIFICATION_WM_MOUSE_ENTER <class_MainLoop_NOTIFICATION_WM_MOUSE_ENTER>`:
  a callback that triggers when the mouse enters the window in the operating
  system that displays the game content.

And many of the callbacks that *do* exist in Nodes don't have any dedicated
methods, but are still quite useful.

- :ref:`Node::NOTIFICATION_PARENTED <class_Node_NOTIFICATION_PARENTED>`:
  a callback that triggers anytime a node is added as a child to another node.

- :ref:`Node::NOTIFICATION_UNPARENTED <class_Node_NOTIFICATION_UNPARENTED>`:
  a callback that triggers anytime a node is removed as a child from another
  node.

- :ref:`Popup::NOTIFICATION_POST_POPUP <class_Popup_NOTIFICATION_POST_POPUP>`:
  a callback that triggers after a Popup node completes any ``popup*`` method.
  Note the difference from its ``about_to_show`` signal which triggers just
  *before* its appearance.

One can access all of these custom notifications from the universal
``_notification`` method. 

..note::
  Methods in the documentation labeled as "virtual" are also intended to be
  overridden by scripts.
  
  A classic example is the
  :ref:`_init <class_Object__init>` method in Object. While it has no
  NOTIFICATION_* equivalent, the engine still calls the method. Most languages
  (with the exception of C#) rely on it as a constructor.

So, in which situation should one use each of these notifications or
virtual functions?

_process vs. _physics_process vs. *_input
-----------------------------------------

Use ``_process`` when one needs a framerate-dependent deltatime between
frames. If code that updates object data needs to update as frequently as
possible, this is the right place. Recurring logic checks and data caching,
should execute here.

Use ``_physics_process`` when one needs a framerate-independent deltatime
between frames. If code should update consistently over time, regardless
of whether time advances quickly or slowly, this is the right place.
Recurring kinematic and object transform operations should execute here. 

While certainly possible, to achieve the best performance, one should avoid
making input checks during these callbacks. ``_process`` and
``_physics_process`` will trigger at every opportunity (they do not "rest" by
default). In contrast, ``*_input`` callbacks will trigger only on frames in
which the engine has actually detected the input.

One can check for input actions via the
:ref:`InputEventAction <class_InputEventAction>` event. If one wants to use
delta time, one can fetch it from the Performance singleton as needed.

.. tabs::
  .. code-tab:: gdscript GDScript

  # Called every frame, even when no input is detected
  func _process(delta):
      if Input.action_just_pressed("ui_select"):
          print(delta)

  # Called during every input event
  func _unhandled_input(event):
      match event.get_class():
          "InputEventAction":
              print(Performance.get_monitor(Performance.TIME_PROCESS))

  .. code-tab:: csharp

  public class MyNode : public Node {

      // Called every frame, even when no input is detected
      public void _Process(float delta) {
          if (GD.Input.ActionJustPressed("UiSelect")) {
              GD.Print(string(delta));
          }
      }

      // Called during every input event. Equally true for _input().
      public void _UnhandledInput(InputEvent event) {
          switch (event.GetClass()) {
              case "InputEventAction":
                  GD.Print(string(GD.Performance.GetMonitor(GD.Performance.TIME_PROCESS)));
                  break;
              default:
                  break;
          }
      }

  }

_init vs. initialization vs. export
-----------------------------------

If the script initializes its own node subtree, without a scene,
that code should execute here. Other property or SceneTree-independent
initializations should also run here. This triggers before ``_ready`` or
``_enter_tree``, but after a script creates and initializes its properties.

Scripts have three types of property assignments that can occur during
instantiation:

.. tabs::
  .. code-tab:: gdscript GDScript
  
  # "one" is an "initialized value". These DO NOT trigger the setter.
  # If someone set the value as "two" from the Inspector, this would be an
  # "exported value". These DO trigger the setter.
  export(String) var test = "one" setget set_test

  func _init():
      # "three" is an "init assignment value".
      # These DO NOT trigger the setter, but...
      test = "three"
      # These DO trigger the setter. Note the `self` prefix.
      self.test = "three"
  
  func set_test(value):
      test = value
      print("Setting: ", test)

  .. code-tab:: csharp

  // "one" is an "initialized value". These DO NOT trigger the setter.
  // If one set the value as "two" from the Inspector, this would be an
  // "exported value". These DO trigger the setter.
  [Export]
  public string Test = "one"
  {
      get;
      set
      {
          Test = value;
          GD.Print("Setting: " + Test);
      }
  }

  public void _Init() {
      // "three" is an "init assignment value".
      // These DO (NOT?) trigger the setter.
      Test = "three";
  }

  When instantiating a scene, property values will set up according to the
  following sequence:

  1. **Initial value assignment:** instantiation will assign either the
     initialization value or the init assignment value. Init assignments take
     priority over initialization values.
  2. **Exported value assignment:** If instancing from a scene rather than
     a script, Godot will assign the exported value to replace the initial
     value defined in the script.

  Instancing a script versus a scene will therefore affect both the
  initialization *and* the number of times the engine calls the setter.
  
_ready vs. _enter_tree vs. NOTIFICATION_PARENTED
------------------------------------------------

When instantiating a scene connected to the first executed scene, Godot will
instantiate nodes down the tree (making ``_init`` calls) and build the tree
going downwards from the root. This causes ``_enter_tree`` calls to cascade
down the tree. Once the tree is complete, leaf nodes call ``_ready``. A node
will call this method once all child nodes have finished calling theirs. This
then causes a reverse cascade going up back to the tree's root.

When instantiating a script or a standalone scene, nodes are not
added to the SceneTree upon creation, so no ``_enter_tree`` callbacks
trigger. Instead, Only the ``_init`` and subsequent ``_ready`` calls occur.

If one needs to trigger behavior that occurs as nodes are parented, regardless
of whether it occurs as part of the main/active scene or not, one can use the
:ref:`PARENTED <class_Node_NOTIFICATION_PARENTED>` notification. For example,
here is a snippet that automatically connects a node's method to a custom
signal on the parent node without failing. Useful on data-centric nodes that
one might create dynamically at runtime.

..tabs::
  ..code-tab:: gdscript GDScript

  extends Node

  var parent_cache

  func connection_check():
      return parent.has_user_signal("interacted_with")
      
  func _notification(what):
      match what:
          NOTIFICATION_PARENTED:
              parent_cache = get_parent()
              if connection_check():
                  parent_cache.connect("interacted_with", self, "_on_parent_interacted_with")
          NOTIFICATION_UNPARENTED:
              if connection_check():
                  parent_cache.disconnect("interacted_with", self, "_on_parent_interacted_with")
  
  func _on_parent_interacted_with():
      print("I'm reacting to my parent's interaction!")

  ..code-tab:: csharp

  public class MyNode extends Node {

      public Node ParentCache = null;

      public void ConnectionCheck() {
          return ParentCache.HasUserSignal("InteractedWith");
      }

      public void _Notification(int What) {
          switch (What) {
              case NOTIFICATION_PARENTED:
                  ParentCache = GetParent();
                  if (ConnectionCheck())
                      ParentCache.Connect("InteractedWith", this, "OnParentInteractedWith");
                  break;
              case NOTIFICATION_UNPARENTED:
                  if (ConnectionCheck())
                      ParentCache.Disconnect("InteractedWith", this, "OnParentInteractedWith");
                  break;
          }
      }
       
      public void OnParentInteractedWith() {
          GD.Print("I'm reacting to my parent's interaction!");
      }
  }
