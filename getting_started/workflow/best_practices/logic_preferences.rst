.. _doc_logic_preferences:

Logic preferences
=================

Ever wondered whether one should approach problem X with strategy Y or Z?
This article covers a variety of topics related to these dilemmas.

Dynamic loading vs. preloading
------------------------------

In GDScript, there exists the global :ref:`preload <class_@GDScript_preload>`
method. It loads resources as early as possible, in order to front-load the
"loading" operations and avoid loading resources while in the middle of
performance-sensitive code.

Its counterpart, the :ref:`load <class_@GDScript_load>` method, loads a
resource only when the load statement is reached. That is, it will load a
resource in-place which can cause slowdowns then it occurs in the middle of
sensitive processes. The ``load`` function is also an alias for
:ref:`ResourceLoader.load(path) <class_ResourceLoader_load>` which is
accessible to *all* scripting languages.

So, when exactly does preloading occur versus loading, and when should one use
either? Let's see an example:

.. tabs::
  .. code-tab:: gdscript GDScript

  # my_buildings.gd
  extends Node

  # (note how constant scripts/scenes have a diferent naming scheme than
  # their property variants).
  
  # This value is a constant, so it is created when the Script object loads.
  # The script is preloading the value. The advantage here is that the editor
  # can offer autocompletion since it must be a static path.
  const BuildingScn = preload("building.tscn")

  # 1. The value is preloaded, so it will be loaded as a dependency of the 
  #    'my_buildings.gd' script file. However, because this is a property
  #    rather than a constant, the object won't copy the preloaded PackedScene
  #    resource into the property until the script is instantiated with .new().
  # 
  # 2. The preloaded value is inaccessible from the Script object alone. As
  #    such, preloading the value here actually does not benefit anyone.
  # 
  # 3. Because the user exports the value, if this script stored on 
  #    a node in a scene file, the scene instantation code will overwrite the
  #    preloaded initial value anyway (wasting it). It's usually better to
  #    provide null, empty, or otherwise invalid default values for exports.
  # 
  # 4. The case in which one successfully uses "office.tscn" will be when
  #    one instantiates "my_buildings.gd" independently with .new().
  export(PackedScene) var a_building = preload("office.tscn")

  # Uh oh! This results in an error!
  # Constants must be assigned constant values. Because `load` performs a
  # runtime lookup by its very nature, one cannot use it to initialize a
  # constant.
  const OfficeScn = load("office.tscn")

  # Successfully loads and only when one instantiates the script! Yay!
  var office_scn = load("office.tscn")

  .. code-tab:: csharp

  using System;
  using GD;

  public class MyBuildings : public Node {

      public const PackedScene Building { get; set; }
      
      public PackedScene ABuilding = GD.load("office.tscn") as PackedScene;

      public void MyBuildings() {
          // Can assign the value during initialization or during a constructor.
          Building = ResourceLoader.load("building.tscn") as PackedScene;

          // C# and other languages have no concept of "preloading".
      }
  }

Preloading allows the script to handle all of the loading the moment one loads
the script. However, there are times when one would and wouldn't want the
script to perform these loads.

1. If one cannot determine when the script might load, then preloading a
   resource, especially a scene or script, could result in subsequent loads one
   does not expect. This could lead to an unintentional, additional,
   variable-length load time to the original script.

2. If something else will potentially replace the value (like a scene's
   exported initialization), then preloading the value has no meaning. This
   point isn't a significant factor if one intends to always create the script
   independently.

3. If one wishes only to 'import' another class resource (script or scene),
   then using a preloaded constant is often the best course of action. However,
   in exceptional cases, one my wish not to do this:

    1. If the 'imported' class is liable to change, then it should be a property
       instead, initialized either using an ``export`` or a ``load`` (and
       perhaps not even initialized until later).

    2. If the script requires a great many dependencies, and one does not wish
       to consume so much memory, then one may wish to dynamically load and
       unload various dependencies as circumstances change. If resources are
       preloaded into constants, then the only way to unload these resources
       safely would be to unload the entire script. If they are instead loaded
       properties, then one can merely set them to ``null`` and remove all
       references to the resource entirely (which, as a
       :ref:`Reference <class_Reference>`-extending type, will cause the
       resources to automatically delete themselves from memory safely).

Large levels: static vs. dynamic
--------------------------------

If one is creating a large level, under what circumstances should they create
the level as one static space versus loading the level in pieces and
dynamically instancing and deleting / loading and unloading the various
pieces of the world?

Well, the simple answer is just, "when the performance requires it." The
dilemma associated with the two options is one of the age-old programming
choices: does one optimize memory over speed, or vice versa?

Of course, the *simplest* way to create a space is to simply load the space
as one level. That's not to say that one should design spaces as one scene. No,
Godot is at its strongest when each small section of a game is isolated into
its own scene file. But if one were to say, "I want to create an entire
spaceship, complete with hallways, spacecraft, offices, bedrooms, and NPCs
roaming about!" then each of those things could, by themselves, comprise a
multitude of sub-scenes.

The question is then: "Well, if I want to create the spaceship, should I just
make the entire spaceship at once, or should I load only the pieces of the
spaceship that are visible to the player(s) and then modify these at runtime
to show/hide and/or create/destroy only the things that our users need?"

1. **Show everything at once:** If the developer expects users to have a high-
   powered device and doesn't care to support low-end platforms, then one
   *could* simply create the whole thing. But there's a very strong likelihood
   that this would lead to performance issues if the assets are high in memory
   and/or processing requirements. In addition, its generally considered poor
   programming and inconsiderate of the user's machine to force their device
   to work unnecessarily hard.