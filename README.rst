|buildstatus|_
|coverage|_

About
=====

CAN BUS tools.

- `DBC`_, `KCD`_, SYM and CDD file parsing.

- CAN message encoding and decoding.

- Simple and extended signal multiplexing.

- Diagnostic DID encoding and decoding.

- ``candump`` output decoder.

- Node `tester`_.

- `C` source code generator.

- CAN bus monitor.

Project homepage: https://github.com/eerimoq/cantools

Documentation: http://cantools.readthedocs.org/en/latest

Installation
============

.. code-block:: python

    pip install cantools

Example usage
=============

Scripting
---------

The example starts by parsing a `small DBC-file`_ and printing its
messages and signals.

.. code-block:: python

   >>> import cantools
   >>> from pprint import pprint
   >>> db = cantools.database.load_file('tests/files/motohawk.dbc')
   >>> db.messages
   [message('ExampleMessage', 0x1f0, False, 8, 'Example message used as template in MotoHawk models.')]
   >>> example_message = db.get_message_by_name('ExampleMessage')
   >>> pprint(example_message.signals)
   [signal('Enable', 7, 1, 'big_endian', False, 1.0, 0, 0.0, 0.0, '-', False, None, {0: 'Disabled', 1: 'Enabled'}, None),
    signal('AverageRadius', 6, 6, 'big_endian', False, 0.1, 0, 0.0, 5.0, 'm', False, None, None, ''),
    signal('Temperature', 0, 12, 'big_endian', True, 0.01, 250, 229.53, 270.47, 'degK', False, None, None, None)]

The example continues `encoding`_ a message and sending it on a CAN
bus using the `python-can`_ package.

.. code-block:: python

   >>> import can
   >>> can_bus = can.interface.Bus('vcan0', bustype='socketcan')
   >>> data = example_message.encode({'Temperature': 250.1, 'AverageRadius': 3.2, 'Enable': 1})
   >>> message = can.Message(arbitration_id=example_message.frame_id, data=data)
   >>> can_bus.send(message)

Alternatively, a message can be encoded using the `encode_message()`_
method on the database object.

The last part of the example receives and `decodes`_ a CAN message.

.. code-block:: python

   >>> message = can_bus.recv()
   >>> db.decode_message(message.arbitration_id, message.data)
   {'AverageRadius': 3.2, 'Enable': 'Enabled', 'Temperature': 250.09}

See `examples`_ for additional examples.

Command line tool
-----------------

The decode subcommand
^^^^^^^^^^^^^^^^^^^^^

Decode CAN frames captured with the Linux program ``candump``.

.. code-block:: text

   $ candump vcan0 | cantools decode tests/files/motohawk.dbc
     vcan0  1F0   [8]  80 4A 0F 00 00 00 00 00 ::
   ExampleMessage(
       Enable: 'Enabled' -,
       AverageRadius: 0.0 m,
       Temperature: 255.92 degK
   )
     vcan0  1F0   [8]  80 4A 0F 00 00 00 00 00 ::
   ExampleMessage(
       Enable: 'Enabled' -,
       AverageRadius: 0.0 m,
       Temperature: 255.92 degK
   )
     vcan0  1F0   [8]  80 4A 0F 00 00 00 00 00 ::
   ExampleMessage(
       Enable: 'Enabled' -,
       AverageRadius: 0.0 m,
       Temperature: 255.92 degK
   )

Alternatively, the decoded message can be printed on a single line:

.. code-block:: text

   $ candump vcan0 | cantools decode --single-line tests/files/motohawk.dbc
     vcan0  1F0   [8]  80 4A 0F 00 00 00 00 00 :: ExampleMessage(Enable: 'Enabled' -, AverageRadius: 0.0 m, Temperature: 255.92 degK)
     vcan0  1F0   [8]  80 4A 0F 00 00 00 00 00 :: ExampleMessage(Enable: 'Enabled' -, AverageRadius: 0.0 m, Temperature: 255.92 degK)
     vcan0  1F0   [8]  80 4A 0F 00 00 00 00 00 :: ExampleMessage(Enable: 'Enabled' -, AverageRadius: 0.0 m, Temperature: 255.92 degK)

The dump subcommand
^^^^^^^^^^^^^^^^^^^

Dump given database in a human readable format:

