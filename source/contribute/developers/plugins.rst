Freeseer Plugin System
======================

There are a number of situations where users may want to adapt Freeseer to 
their own individual needs without needing to make and maintain custom changes 
independently of the main project. In order to make things as straightforward
as possible, Freeseer provides a plugin management framework which allows users
to easily extend certain features of the software in order to easily customize
it to their needs.

The Freeseer plugin management system is based on the `Yapsy plugin system 
<http://yapsy.sourceforge.net/>`_ which is a minimal plugin system that works 
out-of-the-box but is quite extensible. The Yapsy `documentation 
<http://yapsy.sourceforge.net/>`_ is well written and easy to follow, there are
also some good `tutorials <http://ralsina.me/weblog/posts/BB923.html>`_ online.
This document aims to provide everything you need to know to create and use 
Freeseer plugins.

Freeseer Plugin System Setup
****************************

All of the logic needed to find, load and activate plugins are provided by the
Yapsy ``PluginManager`` class. For the Freeseer project we use the singleton
version of the PluginManager in our ``PluginManager`` class. The Freeseer 
``PluginManager`` class takes a profile (which is an instance of 
``freeseer.framework.config.profile:Profile``) as an argument. The code for the 
Freeseer PluginManager class is found in ``src/freeseer/framework/plugin.py``\ .

.. code-block:: python

  class PluginManager(QtCore.QObject):
      '''
      Plugin Manager for Freeseer

      Provides the core functionality which enables plugin support in.
      '''

      def __init__(self, profile):
        ...
        self.plugmanc = PluginManagerSingleton.get()
        self.configfile = profile.get_filepath('plugin.conf')
        ...

