==================
Writing runners
==================

Table of contents
=================

* `Overview`_
* `Writing a JSON runner`_
* `Writing a Python runner`_

Overview
======

Runners in Lutris can be written in two ways. The recommended way for simple usecases is to write a runner in JSON in
the `share/lutris/json/` directory. For more complex cases, you can write a runner as a Python class in
`lutris/runners/`.

Writing a JSON runner
=====================

A JSON runner should be in the following format

.. code-block:: JSON
  {
      "human_name": "A human-readable name for your runner, such as PCSX2 or Citra",
      "description": "A description for what your runner does, such as 'A Nintendo 3DS emulator'",
      "platforms": ["A list", "of each of the platforms", "your runner supports, such as:", "Nintendo 3DS", "Sony
      Playstation Portable"],
      "runner_executable": "TODO",
      "runnable_alone": "TODO",
      "game_options": [
          {
              "option": "main_file",
              "type": "file",
              "label": "The file that will be passed to the runner - that is, the executable of the game",
              "default_path": "game_path",
          }
      ],
      "runner_options": [
          {
              "option": "runner_option",
              "type": "option_type",
              "label": "An option for your runner",
              "help": "A description of the option",
              "default": "A default value for the option",
              "argument": "--the_argument_to_pass"
          }
      ]
  }

By default, the runner will always use the main_file option to select a game. If this is undesired, you may insert
"entry_point_option" above the game options to indicate a different game option.

A runner option's value will be inserted after the corresponding argument. For example, a string option `option_1` with the
argument field `--argument` will result in the executable being `executable --argument <option_1>`. `--argument` may
only be used in runner options, not game options.

The optional "advanced" field in a runner option will make the option be hidden behind an "advanced settings" toggle.

The types that an option may take are as follow:

choice
------

A selection between multiple choices. A choice option should have the following JSON structure:

.. code-block:: JSON
  {
      "option": "option_name",
      "type": "choice",
      "label": "Label",
      "choices": [
          ["Label 1", "choice_1"],
          ["Label 2", "choice_2"],
      ],
      "default": "off",
      "argument": "--argument"
  }

When the user selects a label, the corresponding choice will be passed as the argument.

If the default is "off", no default is selected.

choice_with_entry
-----------------

TODO

choice_with_search
------------------

TODO

bool
----

This is a simple boolean option.

range
-----

TODO

directory_chooser
-----------------

A directory. This can be useful as an entry point if your runner expects to run a game from a directory instead of a ROM/ISO file

file
----

A file. Will often be used as an entry point, like with ROM files.

multiple
--------

Multiple files

label
-----

TODO

mapping
-------

TODO

Game options are game-specific, and include things like the game's ROM file, or command line arguments. Most runners
will only need to specify the ROM file (and indeed, the game option specified as the entry point is currently the only
one that works).

Runner options are for the behaviour of the runner, and may include whether the game will be launched fullscreen, whether to
show a GUI (like with many emulators), and a provided BIOS file.

It is worth noting that a JSON runner can only launch a game when the entry point option is provided as a direct
argument to the game. That is, `runner_exe <main_file>` will work, but `runner_exe --game <main_file>` will not.

Writing a Python runner
=======================

A Python runner will inherit the Runner class. It should have the following format


.. code-block:: python
   from gettext import gettext as _ # This allows internationalised strings with _("")
   from lutris.runners.runner import Runner
   from lutris.util import system

   class <your_runner>(Runner):
       human_name = _("A human-readable name")
       platforms = [_("A list of platforms"), _("Nintendo 3DS"), _("Sony Playstation Portable")]
       description = _("A description of your runner")
       runnable_alone: True # Or false, depending on your runner
       runner_executable: "TODO"
       game_options = [
           # These are the same as in a JSON runner. Labels and help should be internationalised with _("")
       ]
       runner_options = [
           # These are the same as in a JSON runner. Labels and help should be internationalised with _(""). Do not include the argument field. Boolean defaults should be Python booleans, not strings.
       ]

       def play(self):
           # Code to use the arguments to run the game through the runner

Here is a conventional way of implementing play

.. code-block:: python
   def play(self):
       params = [self.get_executable()]
       if self.runner_config.get("fullscreen"):
           params.append("--fullscreen") # This is useful for any boolean option which is false by default
       bios = self.runner_config.get("bios")
       if not system.path_exists(bios):
           return "{error": "FILE_NOT_FOUND", "file": bios}
       params += ["--bios", bios]

       rom = self.game_config.get("main_file")
       if not system.path_exists(rom):
           return {"error": "FILE_NOT_FOUND", "file": rom}
       params.append(rom)
       # If the runner expects an option like --rom, you might instead do
       # params += ["--rom", rom]
       return {"command", params}

As you can see, play needs to manually evaluate all of the runner and game options and work out how they're going to
affect the final command created.
By the end, params might look something like `emulator --fullscreen --bios /path/to/bios /path/to/rom`.
