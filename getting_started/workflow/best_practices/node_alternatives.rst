.. _doc_node_alternatives:

When and how to avoid using nodes for everything
================================================


Nodes are cheap to produce, but even they have their limits. A project may
have tens of thousands of nodes all doing things. The more complex their
behavior though, the larger the strain each one adds to a project's
performance.

Godot provides more lightweight objects for creating APIs which nodes use.
Be sure to keep these in mind as options when designing how you wish to build
your project's features.

1. :ref:`Object <class_Object>`: The ultimate lightweight object, the original
   Object must have its memory managed manually. With that said, it isn't too
   difficult to create one's own custom data structures, even node structures,
   that are also lighter than the :ref:`Node <class_Node>` class.
    - Example: see the :ref:`Tree <class_Tree>` node. While it allows for
    a highly customizable table of content with an arbitrary number of columns,
    the data that it uses to generate its visualization is actually a tree of
    :ref:`TreeItem <class_TreeItem>` Objects.
    - Advantages: Simplifying one's API to smaller scoped objects helps improve
    its accessibility improve iteration time. Rather than working with the
    entire Node library, one creates an abbreviated set of Objects from which
    a node can generate and manage the appropriate sub-nodes.
    - Note: One should be careful when handling them. One can store an Object
    into a variable, but these references can suddenly become invalid. For
    instance, if the object which created it suddenly decides to delete it,
    automatically triggering an error state when the variable is next accessed.
2. :ref:`Reference <class_Reference>`: Only slightly more complex than Object.
   They can manage their own memory safely by automatically deleting themselves
   when all references to them are inaccessible. These are useful in the
   majority of cases where the data in one's custom classes.
    - Example: see the :ref:`File <class_File>` object. It functions
      just like a regular Object except that it does not need to be manually
      deleted.
    - Advantages: same as the Object.
3. :ref:`Resource <class_Resource>`: Only slightly more complex than Reference.
   They automatically know how to serialize/deserialize (i.e. save and load)
   their object properties to/from Godot resource files.
    - Example: Scripts, PackedScene (for scene files), and other types like
      each of the :ref:`AudioEffect <class_AudioEffect>` classes. Each of these
      can be save and loaded, therefore they extend from Resource.
    - Advantages: Much has
    :ref:`already been said <http://docs.godotengine.org/en/latest/getting_started/step_by_step/resources.html#creating-your-own-resources>`
    on Resource's advantages over traditional data storage methods.
    In the context of using Resources over Nodes though, their main advantage
    is in Inspector-compatibility. While nearly as lightweight as
    Object/Reference, they can still display and export properties in the
    Inspector. This allows them to fulfill a purpose much like sub-Nodes on
    the usability front, but also improve performance if one plans to have
    many such Resources/Nodes in their scenes.