.. code-block:: text

   $ cantools dump tests/files/motohawk.dbc
   ================================= Messages =================================

     ------------------------------------------------------------------------

     Name:       ExampleMessage
     Id:         0x1f0
     Length:     8 bytes
     Cycle time: - ms
     Senders:    PCM1
     Layout:

                             Bit

                7   6   5   4   3   2   1   0
              +---+---+---+---+---+---+---+---+
            0 |<-x|<---------------------x|<--|
              +---+---+---+---+---+---+---+---+
                |                       +-- AverageRadius
                +-- Enable
              +---+---+---+---+---+---+---+---+
            1 |-------------------------------|
              +---+---+---+---+---+---+---+---+
            2 |----------x|   |   |   |   |   |
        B     +---+---+---+---+---+---+---+---+
        y               +-- Temperature
        t     +---+---+---+---+---+---+---+---+
        e   3 |   |   |   |   |   |   |   |   |
              +---+---+---+---+---+---+---+---+
            4 |   |   |   |   |   |   |   |   |
              +---+---+---+---+---+---+---+---+
            5 |   |   |   |   |   |   |   |   |
              +---+---+---+---+---+---+---+---+
            6 |   |   |   |   |   |   |   |   |
              +---+---+---+---+---+---+---+---+
            7 |   |   |   |   |   |   |   |   |
              +---+---+---+---+---+---+---+---+

     Signal tree:

       -- {root}
          +-- Enable
          +-- AverageRadius
          +-- Temperature

     Signal choices:

       Enable
           0 Disabled
           1 Enabled

     ------------------------------------------------------------------------

The generate C source subcommand
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Generate `C` source code from given database. The generated code
contains encode and decode functions for all messages.

.. code-block:: text

   $ cantools generate_c_source tests/files/motohawk.dbc
   Successfully generated motohawk.h and motohawk.c.

See `motohawk.h`_ and `motohawk.c`_ for the contents of the generated
files.

Known limitations:

- Multiplexed signals are ignored.

- No signal scaling.

- Maximum signal size is 64 bits.

The monitor subcommand
^^^^^^^^^^^^^^^^^^^^^^

Monitor CAN bus traffic in a text based user interface.

.. code-block:: text

   $ cantools monitor tests/files/motohawk.dbc

.. image:: https://github.com/eerimoq/cantools/raw/master/docs/monitor.png

The menu at the bottom of the monitor shows the available commands.

- Quit: Quit the monitor. Ctrl-C can be used as well.

- Filter: Only display messages matching given regular
  expression. Press <Enter> to return to the menu from the filter
  input line.

- Play/Pause: Toggle between playing and paused (or running and freezed).

- Reset: Reset the monitor to its initial state.

Contributing
============

#. Fork the repository.

#. Install prerequisites.

   .. code-block:: text

      pip install -r requirements.txt

#. Implement the new feature or bug fix.

#. Implement test case(s) to ensure that future changes do not break
   legacy.

#. Run the tests.

   .. code-block:: text

      make test

#. Create a pull request.

.. |buildstatus| image:: https://travis-ci.org/eerimoq/cantools.svg?branch=master
.. _buildstatus: https://travis-ci.org/eerimoq/cantools

.. |coverage| image:: https://coveralls.io/repos/github/eerimoq/cantools/badge.svg?branch=master
.. _coverage: https://coveralls.io/github/eerimoq/cantools

.. _small DBC-file: https://github.com/eerimoq/cantools/blob/master/tests/files/motohawk.dbc

.. _python-can: https://python-can.readthedocs.io/en/master/

.. _DBC: http://www.socialledge.com/sjsu/index.php?title=DBC_Format

.. _KCD: https://github.com/julietkilo/kcd

.. _tester: http://cantools.readthedocs.io/en/latest/#cantools.tester.Tester

.. _encoding: http://cantools.readthedocs.io/en/latest/#cantools.database.can.Message.encode

.. _encode_message(): http://cantools.readthedocs.io/en/latest/#cantools.database.can.Database.encode_message

.. _decodes: http://cantools.readthedocs.io/en/latest/#cantools.database.can.Database.decode_message

.. _examples: https://github.com/eerimoq/cantools/blob/master/examples

.. _motohawk.h: https://github.com/eerimoq/cantools/blob/master/tests/files/c_source/motohawk.h

.. _motohawk.c: https://github.com/eerimoq/cantools/blob/master/tests/files/c_source/motohawk.c