Yapsy provides a ``PluginFileLocator`` class which allows the plugin system to 
locate plugin files that are accessible via the file system. The locator can
only find plugin files that are available to Python. [#Note1]_ 

.. code-block:: python

  class PluginManager(QtCore.QObject):
        
         ...
        locator = self.plugmanc.getPluginLocator()
        locator.setPluginInfoExtension("freeseer-plugin")

The locator has be configured to search for plugin files with the
``.freeseer-plugin`` extension and placed in a plugin directory (or one of its
sub-folders). Freeseer searches 3 directory paths for plugins: 

1. The user's ``$HOME/.freeseer/plugins`` directory (or the Windows equivalent).

2. If you've cloned Freeseer, in the ``src/freeseer/plugins/`` directory.

3. The Freeseer instaled plugins directory in ``site-packages/freeseer/plugins``

The Yapsy ``IPlugin`` class defines the minimal interface needed for Yapsy
plugins. In  ``freeseer/framework/plugin.py`` the ``IBackendPlugin`` class extends 
``IPlugin`` and defines the interface for all freeseer plugin classes. All
Freeseer plugins inherit from the ``IBackendPlugin`` class. 

.. code-block:: python
  
  class IBackendPlugin(IPlugin):
    ...
    CATEGORY = "Undefined"
    ...

  class IAudioInput(IBackendPlugin):
    CATEGORY = "AudioInput"
    ...

Each ``IPlugin`` subclass needs a CATEGORY attribute defined. The ``IBackendPlugin``
class defines the CATEGORY attribute as "Undefined" and there are a number of
plugin category classes that have been defined that inhert from ``IBackendPlugin``.
When you are writing your own Freeseer plugin, unless you are writing some new
feature for the main code base that needs a new CATEGORY, you will usually 
extend one of the existing category classes in your plugin and will not need to 
override the CATEGORY attribute. 

If you are creating a new category you will need to override this attribute 
and you will need to add the CATEGORY name and class name to the 
PluginManager's category filter.

.. code-block:: python

  class PluginManager(QtCore.QObject):
        ...

        self.plugmanc.setCategoriesFilter({
            "AudioInput": IAudioInput,
            "AudioMixer": IAudioMixer,
            "VideoInput": IVideoInput,
            "VideoMixer": IVideoMixer,
            "Importer":   IImporter,
            "Output":     IOutput})
        self.plugmanc.collectPlugins()

Just add a ``"key": value`` pair to the dictionary that is passed to the
``setCategoriesFilter()`` method, where
the ``"key"`` is the string name of the CATEGORY, and the value is the class name
of ``IPlugin`` subclass for that category.

Yapsy provides a number of useful ``PluginManagerDecorator``\ s. Freeseer's plugin system
uses the ``ConfigurablePluginManager`` which allows the plugin to save and load
the active plugins and their settings to a confiuragtion file.

.. code-block:: python

  from yapsy.ConfigurablePluginManager import ConfigurablePluginManager 
  ...
  class PluginManager(QtCore.QObject):
      ...
      PluginManagerSingleton.setBehaviour([ConfigurablePluginManager])

Many of the Freeseer plugins, such as the video and audio plugins, use the 
``ConfigurablePluginManager`` to save the active plugins.

How to Build a Freeseer Plugin
******************************
 
It is fairly straightforward to write a Freeseer plugin, and Yapsy provides two
ways to create plugins. In both cases you need a freeseer metadata file with the 
``.freeseer-plugin`` extension and an other file or directory. The basic steps
for creating a new plugin are: 

1. Write a metadata file ``plugin-module-name.freeseer-plugin``, a metadata 
   file has the following format:

.. code-block:: none

    [Core]
    Name = Plugin Module Name
    Module = pluginmodulename

    [Documentation]
    Author = Author Name
    Version = 3.0.9999
    Website = http://fosslc.org
    Description = Brief plugin description         

2. Create the plugin file(s). 
   
Step 2 breaks down into the following parts:   

A. If you are creating a single file plugin simply create a file with the 
   same name as your metadata file.

.. code-block:: none

    plugin-name.freeseer-plugin
    plugin-name.py

B. If you are creating a multi-file plugin:

   1. Add a directory ``plugin-module-name`` (the ``plugin-name`` must be the 
      same for both the config file and the directory) 
  
   2. In the new plugin directory write a class in the ``__init__.py`` file of the
      ``plugin-module-name`` directory that extends one of the ``IBackendPlugin`` 
      category sub-classes, and override the ``name`` class atribute with the new 
      plugin name: ``name = "Plugin Module Name"``

   3. Add other useful plugin code, such as a ``widget.py`` to the ``plugin-name`` 
      directory as needed and call it in your new plugin class.

How to access a Freeseer Plugin
*******************************

Any modules that need to access the plugins will need to import the
``PluginManager``\ .

There are a number of ways to access the plugins that have been located by the
``PluginManager``\ . It is possible to iterate over all of the plugins or all of
the plugins in a given category, or to acces a plugin by the specific plugin 
name. While Yapsy does provide the ``getAllPlugin()`` or ``getPluginsOfCategory()``
and ``getPluginByName()`` methods, the Freeseer ``PluginManager`` provides a number 
of accessor methods and it is recommended that you use these for accessing 
the plugins.

.. code-block:: python

    get_plugin_by_name(self, name, category)  
    get_all_plugins(self)
    get_plugins_of_category(self, category)
    get_audioinput_plugins(self)
    get_audiomixer_plugins(self)
    get_videoinput_plugins(self)
    get_videomixer_plugins(self)
    get_importer_plugins(self)
    get_output_plugins(self)

When you call any of the above methods you receive a ``PluginInfo`` object, or
a list of ``PluginInfo`` objects, which contains meta information for the
plugin(s). Each ``PluginInfo`` object has an attribute ``plugin_object`` which
returns an instance of the plugin which you can then use. 

An example of a class that calls a plugin by name and uses the 
``.plugin_object`` attribute to access the plugin object:

.. code-block:: python
  
  from freeseer.framework.plugin import PluginManager
  ...

  class QtDBConnector():
     
     def __init__(self, configdir, talkdb_file="presentations.db"): 
        
        ...
        self.plugman = PluginManager(self.configdir) 
     
     ...

     def add_talks_from_rss(self, rss):
        """Adds talks from an rss feed."""
        entry = str(rss)
        plugin = self.plugman.get_plugin_by_name("Rss FeedParser", "Importer")
        feedparser = plugin.plugin_object
        feedparser.parse(entry)

        if not feedparser.get_presentations_list():
            log.info("RSS: No data found.")

        else:
            for presentation in feedparser.get_presentations_list():
                talk = Presentation(presentation["Title"],
                                    presentation["Speaker"],
                                    presentation["Abstract"],  # Description
                                    presentation["Level"],
                                    presentation["Event"],
                                    presentation["Room"],
                                    presentation["Time"])
                self.insert_presentation(talk)

An example of accessing plugins by one of the category methods:

.. code-block:: python

  ...
  
  n = 0  # Counter for finding Audio Mixer to set as current.
  plugins = self.plugman.get_audiomixer_plugins()
  for plugin in plugins:
     self.avWidget.audioMixerComboBox.addItem(plugin.plugin_object.get_name())
         if plugin.plugin_object.get_name() == self.config.audiomixer:
            self.avWidget.audioMixerComboBox.setCurrentIndex(n)
         n += 1
   ...

Other Yapsy Resources
*********************

.. seealso::

  * http://yapsy.sourceforge.net/

  * http://ralsina.me/weblog/posts/BB923.html

  * http://stackoverflow.com/questions/5333128/yapsy-minimal-example

.. rubric:: Footnotes

.. [#Note1] So it is important to make sure that all directories leading to 
            plugin code have an ``__init__.py`` file in them. 
