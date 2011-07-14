---
layout: bits
title: Soft-drink vending machine high level design
---
During one of my job interviews I was asked to present a high level design of a soft-drink vending machine. So I sat down and tried to think what would the perfect soft-drink machine look like. This is the version I came up with. The task itself was limited to 30 minutes, so this is not a full-fledged design.

## General design

A typical soft-drink vending machine is built as a standalone and a self-contained, tamper-proof device, connected physically only to a power outlet. The machine’s sole purpose is to issue/release a soft-drink to end-users in exchange for a cleared payment procedure.
The machine should communicate with the operations centre and transfer (via e.g. a wireless link) a set of status information, such as the number and kind of goods left, any problems it observed (power outage, cash box overfill, mechanical jams) or the fact that the doors were opened, which could either mean that the technician arrived to resupply the unit (which is good) or that somebody tried to steal from it.
The main components include:

* a storage area with a set of sensors (infrared or weight based), used to count the number of goods left

* battery used to sustain the unit’s operation during power outage

* money accepting and verification unit; depending on requirements it can either accept coins, cash, credit cards, RFID tags or any other kind of token, such as magnetic keys or proximity cards

* a user interface unit used to interact with the machine; it can range from a simple keyboard, without any display up to a hi-res touchscreen display with a set of stereo speakers that can, during standby, serve as a advertisement 

* a gyroscope, an accelerometer and a GPS unit used to locate the machine and report any misbehavior (changing the position of the device in order to get to the goods locked inside).

* a two way wireless link (e.g. a GPRS modem) used for a bi-directional communications with the operations centre

* an internal storage used to keep operation logs or usage statistics used for offline billing and reporting (e.g a flash card or a tape drive).

## Main use cases explaining the functionality

The list of most commonly met use-case scenarios.

### Normal (idle) operation

* Machine displays (if it is capable of) a welcome message and waits for the user to start the operation - choose a drink to be sold.

* It periodically reports its current status (a keep-alive message) including e.g. current stock levels, environmental conditions, its geographical position etc.

### Resupplying with a fresh batch of drinks

* A worker arrives at the machine.

* Machine is put into service mode.

* Doors are opened.

* Supplies are refilled.

* Doors are closed.

* Machine is set to normal mode.

* Before going on-line, machine performs a set of checks, notes the current number of supplies and reports its current status to the operations centre together with the fact that it went a resupply operation.

* Before leaving the site, worker tests the operation by going through a regular process of buying a drink.

### Working under a power failure situation

* The machine continues to operate as usual until the energy levels drop to a certain value - e.g. 20%.

* After reaching the “service charge” level (20%), the machine goes into service mode. It does not sell any drinks. However, it still reports its status to the operations center, including the battery level.

* Once the batteries are almost discharged, the machine shuts down. It locks itself down, preventing unauthorized access.

* When the power is restored, the machine starts up and performs a set of self-tests.

* If the tests are completed without errors, the machine goes into normal/idle mode.

* If any of the tests fails, the machine continues to operate in the service mode and notifies the operations centre about its condition - presumably causing a technician to arrive.

### Abnormal operation

* If any abnormal conditions are observed, the machine goes into service mode.

* A status message is sent to the operations centre.

* The operations centre may either remotely shut down/lock the machine, force a restart or give “a green light” and restore its normal operations.

* Machine periodically performs a set of self-tests. If the tests complete without errors, a normal mode is restore.

* Description of a process of buying a drink

### User chooses a drink.

* If the drink is not available an appropriate message is displayed.

* If the drink is available, a required amount of money is displayed.

* User inserts coins/notes/cards or any other token (a magnetic key, an RFID tag etc.)

* A payment verification takes place. Depending on the payment method the process may include checking if the coins are of correct size, weight, if they interact with magnetic waves. If a banknote is inserted, it may be additionally checked using UV light. Other means of payment, like credit/debit cards, RFID tags or magnetic keys, require connecting to a central database in order to clear the transaction or alternatively - locally store the transaction information for a future, offline billing.

* Once the payment is cleared, the drink is released.

* The stocks are recalculated.

* The transaction status is sent to the operations center either after each operation, in batches (e.g. after 10 operations) or after a certain time elapsed (every 10 minutes) - depending on the required granularity.
